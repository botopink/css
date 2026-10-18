# Decision 330 — `?T` is read with TypeScript's operators and has no methods; `result` goes; a type declared in a type is its associated type.

A site's `Page` may carry a stylesheet (`css: ?string`) and a hook that scopes it (`scoper: ?fn(string) -> string`); the home page has both, the about page neither. Every read of those optionals is an operator, never a method:

- `home.scoper?.(home.css ?? "")` calls the hook when the page holds one, and `?? "unscoped"` gives the about page its default — the first line is the home sheet scoped to `a1`, the second `unscoped`;
- `sheetOf(home)?.nodes?.[0]` reads the first node through an optional sheet: the index already answers `?Node`, and through `?.` it stays `?Node`, never `??Node` — present for the home page, absent for the about page;
- `rule?.selector` reads a field of an optional `Page.Rule` as a `?string`, defaulted by `??`;
- `home.css!` asserts the home page's sheet is there; a `null` at that point would abort, located, alike on every target.

`parse` answers a `Result`, and `sheetOf` reads it by its own methods, `parsed.isError()` and `parsed.unwrapOr(…)` — there is no `result` namespace to call. `Page.Rule` is declared in the body of `Page`, so it is `Page`'s associated type: `firstRule`, a static of `Page`, answers `?Page.Rule`, and `main` names it through the owner.

Cells of this repository that prove it: none yet. botopink-lang proves the rule in `modules/optional_operators`, `modules/optional_call_plain_answers`, `modules/optional_chain_record_fields`, `modules/bang_never_null`, `modules/nullish_never_null`, `modules/nullish_beside_logical`, `modules/result_namespace_removed` and `modules/assoc_type_declared_in_body`.

[draft/main.bp](draft/main.bp)
