---
name: sota
description: >
  State-of-the-art research protocol for tier=large decisions. Produces 3 distinct
  approaches (pragmatic / industry-standard / ambitious), tradeoff comparison, and
  one recommended approach with explicit justification. Use when the user invokes
  /sota, says "research the SOTA", "design phase", "explore approaches for X", or
  before any new module/service/subsystem, new external dependency, architecture
  decision with ≥2 viable approaches, or data model design.
disable-model-invocation: true
allowed-tools: Read Write Edit Bash WebFetch WebSearch Agent
---

You are running the **design phase** for a tier=large decision. No code yet. Output a research dossier the user can approve before implementation.

Today's date: !`date +%Y-%m-%d`

## When to skip

- tier=trivial/small/medium (typo, rename, one-liner, single-file feature, multi-file refactor with known approach) → tell the user this is overkill, propose a one-paragraph design instead
- The user has already settled on an approach → skip research, go straight to plan
- Pure bug fix with one obvious resolution → skip

If unsure whether the task is actually tier=large, ask the user to clarify the decision in one sentence before researching.

## Inputs to gather first (ask if missing)

1. **Slug** — kebab-case identifier for this design decision (≤30 chars). If
   invoked from `/ship`, slug is inherited; standalone, ask.
2. **The decision** — one sentence. What are we picking between?
3. **Constraints** — language/runtime, deploy target, team size, performance budget, must-integrate-with X, license restrictions
4. **Non-goals** — what we explicitly aren't optimizing for
5. **Time-to-rough-prototype** — hours, days, or weeks?

Without these, research drifts.

## Protocol

### 1. Research current state (use today's date)

The `deep-research` skill is banned here (token cost) — research inline with
targeted WebSearch/Context7 calls, cap sources per approach.

Search recent (last 12 months) sources:
- WebSearch / WebFetch for blog posts, RFCs, post-mortems, conference talks
- Library docs via `mcp__plugin_context7_context7__query-docs` for any library mentioned
  (Context7 plugin, installed separately; without it, fetch the official docs)
- Well-known reference architectures from major vendors (AWS, GCP, Azure) where relevant

Track every URL you read so the user can verify.

### 2. Produce 3 distinct approaches

Each approach must come from a genuinely different angle. Cookie-cutter "minimal / medium / maximal" of the same architecture doesn't count.

Suggested framing:
- **A — Pragmatic / minimal**: smallest viable thing. Fewest moving parts. What ships in days.
- **B — Industry-standard**: what most teams in 2026 would pick. Boring tech, known operational profile.
- **C — Ambitious / novel**: leans on a recent technique or emerging tool. Higher upside, higher risk.

For EACH approach, output:
- **Sketch** (3-6 lines): core components and their relationship
- **Key dependencies**: libraries, services, runtime, hosted services
- **Failure modes**: what breaks under load, partial failure, ops mistake
- **Cost vector**: dev time, ongoing ops complexity, lock-in, latency, $/month at modest scale
- **When it's the right pick** / **when it isn't**
- **Source links** (3+ per approach where possible)

### 3. Synthesize a recommendation

ONE approach, named. Explicit reference to the alternatives:
- Why **not** A: ...
- Why **not** C: ...
- Why **B** (or whichever): ...

State residual risk — what could make us regret this in 6 months — and what early signal would tell us we picked wrong.

### 4. Surface to user, wait for approval

Output structure:

```
## Decision
<one-line restatement of what we're picking>

## Constraints + non-goals
<bulleted, from inputs gathered>

## Approaches

### A — <name>: <one-line>
<sketch / deps / failure modes / cost / source links>

### B — <name>: <one-line>
<...>

### C — <name>: <one-line>
<...>

## Recommendation: <chosen letter + name>

<2-4 sentence justification referencing why-not-others>

**Residual risk**: <what could make us regret it>
**Early signal**: <metric or observation that would prove it wrong>

## References
<flat list of all URLs consulted>
```

Then ask: *"Approve to proceed with <recommendation>, or push back on the framing?"*

### 5. Persist dossier on approval

Once the user approves (explicit "approved" / "go" / "ok"), write the dossier to
`.workflow/<slug>/sota.md` (create dir if missing). Use this frontmatter:

```yaml
---
slug: <slug>
date: <YYYY-MM-DD>
phase: sota
status: approved
chosen: <A | B | C>
chosen_name: <name of chosen approach>
---
```

The file body is the full dossier (Decision / Constraints / Approaches / Recommendation / References) as surfaced above. Persistence enables downstream skills (`/premortem-lite --target sota`, `/spec`) to attack or read the design without re-running research.

Suggest commit:

```
git add .workflow/<slug>/sota.md
git commit -m "docs(sota): <slug> — design dossier, chose <chosen>"
```

## Hard constraints

- Do NOT write any production code in this phase. Code = next round, after approval.
- Do NOT propose 3 minor variations of the same architecture. If you can't find 3 genuinely distinct angles, surface that fact ("only one viable approach found because <constraint>") and recommend skipping the design phase.
- Do NOT skip the references section — every claim about "X is faster" or "Y is the standard" needs a link.
- Doubt about a library version, API surface, or operational profile → use Context7 or fetch docs, never guess.
- Cap dossier at ~80 lines. Tighter is better — this is for human decision-making, not exhaustive coverage.

## Delegation

For genuinely independent research threads (e.g. "compare 4 message queues" — each queue is its own thread), launch parallel `Agent` calls with `general-purpose` subagent type, ONE per thread, each with a self-contained brief. Synthesize their findings into the dossier yourself; do NOT forward agent reports raw.

## After approval

Hand off to implementation. The next round operates under tier=large gates (atomic commits per logical unit, post-implementation review via `comprehensive-review:code-reviewer` agent, security scan via `comprehensive-review:security-auditor` if attack surface). Do NOT continue coding inside this skill — exit.
