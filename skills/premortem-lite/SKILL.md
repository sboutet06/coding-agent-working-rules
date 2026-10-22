---
name: premortem-lite
description: >
  Lightweight structural pre-mortem of a design dossier (sota.md) or feature
  spec (spec.md) BEFORE implementation. Spawns an INDEPENDENT sub-agent that
  attacks the target's structure across 13 axes (logic, contracts, blast
  radius, prerequisites, evolution, observability, etc.) and produces a
  citation-grounded findings file. Use when user invokes /premortem-lite,
  says "attack this plan", "find structural holes in spec", "pre-mortem the
  design", or auto-invoked by /ship at tier=large. Distinct from the heavy
  /premortem protocol (which executes Python sims/probes against the plan)
  — this is a fast, judgment-based first pass that produces an authoring
  prompt, not a verdict.
disable-model-invocation: true
allowed-tools: Read Write Edit Bash Grep Glob Agent
---

You orchestrate a structural pre-mortem attack on a design or spec artefact:
spawn an independent sub-agent, receive findings, write them to
`.workflow/<slug>/premortem.md`, surface to user for triage.

Shared conventions (findings format, file handoffs, model pins, /ship
contract): see `../CONVENTIONS.md`.

Today's date: !`date +%Y-%m-%d`

## The 7 mitigations (design invariants — stated once, hard rules)

1. **Independence.** The attack runs in a fresh sub-agent; the main thread
   orchestrates and writes, it never attacks — the context that read
   /sota's design or /spec's framing cannot honestly attack them.
2. **Anti-strawman.** Target schema is gated: missing required sections →
   refuse and redirect to /spec or /sota. Never attack an under-specified
   target — you would attack a strawman of your own interpretation.
3. **Citation-grounded YAML.** No prose narratives, no fictional dates.
   Every finding cites the target verbatim; no quote = rejected at
   validation, not logged.
4. **Coverage reported.** A clean report with low coverage is
   uninterpretable — always surface the coverage number.
5. **Sincere quota.** Min 3 / max 8 findings are guidelines, not targets:
   no padding with cosmetic findings, no capping away obvious issues.
6. **No auto-patch.** The skill is read-only on the target: it produces
   findings, never modifications. Every finding is user judgment — even
   when "the suggested clarification feels obvious", you record, the user
   triages. Findings are an authoring prompt, not a verdict.
7. **Tier-gated.** trivial/small → refuse. medium → opt-in (`--premortem`
   flag or `--target spec`). large → default-on, `--no-premortem` opt-out.

## When to skip / redirect

- **tier=trivial / tier=small** → exit immediately (mitigation #7):
  *"target tier <X> — pre-mortem skipped. Run /spec + tests, ship."*
- **Target schema incomplete** → reject per mitigation #2 (see Step 1).
- **`.workflow/<slug>/premortem.md` already exists with `status: triaged`**
  → exit silently. Re-run requires explicit `--rerun` flag.
- **User asks for the heavy executable pre-mortem** (Python sims, external
  probes) → redirect to the standalone `/premortem` protocol (a separate
  `premortem.md`, not included in this repo). Deeper, more expensive, produces
  executable artefacts; this skill is the lightweight cousin.

## Inputs (ask if missing)

1. **Slug** — kebab-case, ≤30 chars. Inherited from `/ship`; standalone, ask.
2. **Target** — `sota` or `spec`. Default: `sota` if
   `.workflow/<slug>/sota.md` exists and `spec.md` does not; else `spec` if
   `spec.md` exists; else exit: *"no target artefact found at
   `.workflow/<slug>/{sota,spec}.md`. Run /sota or /spec first."*
3. **Tier** — read from target frontmatter. small/trivial → exit
   (mitigation #7).

Flags:
- `--target <sota|spec>` — override default target.
- `--rerun` — allow operating on an existing triaged premortem.md; archives
  the prior file first (rename to `premortem-<prior-date>.md` — never
  destroy history silently).

## Step 1: validate target schema (mitigation #2)

Read the target file. Verify the minimal section schema needed to be
attackable:

**For `sota` target**, required h2 sections: `Decision`;
`Constraints + non-goals` (or separately); `Approaches` (≥3 h3
sub-approaches A/B/C); `Recommendation` (chosen approach + justification +
**Residual risk** + **Early signal**); `References`.

**For `spec` target**, required h2 sections: `User story`;
`Acceptance criteria` (≥1 falsifiable AC); `Happy path`; `Edge cases` (≥3
at tier=medium+); `Non-goals`; `Non-functional requirements`;
`Open questions`.

Detection commands:

```bash
grep -E '^## ' ".workflow/<slug>/<target>.md"        # section presence
grep -cE '^- \*\*EDGE' ".workflow/<slug>/spec.md"    # count edges
grep -cE '^### [A-C] —' ".workflow/<slug>/sota.md"   # count approaches
```

If any required section is missing or empty, refuse: *"Target schema
incomplete. Missing or empty sections in `.workflow/<slug>/<target>.md`:
<list>. Pre-mortem refuses to attack an under-specified target — it would
inevitably attack a strawman of my own interpretation rather than the target
itself. Send back to /<spec|sota> to populate these sections, then
re-invoke."*

## Step 2: spawn independent sub-agent for the attack (mitigation #1)

Use `Agent` with `subagent_type: general-purpose`, **model: opus** (design
judgment — see `../CONVENTIONS.md` §Model pins). Pass the target PATH — the
sub-agent Reads the file itself. Independence is about fresh context (no
prior exposure to the artefact's framing), not withheld bytes.

Brief template:

```
You are a STRUCTURAL PRE-MORTEM REVIEWER. You attack the structure of a
design/spec artefact BEFORE any code is written. Your goal: find structural
failures that would survive any quality of implementation because they are
wired into the form of the document itself.

You are an INDEPENDENT reviewer — you have no prior context on this artefact
or its author. Approach it adversarially. Look for what is NOT said as much
as what IS said.

Read the target at .workflow/<slug>/<target>.md. Do not read anything else
about this feature — the target is your entire input.

## Scope — the 13 structural axes

Pick the 3-5 MOST RELEVANT axes for this target. Quality over quantity.
Cosmetic findings are noise; structurally important findings are signal.

A1. Logic and invariants — does the artefact contradict itself in some
    subregime?
A2. Contracts and interfaces — are boundaries specified on both sides
    (producer / consumer)? null vs absent? ordering?
A3. Blast radius and topology — if component X fails, what falls with it?
    Implicit atomicity? SPOF? Retry amplification?
A4. Prerequisites and external dependencies — what must exist before line
    one? Account, quota, license, certificate, network access overlooked?
A5. Physical anchors — what does the artefact structurally need from the
    physical world? GPU? Intra-DC latency? Regional residency?
A6. Evolution and structural debt — does the artefact create debt payable
    in 6-24 months? Frozen schema, premature or missing abstraction?
A7. Conceptual observability — does the artefact specify how to know it
    works? Silent failure possible? SLIs absent?
A8. System boundaries — gray zones of responsibility? Upstream validation
    not assigned? Out-of-scope exceptions?
A9. Data model consistency — entities / relations consistent? Orphans?
    Nullability? Uniqueness?
A10. Temporal assumptions and lifecycle — order, frequency, lifetime
     explicit? Read/write ratio assumed? Retention absent?
A11. Reversibility and exit paths — cost of return if the founding
     hypothesis fails? Frozen data format? Undocumented lock-in?
A12. Composability with the existing — coherence with the ecosystem
     already in place? Naming, error handling, auth, packaging
     incompatible?
A13. Implicit consumer model — who consumes, with what patterns? API
     designed for one use, deployed for another?

## Output format — STRICT YAML

Return findings as a single YAML block. Every finding MUST have a verbatim
citation from the target text — if you cannot quote the exact phrase, you
are inferring not finding: do not log it.

Minimum 3 findings if any structural risk exists, maximum 8 — but the quota
is sincere: no padding with cosmetic findings to hit 3, no skipping obvious
issues to stay under 8. `findings: []` is valid if the target genuinely is
clean — say so honestly and list what you inspected in
`sections_attacked_list`.

```yaml
target: <sota | spec>
target_path: .workflow/<slug>/<target>.md
sections_total: <N>
sections_attacked: <M>
sections_attacked_list: [<section 1>, <section 2>, ...]
sections_skipped:
  - name: <section name>
    reason: <"trivial, no structural surface" | "covered by /audit downstream" | ...>
findings:
  - id: F1
    axis: A<n>
    severity: critical | warning | info
    title: <one-line>
    citation: |
      <verbatim phrase from the target, with section name + approx location>
    paraphrase: |
      <how you read it / what the artefact is asserting>
    structural_risk: |
      <2-4 lines: the failure mode this enables. State the mechanism —
      ambiguity, missing boundary, undefined ordering. NAME the failure.>
    suggested_clarification: |
      <concrete 1-3 sentence proposal to close this. Do NOT apply it
      yourself; the user triages.>
  - id: F2
    ...
out_of_scope_observations:
  - <issue belonging to another review pass — security, perf, UX, code
    quality — and which downstream pass should catch it>
```

## Rules of engagement

- **No fictional narratives.** Do NOT write "In Feb 2027 two teams will
  implement metadata differently because…" — that is the heavy /premortem
  protocol's territory (and even there its most-criticized aspect). State
  the structural ambiguity directly: "metadata's null-vs-absent semantics
  are unspecified — two consumers can read divergently".
- **Distinguish severity.**
  - `critical` — target promises something it cannot deliver under stated
    constraints, OR a structural ambiguity that will produce divergent
    implementations.
  - `warning` — a structural gap that a reasonable implementer might fill
    in a way the author would not endorse.
  - `info` — clarification would help correctness but the target is
    self-consistent.
- **Stay in scope.** You attack STRUCTURE — not security (security-auditor
  downstream), scalability (system-pressure-reviewer), user input pressure
  (user-pressure-reviewer), or implementation bugs (code-reviewer). Deflect
  those into `out_of_scope_observations:`, never log them as structural
  findings.
```

After the agent returns:

1. **Validate the YAML parses.** If malformed, request one re-run with the
   parse error pasted in; still malformed → escalate to user with raw output.
2. **Sanity-check citations** (mitigation #3): grep the target for each
   cited phrase. No match within reasonable tolerance (trivial
   whitespace/punctuation allowed) → mark the finding
   `citation_unverified: true` when writing the file — do NOT drop it
   silently.
3. **Compute coverage** (mitigation #4):
   `coverage_pct = sections_attacked / sections_total`.

## Step 3: write premortem.md

Write to `.workflow/<slug>/premortem.md` (create dir if missing).
Frontmatter per `../CONVENTIONS.md` §Findings files (`phase: premortem`),
plus: `target`, `target_path`, `findings_count`, `findings_info`,
`sections_total`, `sections_attacked`, `coverage_pct`.

Body structure:

```markdown
# Pre-mortem (lite) — <slug> — <date>

Attack target: `.workflow/<slug>/<target>.md`

## Coverage
- Sections in target: <M>; attacked: <K> (<coverage_pct>%)
- Sections attacked: <list>; skipped (with reason): <list>

## Findings

### F1 — A<n> — 🔴/⚠️/ℹ️ <title>
**Citation** (from target):
> <verbatim phrase>
**Paraphrase**: <reviewer's reading>
**Structural risk**: <2-4 lines>
**Suggested clarification**:
> <concrete proposal>
**Triage decision**: _(populated by user)_
- [ ] accept (no change — rationale: …)
- [ ] patch target (edited `<target>.md` — link to diff/commit)
- [ ] reject (finding wrong — rationale: …)

(repeat per finding)

## Out-of-scope observations
- <observation> — should be caught by <downstream pass>

## Triage notes
(user populates — link commits / decisions)
```

If `findings: []` (sincere clean report), the Findings section reads: *"No
structural findings on <K>/<M> sections inspected. Target appears internally
consistent on the axes attacked. This is NOT a guarantee — the structural
attack surface inspected was clean. Downstream passes (/verify behavior,
/audit) still apply."*

## Step 4: surface to user

≤15-line summary:

```
Pre-mortem (lite) on <target>: .workflow/<slug>/premortem.md
🔴 Critical: <N>  ⚠️ Warning: <N>  ℹ️ Info: <N>
Coverage: <K>/<M> sections attacked (<pct>%)
Findings to triage:
- F1 — A<n> — <title>
…
Each finding cites the target verbatim and suggests a clarification. Triage
via the file's "Triage decision" blocks OR by editing the target directly.
When all triaged, mark file `status: triaged`. Disagreeing with the
reviewer's reading is expected — pre-mortem is an authoring prompt, not a
verdict.
```

Do NOT auto-patch the target (mitigation #6). If invoked by `/ship`: return
control — /ship resumes from PREMORTEM_UNRESOLVED (findings) or skips ahead
(clean).

## Relationship to other skills

- `/sota` and `/spec` produce the targets this skill attacks (pre-spec at
  tier=large, pre-test at tier=medium with `--premortem`).
- `/verify` (gen) is the behavioral attack from the spec — complementary to
  this structural attack.
- `/audit` runs POST-implementation — too late for pre-impl structural
  findings, which is why /premortem-lite exists.
- Heavy `/premortem` (Python sims/probes) — the expensive cousin.
