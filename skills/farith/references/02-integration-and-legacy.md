# Integration tests and applying this to legacy codebases

This file covers: how much integration testing is actually needed once pure
functions and Command Verification are in place, and how to adopt this
convention incrementally in an existing codebase (strangler pattern).

## 5. Integration tests should be minimal, not exhaustive

Composing already-verified pure functions does **not** need to be re-tested for
its internal computation — `compose(A, B)` where A and B are each verified by
verification arithmetic will behave exactly as their composition predicts.
Writing large integration suites that re-walk this is redundant.

What legitimately still needs a (small number of) integration test(s):
1. **Wiring mistakes** — e.g. `compose(B, A)` instead of `compose(A, B)`, argument
   order swapped. Prefer fixing this with types (nominal/newtype-style types so
   `UserID` and `OrderID` can't be silently swapped, in whatever form the language
   supports — a wrapper type, a branded/opaque type, a value class) over testing
   it. Only test what the type system in use genuinely cannot express. In a
   language with no static type system at all, this category necessarily grows —
   see the note at the end of this section.
2. **The Core/Shell boundary itself** — the one seam where pure Command-producing
   code meets the real execution layer. This is the only place true
   non-determinism enters, so a minimal smoke/contract check here is legitimate.
3. **Composition order as a spec** — if "A then B, in that order" is itself a
   requirement (not just an implementation detail), one or two tests of that fact
   are fine. This is a spec test, not exhaustive case coverage.

**The stronger the type system, the smaller this category gets.** Prefer
expanding what the type system guarantees over expanding integration test
coverage. In weakly/dynamically typed contexts, more of this category is
unavoidable — but keep pushing what can be pushed into types, or into
equivalent static checks the language/ecosystem offers (schema validation,
linters, contract checkers).

## 12. Applying this to existing (legacy) projects — strangler pattern

§1–11 (see the other reference files) describe the target state. Most existing
codebases are nowhere near it — logic and effects are tangled, tests (if any)
retrace implementation, there's no design-doc layer. Do not attempt a big-bang
rewrite to reach the target state; apply the **strangler fig pattern**: grow the
disciplined core incrementally around/through the legacy code, rather than
pausing feature work to rearchitect everything up front.

Practical approach:

- **Don't retrofit tests onto untouched legacy code just to "add coverage."** A
  characterization test written against tangled legacy logic will itself violate
  §1 (retraces implementation instead of deriving from spec) — writing it isn't
  progress toward this convention, it's just formalizing the current behavior,
  warts included. Only worth doing as a deliberate safety net immediately before
  extracting/refactoring that specific piece — not as a blanket initiative.
- **New code follows the full convention**, regardless of how the surrounding
  legacy code is structured. Don't let "the rest of the codebase doesn't do this"
  become a reason to skip it for new work.
- **When touching existing code (bug fix, feature addition, refactor), extract
  the piece being touched into a pure function + Command boundary as part of that
  change**, rather than patching in place. This is the strangler mechanism: over
  many small touches, the pure/Command-verified core grows and the untested
  tangled shell shrinks, without ever requiring a dedicated big-bang migration.
  (See the interaction note in SKILL.md about when this extraction is and isn't
  warranted — don't extract a Command boundary for a trivial, branch-free effect.)
- **Design docs can be introduced piecemeal**, scoped to the module/feature being
  touched — not retroactively written for the entire existing system.
- **Expect a long transition period with two kinds of tests coexisting**: legacy
  characterization-style tests (implementation-coupled, lower value, candidates
  for eventual deletion once the code they cover is strangled out) and new
  Command-Verification-style tests (design-doc-mapped, per this convention).
  Don't treat their coexistence as a failure — it's the expected intermediate
  state. Flag legacy tests in comments as such (e.g. `LEGACY: characterization
  test, not spec-derived`) so the distinction stays visible rather than blending
  the two styles together silently.
- **Don't force DI/mocks out of legacy code reflexively.** DI/mocking is usually
  unnecessary once effects are separated — but ripping out DI from legacy code
  that hasn't been separated yet just to "follow the convention" removes the only
  seam that code currently has, without providing the Command-based alternative
  first. Do the separation, then the DI naturally becomes unnecessary as a
  consequence — not the other way around.

## Scope note: non-functional requirements are out of scope

This convention (verification arithmetic, Command Verification, the
requirements→design→test mapping) applies to **behavioral correctness** — does
the function/system do the right thing. It does not apply to **non-functional
requirements** (response time, throughput, memory usage, and similar performance
characteristics). These are the responsibility of benchmarking, not of
verification-arithmetic-style tests:

- A performance target ("respond within 100ms") is not something a pure
  function's input/output can verify — it's a property of execution, measured
  empirically, not derived from spec by calculation.
- Don't force performance requirements into the requirements→design→test mapping
  chain. Keep them in benchmarks, load tests, or profiling instead, tracked
  separately from the correctness test suite this convention governs.
- If a "test" in this suite is actually asserting on timing/throughput, it
  doesn't belong here — move it to the appropriate benchmarking tooling.
