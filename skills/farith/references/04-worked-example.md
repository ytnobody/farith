# Worked example: requirements → design doc → test → implementation

This file covers: one concrete, end-to-end walkthrough of the whole chain
(§6/§7's "write a verifying comment," §8-11's requirements→design→test mapping,
§10's ambiguity handling, §1-3's pure-function/Command split) applied to a single
small decision. Code is Python for concreteness; the convention itself is
language-agnostic — see `01-pure-functions-and-commands.md` for the same ideas in
other paradigms.

## Level 0: requirements (business intent)

> When placing an order, pick the cheaper of the two shipping carriers that
> currently have the item in stock, and save the order with that carrier.

This is Level 0 — too coarse to map into tests directly (§11). What happens if
neither carrier has stock? What breaks a tie? Not stated.

## Design doc (Level 1/2 — reviewed before any test is written)

**Behavior spec:**

| stockA | stockB | priceA | priceB | → Command |
|---|---|---|---|---|
| true | true | any | any | `saveOrder` with the cheaper of A/B |
| true | true | equal | equal | `saveOrder` with A (tie-break) — **ASSUMPTION**, requirement doc silent on ties, needs confirmation |
| true | false | any | any | `saveOrder` with A |
| false | true | any | any | `saveOrder` with B |
| false | false | any | any | `escalate`, reason `no_stock` |

**Interface spec:**

```
decide_shipping(state, input) -> Command
Command = {op: "saveOrder", carrier: "A"|"B", price: number}
        | {op: "escalate", reason: string}
```

`decide_shipping` is the pure decision function (§1-2). `execute(cmd, db,
notifier)` is the thin execution layer — no branching logic of its own, so
nothing in it is worth testing beyond a smoke check (§5).

The tie-break row is flagged as an **ASSUMPTION** per §10 rather than silently
decided, and goes into the handoff list below.

## Test (written from the design doc, before the implementation exists — §8)

```python
# Verifies: when both carriers have stock, the cheaper one is chosen.
def test_picks_cheaper_in_stock_carrier():
    state = {"stockA": True, "stockB": True, "priceA": 12.0, "priceB": 9.5}
    assert decide_shipping(state, {}) == {"op": "saveOrder", "carrier": "B", "price": 9.5}

# Verifies: when only one carrier has stock, that one is chosen regardless of price.
def test_picks_only_in_stock_carrier_even_if_pricier():
    state = {"stockA": True, "stockB": False, "priceA": 12.0, "priceB": 9.5}
    assert decide_shipping(state, {}) == {"op": "saveOrder", "carrier": "A", "price": 12.0}

# Verifies: when neither carrier has stock, the decision escalates instead of guessing.
def test_escalates_when_no_stock():
    state = {"stockA": False, "stockB": False, "priceA": 12.0, "priceB": 9.5}
    assert decide_shipping(state, {}) == {"op": "escalate", "reason": "no_stock"}

# Verifies: a price tie with both in stock prefers carrier A.
# ASSUMPTION: requirement doc doesn't state a tie-break rule — see handoff list.
def test_tie_prefers_carrier_a():
    state = {"stockA": True, "stockB": True, "priceA": 10.0, "priceB": 10.0}
    assert decide_shipping(state, {}) == {"op": "saveOrder", "carrier": "A", "price": 10.0}
```

Each expected value above came from the design doc's table, not from running
any code — this is the verification arithmetic itself (SKILL.md's core
question). Each test case carries its §7 comment.

## Implementation (written after the tests, to make them pass)

```python
def decide_shipping(state, input):
    a_ok, b_ok = state["stockA"], state["stockB"]
    if not a_ok and not b_ok:
        return {"op": "escalate", "reason": "no_stock"}
    if a_ok and not b_ok:
        return {"op": "saveOrder", "carrier": "A", "price": state["priceA"]}
    if b_ok and not a_ok:
        return {"op": "saveOrder", "carrier": "B", "price": state["priceB"]}
    if state["priceB"] < state["priceA"]:
        return {"op": "saveOrder", "carrier": "B", "price": state["priceB"]}
    return {"op": "saveOrder", "carrier": "A", "price": state["priceA"]}
```

## Execution layer (Command Verification stops here — §3)

```python
def execute(cmd, db, notifier):
    if cmd["op"] == "saveOrder":
        db.save_order(carrier=cmd["carrier"], price=cmd["price"])
    elif cmd["op"] == "escalate":
        notifier.alert(cmd["reason"])
```

No test asserts what `db.save_order` or `notifier.alert` actually do — that's
the database's and notifier's own correctness, out of scope (§3). The four
tests above are the entire correctness test suite for this decision; `execute`
gets, at most, a single wiring smoke test (§5).

## Handoff list (per §10)

- **ASSUMPTION**: tie-break prefers carrier A when prices are equal. Not stated
  in the Level 0 requirement — needs confirmation from whoever owns it. Until
  confirmed, both the design doc row and `test_tie_prefers_carrier_a` are
  provisional, not settled (§10).
