---
name: verify
description: >
  Generate failing test suite from a spec via INDEPENDENT agent (no impl access),
  then run adversarial + vision grading post-impl. Two modes — default generates
  failing tests, --grade runs post-impl quality pass. Use when user invokes
  /verify <slug>, says "write tests for spec X", "grade the implementation",
  or after /spec completes. Often invoked indirectly by /ship — exit cleanly so
  /ship can chain. Enforces the separation: spec author ≠ test author ≠ impl
  author.
disable-model-invocation: true
allowed-tools: Read Write Edit Bash Grep Glob Agent
---

You generate the verification artifacts for a feature spec. Two modes:

- `/verify <slug>` (default) → generate failing tests from spec
- `/verify <slug> --grade` → post-impl adversarial + vision pass

Shared conventions (artifact layout, findings format, file handoffs, model
pins, /ship contract): see `../CONVENTIONS.md`.

Today's date: !`date +%Y-%m-%d`

## Mode A: default (test generation)

### 1. Validate spec

Read `.workflow/<slug>/spec.md`. Reject if: file does not exist (run
`/spec <slug>` first); frontmatter `status: draft` (approve spec first);
acceptance criteria missing or aspirational ("must be fast" without number);
edge cases lack concrete fixtures; open questions unresolved; stakeholder
edges fewer than 3 at tier=medium+.

On any failure: list the gaps explicitly, do NOT proceed. Send user back to
`/spec` to fix.

### 2. Capture the test baseline (green-before)

If `.workflow/<slug>/baseline.md` does not exist (i.e. /ship didn't already
write it at bootstrap), capture it now: run the FULL suite once and record the
set of already-failing tests (ideally empty). This is the reference for the
--grade regression gate — only tests that go red *after* this point count as
regressions. If the baseline is non-green, say so; the user may want to start
from green.

### 3. Determine test surface

Map spec to test medium based on what it describes:

| Spec describes | Test surface |
|---|---|
| UI feature (component, page, interaction) | Playwright E2E + screenshot per state |
| HTTP endpoint | Integration test against real test backend (no mocks at boundary) |
| Library function / pure logic | Property-based (Hypothesis / fast-check) + unit per AC |
| Backend service / job | Integration against real DB / queue (test instance) |
| CLI tool | Subprocess test with stdin / stdout / file fixtures |
| Migration | Forward + rollback + idempotency tests |

Mixed (e.g. UI + endpoint): generate all relevant surfaces.

If unclear → ask user.

Detect test framework by reading project files:

```bash
test -f package.json && grep -E '"(vitest|jest|playwright|mocha)"' package.json   # JS/TS
test -f pyproject.toml && grep -E '(pytest|hypothesis|playwright)' pyproject.toml # Python
test -f go.mod && echo "stdlib testing"                                           # Go
test -f Cargo.toml && grep -E '(criterion|proptest)' Cargo.toml                   # Rust
```

If multiple frameworks → use the one with most existing tests.

### 4. Spawn INDEPENDENT test author agent

CRITICAL: this agent must NOT see implementation source — no impl paths in the
brief, ever ("just so it has context" defeats the entire purpose). Use `Agent`
with `subagent_type: general-purpose`, **model: sonnet** (translating an
approved spec into tests is standard work — see `../CONVENTIONS.md` §Model
pins). Per File Handoffs, pass PATHS, not pasted content — the spec file is
the one artifact the agent reads:

- Path to `.workflow/<slug>/spec.md` (its ONLY source of truth)
- Tier
- Test framework + conventions discovered above
- Where tests should live (`tests/<slug>/`, `e2e/<slug>.spec.ts`, etc.)
- Where fixtures should live (`tests/fixtures/<slug>/`)

Brief template:

```
You are writing a FAILING test suite from a spec. You will NOT see
implementation source. Read the spec at .workflow/<slug>/spec.md — it is
your only source of truth.

Each test name maps to an acceptance criterion / edge ID:
  test_AC1_<short_description>, test_AC2_..., test_EDGE1_..., ...

Use fixtures verbatim from the spec (exact strings, exact byte values).
Place fixtures in tests/fixtures/<slug>/.

Use the project's test framework: <discovered framework>.
Follow existing test conventions: <list a few example test files for
pattern matching>.

Do NOT look in src/, app/, lib/, server/src/, client/src/. If the spec is
ambiguous about an interface, choose a sensible default and add a comment
marking the assumption.

Return a list of files created and a brief summary of the test count
per AC / EDGE.
```

Use `isolation: worktree` if generating many files (>5) so the work is
isolated and reviewable as a single diff.

### 5. Run the suite

After agent returns, run the suite scoped to the slug (adapt to detected
framework: `npm test -- "<slug>"` / `pytest -k "<slug>"` /
`go test ./... -run "<slug>"` / `cargo test "<slug>"`).

Expected: ALL fail (impl doesn't exist or doesn't match interface). Capture
exit code + summary line.

If any test passes unexpectedly → likely tautological (a test that just calls
the impl and asserts "no error"); flag for user review: *"Test X passed
without implementation. Likely tautological. Review before proceeding."*

### 6. Commit tests

Suggest commit `test(<slug>): failing test suite from spec` with body:
"Tests authored from .workflow/<slug>/spec.md by independent agent (no impl
access). Implementer must not edit tests; surface concerns instead."

### 7. Hand off to implementer

Output to user:

> Tests in place. <N> tests, all failing as expected.
> Files: <test files> / <fixture files>
> Implement against tests until green. Do NOT edit tests — if a test seems
> wrong, surface it (we'll decide spec change vs test fix together).
> When green, run `/verify <slug> --grade` (or re-invoke `/ship <slug>`).

If invoked by `/ship`: return control (implementation is multi-turn chat, not
this skill's job).

---

## Mode B: `--grade` (post-impl quality pass)

### 1. Regression gate (green-after)

Run the **FULL** suite (not slug-scoped) and diff against
`.workflow/<slug>/baseline.md`. The impl phase only proved the feature's own
tests pass — it may have broken an unrelated test.

- Any feature test still red → exit: *"X tests still failing. Finish
  implementation first."* Don't grade an incomplete impl.
- Any test red now that was NOT red in the baseline is a 🔴 regression
  introduced by this feature: surface it and do NOT proceed to grading until
  resolved. (Pre-existing baseline reds are not yours — don't block on them.)
- No baseline file → warn, fall back to "suite all green" as the gate.

### 2. Adversarial pass — independent agent

Spawn `Agent` (subagent_type: general-purpose, fresh context, **model:
sonnet**; escalate to opus if concurrency / crypto / auth is touched). Per
File Handoffs: pass the spec PATH + implementation file paths (list only —
agent reads what it needs); it writes findings to a file and returns a
compact summary.

Brief template:

```
You are an adversarial reviewer. Read the spec at .workflow/<slug>/spec.md.
The implementation is at:
  - <list paths>
The test suite passes.

Your job: find an input that satisfies the spec but breaks in practice.
Use: property-based reasoning (what invariants does the impl assume?);
fuzz reasoning (what input shapes haven't been considered?); edge-of-spec
reasoning (boundary values just inside / outside spec); concurrency
reasoning (if applicable).

For pure functions / library code: prefer property-based test framework
(Hypothesis / fast-check / proptest) over freeform fuzzing — much higher
hit rate. If property-based testing isn't set up, suggest adding it
(don't write the test, just suggest).

WRITE your findings to .workflow/<slug>/grade-adversarial-<YYYY-MM-DD>.md,
one entry per finding:
  🔴/⚠️ <axis>: <input> | <expected per spec> | <actual> | <reproducer>

Then RETURN only a ≤15-line summary: 🔴/⚠️ counts, one line per 🔴 AND ⚠️
finding, and the file path. If you find no issues, say so explicitly.
```

### 3. Vision pass (UI features only)

Detect UI changes:

```bash
git diff --name-only $(git merge-base HEAD main) | \
  grep -E '\.(tsx|jsx|vue|svelte|html|css|scss)$'
```

If UI changes detected:

1. Run Playwright tests with screenshot capture (project should already
   have this configured if Playwright tests exist).
2. Spawn vision agent (`Agent`, subagent_type: general-purpose, **fresh
   context** — never the same agent as the adversarial pass (single context =
   correlated failure modes) — multimodal, **model: opus**) with:
   - Screenshots from Playwright run (paths)
   - Design reference: Figma frame URL or mockup PNG path from spec's
     "Design reference" section. If none → fall back to spec's "Happy
     path" section as design intent.
   - Spec path (`.workflow/<slug>/spec.md`)

Brief template:

```
Compare each screenshot against the design reference. Look for: layout
differences (alignment, spacing, hierarchy); broken affordances (button
looks unclickable, hidden hover state); missing states (loading, empty,
error not visible); color / typography divergence; responsive issues if
multiple viewports captured.

Be specific: name selectors, pixel locations, screenshot file.
Don't grade on subjective taste — grade against the reference.

If a baseline screenshot exists from prior version, mention pixel-diff
results first (cheaper, unfoolable). Use vision only for semantic
differences.

WRITE your findings to .workflow/<slug>/grade-vision-<YYYY-MM-DD>.md,
one entry per finding:
  🔴/⚠️ <viewport>: <element> at <screenshot:location> | <issue> | <fix>

Then RETURN only a ≤15-line summary: 🔴/⚠️ counts, one line per 🔴 AND ⚠️
finding, and the file path. If you find no issues, say so explicitly.
```

### 4. Write findings file

From the agents' summaries (open a per-agent file only on-demand), aggregate
into `.workflow/<slug>/grade-<YYYY-MM-DD>.md` — frontmatter per
`../CONVENTIONS.md` §Findings files (`phase: grade`). Body: `# Grade — <slug>
— <date>`, then `## Adversarial findings` and `## Vision findings (if UI)`
sections with one line per finding (`🔴 #1 — <axis>: <one-line>  (detail:
grade-adversarial-<date>.md)`), a `## No findings` note if applicable, and a
`## Triage notes` section the user populates as they resolve.

### 5. Surface to user

Show the file path + summary (≤15 lines):

```
Grade complete: .workflow/<slug>/grade-<date>.md
🔴 Critical: <N>  ⚠️ Warning: <N>
✅ Regression gate: <clean | N regressions vs baseline>
✅ Adversarial pass: <clean | findings>
✅ Vision pass: <clean | findings | n/a if no UI>
Triage findings, fix or accept each. When done, mark file
`status: triaged` and re-invoke /ship <slug> to continue to audit phase.
```

DO NOT auto-fix findings — never, not even a 🔴 where "it's obvious what to
do". Each is a user decision (see `../CONVENTIONS.md`).

If invoked by `/ship`: return control. /ship handles the resume.

---

## Hard constraints

- **Test author MUST be independent context.** No impl source paths in the
  brief. Hard rule.
- **Implementer MUST NOT edit tests.** If invoked in --grade mode and tests
  have been edited since gen commit (`git log` on test files), warn user
  and ask if intentional.
- **Adversarial and vision graders MUST each be fresh context** (separate
  Agent invocations).
- **Regression gate before grading.** Full suite vs baseline; new reds block.
- **Adversarial mandatory at tier=large.** Optional at tier=medium but
  default to running it unless user says skip.
- **Property-based preferred** for pure functions / library code.
- **Never skip spec validation** ("the spec looked fine" is how gaps ship).
- **Spec changes mid-impl** require version bump in spec frontmatter (e.g.
  `version: 2`) or explicit "spec amended" line in commit message.
- **Model pins + file handoffs** per `../CONVENTIONS.md` — sonnet test
  author / adversarial, opus vision; agents write findings files and return
  ≤15-line summaries.
