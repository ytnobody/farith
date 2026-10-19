# Pure functions, Commands, and why DI/mocking mostly disappears

This file covers: pushing logic into pure functions, separating "deciding what to
do" from "doing it," Command Verification as the testing strategy for effects, and
why DI/mocking is usually a symptom rather than a legitimate design choice.

## 1. Push logic into pure functions

Most business logic can and should be a pure function: same input → same output,
no hidden state, no I/O.

- Bad: `obj.increment()` — mutates hidden internal state
- Good: `newState = increment(state)` — state is an explicit argument and explicit
  return value

This applies even to stateful logic and state machines: `f(state, input) ->
(newState, output)`. Once state is explicit, the entire transition table can be
written down as a spec and verified by verification arithmetic — expected
transitions can be derived from the spec, not from running the code.

**Corollary:** if you find yourself thinking "this can't be a pure function," that
feeling is usually a signal that a side effect has leaked into the logic and needs
to be pulled out, not a reason to give up on purity.

**Immutability matters too, regardless of language.** Passing state as an argument
doesn't help if the callee can still mutate it in place through aliasing. Any
language where a composite value (array, map/dict, object, struct) is passed by
reference rather than by value is at risk here — the signature looks pure but the
caller's data gets mutated anyway. Treat state as immutable data: copy-on-write,
persistent data structures, or an explicit "don't mutate what you were handed"
discipline, whichever fits the language.

## 2. Separate "deciding what to do" from "doing it"

This is the key structural move. Don't let a pure function perform a side effect —
have it return a description of the side effect instead.

```
pure function: (state, input) -> Command   // "what should be requested" — just data
execution layer: Command -> actual effect   // thin, dumb, no logic
```

A `Command` is plain data: a procedure name / operation type plus parameters. This
is the same idea as the Command pattern, Free Monad / Interpreter pattern, Elm's
`Cmd`, or treating `IO` as a value in Haskell — applicable in any language that can
represent "an operation plus its parameters" as a value (a struct, a record, a
plain object/dict, a tuple, an enum case with payload, …).

**Single Command by default; a list only when actually needed.** A pure function
returns one `Command` unless the situation genuinely requires expressing an
ordered sequence of effects (e.g. "write to DB, then send an email"). In that
case, return `Command[]` (an ordered list) rather than inventing a special
multi-effect type. Don't default to list-shaped Commands "just in case" — that
adds a layer of sequencing/ordering concerns to verify for no benefit when a
single Command would do. When a list is genuinely needed, the order itself becomes
part of what Command Verification (below) checks (assert the full sequence, not
just its members).

Once effects are represented as data:
- The decision logic (`buildCommand(state, input) -> Command`) is a pure function
  and fully verifiable by verification arithmetic.
- The execution layer has no branching logic of its own, so there's nothing in it
  worth testing.

**Retry, timeout, and error-handling logic belong in the pure layer, not the
execution layer.** These are frequently mistaken for "properties of execution" and
left untested inside the shell — but they are decisions ("given this state and
this failure, what should happen next: retry, give up, escalate to a different
Command?"), which means they fit the same `(state, input) -> output` shape as
everything else in §1. Model them the same way:

```
(state, error) -> nextAction   // nextAction: retry-with-backoff(n), give-up, emit different Command, etc.
```

This function is pure and testable by verification arithmetic exactly like any
other decision logic — "given 2 prior failures and this error type, the next
action should be retry with 4s backoff" is a spec-derivable expectation, no
different from §1's examples. What remains in the execution layer is purely
mechanical: sleep for the duration it's told, call the Command it's told, report
the raw result back. If you find yourself wanting to test "does retry happen after
3 failures," that's a signal the retry decision hasn't been extracted into a pure
function yet — extract it, then test that function, not the execution layer.

**Important distinction: whether a timeout/failure occurs is a side effect; how to
respond to one is not.** Whether an operation actually times out is inherently
non-deterministic and belongs entirely to the execution layer (nothing to verify
by verification arithmetic there — it's not testable and shouldn't be). What the
pure decision function tests is the response given that a timeout/error was
*already reported as input*: `(state, {type: "timeout"}) -> nextAction`. Never try
to test "does a timeout actually happen after N seconds" — that's testing the side
effect's own behavior. Test only "given a reported timeout, is the chosen next
action correct."

## 3. Command Verification — the testing strategy for effects

This is the piece that doesn't have an established name in the literature
(closest relative: Functional Core, Imperative Shell by Gary Bernhardt — but that
describes the architecture, not this specific testing rule). Working name for this
skill: **Command Verification**.

> Command Verification: without ever actually executing the side effect, verify
> that the pure function assembled the *correct Command* — correct
> procedure/operation, correct parameters. Do **not** verify that executing the
> Command produced the expected result.

Why the result is out of scope:
- The side-effect target's own correctness (does the database honor ACID, does
  the API behave per its docs) is the responsibility of whoever builds that
  system — testing it is redundant and it's also inherently non-deterministic, so
  it can never be a real verification arithmetic anyway.
- What *is* your responsibility is whether you're calling it correctly: right
  query, right bind parameters, right request shape, right transaction
  boundaries. That's exactly what a Command captures.
- No mocks, no stubs, no real call needed — just assert the Command value equals
  the expected Command value.

```
assert(buildCommand(state, input) == expectedCommand)
```

## 4. Why this makes DI and mocking mostly unnecessary

DI/mocking is frequently reached for reflexively ("testable = has DI") rather than
as a considered choice. Once side effects are separated as in §2–3, most of the
reasons to reach for a DI container or mocking library disappear:

- There's no hidden dependency to inject — the pure decision function takes
  everything it needs as arguments and returns data.
- There's nothing meaningful to mock — you're not calling the DB from the
  function under test, you're asserting on a plain value.

Treat heavy reliance on DI/mocking as a **symptom**: it usually means side effects
were never actually separated from logic, and DI/mocks are being used to paper
over that, not as a legitimate design choice. When reviewing code (your own output
included), ask "would this DI/mock still be needed if the effect were expressed as
a Command instead?" before adding it.

A common failure pattern to watch for: a mock's return value is configured by the
test author, and then the test asserts against that same configured value. This is
self-referential and verifies nothing — it's the DI/mock-era version of the "copy
the implementation's output into the expected value" tautology.
