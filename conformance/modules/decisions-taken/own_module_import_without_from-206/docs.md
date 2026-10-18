# Decision 206 — `import {…} from "<name>"` resolves only to a package — std, a bundled package or a declared dependency.

A consumer of `css` with a module of its own, `theme`. The program's imports name where each name comes from: `io.terminal` from std and `sheet.scope` from the dependency `css`, both with `from`; the stylesheet `card` and the colour `accent` from the package's own module `theme`, by its path in braces and with no `from` — `accent` bound under another name with `as`. The program scopes the card's sheet with `scope("card", card)` and prints the served sheet, each compound carrying `[data-s="card"]`, then the brand colour.

Writing `from "theme"` for the package's own module would be refused at the source string, with the brace form as its fix; a `from "css"` keeps meaning the dependency even where the package has a module of that name.

Cells of this repository that prove it: none yet. `test/scope_css_example_test.bp` imports `sheet.scope` by its path, the library's own module; botopink-lang proves the rule in `modules/import_own_module_with_from`, `modules/import_bundled_package_beside_own_module` and `modules/import_module_path_in_braces`.

[draft/main.bp](draft/main.bp)

[draft/theme.bp](draft/theme.bp)
