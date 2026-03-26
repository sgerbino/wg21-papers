# WG21 Proposal Evaluation Rubric

## Paper Metadata

| Field     | Value               |
| --------- | ------------------- |
| Paper     | P3984R0             |
| Title     | A type-safety profile |
| Type      | Language            |
| Evaluator | WG21_EVAL rubric/AI |
| Date      | 2026-03-25          |

---

## Checklist

### 1. Evidence of Need

**Applies to:** Both | **Ref:** P4133 3.1 / 4.1 | **Failure modes:** A2

**Score:** 1 / 2
**Notes:**
Section 1 provides a historical and philosophical motivation: C
compatibility concerns and the zero-overhead principle prevented
complete type and resource safety. The paper argues that coding
guidelines are "labor intensive, error prone, and basically not
feasible at scale." The fundamental assumptions (guaranteed properties,
opt-in, validated by software) are stated clearly. However, the
evidence is entirely argumentative. No real-world code demonstrates
current pain or proposed relief. No structured cost/benefit analysis
quantifies the cost of adding profiles to the language (compiler
complexity, teaching burden, interaction surface) against the
demonstrated benefit. The claim that "important C++ users" need this
is not substantiated with data, survey results, or named stakeholders.

---

### 2. Implementation

**Applies to:** Both | **Ref:** P4133 3.2 / 4.2 | **Failure modes:** A1, C2

**Score:** 1 / 2
**Notes:**
The abstract states that "what is described has been prototyped" and
cites three sources. Section 5 reports that "an experimental
implementation is being conducted in a major C++ compiler" and that
"an implementation guide is being written." The C++ Core Guidelines
checks in clang-tidy partially overlap with the profile's rules. But
the paper does not present implementation results: no compiler output,
no diagnostics, no benchmarks of analysis time, no edge-case coverage,
no interaction analysis with existing language features (modules,
concepts, coroutines, constexpr). The reader is told prototypes exist
but shown nothing from them.

---

### 3. Teaching Story (Language Only)

**Applies to:** Language | **Ref:** P4133 4.3

**Score:** 0 / 2
**Notes:**
The paper does not address how developers encounter profiles for the
first time, what diagnostic messages look like when a profile rejects
code, or which existing mental models profiles reinforce or break. The
conservative "assume all branches execute" rule (Section 2.6) will
reject code that experienced developers consider obviously safe - the
paper does not discuss how that rejection is communicated or what the
developer's recovery path looks like. The `[[not_invalidating]]`
attribute introduces a new concept (compiler-verified annotation that
speeds analysis) with no discussion of how it would be taught or
discovered.

---

### 4. Steel Man Against Standardization / Adoption

**Applies to:** Both | **Ref:** P4133 3.3 / 4.4 | **Failure modes:** D2

**Score:** 1 / 2
**Notes:**
Section 1.1 "Alternatives to profiles" addresses why a framework is
needed over ad-hoc facilities (libraries, static analysis, compiler
options, pragmas). The section argues that variety is "a recipe for
chaos" and that a standard way of specifying guarantees is necessary.
This is a reasonable argument for standardization in general, but it
does not confront the strongest arguments against this specific
profile design:

- The conservative analysis rejects valid code; the false positive
  rate is unknown and unmeasured.
- The "trusted code" escape hatch means the guarantee has a hole
  wherever abstraction implementations live - and that is where the
  bugs are.
- Existing static analyzers (clang-tidy, MSVC analysis, PVS-Studio)
  already cover a large fraction of these checks without language
  changes.
- The concurrency exclusion (Section 4) means the guarantee does not
  hold in multithreaded programs - the majority of production C++.

None of these counter-arguments are stated at their strongest and
defeated with evidence.

---

### 5. Steel Man of Competing Designs

**Applies to:** Both | **Ref:** P4133 3.4 / 4.5 | **Failure modes:** D1, D4

**Score:** 1 / 2
**Notes:**
Section 1.1 enumerates categories of alternatives (libraries, static
analysis, compiler options, pragmas, coding guidelines) and dismisses
them collectively as incomplete, platform-specific, and incompatible.
The paper does not present any competing design at its strongest.
Missing alternatives that address the same problem:

- **Borrow-checker / ownership model** (Rust-style, or cppfront/Circle
  approaches): provides stronger guarantees with a different tradeoff.
- **Contract-based approach**: contracts (P2900) could express some of
  the same preconditions with different ergonomics.
- **Extended static analyzer integration**: standardize the analysis
  rules without a language-level profile framework.
- **Do nothing**: rely on existing guidelines + tooling, which have
  improved steadily.

None of these is presented at its strongest, and the paper does not
justify the profile approach against each.

---

### 6. Post-Adoption Metrics

**Applies to:** Both | **Ref:** P4133 3.5 | **Failure modes:** A4

**Score:** 0 / 2
**Notes:**
No success criteria are defined. The paper does not state what would
constitute successful adoption of the type-safety profile. Possible
metrics - percentage of a codebase verifiable under the profile,
reduction in type-safety-related CVEs, adoption rate across major
implementations, developer satisfaction surveys, false positive rate
in practice - are not discussed.

---

### 7. Retrospective Trigger

**Applies to:** Both | **Ref:** P4133 3.6 | **Failure modes:** A4

**Score:** 0 / 2
**Notes:**
No retrospective plan. The paper does not define when the committee
should revisit whether the profile's design decisions were correct -
for example, whether the conservative analysis is too aggressive,
whether `[[not_invalidating]]` is sufficient, or whether the
concurrency exclusion is acceptable long-term.

---

### 8. Decision Record

**Applies to:** Both | **Ref:** P4133 3.7 | **Failure modes:** B1, B2, E1

**Score:** 1 / 2
**Notes:**
Several design decisions are documented with rationale:

- Section 2.3: whether to trust constructors/destructors (two options
  presented with tradeoffs).
- Section 2.6: the "assume all branches execute" portability rule.
- Section 2.6: the `[[not_invalidating]]` attribute as a
  compiler-verifiable optimization.
- Section 1: the subset-of-superset strategy.

Missing: alternatives considered and rejected for each decision
(e.g., why not a flow-sensitive analysis for the portability rule?),
dissenting views or objections raised during design review, and
conditions under which decisions should be revisited. The constructor
trust decision (Section 2.3) is presented as an open question rather
than a resolved design choice.

---

### 9. Domain Coverage Attestation

**Applies to:** Both | **Ref:** P4133 3.8 | **Failure modes:** A3, D3, G1, G2

**Score:** 0 / 2
**Notes:**
No mention of which domains were consulted. The type-safety profile
would affect embedded systems (where casts and pointer arithmetic are
common), game engines (where performance-sensitive code uses unions
and raw pointers), financial systems (where correctness guarantees
are valued but legacy code is extensive), scientific computing (where
pointer arithmetic on arrays is pervasive), and systems programming
(where the "trusted code" boundary is the majority of the codebase).
The paper does not state which of these domains informed the design
or acknowledge coverage gaps.

---

### 10. Prediction Registry

**Applies to:** Both | **Ref:** P4133 3.9 | **Failure modes:** A4

**Score:** 0 / 2
**Notes:**
The paper makes several implicit predictions that are not recorded
with falsifiable criteria:

- The type-safety profile can guarantee safety "without imposing
  inessential overheads."
- The conservative analysis ("assume all branches execute") will be
  practical for real codebases.
- `[[not_invalidating]]` will be sufficient to reduce false positives
  to acceptable levels.
- The profile framework will converge across implementations without
  "inessential incompatibilities."

None of these are recorded with criteria for testing them or a revisit
date.

---

## Score Summary

| Metric     | Value      |
| ---------- | ---------- |
| Total      | 5 / 20     |
| Percentage | 25%        |
| Rating     | Needs Work |

---

## Executive Summary

P3984R0 presents a coherent design for type-safety and resource-safety
profiles, built on a well-articulated subset-of-superset strategy. The
technical content - definitions of object and resource, the
invalidation analysis rules, the `[[not_invalidating]]` attribute, and
the portability rule for conservative analysis - is the paper's
strength. Stroustrup's design intuition is evident in the clean
separation between supersetting (hardened libraries) and subsetting
(static analysis), and in the pragmatic choice to trust constructors
and destructors rather than attempt full verification.

However, the paper operates almost entirely in the space of design
rationale and architectural description. The P4133 evidence
requirements - real-world code, implementation results, teaching
story, steel man arguments, success metrics, retrospective commitment,
domain attestation, and falsifiable predictions - are largely absent.
The paper acknowledges that "we need implementer input" (Section 2.6)
and that "a few simple profiles" must be tried out (Section 5), but
does not supply the empirical evidence that would support advancing the
design.

**Overall readiness:** The design vision is clear but the evidence
base is thin. The paper reads as an architectural sketch - valuable
for setting direction, but not yet accompanied by the implementation
results and structured evidence the committee needs to evaluate the
guarantee's practical reach.

**Strongest areas** (items scoring 2): None.

**Critical gaps** (items scoring 0): Teaching story (item 3),
post-adoption metrics (item 6), retrospective trigger (item 7),
domain coverage attestation (item 9), prediction registry (item 10).

---

## Improvement Tips

### 1. Evidence of Need (P4133 4.1)

**Scored 1.** Show real-world code. Take a representative function
from a production codebase, show the type-safety bug it contains
today, and show how the profile would have caught it. Quantify the
frequency of such bugs - CVE databases, bug trackers, and internal
defect data are available. Replace "important C++ users" with named
organizations or published statements of need.

### 2. Implementation (P4133 4.2)

**Scored 1.** Present results from the prototypes. Show compiler
output: a function that passes the profile, a function that fails, and
the diagnostic message produced. Report analysis time overhead on a
real translation unit. Document which edge cases were tested and which
remain open. If the experimental compiler implementation is not yet
public, summarize what it has demonstrated.

### 3. Teaching Story (P4133 4.3)

**Scored 0.** Add a section addressing three questions. First: how
does a developer encounter profiles for the first time - is it a
compiler flag, a source annotation, or a project configuration?
Second: when the profile rejects valid code (e.g., the "assume all
branches execute" rule rejects a function with a conditional pointer
return), what does the error message say and what is the developer's
recovery path? Third: which existing mental models (RAII, const
correctness, move semantics) do profiles reinforce, and which do they
break (e.g., "my non-const function doesn't invalidate, but the
profile assumes it does")?

### 4. Steel Man Against Adoption (P4133 4.4)

**Scored 1.** Confront the strongest objections directly. The
concurrency exclusion means the guarantee does not hold in
multithreaded programs - address why a partial guarantee is still
valuable. The false positive rate is unknown - commit to measuring it
or explain why measurement is premature. The "trusted code" escape
hatch means the guarantee has a hole at the abstraction boundary -
argue why this boundary is manageable in practice.

### 5. Steel Man of Competing Designs (P4133 4.5)

**Scored 1.** Present each competing approach at its strongest.
Rust's borrow checker provides stronger guarantees with compile-time
enforcement and no escape hatch in safe code - why is the profile
approach better for C++? Contract-based approaches (P2900) could
express some preconditions with different ergonomics - why is a
profile framework preferable? Standardizing analysis rules without a
framework avoids the language-change cost - why is the framework
necessary? The "do nothing" option leverages steadily improving
tooling - why is that insufficient?

### 6. Post-Adoption Metrics (P4133 3.5)

**Scored 0.** Define measurable success criteria. Examples: the
type-safety profile verifies at least N% of functions in a
representative codebase, the false positive rate is below M%,
at least two major implementations ship within two releases,
type-safety-related CVE frequency declines measurably in codebases
that adopt the profile.

### 7. Retrospective Trigger (P4133 3.6)

**Scored 0.** Add a mandatory look-back commitment. Suggest two
releases or six years, whichever comes first. Define the questions:
did the conservative analysis prove practical or too aggressive? Did
`[[not_invalidating]]` reduce false positives sufficiently? Did the
concurrency exclusion remain acceptable? Did implementations converge?

### 8. Decision Record (P4133 3.7)

**Scored 1.** For each design decision (trust constructors, assume
all branches execute, `[[not_invalidating]]`), document the
alternatives that were considered and why they were rejected. Record
any dissenting views. State conditions under which each decision
should be revisited - e.g., "if the false positive rate exceeds X%,
revisit the conservative analysis rule."

### 9. Domain Coverage Attestation (P4133 3.8)

**Scored 0.** List the domains affected by the type-safety profile:
embedded, games, finance, scientific computing, systems programming,
web services. State which domains provided input during design. The
profile bans pointer arithmetic - embedded and scientific computing
domains rely on it heavily. Acknowledge this gap and describe plans to
gather input from those domains.

### 10. Prediction Registry (P4133 3.9)

**Scored 0.** Record the paper's key predictions with falsifiable
criteria. "The profile can guarantee safety without inessential
overhead" - testable by benchmarking profiled vs. unprofiled code.
"The conservative analysis will be practical" - testable by measuring
the false positive rate on real codebases. "Implementations will
converge" - testable by comparing results across compilers. Each
prediction should have a revisit date.
