---
name: audit
description: >
  Multi-axis code audit via parallel agent fan-out. Spawns code-reviewer,
  security-auditor, architect-review, system-pressure-reviewer, and
  user-pressure-reviewer in parallel; aggregates findings into one
  risk-tiered table at .workflow/<slug>/audit-<date>.md. Use when user invokes
  /audit, says "full audit", "audit the diff", "deep review", or after
  tier=medium+ implementation has passed grade. Mandatory at tier=large.
  Distinct from the built-in /review (PR review); /audit is the heavy multi-axis pass — don't run it
  on every commit, reserve it for pre-merge of substantive work.
disable-model-invocation: true
allowed-tools: Read Write Edit Bash Grep Glob Agent
---

You orchestrate a multi-axis review by fanning out to specialized agents in
parallel, then aggregating their findings into one ranked artifact.

Shared conventions (findings format, file handoffs, model pins, /ship
contract): see `../CONVENTIONS.md`.

Today's date: !`date +%Y-%m-%d`

## Inputs

Either:

- `/audit <slug>` — audit the diff for a feature with frontmatter spec
- `/audit` (no slug) — audit current branch diff vs main / target branch
- `/audit --files <glob>` — audit specific paths

If invoked without context, ask user what's being audited.

## Step 1: determine scope

```bash
# If slug given: find files referencing slug + impl files from latest commit
git log --name-only --pretty=format: -- "*<slug>*" | sort -u

# If no slug: branch diff vs main (or detected base branch)
git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main
git diff --name-only <base>..HEAD

# If --files: use the glob
ls <pattern>
```

**Bound the scope: ≤50 files per audit run.** If more, ask user to narrow or
split — each agent has finite context.

## Step 2: classify what review axes apply

Detect attack surface + complexity dimensions to decide which agents to
spawn, and pin each agent's model (see `../CONVENTIONS.md` §Model pins:
sonnet is the measured default reviewer; security gets opus unguarded):

| Detection | Agent triggered | Model |
|---|---|---|
| Always | `comprehensive-review:code-reviewer` | sonnet |
| Public HTTP / IPC / parsing untrusted input / auth / crypto code touched | `comprehensive-review:security-auditor` | opus (unguarded) |
| tier=large OR new module / service boundary | `comprehensive-review:architect-review` | sonnet |
| Long-running process / background job / request handler / storage / cache | `system-pressure-reviewer` | sonnet |
| Public API endpoint / UI component / form / file upload | `user-pressure-reviewer` | sonnet |

Detection heuristics:

```bash
# Attack surface
git diff --name-only <base>..HEAD | grep -iE \
  '(routes?/|controller|handler|auth|crypt|password|token|session|parse|deserial|sql|template)'

# Long-running / pressure surface
git diff --name-only <base>..HEAD | grep -iE \
  '(server|worker|job|queue|scheduler|cache|cron|stream)'

# UI / public API surface
git diff --name-only <base>..HEAD | grep -iE \
  '(routes?/|api/|controllers?/|components?/|pages?/|forms?/|upload)'
```

Always include code-reviewer. Add others based on triggers. If unclear, err
toward including (a wasted review costs tokens; a skipped one costs a missed
bug). Never skip an agent because "I think the code is clean" — that's
exactly when audit is most likely to find something.

State the agent set explicitly to user before spawning:

> Spawning 4 parallel agents:
> - code-reviewer (always)
> - security-auditor (auth code touched)
> - system-pressure-reviewer (background job code touched)
> - user-pressure-reviewer (public API endpoint added)

## Step 3: parallel fan-out

Send ONE message containing multiple `Agent` tool calls — they execute in
parallel (sequential dispatch wastes wall clock). Each agent runs in its own
fresh context; never share one axis's findings as input to another —
correlated bias defeats the orthogonality. Pin the model per the Step 2
table.

Per File Handoffs (`../CONVENTIONS.md`): each brief passes PATHS, and each
reviewer WRITES its findings file and returns only a compact summary.

Brief template:

```
You are reviewing the diff for <slug or branch>.

Scope: <list of files>
Spec (if applicable): read .workflow/<slug>/spec.md
Tier: <small | medium | large>
Base branch: <main / target>

Your axis: <code-reviewer / security-auditor / architect-review /
            system-pressure-reviewer / user-pressure-reviewer>

WRITE your findings to .workflow/<slug>/audit-<your-axis>-<YYYY-MM-DD>.md,
one entry per finding:
🔴/⚠️ <file:line> — <axis>: <one-line summary>
   Detail: <2-3 lines>
   Recommendation: <action>

Then RETURN only a compact summary (≤15 lines): 🔴/⚠️ counts, one line per
🔴 AND ⚠️ finding, and the file path. Do NOT dump full findings into your
reply.

If clean: write the file stating "No findings" and say so in the summary.
Do NOT include ✅ entries for what looks fine. Only flag issues.
```

## Step 4: aggregate

Wait for all agents to return. Work from the SUMMARIES — open a per-reviewer
file only on-demand (when the user drills into a finding). Then:

1. **Normalize** each finding to `{file, line, severity, axis, summary,
   recommendation}`.
2. **Dedupe** by `(file, line, summary-keyword)`. If two agents flag the
   same line for related issues, merge into one finding listing both axes
   under a `cross_validated_by:` field — these are higher-credibility and
   get visual emphasis (named axes in the heading).
3. **Sort** by severity (🔴 first), then by file path, then by line.
4. **Count** per severity per axis.

## Step 5: write aggregated findings file

Write to `.workflow/<slug>/audit-<YYYY-MM-DD>.md` (or
`.workflow/_orphan/branch-<branch-name>-audit-<date>.md` if no slug).
Frontmatter per `../CONVENTIONS.md` §Findings files (`phase: audit`), plus
`agents_run: [...]` and `cross_validated: <N>`. Body:

```markdown
# Audit — <slug> — <date>

## Summary

| Axis | 🔴 | ⚠️ | Detail file |
|---|---|---|---|
| code-reviewer | <N> | <N> | audit-code-reviewer-<date>.md |
| ... | | | |
| **TOTAL** | <N> | <N> | <N> cross-validated |

## Critical findings (🔴)

### #1 — <file:line> — flagged by [code-reviewer, security-auditor]

<summary>

**Recommendation**: <...>

## Warnings (⚠️)

(same format)

## Triage

(populated by user as they resolve each finding)
```

## Step 6: surface to user

≤15 lines:

```
Audit complete: .workflow/<slug>/audit-<date>.md

🔴 Critical: <N>  ⚠️ Warning: <N>  Cross-validated: <N>
By axis: code-reviewer <N>🔴 <N>⚠️ | security-auditor <N>🔴 <N>⚠️ | ...

Triage findings, fix or accept each. When done, mark file
`status: triaged`. If invoked by /ship, /ship will detect that and
proceed to "ready to merge".
```

DO NOT auto-fix, and do NOT filter findings before showing the user ("this
one isn't important" is the user's call). If invoked by `/ship`: return
control; /ship handles the resume.
