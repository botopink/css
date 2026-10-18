# Decision 594 — Every library runs at build on the wasm comptime runtime (434), jhonstart first; a function only Erlang or Node can run is a located error there.

`css` is botopink alone — no host cell, no sidecar — so everything it does has a wasm form and runs at build. The program scopes one sheet twice: `comptime scoped(…)` runs `css`'s `scope` at build, on the wasm comptime runtime, whatever `--target` the program is built for, and splices the served text as a literal; `scoped(source)` runs the same call at run time on the program's target. The first line printed is the sheet with `[data-s="a1"]` on each compound, before the `::after` pseudo-element; the second, `true`, says the two answers are one string.

Were `css` to reach a function only Erlang or JavaScript can run, the `comptime` line would be refused at build, located, naming that function — not run on the target's VM.

Cells of this repository that prove it: none yet. The library's suites run `scope` on erlang and commonJS (`test/scope_test.bp`), not at build.

[draft/main.bp](draft/main.bp)
