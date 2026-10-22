---
name: spec
description: >
  Produce a falsifiable feature spec at .workflow/<slug>/spec.md. Use when user invokes
  /spec, says "write a spec for X", "let's spec out Y", or starts tier=medium+
  feature work without an existing spec. Acceptance criteria must be testable
  (numeric where possible). Edge cases require concrete fixture values, not
  abstract descriptions. Stakeholder-supplied edges mandatory at tier=medium+.
  Often invoked indirectly by /ship — exit cleanly so /ship can chain.
disable-model-invocation: true
allowed-tools: Read Write Edit Bash Grep Glob
---

You produce a feature spec — a durable, falsifiable artifact at
`.workflow/<slug>/spec.md` — before any implementation. No code. No tests yet.

Today's date: !`date +%Y-%m-%d`

## When to skip / redirect

- tier=trivial (typo, rename, one-liner) → exit, no spec needed.
- tier=small bug fix → write a 5-10 line spec stub (repro + expected) instead
  of full ceremony.
- Spec already exists at `.workflow/<slug>/spec.md` → ask: amend in-place, bump version,
  or archive + replace?
- Architecture decision rather than a feature → exit, suggest `/sota`.

## Pre-read: decision log

Before asking inputs, check for `.workflow/<slug>/decisions.md` (produced by an
optional `grill-me` interview). If present and `status: resolved` or
`status: partial`, read it. Treat each logged decision (Q / Answer /
Rationale) as **authoritative** — do NOT re-ask those trade-offs. The user
already settled them; re-questioning erodes trust.

Partial logs (`status: partial`): logged decisions are settled, unlogged
branches are open for /spec to question normally.

Surface in your turn: *"I read `.workflow/<slug>/decisions.md` — N decisions
settled (D1, D2, ...). I won't re-ask them. Still missing: <inputs>"*.

## Inputs (ask if missing)

Required:

1. **Feature title** + **slug** (kebab-case, ≤30 chars). If slug not given,
   propose one and confirm.
2. **User story**: "As <role>, I want <capability>, so that <outcome>".
3. **Where it lives** (which module / route / component). Read+grep the
   codebase first to propose; ask only if ambiguous.
4. **Hard constraints** (perf budget, browser support, integration points,
   compliance).
5. **Tier** (small / medium / large). Default medium if unstated.
6. **Stakeholder-supplied edge cases** — mandatory at tier=medium+. Ask
   explicitly: *"What edges have bitten you (or production users) on
   similar features? At least 3."*

Optional but useful:

7. Related ticket / PR / design doc URLs.
8. Existing similar features in the codebase to mirror patterns.
9. Design reference (Figma, mockup) for UI features.

## Codebase-aware drafting

Before drafting, search the codebase:

```bash
# Existing similar features to mirror
grep -rl "<related-keyword>" --include="*.{ts,tsx,py,go}" -l

# Existing specs to follow same template variant
ls .workflow/*/spec.md 2>/dev/null

# Conventions: read CLAUDE.md (project-level if exists)
test -f CLAUDE.md && head -50 CLAUDE.md
```

If a similar spec exists, use the same template variant + section ordering.

## Spec template

Write to `.workflow/<slug>/spec.md` (create `.workflow/<slug>/` if missing):

```markdown
---
slug: <slug>
date: <YYYY-MM-DD>
tier: small | medium | large
status: draft
related: [<ticket-url>, <pr-url>]
---

# <Feature Title>

## User story
As ..., I want ..., so that ...

## Acceptance criteria
AC1. <falsifiable statement; numeric where possible>
AC2. ...
ACn. ...

## Happy path
1. User does X
2. System responds with Y
3. ...

## Edge cases

- **EDGE1** [stakeholder]: <name> — fixture: <concrete value>; expected:
  <concrete behavior>
- **EDGE2** [stakeholder]: ...
- **EDGE3** [stakeholder]: ...
- **EDGE4** [llm-suggested]: ... (supplementary)
- ...

(Mandatory: ≥3 edges marked [stakeholder] from real domain knowledge.
LLM-suggested edges are supplementary.)

## Non-goals
- Explicitly NOT covered: ...

## Non-functional requirements
- Perf: <p50/p95 number, or "not measured, target unknown">
- Observability: <metrics / logs / traces required>
- Security: <auth requirements, attack surface, threat model>
- Accessibility: <WCAG level, screen-reader behavior>

## Open questions
- [ ] ... (force explicit "no unknowns" — empty section requires confirmation)

## Design reference
- Figma / mockup: <url or "none — design intent in Happy path section">

## Provenance
- Author: <user>
- Source: <ticket / conversation / design doc>
```

## Iteration

1. Draft per template. Show to user.
2. **Push back hard on aspirational language**:
   - "fast" → demand p50/p95 number, OR explicit "not measured, target
     unknown" (latter is fine if honest).
   - "user-friendly" → demand click count, time, or measurable proxy.
   - "robust" → demand failure scenarios it must survive.
   - "scalable" → demand concurrency or throughput target.
   - "secure" → demand threat model + auth requirements.
3. Verify each AC is falsifiable: *"could a test prove this false?"*. If no
   → rewrite (an AC paraphrasing the user story — "the user gets X" — isn't).
4. Verify edge fixtures are concrete (`""`, `"𝕏𝕐𝕑"`, `2026-02-29`, `-0.0`),
   not abstract ("invalid input"), not the happy path with one variable
   changed. The bitey 5 beat a "comprehensive" 20.
5. Cap spec at ~120 lines. Overflow signals unfocused feature — propose
   splitting into multiple specs.

## Approval gate

Once user is satisfied, ask explicitly:

> Approve as-is? (frontmatter flips to `status: approved`, file is committed.)

On confirmation:

1. Update frontmatter: `status: approved`.
2. Write file.
3. Suggest commit:

   ```
   git add .workflow/<slug>/spec.md
   git commit -m "docs(spec): <feature title>

   Spec at .workflow/<slug>/spec.md.

   Refs: <ticket-url>"
   ```

4. If invoked by `/ship`: return control. `/ship` will auto-continue to
   test gen.
5. If invoked standalone: tell user *"spec approved. Next: `/verify <slug>`
   to generate failing tests, or `/ship <slug>` to run the full lifecycle."*

## Hard constraints

- **No code in this phase.** Code = next round.
- **Acceptance criteria MUST be falsifiable.** Aspirational ACs are rejected.
- **Edge fixtures MUST be concrete** (real values, real strings with their
  exact bytes). No "invalid input" placeholder.
- **Stakeholder edges mandatory at tier=medium+.** If user can't supply 3,
  surface the gap explicitly: *"no production edges supplied = unknown
  unknowns; proceeding at risk. Add to spec under 'Open questions' for
  later resolution."* Don't gate-block, but record the gap.
- **Open-questions section mandatory** (may be empty after resolution, but
  presence is required).
- **Never write code or tests in this skill.** Exit cleanly so `/verify` (or
  `/ship`) can take over.
- **Frontmatter is parseable**: `/ship` greps `^status:` to detect phase.
  Don't break the `key: value` format.

## Anti-patterns (do NOT do)

- Padding the spec with framing prose ("This feature is important
  because...") or rephrasing the user story in other sections — purpose
  belongs in the user story, once.
- Inventing constraints ("must support 1M req/s") not supplied by user.

(AC/edge quality rules live in §Iteration, stated once there.)
