---
name: farith
description: >-
  Coding and testing convention centered on "verification arithmetic" — tests must independently verify correctness, not just retrace implementation. Use this skill whenever writing or reviewing test code, designing new functions/modules, discussing test strategy, refactoring for testability, or when the user asks about unit tests, integration tests, mocks, DI, pure functions, side-effect separation, or "how should I test this." Also apply when writing implementation code itself (not just tests) — logic should be designed as pure functions from the start so that Command Verification is possible later. Trigger even if the user doesn't use these exact terms — e.g. "this test feels pointless," "how do I test this DB call," "should I mock this," "this function has too many dependencies" are all signals to consult this skill. Language-agnostic — applies equally to statically and dynamically typed languages, OOP and functional styles.
---

# Pure Function Testing / Command Verification

A coding and testing convention built on one principle, applicable in any
language or paradigm: **a test must be verification arithmetic — it must derive
the expected result independently of the implementation under test, not retrace
what the implementation already did.**

Everything in this skill follows from that one constraint. Apply this both when
writing implementation code and when writing/reviewing tests.

## §6: The core question to ask, always

For any test: **"What is this verifying, independently of the code under test?"**

If the expected value was derived by running the implementation and copying its
output, it is not a test — it's a tautology. If you can't answer the question,
don't write the test yet; fix the design first.

## How this skill is organized

The full convention is split across three reference files, each loaded only when
the situation actually calls for it. Read the file before acting on it in depth —
the one-line summary below is enough to recognize *when* that is:

- **`references/01-pure-functions-and-commands.md`** — push logic into pure
  functions; separate "deciding what to do" from "doing it"; Command
  Verification (assert the Command a pure function would produce, never the
  result of executing it); why DI/mocking becomes mostly unnecessary once this
  is done. Read this when writing or reviewing any function that currently
  mixes a decision with a side effect, or when a test wants a mock.
- **`references/02-integration-and-legacy.md`** — how little integration
  testing is actually needed once pure functions + Commands are in place, and
  what legitimately still needs one; how to adopt this convention incrementally
  in an existing codebase via the strangler pattern instead of a rewrite. Read
  this when deciding how many integration tests a change needs, or when
  touching code in a codebase that doesn't already follow this convention.
- **`references/04-worked-example.md`** — one concrete requirements → design
  doc → test → implementation walkthrough tying §1–3, §6–8, §10–11 together in
  real code. Read this when the abstract description above isn't enough to see
  how the pieces fit, or before applying this convention to a new decision for
  the first time.
- **`references/03-design-doc-and-mapping.md`** — write the test before the
  implementation; treat test code as a mapping of the requirements (and test
  edits as spec changes); handle ambiguous requirements by flagging them
  instead of silently deciding; insert a reviewed design-doc layer between
  Level-0 requirements and tests; keep the design doc as source of truth over
  test comments. Read this when starting work from a vague requirement, or when
  a test needs to change and it's unclear why.

## §7: Every test case must have a comment stating what it verifies

Each test case (not just the file/suite) needs a comment stating, in plain
language, what it is verifying — independent of how the implementation happens to
work. This is a direct enforcement mechanism for the core question above: if you
can't write that comment without describing the implementation's internals, you
likely don't have a real verification arithmetic yet.

Guidelines for the comment:
- State the expected behavior/property, not the mechanics ("returns the cheaper
  of two shipping options when both are in stock" — not "calls compare() and
  returns the result").
- For Command Verification tests, state which Command is expected and why, not
  just "checks the mock was called."
- If you can't summarize what's being verified in one line, the test is probably
  checking too many things at once — split it.
- This comment should make the test's intent readable even with the
  implementation deleted (the same bar as the checklist item below).

```
// Verifies: given a state at the boundary (0 items), removing one more
// item does not throw and leaves state unchanged (idempotent floor).
assert(remove(state0, item) == state0)
```

## How this interacts with anti-over-engineering conventions (e.g. YAGNI)

This convention asks for structure (pure functions, Command types, a design-doc
layer) that a YAGNI-minded review — including this user's `ponytail` convention —
would otherwise flag as unneeded abstraction. The two are not actually in
conflict; they're scoped differently, and one question resolves which applies:

**Is there a decision here that could be wrong?** — i.e. a branch, a choice
between options, a retry/backoff policy, a state transition, anything where
"given this input, is the output correct" is a meaningful question with more
than one possible answer.

- **No branch, no decision, one obvious thing to do** (e.g. a single
  pass-through call to a logging library, a one-line wrapper with no
  conditional logic): leave it inline. Do not invent a pure
  decision-function-plus-Command split just to have something to test — that
  manufactures the exact "interface/abstraction nobody asked for" that
  YAGNI-style review correctly rejects. This skill never mandates extracting a
  Command boundary where there is nothing to decide.
- **A real decision exists** (what to retry, which option to pick, what the
  next state should be, what query/request to build from inputs): extracting it
  into a pure function and verifying it via Command Verification is not
  speculative complexity — it is what makes that decision logic testable at
  all. Here, this convention's structure takes priority over a "fewest lines"
  instinct, because untestable decision logic is a bigger liability than the
  few extra lines the extraction costs.

When genuinely unsure which case applies, default to asking the question above
out loud before writing either the abstraction or the inline version.

## Scope note: non-functional requirements are out of scope

This convention applies to **behavioral correctness**, not performance
(response time, throughput, memory). See
`references/02-integration-and-legacy.md` for the full scope note — the short
version: performance claims belong in benchmarks, not in this test suite.

## Quick checklist for reviewing a test

- [ ] Is the expected value derivable from the spec/requirements independently
      of the implementation? (If it was derived by running the code, it's not
      verification arithmetic.)
- [ ] If this test involves a side effect: does it check the *Command* (what
      would be requested), not the *result* of executing it?
- [ ] If this is an integration test: is it covering a wiring mistake or
      Core/Shell boundary that types genuinely can't catch — or could a
      stronger type make this test unnecessary?
- [ ] If this test uses a mock: would it still be needed if the effect were
      expressed as a Command/data value instead of an interface call?
- [ ] Could you delete the implementation, keep only the test, and still know
      what behavior is being guaranteed?
- [ ] Does this test case have a comment stating what it verifies, in terms of
      behavior/spec rather than implementation mechanics?
- [ ] Was this test written before the implementation it tests (or could its
      expected value have been derived without ever running the
      implementation)?
- [ ] Does this test trace back to a reviewed design doc (not straight to a
      Level-0 requirements doc with the detail invented along the way)?
- [ ] Would a behavior-preserving refactor (no requirement/design change)
      leave this test untouched?
- [ ] If this is legacy/existing code being touched: does the change extract a
      pure function + Command boundary as part of the touch (rather than patch
      in place) — but only where §"How this interacts with anti-over-engineering
      conventions" says a real decision exists to extract?
- [ ] If this test's assertion changed in this diff: is there an explicit,
      stated requirement change justifying it (not just "implementation
      changed, so I updated the test")?
- [ ] If this test's assertion/comment changed: did the corresponding design
      doc item change in the same edit (design doc as source of truth)?
- [ ] If this test relies on an assumption about an ambiguous requirement, is
      that flagged in the comment and included in the handoff list — rather
      than silently decided?

## A note on revising this skill itself

The design-doc review process described in the reference files applies to
product code, not to this skill. Revising this skill is a change to the rule
itself, not an application of it — no review step is required. Once revised,
simply follow the new version going forward.

## Terminology reference

- **verification arithmetic** — the guiding principle: derive expected results
  independently, don't retrace the implementation.
- **Functional Core, Imperative Shell** (Gary Bernhardt) — the overall
  architecture this convention produces.
- **Command Pattern / Free Monad / Interpreter Pattern** — the mechanism for
  turning "do a side effect" into "return data describing a side effect."
- **Command Verification** — (working term coined in this convention) testing
  that the correct Command was assembled, without executing it or checking its
  result.
- **State/Reducer pattern** — `(state, input) -> newState` as the way to keep
  stateful logic pure.
- **Test code as mapping** — test code is requirements translated into
  executable form; the mapping direction (requirements → tests →
  implementation) must not invert.
