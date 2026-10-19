# css — conformance

`css` is the base for building CSS (decision 338): a reader of a stylesheet into a typed `Sheet`, its renderer, and
`scope(id, css)`, which appends `[data-s="<id>"]` to each compound of each selector. It is a library of its own (326), pure
botopink with no host cell and no sidecar, importing std and nothing else, so it compiles for erlang and commonJS and runs
at build on the wasm comptime runtime (594). It knows no framework and no file kind: the scope id is a string its caller
hands it, and the layers above it (`rakun-style`, `rakun-stylesheet`, `rakun-bpp`) own everything that is not CSS (338).
This reference describes the library as its `src/` declares it today. Every example is a cell (decisions 461, 465): the
examples that exist are the decision cells of [`modules/decisions-taken/`](modules/decisions-taken/), each a `docs.md`
beside its program. A table, a grammar or a wire format is not a program and is fenced as `text`.

| Section | What |
|---|---|
| *Package layout* | the three modules, the imports, the manifest |
| *sheet* | `Sheet`, `Node`, `parse`, `render`, `scope`, `scopeSheet` |
| *selector* | scoping one selector list: `scopeSelector`, `validScopeId`, `ScopedText` |
| *chars* | the character walk the reader and the scoper share |
| *Refusals* | what a sheet or an id is refused for, and how it is named |
| *Departures from the spelling in force* | what `src/` still writes in a retired form |

| Directory | What it holds |
|---|---|
| `modules/decisions-taken/` | The ledger's index: one directory `<slug>-<N>/` per decision of [`../decisions-taken.md`](../decisions-taken.md) that fixes program behaviour, each a `docs.md` walking through one example program of the decision (front 166). |

## Package layout

`src/root.bp` lists the modules `chars`, `selector` and `sheet`, listed so a module precedes the siblings that import it.
A consumer writes `import {sheet.scope, sheet.parse} from "css";` — a name comes from a module of the package by its path in
braces, and from a package with `from`; a bare name with neither is `shorthand-import`
([own_module_import_without_from-206](modules/decisions-taken/own_module_import_without_from-206/draft/main.bp),
[shorthand_import_refused-337](modules/decisions-taken/shorthand_import_refused-337/draft/main.bp)).
A program that imports `css` declares it in `dependencies` (326, 242). The cells' workspace is
[`modules/botopink.json`](modules/botopink.json): `css` as a path dependency, the library's two targets.

A sheet is read as its codepoints, the same unit on every target. Every refusal names a byte offset of the UTF-8
text. Because `css` has no host cell, a `comptime` call of `scope` answers the same bytes at build as at run time on every
target ([libraries_run_at_build_on_wasm-594](modules/decisions-taken/libraries_run_at_build_on_wasm-594/draft/main.bp)).
A `?T` the caller holds, a sheet or a hook to scope it, is read with `??`, `?.` and `!`
([optional_typescript_operators-330](modules/decisions-taken/optional_typescript_operators-330/draft/main.bp)).

## sheet

`parse(css)` answers `Result<Sheet, string>`. A `Sheet(nodes)` is an array of `Node`, and every byte of the text lands in
exactly one node, so `render(parse(css))` is `css` byte for byte. A `Node` is one piece of a sheet, `at` the byte offset of
its first byte:

```text
Space(text)                         whitespace, verbatim
Comment(text)                       /* … */ verbatim
Declaration(text)                   `prop: value;` verbatim, the `;` included when written
AtStatement(name, text)             a block-less at-rule (@import …;, @layer a, b;), name without the @
Rule(selector, at, body)            selector is the raw text up to `{`, trailing space included
AtRule(name, prelude, at, body)     @<name><prelude>{<body>}, prelude raw
Keyframe(selector, at, body)        a step of @keyframes (from, 50%): never scoped
```

The reader tells what a block may hold by where it is: the sheet itself, the body of a rule-holding at-rule (`@media`,
`@supports`, `@layer`, `@container` and any at-rule not named below), a rule's body (declarations and nested rules, CSS
nesting), the body of a declaration-holding at-rule (`@font-face`, `@page`) and `@keyframes`' steps.

- `render(sheet)` answers the text of a sheet, and `renderNodes(nodes)` the text of a node array.
- `scope(id, css)` answers `Result<string, string>`: `parse`, then `scopeSheet`, then `render`. It rewrites selectors and
  nothing else: declarations, values, comments and at-rule preludes pass through byte for byte. Keyframe steps are not
  selectors and are left alone.
- `scopeSheet(id, sheet)` answers `Result<Sheet, string>`: the sheet with every element selector scoped to `id`.

The scope is written as `[data-s="<id>"]` appended to each compound — before a pseudo-element, which must stay last —,
and the compound's shape decides the rest (see *selector*). The id is written into an attribute value, so it is `[a-z0-9-]+`.
The decision cells show `scope` at work: [own_module_import_without_from-206](modules/decisions-taken/own_module_import_without_from-206/draft/main.bp)
scopes a card's sheet and prints the served text, each compound carrying `[data-s="card"]`.

## selector

`scopeSelector(id, selector, at)` answers a `ScopedText(text, problem)`: the selector list `selector` (a rule's raw prelude,
its trailing space included) scoped to `id`, `at` its byte offset in the sheet so that a refusal names the sheet's offset.
`problem` is `""` when it succeeded. `validScopeId(id)` is true for `[a-z0-9-]+`.

A selector list is compounds and separators: whitespace, the combinators `>`, `+`, `~` and the list's `,`. Each compound
gets `[data-s="<id>"]` before its first pseudo-element (`::before`, and the legacy one-colon `:before`, `:after`,
`:first-line`, `:first-letter`), otherwise at its end. Then, by the compound's shape:

```text
:global(sel)       the whole compound: written as `sel`, unscoped
:is(…) / :where(…) each argument is a selector list scoped inside; the compound gets no attribute of its own
                   (the element is the argument's; :where's zero specificity is kept)
&                  CSS nesting: the compound is left alone, its parent is scoped
comments only      a separator
```

## chars

`chars` is the character walk the reader and the scoper share. A skip function answers the index just past the construct it
read, or `-1` when the text ends inside it.

- `charAt(cs, i)` is the character at `i`, `""` outside the text (a negative index is outside); `textOf(cs, begin, end)` is the
  text of `cs[begin..end)`.
- `byteOffset(cs, i)` counts the UTF-8 bytes before the character at `i`; `problemAt(what, byte)` writes the one shape of
  every refusal, `css: <what> at byte <n>`.
- `isNameChar(c)` and `isSpace(c)` classify a character.
- `opensComment(cs, i)` tests for `/*` at `i`; `skipComment`, `skipString` and `skipEscape` skip a comment, a string and an
  escape. A string ends at its closing quote; an unescaped line break inside it ends it without a close (CSS's bad-string),
  and an escaped one continues it. An escape is one character, or up to six hex digits and the one whitespace that ends them.

## Refusals

A sheet that does not parse is an `Error` naming the byte offset, never a truncated sheet: an unclosed `{`, `(` or `[`, an
unexpected `}`, `)` or `]`, an unterminated string or comment, a dangling `\`, a selector with no block, a declaration outside
a rule, an empty selector, a rule inside a declaration-holding at-rule such as `@font-face`. `scope` adds three more: a
selector that already holds a `[data-s…]` attribute (a sheet is scoped once), a `:global` that is not a whole compound (`.a:global(.b)`),
with no argument or inside another pseudo-class's argument, and an id outside `[a-z0-9-]+`.

## Departures from the spelling in force

`src/` still writes these forms, and its codemod is pending; this reference describes the API in the spelling in force.

- `src/root.bp` declares its modules with `pub mod`, and `botopink.json` carries `"entry"` and `"files"`; every `.bp` under
  `src/` is a module now (592 (1)), a library has no entry (733) and `files` is the tree (703).
