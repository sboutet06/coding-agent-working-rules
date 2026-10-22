# Shared skill conventions

Single source for conventions shared by `/ship`, `/audit`, `/verify`,
`/premortem-lite` (and read by `/spec`, `/sota`).
Skills reference this file instead of restating it.

## Artifact layout

All feature artefacts live under `.workflow/<slug>/` (slug = kebab-case, ≤30
chars): `sota.md`, `decisions.md`, `premortem.md`, `spec.md`, `baseline.md`,
`plan.md`, `progress.md`, `grade-*.md`, `audit-*.md`. Orphan
artefacts (no feature slug) go under `.workflow/_orphan/`. Tests are code, not
workflow artefacts — they live in `tests/<slug>/`, `e2e/<slug>.spec.ts`,
`tests/fixtures/<slug>/`.

Every artefact carries YAML frontmatter with greppable `key: value` lines —
`/ship` detects phase via `head -10 <file> | grep -E "^status:"`. Never break
the `key: value` format. Multiple grade/audit files per slug = re-runs after
fixes; latest by date wins.

## Findings files (grade / audit / premortem)

Frontmatter (extra keys per skill are fine; these are the contract):

```yaml
---
slug: <slug>
date: <YYYY-MM-DD>
phase: grade | audit | premortem
status: pending      # → triaged once the user resolves all 🔴
findings_critical: <N>
findings_warning: <N>
---
```

- Severity emoji: 🔴 critical, ⚠️ warning, ℹ️ info.
- **Never auto-fix a finding.** Each is a user judgment — the skill records,
  the user triages (fix / accept / reject, rationale in a Triage section).
- Summary surfaced to the user: ≤15 lines — counts, one line per 🔴/⚠️, path.

## File handoffs

Everything an agent returns in-context becomes the controller's resident input
tokens, re-read on every later turn — so artifacts move as **files**, never as
text dumped into context. Projected (`docs/measurements/RESULTS.md` §Optim #2, 71 real
reviewer outputs, median 4709 tok each): a 5-reviewer audit returning findings
in-context leaves **23.5k tokens resident** in the controller vs **750** with
file-handoff — **96% lighter**; over an ~8-turn triage that is ~182k input
tokens avoided.

- **Each reviewer / grader / adversarial agent WRITES its findings** to a
  uniquely-named file (`audit-<reviewer>-<date>.md`, `grade-<agent>-<date>.md`)
  and **returns a compact summary**: status, 🔴/⚠️ counts, **one line per
  critical AND important finding**, and the path. Completeness of serious
  findings is the hard requirement — measured: a generic "≤15-line summary"
  instruction dropped ~5% of serious defects (`docs/measurements/RESULTS.md`,
  fidelity 0.95). Keep prose short but carry EVERY 🔴/⚠️ headline; only minor
  findings may stay file-only.
- **The controller aggregates from the summaries** and opens a per-agent file
  only on-demand (user drills into a finding). Eagerly re-reading every file
  defeats the handoff.
- **Dispatch briefs pass PATHS, not pasted content**: agents Read
  `.workflow/<slug>/spec.md` and impl/diff paths themselves. Never paste
  session history into a fresh agent's brief.

## Model pins

**Pin the model on every agent dispatch — never inherit the session model.**
Measured on a planted-defect audit battery (N=5, Wilson CI, June 2026 model
versions; full numbers in `docs/measurements/RESULTS.md` in the repo):

| reviewer variant | recall | false-positive (decoys) | $/review |
|---|---|---|---|
| opus (no guard) | 0.84 | 0.47 | 0.41 |
| opus + precision guard | 0.72 | 0.00 | 0.41 |
| **sonnet** | **0.75** | **0.13** | **0.23** |
| haiku | 0.68 | 0.00 | 0.07 |

Opus has best raw recall but over-flags (3.6× sonnet's FPs); the precision
guard zeroes its FPs but drops recall to 0.72 — sonnet at 1.8× the cost.

| Role | Model |
|---|---|
| sota, premortem-lite, spec, planning (design judgment) | opus |
| /verify test author; grade adversarial pre-check | sonnet |
| audit reviewers (default — measured sweet spot) | **sonnet** |
| security-critical final pass (max recall; human triages FPs) | opus unguarded |
| vision (UI grade, multimodal) | opus |

**Never haiku for review** (recall 0.68, misses DRY/structural defects).
**Never the blunt precision guard on opus** — if you need low FP, use sonnet.

## Leaf-skill contract

If invoked by `/ship`: return control cleanly — no continuation past the
skill's output; `/ship` detects filesystem state and resumes. Standalone:
suggest the next step (`/verify <slug>`, `/ship <slug>`, ...).
