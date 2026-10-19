# farith

A Claude Code skill enforcing **verification arithmetic**: a test must derive
its expected result independently of the implementation under test, not
retrace what the implementation already did. The same principle shapes how
implementation code should be written in the first place — as pure functions
with side effects pushed to the edges — so that independent verification is
possible later.

## Install

Copy (or symlink) the skill into your Claude Code skills directory:

```sh
git clone https://github.com/<you>/farith.git
ln -s "$(pwd)/farith/skills/farith" ~/.claude/skills/farith
```

Or just copy `skills/farith/SKILL.md` directly into `~/.claude/skills/farith/`.

## What it does

- Flags tests that just retrace the implementation ("verification theater")
  and pushes for tests that compute the expected value a different way.
- Pushes implementation logic toward pure functions, with I/O and side
  effects separated out, so later verification is actually possible.
- Applies to test-writing, test review, test strategy discussions, and to
  implementation code itself — not just to tests after the fact.

[`skills/farith/SKILL.md`](skills/farith/SKILL.md) is the always-loaded
summary — the core principle, the test-comment rule, the YAGNI/anti-over-engineering
interaction rule, and the review checklist. The full convention is split into
three reference files under `skills/farith/references/`, loaded only when the
situation calls for them:

- [`01-pure-functions-and-commands.md`](skills/farith/references/01-pure-functions-and-commands.md) —
  pure functions, separating decisions from effects, Command Verification,
  why DI/mocking mostly disappears.
- [`02-integration-and-legacy.md`](skills/farith/references/02-integration-and-legacy.md) —
  minimal integration testing, and the strangler pattern for adopting this in
  existing codebases.
- [`03-design-doc-and-mapping.md`](skills/farith/references/03-design-doc-and-mapping.md) —
  test-first, test code as a requirements mapping, handling ambiguity, the
  design-doc review layer.

The convention is language- and paradigm-agnostic: nothing in it assumes a
particular type system, OOP vs. functional style, or language-specific
mutability semantics.

## License

MIT
