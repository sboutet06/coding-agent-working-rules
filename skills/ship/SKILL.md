---
name: ship
description: >
  End-to-end feature lifecycle orchestrator. Detects current phase from filesystem
  state and runs the next step (spec → tests → impl handoff → grade → review).
  Resume by re-invoking — no need to remember the sequence. Use when user says
  /ship <slug>, "ship feature X", "let's build feature Y", "continue feature
  X", or wants to resume any in-progress feature. For tier=trivial work
  (one-line bug fix, typo, rename), exit gracefully and tell the user to just
  code it directly.
disable-model-invocation: false
allowed-tools: Read Write Edit Bash Agent Grep Glob
---

You orchestrate the full feature lifecycle. The user invokes `/ship <slug>` and
you detect which phase the feature is in, then run the next step. Re-invocations
resume from current state.

Shared conventions — artifact layout, findings-file format, file handoffs,
model pins: see `../CONVENTIONS.md`. Summaries below; the details live there.

- **File Handoffs**: subagents WRITE findings/reports to files and return
  ≤15-line summaries carrying every 🔴/⚠️ headline + the path; dispatch briefs
  pass PATHS, never pasted content or session history. Aggregate from
  summaries; open per-agent files only on-demand. (Measured: 96% lighter
  controller context.)
- **Model pins**: pin the model on EVERY dispatch — never inherit the session
  model. opus for design judgment (sota/grill/premortem/spec/plan) + vision +
  the unguarded security pass; **sonnet for everything routine** (test author,
  implementers, per-task review, grade, default audit reviewers — measured
  best recall/precision/cost). Never haiku for review.

Today's date: !`date +%Y-%m-%d`

## Step 1: parse arguments

- First positional arg = `<slug>` (kebab-case, ≤30 chars). If missing, ask.
- `--tier=large` flag → run `/sota` first, then (if a `grill-me` skill is
  installed) run it on the retained approach, then auto-run
  `/premortem-lite --target sota` (unless `--no-premortem` set), then proceed.
- `--grill` flag → run a `grill-me` skill (not included — see README) on the
  slug BEFORE entering spec phase.
  Output `.workflow/<slug>/decisions.md` becomes input to `/spec`. Use when the plan
  is fuzzy or has ≥3 unresolved branches.
- `--premortem` flag → opt-in pre-mortem attack on `spec.md` after spec
  approval, before test gen. Auto-on for `--tier=large` (targets `sota.md`
  instead). Use at tier=medium when architecture has non-trivial structural
  risk you want stressed before coding.
- `--no-premortem` flag → opt-out from pre-mortem even at `--tier=large`.
  Use when /sota dossier is small or design is mechanical.
- `--grade-only` → jump straight to grade phase (skip spec / test gen / review).
- `--audit-only` → jump straight to multi-axis audit.
- `--reset` → wipe state files for the slug after explicit user confirmation
  (DESTRUCTIVE — list files to be removed and ask before deleting).

**Auto-suggest `--grill`**: if the description is under ~100 words with ≥2
fuzz markers (*"maybe"*, *"or"*, *"not sure"*, *"to be seen"*), suggest
`/ship <slug> --grill` ("Fuzzy description, N markers — grill the open
decisions first?") and wait for the answer before proceeding.

If user describes a one-line change (typo, rename, dependency bump, doc
whitespace) when invoking, exit with: *"This looks like tier=trivial — /ship
adds ceremony with no benefit. Just code it directly, write a test if
warranted, commit. Re-invoke /ship for tier=medium+ work."*

## Step 2: detect current phase

Check filesystem state IN ORDER. Stop at the first match.

| Check | Phase |
|---|---|
| `.workflow/<slug>/` does not exist | `NONE` |
| `--tier=large` set, no `.workflow/<slug>/sota.md` | `SOTA_NEEDED` |
| `.workflow/<slug>/sota.md` exists, no `.workflow/<slug>/premortem.md`, pre-mortem expected | `PREMORTEM_NEEDED` (target=sota) |
| `.workflow/<slug>/spec.md` does not exist (and SOTA / pre-mortem either done or not expected) | `SPEC_NEEDED` |
| `.workflow/<slug>/spec.md` exists; frontmatter `status: draft` | `SPEC_DRAFT` |
| `.workflow/<slug>/spec.md` `status: approved`, pre-mortem expected on spec, no `premortem.md` | `PREMORTEM_NEEDED` (target=spec) |
| `.workflow/<slug>/premortem.md` exists, frontmatter `status: pending` | `PREMORTEM_UNRESOLVED` |
| `.workflow/<slug>/spec.md` `status: approved`; pre-mortem done or skipped; no test files reference slug | `SPEC_APPROVED` |
| Test files exist; suite failing; tier=large; no `.workflow/<slug>/plan.md` | `PLAN_NEEDED` |
| `.workflow/<slug>/plan.md` exists; frontmatter `status: needs-arbitration` | `PLAN_UNRESOLVED` |
| Test files exist; suite failing; plan `status: approved` OR tier=medium | `IMPL` |
| Test files exist; suite all-green; no `.workflow/<slug>/grade-*.md` | `TESTS_GREEN` |
| Latest grade file exists; has unresolved 🔴 findings | `GRADE_UNRESOLVED` |
| Latest grade file marked `status: triaged`; no `.workflow/<slug>/audit-*.md` | `GRADED` |
| Latest audit file exists; has unresolved 🔴 findings | `AUDIT_UNRESOLVED` |
| Latest audit file marked `status: triaged` | `AUDITED` |

**Pre-mortem expected** evaluation:
- `--tier=large` set AND `--no-premortem` NOT set → target = `sota` (after /sota done)
- `--premortem` flag set (tier=medium) → target = `spec` (after spec approved)
- Otherwise → pre-mortem skipped entirely, treat as done

Detection commands:

```bash
test -d ".workflow/<slug>"                # workflow dir
test -f ".workflow/<slug>/sota.md"        # sota
test -f ".workflow/<slug>/premortem.md" && head -10 ".workflow/<slug>/premortem.md" | grep -E "^status:"
test -f ".workflow/<slug>/spec.md" && head -10 ".workflow/<slug>/spec.md" | grep -E "^status:"
grep -rl "<slug>" tests/ server/tests/ e2e/ 2>/dev/null   # test files
# run tests scoped to slug (detect runner from package.json / Makefile / pyproject.toml)
npm test -- "<slug>"     # or: pytest -k "<slug>"; cargo test "<slug>"; etc.
ls .workflow/<slug>/grade-*.md 2>/dev/null | sort | tail -1
ls .workflow/<slug>/audit-*.md 2>/dev/null | sort | tail -1
```

Report detected phase to user in one line before acting:
*"Detected phase: TESTS_GREEN. Auto-running grade phase."*

## Step 3: act on phase

### `NONE` → bootstrap workflow dir, dispatch

`mkdir -p ".workflow/<slug>"`, then ensure a feature branch — ship commits a
deliberately-RED test suite and many WIP commits; keep them off the default
branch (and AUDITED opens a PR, which needs one). If on the default branch,
`git checkout -b "ship/<slug>"`; if already on a non-default branch, reuse it
(the user already branched). Tell the user which branch ship is working on:

```bash
default_branch="$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|origin/||' || echo main)"
current="$(git branch --show-current)"
```

**Capture the test baseline** (reference for the grade-phase regression gate):
run the full suite once and record the set of already-failing tests to
`.workflow/<slug>/baseline.md` (ideally empty). If non-green, say so; the user
may want to start from green.

Then dispatch: `--tier=large` → **SOTA_NEEDED**; otherwise (medium default) →
**SPEC_NEEDED**.

### `SOTA_NEEDED` → enter design phase

Apply the `/sota` skill protocol (`../sota/SKILL.md`). It persists the
approved dossier to `.workflow/<slug>/sota.md`. On approval, transition:
`--no-premortem` set → **SPEC_NEEDED**; otherwise → **PREMORTEM_NEEDED**
(target=sota).

### `PREMORTEM_NEEDED` (target=sota | spec) → run pre-mortem

Apply the `/premortem-lite` skill protocol (`../premortem-lite/SKILL.md`)
with `--target sota` (after SOTA_NEEDED) or `--target spec` (after spec
approval at tier=medium with `--premortem`). It spawns an INDEPENDENT
sub-agent and writes findings to `.workflow/<slug>/premortem.md`.

After it returns:
- no findings → mark `premortem.md` `status: triaged` yourself, transition:
  target=sota → **SPEC_NEEDED**; target=spec → **SPEC_APPROVED**.
- findings → file stays `status: pending` → **PREMORTEM_UNRESOLVED**.

### `PREMORTEM_UNRESOLVED` → resume pre-mortem triage

Show outstanding findings from `.workflow/<slug>/premortem.md`. Per finding the
user decides: accept (rationale in Triage notes) / patch the target (edit
`sota.md` or `spec.md`) / reject (rationale). When all triaged, write
`status: triaged` in the frontmatter, then: `target: sota` → **SPEC_NEEDED**;
`target: spec` → **SPEC_APPROVED**.

### `SPEC_NEEDED` → enter spec phase

If `--grill` set (or transitioning from a tier=large flow) and a `grill-me`
skill is installed, first apply its protocol: descend the decision tree,
write `.workflow/<slug>/decisions.md`.

Then apply the `/spec` skill protocol (`../spec/SKILL.md`). Ship adds: if
`decisions.md` or `sota.md` exists, /spec treats logged decisions / chosen
design as authoritative (no re-questioning). On approval
(`status: approved`):
- `--premortem` set AND tier=medium AND no `premortem.md` yet →
  **PREMORTEM_NEEDED** (target=spec).
- otherwise → auto-continue to **SPEC_APPROVED**.

### `SPEC_DRAFT` → resume spec iteration

Show the current spec, ask: *"Amend, approve, or abort?"*. On approve, set
frontmatter to `status: approved`, then auto-continue to SPEC_APPROVED.

### `SPEC_APPROVED` → enter test gen phase

Apply the `/verify` skill protocol, default mode (`../verify/SKILL.md`):
independent test author (no impl access, model: sonnet), tests written, run,
all-fail confirmed, committed (`test(<slug>): failing test suite from spec`).

Ship adds: tell the user *"<N> tests in place, all failing as expected.
Implementation is autonomous from here (you're out of the impl loop) — I'll
surface only a fuzzy plan, a hard block, or the grade/audit findings."* Then
auto-continue: tier=large → **PLAN_NEEDED**; tier=medium → **IMPL** (single
autonomous implementer, no decomposition).

### `PLAN_NEEDED` → decompose (tier=large)

Decomposition's measured justification is **context hygiene + recoverability +
falsifiable progress**, NOT defect-catching (`docs/measurements/RESULTS.md` in the repo, optim #3) — so
keep tasks coarse enough to be coherent, not atomic for its own sake.

Spawn a planning agent (fresh context, **model: opus** — design judgment) with
the spec + the failing tests + the impl surface (paths). It produces ordered
tasks; **split only where a reviewer could meaningfully reject one task while
approving its neighbor**, each task ending in an independently-testable
deliverable. Write `.workflow/<slug>/plan.md` with frontmatter `slug`, `date`,
`phase: plan`, `status: approved | needs-arbitration`, `obvious: true|false`,
`tasks_total: <N>`.

Body: per task — title, the files it touches, **Interfaces** (Consumes /
Produces: exact names + types neighboring tasks rely on), the covering tests,
and "done = those tests pass". Plus a verbatim **Global constraints** block from
the spec, implicitly part of every task.

**Self-assess obviousness** (the only planning-time human gate, per the user's
rule "review only if the split isn't obvious"):
- **obvious** = clean Consumes/Produces with no circular dependency, no task
  needs to rewrite a prior task's output, each task maps to a distinct test
  subset → write `status: approved`, auto-continue to **IMPL**.
- **fuzzy** = coupling, a circular interface, or an interface that depends on a
  decision the spec never settled → write `status: needs-arbitration`, list the
  specific ambiguities, transition to **PLAN_UNRESOLVED**.

### `PLAN_UNRESOLVED` → human arbitrates the plan

Surface only the named ambiguities (grill-me style: one decision at a time, with
your recommendation + tradeoff). On each resolution, patch `plan.md`. When the
split is clean, set `status: approved` and auto-continue to **IMPL**.

### `IMPL` → autonomous subagent-driven implementation

Human is OUT of this loop. Read the ledger `.workflow/<slug>/progress.md` (create
if absent); resume at the first task not marked complete. For tier=medium there
is one implicit task = the whole feature.

For each task in order:

1. **Dispatch a fresh implementer subagent** (general-purpose; **model: sonnet**,
   escalate to opus only if the task needs broad design judgment). Hand over
   **files, not pasted content**: the task brief, its Interfaces, the covering
   test paths, the Global constraints. Brief: implement until the covering
   tests pass; do not edit tests; report
   `DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT` + write a report file +
   return a ≤15-line summary + path.
2. **Per-task review gate** (cheap, LOCAL): dispatch a reviewer (**model:
   sonnet**) on the task's diff via a review-package file. This catches local
   defects early but is **structurally blind to cross-task defects** (measured:
   it can't see DRY spanning two tasks) — that is the final audit's job, not
   this gate's. If 🔴/⚠️: fix subagent → re-review → repeat until clean.
3. **Commit** the task. Append one line to `progress.md`:
   `Task N: complete (commits <base7>..<head7>, review clean)`.

Status handling:
- **NEEDS_CONTEXT** → provide the missing context, re-dispatch.
- **BLOCKED** → diagnose: more context / a more capable model / split the task /
  the plan is wrong. Re-dispatch with the change. **Escalate to the human ONLY
  when truly stuck** (this and a fuzzy plan are the sole impl-time human gates).
- Never force the same model to retry unchanged.

When all tasks are complete and the suite is green, auto-continue to
**TESTS_GREEN**. Do not pause to check in between tasks — the ledger is the
record.

### `TESTS_GREEN` → enter grade phase

Apply the `/verify --grade` protocol (`../verify/SKILL.md` Mode B): full-
suite regression gate vs `baseline.md` (new reds block grading), adversarial
agent (sonnet; opus if concurrency/crypto/auth), vision agent (opus) on UI
changes — each writing its findings file and returning a summary — aggregated
into `.workflow/<slug>/grade-<YYYY-MM-DD>.md`.

Ship adds: surface 🔴 to the user (never auto-fix). After the user triages,
they re-invoke /ship; detection sees no unresolved 🔴 → flip the file to
`status: triaged` (write the marker yourself) and proceed.

### `GRADE_UNRESOLVED` → resume grade triage

Show outstanding 🔴 findings from latest grade file. Ask user to address each
in chat. When user says "fixed all" or equivalent, mark file
`status: triaged` and proceed to GRADED phase.

### `GRADED` → enter audit phase

Apply the `/audit` skill protocol (`../audit/SKILL.md`): parallel fan-out
(one message, multiple Agent calls) to code-reviewer (always) +
security-auditor / architect-review / system-pressure / user-pressure per its
trigger table; models per CONVENTIONS (sonnet default, opus unguarded for
security). Each reviewer writes `audit-<reviewer>-<date>.md` + returns a
summary; aggregate into `.workflow/<slug>/audit-<YYYY-MM-DD>.md`. Ship adds:
architect-review is mandatory at tier=large. Surface 🔴 to user.

### `AUDIT_UNRESOLVED` → resume audit triage

Same pattern as GRADE_UNRESOLVED. Show outstanding 🔴, await user resolution,
mark `status: triaged`, proceed to AUDITED.

### `AUDITED` → done

Tell user: *"Lifecycle complete for <slug>. Spec / grade / audit files under
`.workflow/<slug>/`. Ready to merge — open PR with `gh pr create`; PR
description should link spec + both review files."* Exit.

## Phase progression rules

| Transition | Auto-continue? |
|---|---|
| NONE → SOTA_NEEDED | yes (if `--tier=large`) |
| NONE → SPEC_NEEDED | yes (otherwise) |
| SOTA_NEEDED → PREMORTEM_NEEDED (sota) | yes after /sota approval (unless `--no-premortem`) |
| SOTA_NEEDED → SPEC_NEEDED | yes after /sota approval if `--no-premortem` |
| PREMORTEM_NEEDED → PREMORTEM_UNRESOLVED | yes (auto-runs /premortem-lite, if findings) |
| PREMORTEM_NEEDED → SPEC_NEEDED | yes if no findings (target=sota) |
| PREMORTEM_NEEDED → SPEC_APPROVED | yes if no findings (target=spec) |
| PREMORTEM_UNRESOLVED → next | only after user triages all findings |
| SPEC_NEEDED → SPEC_DRAFT | yes (within /spec protocol) |
| SPEC_DRAFT → SPEC_APPROVED | only after explicit user approval |
| SPEC_DRAFT → PREMORTEM_NEEDED (spec) | yes after spec approval if `--premortem` at tier=medium |
| SPEC_APPROVED → PLAN_NEEDED | yes (auto-runs /verify gen, then tier=large) |
| SPEC_APPROVED → IMPL | yes (tier=medium — single autonomous implementer) |
| PLAN_NEEDED → IMPL | yes if the split is obvious |
| PLAN_NEEDED → PLAN_UNRESOLVED | yes if the split is fuzzy (human arbitrates) |
| PLAN_UNRESOLVED → IMPL | only after the human resolves the named ambiguities |
| IMPL → TESTS_GREEN | yes — autonomous subagent-driven impl, human out of the loop |
| TESTS_GREEN → GRADE_UNRESOLVED | yes (auto-runs grade) |
| GRADE_UNRESOLVED → GRADED | only after user triages all 🔴 |
| GRADED → AUDIT_UNRESOLVED | yes (auto-runs audit) |
| AUDIT_UNRESOLVED → AUDITED | only after user triages all 🔴 |
| AUDITED → done | exit |

User-required gates: /sota approval, pre-mortem finding triage, spec approval,
**plan arbitration (only if the split is fuzzy)**, **BLOCKED escalation (only if
an implementer is truly stuck)**, grade triage, audit triage. Implementation
itself is autonomous — the human is out of the impl loop. Everything else flows.

## Hard constraints

- **Never edit tests** after they're written by the independent agent. If user
  asks you to, refuse and explain why (independent-author rule).
- **Never auto-fix** grade or review findings. Each is a user judgment.
- **Always commit between phases**: spec commit, test commit, impl commit(s),
  grade-triage commit, review-triage commit. Use Conventional Commits.
- **Spec is source of truth**: if impl diverges, amend spec in same PR or
  abort. Silent drift forbidden.
- **Per-tier UX**:
  - tier=trivial → exit, redirect to direct coding
  - tier=small → /ship is overkill; suggest /spec stub + write a test
  - tier=medium → full lifecycle; impl is ONE autonomous implementer subagent
    (no decomposition), then grade + audit. `--premortem` optional if
    structural risk on the spec.
  - tier=large → the `--tier=large` pipeline from Step 1 (/sota → [grill-me] →
    premortem-lite), then full lifecycle with decomposed PLAN → autonomous
    subagent IMPL (human arbitrates only a fuzzy plan or a hard block) and
    full agent fan-out on grade and audit.
- **Resume safety**: if filesystem state is ambiguous (e.g. spec and tests
  both exist but stale), ask user to clarify rather than guess. Better to
  pause than mis-resume.
- **`--reset` is destructive**. List files to be deleted (`.workflow/<slug>/`
  contents, tests, fixtures), require explicit "yes, delete all" before acting.
  Never `--reset` a slug whose impl is on a non-current branch (spec might be
  in active use elsewhere). The whole feature lifecycle lives under
  `.workflow/<slug>/`; `rm -rf .workflow/<slug>` after confirmation purges it.

## State files (ship-specific status values)

Layout + frontmatter conventions: `../CONVENTIONS.md`. Per-artifact
status values ship greps:

- `sota.md` — `status: approved` + `chosen: <A|B|C>`; authoritative for /spec
  and /premortem-lite.
- `decisions.md` (optional, from grill-me) — `status: resolved | partial | blocked`;
  authoritative for settled branches in /spec.
- `premortem.md` — `status: pending | triaged` + `target: sota | spec`.
- `spec.md` — `status: draft | approved | archived`.
- `baseline.md` — tests already failing at bootstrap; grade regression
  reference.
- `plan.md` (tier=large) — `status: approved | needs-arbitration` +
  `obvious: true|false`; ordered tasks with Interfaces; authoritative for IMPL.
- `progress.md` — durable IMPL ledger, one line per completed task. On resume /
  after compaction, trust this + `git log`, not memory — re-dispatching
  completed tasks is the most expensive failure mode.
- `grade-<date>.md`, `audit-<date>.md` — `status: pending | triaged`.

## When NOT to use /ship

- tier=trivial work
- Bug fixes that are reproduce-once-fix-once-test (use a regression-test pattern,
  not full lifecycle)
- Refactors with no behavior delta (keep the existing suite green and review the diff)
- Dependency upgrades (use Dependabot/Renovate, not /ship)
- Documentation-only changes
- Commit / PR generation (use your usual commit flow or `gh pr create`)

In all these cases: tell the user explicitly that /ship is overkill and what
the right alternative is.
