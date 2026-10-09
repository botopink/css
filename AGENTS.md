# css

> Repository: `botopink/css` (`git@github.com:botopink/css.git`) · in the meta checkout: `repository/css/`

The `css` library (decision 338): the base for building CSS. A reader of a stylesheet —
rules, at-rules, comments, strings, `{ }` nesting — into a typed `Sheet`, its renderer, and
`scope(id, css)`, which writes `[data-s="<id>"]` into each compound of each selector. The
lowest of the three CSS layers (`css` → `styled` → emilia): it imports `std` and nothing
else, and knows no file kind, no manifest and no framework — the scope id is a string its
caller hands it. Consumers write `import {sheet.scope, sheet.parse} from "css";`.

**A library of its own** (decision 326, the rule for a new shared package). A program that
imports `from "css"` declares it in `dependencies` (decision 242) —
`{ "css": { "git": "https://github.com/botopink/css.git", "branch": "feat" } }`; inside the
meta checkout that entry resolves by name through the `repository/` root
(`repository/css`), elsewhere through the install store.

Pure `.bp` only: no `#[@External]` cell, no `declare fn`, no `.erl` / `.mjs` sidecar, so
every target reads and scopes a sheet to the same bytes.

## Tree

```text
css/
├── AGENTS.md
├── botopink.json      ← "name": "css", "target": "erlang", "targets": ["erlang", "commonJS"], `files` = every src module
├── src/
│   ├── root.bp        ← `pub mod` per module, a module before the siblings that import it
│   ├── chars.bp       ← charAt, textOf, byteOffset, problemAt, isNameChar, isSpace, opensComment, skipComment, skipEscape, skipString
│   ├── selector.bp    ← ScopedText, validScopeId, scopeSelector   (one selector list scoped)
│   └── sheet.bp       ← Node, Sheet, parse, render, renderNodes, scopeSheet, scope
└── test/
    ├── scope_test.bp              ← the forty selector cases (literal pairs), their digest, every refusal with its byte offset
    ├── sheet_test.bp              ← node shapes, raw selector and offset, render(parse(css)) == css over a corpus
    └── scope_css_example_test.bp  ← front 119's `scope-css-example.bp` as a suite case
```

## The sheet

`parse(css) -> @Result<Sheet, string>`; `Sheet(nodes: Node[])`. Every byte of the text lands
in exactly one node, so `render(parse(css))` is `css` byte for byte.

| `Node` | Holds |
|---|---|
| `Space(text)`, `Comment(text)` | the text verbatim |
| `Declaration(text)` | `prop: value;` verbatim; the last one of a block may run to the `}` (its space included) |
| `AtStatement(name, text)` | a block-less at-rule (`@import …;`, `@layer a, b;`) verbatim; `name` without `@` |
| `Rule(selector, at, body)` | `selector` raw up to the `{`, its trailing space included; `at` its byte offset |
| `AtRule(name, prelude, at, body)` | `@<name><prelude>{<body>}`, the prelude raw |
| `Keyframe(selector, at, body)` | a step of `@keyframes` (`from`, `50%`) — never scoped |

A block's context decides what it may hold: the sheet and a top-level rule-holding at-rule
(`@media`, `@supports`, `@layer`, `@container`, and any at-rule not named below) hold rules;
a rule's body holds declarations and nested rules (CSS nesting) — an at-rule inside it holds
the same; `@font-face`, `@page` (and its margin boxes), `@property`, `@counter-style`,
`@font-palette-values`, `@font-feature-values`, `@view-transition`, `@position-try` hold
declarations; `@keyframes` (vendor prefix dropped) holds steps.

Refusals, each `css: <what> at byte <n>` — `n` the byte offset in the UTF-8 text (the walk
reads codepoints, `byteOffset` counts their UTF-8 lengths): `unclosed block` (the `{`),
`unclosed (` / `unclosed [`, `unexpected }` / `)` / `]`, `unterminated string` (the quote; an
unescaped line break ends it, CSS's bad-string), `unterminated comment`, `dangling escape`,
`a selector with no block`, `an at-rule with no ; or block`, `a declaration outside a rule`,
`an empty selector`, `a rule inside an at-rule that holds declarations`. A `;` inside
parentheses is content (`url(data:…;base64,…)`). A sheet that does not parse is never
rendered — no truncated sheet.

## Scoping

`scope(id, css) -> @Result<string, string>` = `parse`, `scopeSheet`, `render`. Only rule
selectors change; declarations, comments, keyframe steps and at-rule preludes pass through.

| Written | Served (`id` = `a1`) |
|---|---|
| `h1`, `.text`, `*`, `div.card#x` | the attribute at the compound's end: `h1[data-s="a1"]` |
| `article > p + p`, `h1, h2` | every compound of every selector of the list |
| `a:hover::after`, `p:before` | before the first pseudo-element (`::x`, or the legacy `:before` `:after` `:first-line` `:first-letter`), after pseudo-classes: `a:hover[data-s="a1"]::after` |
| `:global(sel)` as a whole compound | `sel`, unscoped; the rest of the selector scoped |
| `:is(…)`, `:where(…)` in a compound | each argument scoped inside; the compound gets no attribute of its own (`:where`'s zero specificity kept) |
| `:not(…)`, `:has(…)`, `:nth-child(…)` | the argument untouched; the compound scoped |
| a compound with `&` | left alone — its parent is scoped |
| `.a\:hover`, `.\31 0` | an escape is part of the name (a hex escape ends at its one space) |

Refused: an id outside `[a-z0-9-]+` (it is written inside an attribute value); a selector
that already holds a `[data-s…]` attribute (`scope(s, scope(s, css))` — a sheet is scoped
once); a `:global(…)` that is not a whole compound (`.a:global(.b)`) or sits in another
pseudo-class's argument; a bare `:global`.

## Testing

```sh
../botopink-lang/zig-out/bin/botopink test --target erlang
../botopink-lang/zig-out/bin/botopink test --target commonJS
../botopink-lang/zig-out/bin/botopink format --check src test
```

`zig build test-libs` discovers the package as the cells `css · erlang` and
`css · commonJS`. Every expected text in the tests is a literal; the digest of the forty
outputs is the first 32 hex digits of SHA-256 over the forty served literals joined by a
line feed, pinned as a literal and asserted on both targets.

## Language notes (measured on the compiler this library landed on)

- `botopink format` writes an `if (…) x = y else if (…) {` chain whose first branch is one
  statement on one line and breaks the condition after it mid-expression; the first branch
  is written as a block of two statements instead.
- Every refusal is computed into a `problem` string inside the loops and thrown after
  them (the routing library's note: a `throw` or `try` in a `while` that reassigns a `var`
  does not compile on erlang).

## Local gate

`scripts/git-hooks/pre-commit` is the tracked pre-commit gate, self-contained:
it sources `scripts/git-hooks/lib/runner-standalone.sh` from this repository and
reaches nothing outside it, so a standalone clone, a checkout inside the botopink
meta workspace and a worktree run the same gate. Install it once per clone:

```sh
git config core.hooksPath scripts/git-hooks
```

The repository is one plain package, so the gate's test stage runs
`botopink test --target <t>` at the root on each target `botopink.json` declares
(`erlang`, `commonJS`). Never commit with `--no-verify`; fix the red instead.
`scripts/git-hooks/pre-commit` and `scripts/git-hooks/lib/runner-standalone.sh`
are one text across every library repository: the meta repository's
`hook-integrity` workflow compares the bytes (its check 4), so a change to either
lands in all of them together. CI: `.github/workflows/test.yml` runs the same
package on linux and macos, on each declared target, with the compiler built from
`botopink/botopink-lang` `feat`. The organisation's Actions policy is the maintainer's
setting: until it admits this repository, the workflow does not run here and the local
gate and the meta checkout's `zig build test-libs` are the only verdicts.
