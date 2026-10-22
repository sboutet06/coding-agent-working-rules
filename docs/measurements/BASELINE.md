# Baseline — ship audit/grade quality, before optimisation

> Measurement notes from my private benchmark harness for these skills
> (June 2026, Claude Opus / Sonnet / Haiku versions of that time). The harness
> itself — the scripts and fixtures referenced below — is not published yet.
> `grill-me` and `migrate` are measured here but not included in this repo.

Date: 2026-06-17 · model build: claude-opus-4-8 / sonnet / haiku (CLI 2.1.178)
Battery: `bench/fixtures/planted/` — 10 distinct defects in one Python module
+ its test file. N=3 per tier (floor; use N=5 for go/no-go decisions).

Reproduce: `bash bench/baseline.sh` then read `bench/results.jsonl`.

## What this measures

The grade/audit phase of `/ship` = a reviewer agent over a diff. Today `/ship`
**pins no model**, so the audit inherits the session model (**opus**). This
battery feeds a frozen defect-laden file to the reviewer and counts how many
of 10 known defects it surfaces — cheaply, repeatably, programmatically.

It is the "before" picture for optimisation #1 (Model Selection + `model:` pin).

## Results (N=3, median [min–max])

| tier | detect (any sev) | detect (≥ expected sev) | $/run | s/run | out tok/run |
|---|---|---|---|---|---|
| **opus** (current default) | 9 [9–10] | 7 [7–9] | 0.38 [0.37–0.70] | 64 [58–78] | 4532 |
| **sonnet** | 10 [10–10] | 8 [7–9] | 0.25 [0.24–0.33] | 94 [94–103] | 5528 |
| **haiku** | 8 [8–8] | 7 [7–7] | 0.07 [0.03–0.10] | 31 [29–65] | 3071 |

Per-defect catch (across the 3 runs): see `uv run bench/score.py baseline-<tier>`.

### Reads

- **haiku is blind to structural/judgment defects**: misses DRY duplication
  entirely (0/3) and unbounded-loop mostly (1/3), while nailing pattern-matchable
  ones (hardcoded secret, SQL injection, off-by-one: 3/3). Confirms the doctrine
  "cheapen mechanics, never judgment" — haiku is safe for transcription-tier
  impl, **never** for review.
- **sonnet ≥ opus here on both quality and cost** (10 vs 9 caught, $0.25 vs
  $0.38). On a single-file direct review, paying opus for the audit buys nothing.
  This is the measured case for pinning sonnet on standard review and reserving
  opus for the final whole-branch pass — to be confirmed on a multi-file fixture.
- **detection-at-severity (7–8/10)** is the stricter, noisier metric: defects
  caught but downgraded. This is the number the controller-tier effect reported by
  [superpowers](https://github.com/obra/superpowers/blob/be400204b345c805c70c1df1223ddc971133822d/docs/superpowers/specs/2026-06-10-strict-cost-sdd-design.md) (L2: sonnet controller → reviewers collapse into plan-compliance advocacy) would
  move; not yet exercised here.

## Honest limitations

1. **Single-file, direct review.** This measures raw defect-spotting, NOT
   superpowers' setting where haiku failed 0/10 — that was a multi-turn *task
   reviewer under a controller*, judging severity under authority pressure. Our
   battery does not (yet) reproduce judgment-under-pressure. Don't read "haiku
   8/10 here" as "haiku is fine as a reviewer".
2. **N=3 is a floor.** Variance is real (opus $0.37→0.70, haiku 29→65s). Widen
   to N=5 before any model-selection decision ships.
3. **Keyword/line matching can undercount** a well-worded finding (opus's D9
   race dropped to 1/3 likely on wording, not blindness). The per-defect view
   exposes this; refine `defects.yaml` keywords as needed — never to inflate a
   score.
4. **Quality axis only.** Cost/speed are captured free here. The spec and
   test-author phases have no micro-bench yet (scaffolded in the private harness);
   L2 full-lifecycle smoke not yet built.

## Reliability-layer re-score (2026-06-17, supersedes the raw table above)

Re-scored with `stats.py` (Wilson CI, item discrimination, decoys). The battery
was hardened to 13 defects (+H1 timing-attack, +H2 predictable-token, +H3
weak-hash) and 3 decoys. Free re-score of the existing baseline runs:

- recall (any sev), Wilson 95% CI: opus 0.93 [0.79–0.98], sonnet 1.00 [0.89–1.00],
  haiku 0.80 [0.63–0.90]. decoy-flag rate 0.00 for all (decoys absent from the
  pre-hardening runs — a fresh sweep on the current fixture is needed for a real
  precision number).
- **Item analysis (pooled, 9 takers): 7 of 13 defects are CEILING** (p≥0.90,
  r_pb=0) — the pattern-matchable ones carry no information. The discriminating
  core is D3-authz, D6-dry (r_pb=0.97), D8-loop, **H1-timing (p=0.44, r_pb=0.71)**
  and **H3-weak-hash (p=0.67, r_pb=0.97)**. D9-race shows negative discrimination
  (scoring artifact — co-located with D8; treat as broken, not a real signal).
- **CEILING WARNING fires on opus and sonnet** → confirms "100% is suspect": this
  battery is still too easy at the top. Maintain it as an item bank — keep adding
  hard items (H-series) until the best tier drops below ceiling.

Skill-quality batteries (`skillq/`, judge κ=1.00 on both): grill-me opus/haiku
~0.83 coverage (branch identification doesn't separate them; G6/G7 discriminate);
sota opus 1.00 vs haiku 0.73 (A3 semantic approach + T2 ops tradeoff discriminate).

## Hardened battery — ceiling broken (2026-06-17, N=5, 17 defects + 3 decoys)

Added 4 reasoning-heavy defects (HX4 merge-boundary off-by-one, HX5 pagination
overlap, HX6 user-enumeration timing, HX7 false-success return). Fresh N=5 sweep:

| tier | recall (Wilson 95%) | detection@sev | decoy-flag (FP↓) | $/run |
|---|---|---|---|---|
| opus | 0.84 [0.74–0.90] | 0.59 [0.48–0.69] | **0.47 [0.25–0.70]** | 0.41 |
| sonnet | 0.75 [0.65–0.83] | 0.65 [0.54–0.74] | 0.13 [0.04–0.38] | 0.23 |
| haiku | 0.68 [0.58–0.77] | 0.56 [0.46–0.67] | 0.00 [0.00–0.20] | 0.07 |

- **No ceiling warning** — top tier (opus) recall 0.84 < 0.90: the battery now
  discriminates instead of saturating. Goal "100% suspect" satisfied.
- **Precision signal emerged**: opus over-flags (0.47 decoy rate) — high recall +
  high false positives; haiku the inverse (0.68 / 0.00). The precision/recall
  tradeoff is now measurable, which recall-only could never show.
- Discriminating item bank: HX7 (p=.13, r_pb=.59), HX4 (.27/.39), H1-timing
  (.40/.76), H3-hash (.60/.64), D3/D6/D8; HX6 at FLOOR (p=.07 — legitimately hard,
  will rise as models improve). HX5 saturated (retire/harden next cycle).
- **Known limitation**: decoy N1 (a hardcoded 30s timeout) is debatable — a
  reviewer flagging "make configurable" is not strictly wrong, so part of opus's
  0.47 is contestable, not pure FP. N2/N3 are unambiguous. Tighten N1 to a
  truly-innocuous helper before quoting precision as a headline number.

## Next measurement

After implementing Model Selection (pin sonnet for task review, opus for final
whole-branch), re-run `bench/baseline.sh` and diff `results.jsonl`. Expected
delta: same-or-better detection at lower $/run vs the current opus-inherited
audit.
