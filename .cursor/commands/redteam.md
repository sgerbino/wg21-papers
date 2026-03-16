---
description: Red-team a WG21 paper - report the single worst vulnerability
---

Red-team `$ARGUMENTS` (or the currently open file if none specified).

## Setup

1. Read the target paper in full.
2. Read `CLAUDE.md` in the repo root for style and tone rules.
3. If the paper references source code via GitHub links, read the linked files to verify claims.
4. If the paper quotes another WG21 paper by section number, verify the quote and section number are accurate when the source is accessible.

## Severity Taxonomy

- **CRITICAL** - Factually wrong. A spec reference that does not exist. A quote that is not exact. A code snippet that does not match the linked repo source. A causal logic break where step N does not follow from step N-1. A broken link. A citation number in the body that does not match the corresponding References entry. A cross-reference to a companion paper that states something the companion does not actually say.
- **STRUCTURAL** - A competent hostile reader could use this to invalidate an entire section or the paper's core claim. An unstated assumption the argument depends on. An unaddressed counterargument that threatens the main thesis. A logical gap in the causal chain. An internal contradiction between two sections of the same paper.
- **TONAL** - A sentence hotter than the paper's own disclosure promises. An implied accusation of bad faith. Words like "clearly" or "obviously" doing argumentative work the evidence has not earned. The attack instinct leaking through measured prose. A violation of CLAUDE.md tone rules 39 or 40.
- **COSMETIC** - Awkward phrasing, slightly clunky transition, word choice that is fine but not perfect, minor style nit. **Below threshold. Never report cosmetic findings.**

## Procedure

1. Analyze the paper against all four severity levels.
2. Report **ONLY the single highest-severity finding**. Format:

```
**[SEVERITY]** Section X.Y / paragraph N / "quoted text..."

Problem: <one sentence>
Fix: <one sentence>
```

3. If multiple findings share the highest severity, report the one that is most damaging to the paper's credibility.
4. If no CRITICAL, STRUCTURAL, or TONAL issues remain:

```
**CLEAR** - paper is below threshold. Ship it.
```

## Rules

- One finding per invocation. No lists. No "also consider." Just the single worst thing.
- Do not suggest rewrites beyond the one-sentence fix direction.
- Do not report cosmetic issues. Ever. They are below threshold.
- Do not praise the paper. Do not summarize the paper. Just the finding or CLEAR.
- Verify factual claims against linked sources when feasible rather than assuming correctness.
