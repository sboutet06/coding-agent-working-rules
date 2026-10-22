# Measurement methodology — reliability doctrine for the whole bench

> Measurement notes from my private benchmark harness for these skills
> (June 2026, Claude Opus / Sonnet / Haiku versions of that time). The harness
> itself — the scripts and fixtures referenced below — is not published yet.
> `grill-me` and `migrate` are measured here but not included in this repo.

Every number this bench reports must be defensible. This file states the methods
and why they are the established (SoTA / textbook) choice. The rules here apply
to **all** batteries — the planted-defect code battery (`fixtures/planted/`) and
the skill-quality coverage batteries (`skillq/`).

## 1. Never a bare point estimate — Wilson score CI

Detection rate, recall, decoy rate, coverage are all proportions `k/n`. At our
sample sizes (N=3–10 runs) the **Wald** interval is disqualified: it collapses to
zero width at k=0 or k=n and breaches [0,1]. We use the **Wilson score interval**
(point + lo–hi), the recommended default for small n.

> Wilson (1927); Brown, Cai & DasGupta (2001), *Statistical Science* 16(2),
> "Interval Estimation for a Binomial Proportion" — recommends Wilson or Jeffreys
> for small n. Do **not** bootstrap a plain proportion at small n (the empirical
> distribution is degenerate); bootstrap is for non-proportion metrics only.

Implemented in `stats.wilson`.

## 2. A 100% score is a red flag, not a victory — item discrimination

"A 100% score is suspect: the test is not challenging enough." This is the **ceiling
effect**, and it is formal:

- **CTT**: item discrimination = point-biserial `r_pb` between the item and the
  total score. Its `√(p·q)` factor → 0 as difficulty `p` → 1 (or 0). An item
  everyone passes has zero variance, so `r_pb` is undefined / zero. It carries no
  information about who is more able.
- **IRT**: item information `I(θ) = a²·P(θ)·Q(θ)` → 0 when `P → 1`. A saturated
  item shrinks no uncertainty.

So the scorers compute, **pooled across all runs/tiers (= test-takers)**, each
item's difficulty `p` and discrimination `r_pb`, and flag:
- `p ≥ 0.90` → **CEILING / non-discriminating** (item-analysis convention),
- `p ≤ 0.10` → **FLOOR**,
- `|r_pb| < 0.20` → **weak**,
- `r_pb < 0` → broken item (often a scoring artifact — investigate, don't keep).

A test whose best taker saturates recall triggers a **CEILING WARNING**: it
cannot rank improvements above it → add harder items until the top drops below
ceiling. This makes the bench self-diagnosing.

> Point-biserial: en.wikipedia.org/wiki/Point-biserial_correlation_coefficient.
> IRT information: en.wikipedia.org/wiki/Item_response_theory; "Lost in
> Benchmarks? Rethinking LLM Benchmarking with IRT" (arXiv:2505.15055, 2025)
> quantifies benchmark saturation. Discrimination maximal at p≈0.50 (Crocker &
> Algina); flag p>0.90 / p<0.10.

Implemented in `stats.point_biserial`, surfaced in both scorers' "item analysis".

## 3. Recall alone is gameable — negative controls (decoys)

A detector that flags everything gets recall 1.0. Recall-only therefore cannot
tell a real detector from "flag-everything". We embed **negative controls**:
clean code (planted battery) / settled-or-trivial concerns (skill batteries) that
must **not** be flagged. A finding landing on one is a **false positive**; the
`decoy-flag rate` (with Wilson CI) is the precision guard. Decoys also raise
difficulty (anti-ceiling): over-flaggers score high recall but high decoy rate.

> Precision/recall gameability: en.wikipedia.org/wiki/Precision_and_recall.
> Negative-control evaluation: Ribeiro et al. (2020) CheckList INV tests
> (ACL 2020); Gardner et al. (2020) Contrast Sets (arXiv:2004.02709) report up to
> 25% accuracy drops once the easy ceiling is removed. Match the decoy:defect
> ratio to the deployment base rate — precision is prevalence-dependent.

## 4. Semantic scoring needs a judge — and the judge must be validated

Code findings match by line+keyword (deterministic). Skill-quality coverage is
natural-language, so an **LLM judge** maps surfaced→gold. Two non-negotiables:

- **Judgment is not cheapenable** → judge = most capable model (here `opus`).
  (Cheap reviewers advocate for defects; [superpowers](https://github.com/obra/superpowers/blob/be400204b345c805c70c1df1223ddc971133822d/docs/superpowers/specs/2026-06-10-strict-cost-sdd-design.md)' planted-defect L3 found
  haiku reviewers flagged 0/10 at correct severity.)
- **Validate the judge** against hand labels with **Cohen's κ**; accept at
  κ ≥ 0.61 (substantial, Landis & Koch 1977). The grill-me judge scored κ = 1.00
  vs hand labels on run-1. Re-validate whenever the gold or judge prompt changes.
- **Bias guards**: judge is instructed to ignore wording/length/order (verbosity
  & position bias). For any *pairwise* judging, swap order and count only
  order-consistent verdicts; prefer a panel (PoLL) for high-stakes calls.

> Zheng et al. (2023) MT-Bench (arXiv:2306.05685): position bias (Claude-v1 only
> 23.8% order-consistent), verbosity bias, self-preference; GPT-4 judge agrees
> with humans 85% vs 81% human–human. Liu et al. (2023) G-Eval (arXiv:2303.16634).
> Verga et al. (2024) Panel-of-LLM-judges (arXiv:2404.18796). Cohen (1960) κ.

Implemented in `stats.cohens_kappa`; used in `skillq/score_skillq.py`.

## 5. Before/after comparisons — paired tests + effect size + power

Comparing two skill/model variants on the **same** frozen items is paired data:

- **Paired binary** (item caught under A vs B): **McNemar** on discordant counts
  `b, c`. At `b+c < 25` (our regime) use the **exact binomial**, not χ².
- **Paired scores** (judge/continuous): **Wilcoxon signed-rank**, or paired
  **bootstrap** for non-decomposable metrics.
- Always report an **effect size**: Cohen's `d_z` (0.2/0.5/0.8 = S/M/L) or
  **Cliff's δ** (Romano 2006: <0.147 negligible / <0.33 S / <0.474 M / large).
- **Power**: a *medium* paired effect (d_z≈0.5) needs ~34 paired items at
  α=.05/power=.80; McNemar power scales with the *discordant* rate, so for small
  expected gains budget 150–300+ items. N=3 runs is a **floor for CIs, not enough
  for a significance claim** — widen items/runs before declaring an optim "won".

> McNemar (1947); Edwards continuity correction; en.wikipedia.org/wiki/McNemar%27s_test.
> Cliff (1993); CRAN `effsize`. Berg-Kirkpatrick et al. (2012, EMNLP) paired
> bootstrap for NLP. Dietterich (1998) on comparing classifiers.

Implemented in `stats.mcnemar`, `stats.cliffs_delta`, `stats.bootstrap_ci`.

## 6. Operating rules (apply every run)

- N=3 = CI floor; **N=5+ before any go/no-go**; report ranges, never points.
- Freeze fixtures; pin the model per role during a comparison (else you measure
  routing noise).
- Maintain the battery as an **item bank**: when models improve and items hit the
  ceiling, retire/replace them with harder ones. A benchmark is never "done".
- No silent caps: if coverage is bounded (top-N items, N runs), say so.
