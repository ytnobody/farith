# Test code as a mapping of requirements, and the design-doc layer

This file covers: writing tests before implementation, treating test code as a
mapping of the requirements, handling ambiguous requirements without silently
inventing answers, inserting a reviewed design-doc layer between requirements and
tests, and keeping the design doc as the source of truth.

## 8. Write the test before the implementation

Writing the test first is not just a workflow preference — it structurally
enforces verification arithmetic. If the implementation doesn't exist yet, the
expected value *cannot* be derived by running the implementation and copying its
output; it can only come from the spec. This closes off the most common failure
mode (the mock tautology, and the general "copy the implementation's output into
the expected value" trap) by construction rather than by discipline.

Practical implications:
- Write the test (with its "what this verifies" comment — see SKILL.md §7) from
  the spec/requirements before writing the function body.
- For a Command Verification test, this means deciding what Command *should* be
  produced before writing the code that produces it — which also forces the
  Command's shape (procedure name, parameters) to be designed deliberately,
  rather than discovered by whatever the implementation happened to emit.
- If writing the test first feels hard or unclear, that's often a signal the spec
  itself isn't clear yet — resolve that before writing either the test or the
  implementation.
- This doesn't mandate strict red-green-refactor ceremony; the point is the
  *ordering constraint* (spec → test → implementation), not a specific process
  ritual.

## 9. Test code is a mapping of the requirements document

Following the above to their conclusion: test code is not a byproduct of
implementation, it is a **mapping of the requirements document into an
executable form**.

```
requirements (natural language) --mapping--> test code (executable)
```

This isn't just a metaphor — it has concrete, checkable consequences:

- **Coverage should approach a bijection.** Every requirement should correspond
  to at least one test; every test should trace back to some requirement. A test
  with no traceable requirement is a test whose verification arithmetic basis is
  unclear. A requirement with no test is an unverified spec.
- **Reading all the tests should let you reconstruct the requirements**
  (approximately). The §7 "what this verifies" comments are what make this
  reconstruction possible — collectively they should read like a spec document,
  not like a changelog of implementation calls.
- **Requirements change → tests change. Implementation changes alone → tests
  should NOT change.** If a refactor that doesn't alter behavior forces test
  edits, the tests were coupled to implementation details, not to the spec — a
  violation of the core principle. This is a fast, practical smell test for
  whether a test suite is genuinely doing verification arithmetic or just
  retracing.
- **The mapping direction must stay fixed: requirements → tests →
  implementation** (this is also why §8 orders test-writing before
  implementation). If pressure ever runs the other way — implementation
  dictating what the tests should say — the mapping has inverted, and the suite
  has quietly become the "implementation-tracing" anti-pattern this entire
  convention exists to prevent.

**Corollary: a change to test code is a spec change.** Since test code is the
mapping of the requirements document, editing a test is — by definition —
editing the requirements, not "fixing the test." This gives a hard operational
rule:

- A test edit must be justifiable by pointing to a changed or newly-discovered
  requirement. "The implementation changed so I updated the test to match" is
  not a valid justification — it's the mapping running backwards.
- In review, treat test diffs the way you'd treat requirements-doc diffs: they
  need their own explanation of *what changed in the spec and why*, separate
  from and independent of the accompanying implementation diff.
- If a test needs to change and no requirement changed, stop — either the
  original test was wrong (didn't actually reflect the real requirement, i.e. it
  was never a correct mapping), or the "fix" is actually smuggling in an
  undocumented spec change. Either way, resolve it explicitly rather than
  silently editing the assertion.

## 10. Handling ambiguous or undecided requirements

§8/§9 create a tension: writing tests first, as a mapping of the requirements,
presupposes the requirements already exist — but in practice they're often
incomplete or ambiguous at the point testing starts. Left unhandled, this creates
a real risk (for humans and for AI) of quietly *inventing* requirements to make
the test-writing proceed, which inverts the mapping direction that §9 depends on
(test writer decides the spec, instead of reflecting it).

The rule: **don't silently resolve ambiguity — mark it and surface it.**

- When a requirement is unclear, underspecified, or you're filling a gap with an
  assumption, say so directly in the test's §7 comment (e.g. `ASSUMPTION: treats
  empty input as invalid, not as zero — requirement doc doesn't specify this`).
- Once a batch of tests is written, collect every such marker into an explicit
  **handoff list** — a short list of open questions/assumptions that need
  confirmation from whoever owns the requirements. Don't bury this only in
  scattered code comments; surface it as a visible summary at the end of the
  work.
- Treat an assumption used in a test as *provisional*, not settled. Once the
  requirement owner confirms or corrects it, update both the requirement doc and
  the test together (per §9, this is a real spec change, not a "test fix").
- This also gives a concrete self-check for the case where the test author is an
  AI (the §4/§8 failure mode risk applies here too): if you notice yourself
  deciding an ambiguous point in order to keep writing a test, that's the exact
  moment to stop and flag it via this mechanism instead of quietly picking an
  interpretation.

## 11. Requirements are usually Level 0 — insert a reviewed design document

§9 treats test code as a mapping of "the requirements document," but in practice
most requirements documents stop at **Level 0: business intent** ("optimize
shipping cost") — too coarse to map directly into tests without the test author
silently inventing the missing detail (exactly the risk §10 exists to catch).

Insert an explicit intermediate artifact and extend the mapping chain:

```
Level 0: requirements doc (business intent)
              |  mapping
              v
         design doc (behavior spec + interface spec — Level 1/2)
              |  review (human or a separate model)
              v
         test code (mapping of the design doc)
              |
              v
         implementation
```

- The **design doc** is where behavior specs ("given X in stock, pick the
  cheaper option") and interface specs (function/Command shapes — this is also
  where Command Verification gets its grounding, since Commands are an
  interface-level concern, not usually stated at Level 0) get made explicit.
- The design doc should go through **review — by a human, or by a separate model
  instance from the one that wrote it** — before test code is written from it.
  This catches invented/assumed detail (§10-style ambiguity) at the design
  stage, which is cheaper to fix than catching it after tests and
  implementation both exist.
- **Review depth should scale with the nature of the change, not be applied
  uniformly:**
  - **Trivial edits** (typos, wording/formatting fixes with no behavioral
    change) — light-touch review is enough; confirm the edit really is
    non-behavioral and move on.
  - **Targeted changes to specific items** (a single behavior spec or interface
    changed/added) — review that item in depth: check it against §7-comment/test
    implications, check for newly-introduced ambiguity (§10), check downstream
    items that reference it. Depth, not breadth — the rest of the doc doesn't
    need re-review.
  - **Large-scale or structural changes** (a new feature area, a significant
    rewrite of the doc, many interrelated items changing at once) — treat this
    at the same review level as a brand-new design doc: full read-through, not
    just a diff review, since interactions between changed items are easy to
    miss when reviewing hunks in isolation.
  - Use judgment on which tier a given change falls into, but default to the
    deeper tier when uncertain — an unnecessary deep review costs time, but a
    missed one reintroduces exactly the silent-ambiguity risk §10/§11 exist to
    prevent.
- Once this intermediate artifact exists, §9's "test code is a mapping of the
  requirements document" should be read more precisely as **"test code is a
  mapping of the reviewed design doc"** — with the design doc itself carrying
  the mapping obligation back to Level 0. §9's corollary (test edits are spec
  changes) and §10 (flag ambiguity, don't invent) apply at the design-doc-writing
  stage, not only at the test-writing stage.
- If no separate design doc exists yet and the pressure to write tests is high,
  treat writing the design doc first as the higher priority — writing tests
  straight from Level 0 requirements is where most silent requirement invention
  happens.

## 13. Design doc is the source of truth; test comments are a derived copy

§7 test-verification comments and §11 design docs describe overlapping content
(both state behavior specs), which risks silent drift if not given an explicit
precedence. **The design doc is the source of truth; the §7 comment in the test
is a derived copy of it, not an independent source.**

This resolves the ordering question directly: when a test needs to change (§9's
corollary — this means a spec change), **the corresponding design doc item must
be updated in the same change**, not left to go stale. A test diff without a
matching design doc diff is incomplete, the same way a code diff without a
matching test diff would be under this convention.

- When updating a test's §7 comment, check whether the design doc it's derived
  from needs the same update — treat this as one edit spanning two artifacts,
  not two independent edits.
- If the design doc and a test's §7 comment disagree, the design doc wins; the
  test (and its comment) needs to be brought in line with it, not the reverse.
- If you find yourself repeatedly changing a test comment without ever touching
  the design doc, that's a sign the design doc has stopped being the actual
  source of truth in practice — worth flagging (as a §10-style handoff item)
  rather than letting the drift continue silently.
