# css — conformance/

- `docs.md` — the reference: one section per module of `src/` (`sheet`, `selector`, `chars`), the refusals and the departures from the spelling in force; every example is a cell.
- `modules/decisions-taken/<slug>-<N>/` — one directory per code-bearing row of `../decisions-taken.md` (front 166, decisions 646–651): a `docs.md` (its first line generated, the rest written) and the example under `draft/` (phase 1, not a cell yet) or `src/` with `botopink.json` once promoted. Kept by the meta repository's `scripts/decision-cells.py` and its CI check 7; the subtree is front 166's (decision 651).
- `modules/botopink.json` — the cells' workspace: `css` as a path dependency, the library's two targets; `decision-cells.py compile` builds each draft against it.
