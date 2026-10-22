# Bench results — ship skills, reliability-scored (2026-06-17)

> Measurement notes from my private benchmark harness for these skills
> (June 2026, Claude Opus / Sonnet / Haiku versions of that time). The harness
> itself — the scripts and fixtures referenced below — is not published yet.
> `grill-me` and `migrate` are measured here but not included in this repo.

All numbers: N=3–5, Wilson 95% CI, judge κ-validated. Opus vs Haiku contrast
(the discriminating pair). See METHODOLOGY.md for methods, BASELINE.md for the
code-defect (audit) battery detail.

## Audit (code-defect battery, deterministic scoring, N=5)

| tier | recall | detection@sev | decoy-flag (FP↓) |
|---|---|---|---|
| opus | 0.84 [0.74–0.90] | 0.59 | **0.47 [0.25–0.70]** |
| sonnet | 0.75 [0.65–0.83] | 0.65 | 0.13 |
| haiku | 0.68 [0.58–0.77] | 0.56 | 0.00 |

Ceiling broken (opus < 0.90). 8/17 defects discriminate; HX6 at floor.

## Skill-quality (coverage batteries, opus-judge, N=3)

| skill | opus cov | haiku cov | opus decoy-FP | haiku decoy-FP | judge κ | discriminating items |
|---|---|---|---|---|---|---|
| grill-me (identify) | 0.83 | 0.83 | 0.00 | 0.00 | 1.00 | G6, G7 |
| **sota** | 1.00 | **0.73** | 0.00 | 0.00 | 1.00 | A3 (vector), T2 (ops) |
| premortem-lite | 0.94 | 0.89 | 0.00 | 0.00 | 1.00 | S6 (retention) |
| spec | 1.00 | 0.96 | 0.00 | 0.11 | 1.00 | F8 (concrete fixtures) |
| **verify** | 0.94 | **0.83** | **0.56** | 0.00 | 0.77 | T4 (404), T6 (surface) |
| migrate | 1.00 | 1.00 | 0.22 | 0.00 | 0.73 | (coverage saturated) |
| grill-me-depth | 1.00 | 1.00 | — | — | 1.00 | both FLAG the traps |

grill-me-depth uses ambiguity traps: a vague expiry answer ("it depends… 14 or
30… we'll see") and a contradiction (never-leaves-boundary vs download-to-device).
Both opus and haiku correctly refuse to rubber-stamp — they flag G3 UNRESOLVED
(rule 6) and G6 NEEDS-ARBITRATION (rule 1). The skill's explicit rules carry both
tiers, so this axis doesn't separate models — but it validates the skill's
ambiguity discipline works as written. κ=1.00.

## Cross-cutting findings

1. **Coverage saturates for strong models.** Both tiers enumerate the obvious
   concerns, so raw coverage rarely separates opus/haiku. Discrimination lives in
   (a) the subtle long-tail concerns (A3, S6, F8, T6, G6) and (b) the precision
   dimension below. Per the "100% suspect" rule, saturated coverage items are
   flagged CEILING and are not read as a quality verdict.

2. **Opus over-flags — the single most robust, most actionable signal.** Opus
   posts high recall AND high false positives (verify decoy-FP 0.56, audit 0.47,
   migrate 0.22); haiku is the mirror image (near-zero FP, lower recall). "More
   thorough" opus is also "more trigger-happy". Directly relevant to the coming
   Model Selection optim: an opus reviewer needs a precision guard; a cheap
   reviewer under-flags. Recall-only could never have shown this — the decoys did.

3. **sota is the cleanest coverage discriminator** (opus 1.00 vs haiku 0.73):
   weak models default to keyword approaches and miss the ambitious one.

4. **Judge trustworthy everywhere**: κ ∈ [0.73, 1.00], all ≥ substantial
   (Landis-Koch). premortem's judge was fixed this round (κ 0.57→1.00) via an
   area-coverage instruction + sharper gold wording — the validation loop works.

## Optim #1 — Model Selection (measured 2026-06-18)

Before: ship pinned no model → every dispatch inherited the session model (opus).
The bench was used to choose the per-role model. The audit-reviewer measurement:

| reviewer variant | recall | FP (decoys) | $/review |
|---|---|---|---|
| opus (no guard) | 0.84 | 0.47 | 0.41 |
| opus + precision guard | 0.72 | **0.00** | 0.41 |
| **sonnet** | 0.75 | 0.13 | 0.23 |

The precision guard (added to test the over-flag finding) **zeroed opus's false
positives but dropped recall to 0.72** — no better than sonnet at 1.8× the cost.
Conclusion (overturned the obvious guess that opus = best reviewer): **sonnet is
the best default reviewer**; guarded-opus is dominated. Opus unguarded reserved
for the security-critical final pass (max recall, human triages the FP).

Shipped to `skills/CONVENTIONS.md` (Model pins): test-author, grade, and default
audit → sonnet; sota/grill/premortem/spec + security-auditor + vision → opus;
never haiku for review; the blunt precision guard is rejected. Cost win: the
phases that were silently opus now run sonnet (~½ price) with equal-or-better
review quality.

## Optim #2 — File handoffs (measured 2026-06-18)

`bench/handoff_measure.py`, from 71 real captured reviewer outputs (median 4709
tok each): everything an agent returns in-context becomes the controller's
resident input, re-read every turn. A 5-reviewer audit:

| mode | resident in controller / turn |
|---|---|
| return-in-context | 23,545 tok |
| file-handoff (surface from summaries, read files on-demand) | 750 tok (**96% lighter**) |
| file-handoff but eagerly read all 5 files to aggregate | ~23.5k once, then 750/turn (**~85%** over 8 turns) |

The 96% is the resident-per-turn figure and holds ONLY if ship surfaces 🔴 from
the ≤15-line summaries and opens a full per-agent file on-demand. If it eagerly
reads all files to aggregate, it re-inhales the detail and the saving drops to
~85% (one aggregation read). `skills/CONVENTIONS.md` was corrected to the on-demand pattern.

**Quality, now measured (`bench/handoff_quality.py`, sonnet N=5).** The risk was
summary fidelity — does the returned summary carry every serious defect the full
findings caught? The reviewer emits both `findings` (the file) and
`summary_serious` (the handoff); fidelity = serious defects carried / caught.

| summary contract | fidelity | dropped |
|---|---|---|
| generic "≤15-line summary" | 0.95 [0.86–0.98] | D8-unbounded-loop, D9-race (some runs) |
| **"carry EVERY serious finding; lines govern prose, not inclusion"** | **1.00 [0.94–1.00]** | none (58/58) |

So file-handoff is NOT free by default — a line-capped summary drops ~5% of
serious defects (a measured quality loss). With the completeness-required
contract it is lossless on serious-defect surfacing (1.00). `skills/CONVENTIONS.md` ships the
corrected contract: carry every 🔴/⚠️ headline; only *minor* findings stay in the
file. This is the measured close the cost-only projection lacked.

Shipped to `skills/CONVENTIONS.md` (File handoffs): each reviewer/grader/adversarial
agent writes its findings to a per-agent file and returns only a ≤15-line
summary + path; ship assembles the audit file from summaries and reads detail
on-demand; dispatch briefs pass paths, never pasted file bodies or session
history.

## Optim #3 — task decomposition, pre-build measurement (2026-06-18)

Before building: does reviewing a focused slice (= one decomposed task) catch its
defects better than reviewing the whole file (= monolithic impl, final-only
review)? `bench/slice_review.sh` + `slice_score.py`, sonnet N=3 per slice:

| | focused per-task review | whole-file review |
|---|---|---|
| recall | 0.82 [0.70–0.90] | 0.75 |
| decoy-FP | 0.22 | 0.13 |

Honest verdict — per-task review is NOT a clear defect-catching win:
- recall +0.07 but **CIs overlap** (inconclusive at N=3 — the methodology floor);
- **precision is worse** (FP 0.22 vs 0.13 — small slice → more nit-flagging);
- **structurally blind to cross-task defects**: D6-DRY (duplication spanning
  `register` and `validate_email`, in different slices) is **never** caught even
  focused — the measured "loss of holistic view".

Design consequence: decomposition's justification is **context hygiene +
recoverability + falsifiable progress** (goal #1/#3, already evidenced by optim
#2), NOT defect-catching. Per-task review = a cheap fast LOCAL gate; the **final
whole-branch audit must stay** as the cross-cutting net (it catches what per-task
review cannot — measured). They are complementary, not redundant.

## E2E — full ship lifecycle on a tier=large feature (2026-06-18)

Fixture `bench/e2e/share-service/` (8 ACs, 16 failing tests, attack surface +
concurrency). Ran the whole pipeline autonomously in a throwaway repo:

branch `ship/share-service` → green baseline → PLAN (opus planner: 3 tasks,
obvious → no human gate) → IMPL (3 fresh sonnet implementers + per-task sonnet
review + ledger) → 16/16 green → grade (regression PASS + opus adversarial,
escalated for crypto/auth/concurrency) → audit (5 parallel reviewers, file-handoff).

Outcome:
- **Autonomous impl worked**: human out of the loop, 3 tasks implemented to green,
  clean conventional-commit history + durable ledger.
- **All 3 optims exercised**: model pins (opus planner/adversarial/security-auditor,
  sonnet impl/review/other audit), file handoffs (6 per-reviewer audit files +
  ≤15-line summaries), decomposition + ledger. Plus feature branch, green baseline,
  regression gate.
- **The measured design decision was confirmed in the wild**: the per-task local
  review passed all 3 tasks clean and did NOT catch the concurrency race or the
  memory leak; the adversarial grade + final audit did. Exactly what the slice
  measurement (optim #3) predicted — per-task review is blind to cross-task /
  concurrency defects, the final audit is the net.
- **5 reviewers converged**: all 5 independently flagged the redeem TOCTOU race
  (🔴), 4/5 the rate-limiter memory leak, 3/5 its process-local limit. Multi-
  reviewer agreement = high-confidence findings. New catches beyond the grade:
  `tenant_id=None` authz bypass, and `__pycache__` tracked in git (a real slip
  the orchestrator made — the audit caught it).

Honest scope: N=1 existence proof that the pipeline runs end-to-end and surfaces
real defects — not a distribution. The first-pass code has real 🔴 (race, leak)
by design: that is the surface the grade/audit exist to catch, and they did.

## Known limitations / item-bank maintenance

- Several coverage batteries (migrate, spec, grill-me-depth-clean) sit at the
  ceiling: single-shot enumeration is easy. They still measure precision (decoys)
  and the subtle items; harden the must-surface long tail over time.
- audit decoy N1 (a documented constant) is debatable — part of opus's 0.47 is
  contestable, not pure FP. Tighten before quoting precision as a headline.
- grill-me identification (sees branches) ≠ depth (wrings each dry). The depth
  battery with a full answer-key ceilinged; the ambiguity-trap rerun tests the
  skill's rule-6 (refuse to rubber-stamp a vague/contradictory answer) — the real
  "wring it dry" signal.
