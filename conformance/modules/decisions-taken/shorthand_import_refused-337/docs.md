# Decision 337 — The shorthand import goes; `mod m;` binds the namespace `m` in the module that declares it.

A refused program. The package's module `theme` declares `scoped`, which scopes a sheet with `css`'s `scope` (imported by its path and `from "css"`). `main.bp` writes `import {scoped};` — a name with no module before it and no `from`, the shorthand — and the program is refused at that item, `shorthand-import`, the fix written: `theme` is the one module of the package that declares `scoped` `pub`, so the import to write is `import {theme.scoped};`. No bare name reaches a dependency either: `import {scope};` would not find `css`'s `scope`.

Cells of this repository that prove it: none yet. botopink-lang proves the rule in `modules/shorthand_import`, `modules/shorthand_import_one_candidate`, `modules/shorthand_import_several_candidates`, `modules/mod_binds_namespace`, `modules/mod_namespace_cascade`, `modules/mod_reexport_by_val`, `modules/private_module_through_namespace`, `modules/redundant_module_import` and `modules/declaration_named_like_module`.

[draft/main.bp](draft/main.bp)

[draft/theme.bp](draft/theme.bp)
