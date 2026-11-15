# MapLibre GL JS 6.18 migration — exploration reports (2026-11-15)

Raw reports from exploration agents, input for the migration plan (`prompts/20261115-mgljs6-PLAN.md`, not yet written).

- `A-fork-gljs.md` — inventory of the `@eslopemap/maplibre-gl@5.21.9` fork (terrain-analysis, blend-mode, RTT changes, tests, bugs)
- `B-upstream-gljs.md` — upstream v6.18.0: layer-blend architecture, porting hazards, ESM-only distribution
- `C-style-spec.md` — style-spec fork vs upstream v26.8.0, trial rebase result
- `D-app.md` — slopedothtml usage of the fork and current test state
- `E-trial-rebase.md` — trial cherry-pick of the gl-js fork onto v6.18.0 (may be missing if the run had not finished)
