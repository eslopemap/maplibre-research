# D — App-side impact of rebasing the fork onto MapLibre GL JS v6.18.0

Repo: `/home/user/slopedothtml` @ `8d4274e` (main). All paths relative to repo root unless absolute.
Nothing committed; `git status` clean at the end (see section 6).

## 1. How MapLibre is consumed

Zero-build static app; MapLibre is a **vendored classic UMD script exposing the global `maplibregl`** — not an npm dependency, not an ES import.

| Where | What |
|---|---|
| `deps.json:7-14` | `"name": "@eslopemap/maplibre-gl", "range": "5.21.9"`, files `dist/maplibre-gl.js`, `dist/maplibre-gl.css`, `importMap: {}` (exact pin; exact pins bypass the 7-day `minimumAgeDays` gate) |
| `deps.json:3-4` | CDN/registry bases: `https://cdn.jsdelivr.net/npm`, `https://registry.npmjs.org` |
| `app/vendor/deps.lock.json:6-24` | version `5.21.9`, publishedAt 2026-03-29, sha256 integrity, `sourceUrl` `https://cdn.jsdelivr.net/npm/@eslopemap/maplibre-gl@5.21.9/dist/maplibre-gl.{css,js}` |
| `app/vendor/@eslopemap/maplibre-gl/5.21.9/dist/` | vendored `maplibre-gl.js` (1,057,186 bytes, UMD wrapper, header says `v5.21.9`) + `maplibre-gl.css` (70,024 bytes). Version is in the directory path. |
| `app/index.html:14-15` | `<link rel="stylesheet" href="vendor/@eslopemap/maplibre-gl/5.21.9/dist/maplibre-gl.css">`, `<script src="vendor/@eslopemap/maplibre-gl/5.21.9/dist/maplibre-gl.js">` — **hand-maintained**: `scripts/vendor-deps.mjs:248-251` only rewrites the inline `<script type="importmap">`, not these tags |
| `app/index.html:16` | `vendor/maplibre-contour/0.1.0/dist/index.min.js` (classic script, global `mlcontour`; `deps.json:16-17` range `^0.1.0`) |
| `app/index.html:17-30` | import map: gpxjs 1.1.0, chart.js 4.5.1, @kurkle/color 0.4.0, fflate 0.8.2, **pmtiles 4.4.0**, chartjs-plugin-annotation 3.1.0 (no maplibre entry) |
| `package.json` | maplibre is **not** listed (only `@we-gold/gpxjs` dep + test devDeps). `package-lock.json` has no maplibre. |
| `src-tauri/tauri.conf.json:7,11` | `frontendDist: "../app"` → desktop bundles the same vendored copy; `csp: null` (no CSP, so worker blob URLs are unrestricted) |
| Service worker | none (`serviceWorker` not referenced; `app/manifest.json` has no cache list) |
| `CONTRIBUTING.md:162-171` | "MapLibre note": pinned in deps.json, current pin `5.21.9`, "treated as a vendored classic browser script asset", relies on "forked package shape and current worker/bootstrap behavior" |
| `CONTRIBUTING.md:22`, `README.md:99`, `FEATURES.md:6,165,192` | prose mentions of the fork |
| `reports/20260401-VENDOR-ESM-REPORT.md:91-101,199` | rationale for keeping UMD/classic script; pin 5.21.9 |
| Non-app copies (CDN) | `spike_demo/shared_frontend/index-protocol.html:7,17` and `index-localhost.html:7,17` → `https://unpkg.com/@eslopemap/maplibre-gl@5.21.9/...`; `debug/debug-bisect-*.html` and `debug/debug-color-relief-repro.html` lines 6-7 → `unpkg.com/@eslopemap/maplibre-gl@5.21.2` (+ `maplibre-contour@0.0.5` in bisect-11/12) |
| Historical | `prompts/20260323-IMPLEM-FORKED-MAPLIBRE.md` (5.20.0 upstream → fork 5.21.2), `prompts/20260323-TERRAIN-ANALYSIS-3D-STATUS.md` |

Update tooling:
- `npm run vendor:update` / `just vendor-update` → `node scripts/vendor-deps.mjs update` (downloads from jsDelivr into `app/vendor/<name>/<version>/`, rewrites lock + import maps).
- `npm run vendor:check` / `just vendor-check`.
- `just deps-update` = vendor-update + `scripts/cargo-cooldown-update.sh` + `scripts/npm-cooldown-update.sh`.
- `.github/workflows/deps-update.yml` runs the same bi-monthly and opens PR `deps/update-all` (the fork is exact-pinned so it does not move).

Upgrade checklist (app side): bump `deps.json:8`; run `vendor:update`; hand-edit `app/index.html:14-15`; delete old `app/vendor/@eslopemap/maplibre-gl/5.21.9/`; update `CONTRIBUTING.md:169`, `reports` are historical; optionally `spike_demo/*` URLs.

## 2. Fork-only and upgrade-sensitive API usage

### 2a. `terrain-analysis` layer type (fork-only)

Two layers, both in the initial style (`app/js/main.js:407-408`, re-ensured after `setStyle` at `main.js:475-476`), source `dem-hd` (raster-dem, terrarium, tileSize 512, maxzoom 14: `main.js:329-337`):

- `analysis` — `main.js:355-368`: `type: 'terrain-analysis'`, layout `visibility`, paint **`attribute`** `'slope'`, **`opacity`**, **`color`**, **`blend-mode`**.
- `analysis-relief` — `main.js:370-383`: paint `attribute: 'elevation'`, `opacity`, `color`, `blend-mode`.

Paint property names are **unprefixed** (`attribute`, `opacity`, `color`, `blend-mode`). Vendored 5.21.9 spec (extracted from bundle): `attribute` enum {elevation, slope, aspect}; `opacity` number, zoom expr, transition; `color` color-ramp with expression parameters `elevation|slope|aspect`; `blend-mode` enum {normal, multiply, soft-multiply, screen}, zoom-parameter, no transition.

Runtime mutation, all via `setPaintProperty`/`setLayoutProperty` in `applyModeState` (`app/js/ui.js:87-144`):
- `attribute`: `ui.js:107` ('slope'), `ui.js:129` (`state.mode` = 'slope' | 'aspect')
- `color`: `ui.js:108`, `ui.js:130`
- `opacity`: `ui.js:109-113` and `ui.js:118-122` (zoom `interpolate` expressions for the slope+relief crossfade), `ui.js:131`, `ui.js:139`
- `blend-mode`: `ui.js:112, 123, 132, 140`
- visibility: `ui.js:91-92, 104, 116, 127, 134-136`

Colour expressions (`app/js/constants.js:12-57`):
- `['step', ['slope'], ...]` 13 stops (`constants.js:13-29`)
- `['step', ['aspect'], ...]` (`constants.js:30-37`)
- `['interpolate', ['linear'], ['elevation'], ...]` 17 stops (`constants.js:38-57`); parsed for the legend at `constants.js:126`.
`MAX_STEP_STOPS = 16` (`constants.js:4`).

Upstream `color-relief` layer type is **not** used; "color-relief" is only an app mode name (`app/index.html:141`, `ui.js:55,135,250`, `constants.js:38,63`). `debug/debug-color-relief-repro.html` is a repro page only.

Known fork constraint: terrain-analysis must sit in the initial style right after hillshade or it disappears in 3D terrain (`prompts/20260323-TERRAIN-ANALYSIS-3D-STATUS.md`, repro `debug/debug-bisect-14.html`). Two separate raster-dem sources (`dem-terrain`, `dem-hd`) are used deliberately (`main.js:390-397` comment: one source = one shared TileManager).

### 2b. `blend-mode` (fork-only)

- Only ever set on the two **terrain-analysis** layers. **No raster layer in the app sets `blend-mode`** (all `type: 'raster'` entries in `app/js/layer-registry.js:39-438` and `app/js/tauri-bridge.js:193-222` have no blend paint).
- Values used: `'multiply'`, `'soft-multiply'`, `'normal'`. `'screen'` is never used.
- Rule (duplicated 3×): `state.multiplyBlend ? (state.slopeOpacity > 0.7 ? 'multiply' : 'soft-multiply') : 'normal'` — `main.js:365`, `main.js:380`, `ui.js:88`.
- UI: one checkbox `#multiplyBlend` "Multiply blend" (`app/index.html:157`), handler `main.js:1030-1034` → `applyModeState`; restored from persisted settings at `main.js:203`. Also re-evaluated whenever analysis opacity changes (threshold 0.7) since those paths call `applyModeState`.
- State/persistence: `state.multiplyBlend: true` default (`app/js/state.js:33`), persisted key (`app/js/persist.js:75`). `state.layerSettings` comment mentions a per-layer `blend?` field (`state.js:29`) — **not implemented**.
- Debug panel prints `l.paint?.['blend-mode']` from `map.getStyle()` (`main.js:609-610`) — depends on the property name.
- Not in URL hash / bookmarks as a separate key beyond the persisted settings (no `blend` in `ui.js` URL parsing).

Migration note: with upstream v6 naming the equivalent on rasters is `raster-layer-blend`; the app only needs the terrain-analysis blend property. If the rebased fork renames it (e.g. `terrain-analysis-layer-blend` / `layer-blend`) touch `main.js:365,380,609` and `ui.js:112,123,132,140`; `soft-multiply` is a fork-only value that must survive or be replaced.

### 2c. Private MapLibre internals

- `app/js/dem.js:40-47, 60-68`: `map.style.tileManagers[sourceId]`, `.getRenderableIds()`, `.getTileByID(id)`, `tile.dem` (`.get(x,y)`, `.dim`), `tile.tileID.canonical`, `tileManager.maxzoom/minzoom`. Used for cursor elevation/slope (`queryLoadedElevationAtLngLat`). Fails soft (returns null) if the shape changes → cursor readout silently disappears. Unit test `tests/unit/dem.test.mjs` only covers `sampleElevationFromDEMData` with mocks, not the internals.
- No `map.painter`, `map._render`, `map.style._*`, no custom (`type: 'custom'`) layers (the old custom WebGL layer was removed).
- App attaches its own expando props on the map: `map.__activeStyleMode`, `map.__nativeBasemapLayerIds`, `map.__ensureBasemapStyle` (`main.js:645-657`), `map.__nativeOpacityDefaults` (`layer-engine.js:73-78`).
- Tests call `map.fire('click'|'dblclick'|mouse…, {...})` with synthetic events (`tests/e2e/helpers.js:95,105,214`), `map.loaded()`, `map.isMoving()`.

### 2d. Protocols, terrain, projection

- `maplibregl.addProtocol('pmtiles', protocol.tile)` / `removeProtocol` (`app/js/pmtiles-protocol.js:24,41`), with `pmtiles@4.4.0`.
- `mlcontour.DemSource({url, encoding:'terrarium', maxzoom:12, worker:true})` + `demContourSource.setupMaplibre(maplibregl)` + `contourProtocolUrl(...)` (`main.js:296-325`) — maplibre-contour 0.1.0 registers its own protocol through `maplibregl.addProtocol`.
- `map.setTerrain({source:'dem-terrain', exaggeration})` / `setTerrain(null)` (`ui.js:39-45`); custom `Terrain3DControl` IControl (`main.js:507-553`) using CSS classes `maplibregl-ctrl maplibregl-ctrl-group`.
- No globe / `setProjection` / sky / light.

### 2e. General API surface (v5→v6 check-list)

- Import style: global `maplibregl` from UMD classic script; `maplibregl.Map`, `.NavigationControl`, `.GeolocateControl`, `.ScaleControl`, `.AttributionControl`, `.Marker`, `.LngLatBounds`, `.addProtocol`, `.removeProtocol` (`main.js:481,495,501,555,557`; `tracks.js:954,965`; `track-edit.js:123`; `spike_demo/shared_frontend/app.js:48,95`). **If v6 ships ESM-only this is the largest app-side change** (index.html script tag, `mlcontour.setupMaplibre(maplibregl)`, every `maplibregl.*` reference, Playwright tests evaluating the global `map`).
- Map constructor options (`main.js:481-491`): `container, center, zoom, bearing, pitch, maxTileCacheZoomLevels: 20, attributionControl: false, style, canvasContextAttributes: {antialias: true}`.
- Controls: `NavigationControl({visualizePitch, visualizeRoll, showZoom, showCompass})`, `GeolocateControl({positionOptions, trackUserLocation, showUserLocation, showAccuracyCircle})`, `ScaleControl({unit, maxWidth})`, `AttributionControl()`.
- Style: `version: 8`, remote `glyphs`/`sprite` from vectortiles.geo.admin.ch (`main.js:386-388`); `map.setStyle(urlOrObject)` to swap to the SwissTopo vector style and back (`main.js:655`), `style.load` handlers (`main.js:653,659`), `map.getStyle()` (`main.js:598,663`).
- **Global state** (v5.6+ feature): `map.setGlobalStateProperty` (`tracks.js:536-538`, wrapper `ui.js:19-22`), `['global-state', …]` expressions (`constants.js:167`, `main.js:347`, `tracks.js:187,265-268`).
- Hillshade: `hillshade-method` (`standard|basic|combined|multidirectional|igor`), `hillshade-exaggeration` expression, shadow/highlight/accent colours (`main.js:340-352`).
- Layer types used: background, hillshade, terrain-analysis, raster, fill, line, symbol, circle; sources raster, raster-dem, vector, geojson.
- Methods (counts in app/js): getLayer 32, getSource 25, getCanvas 23, triggerRepaint 17, setPaintProperty 15, addLayer 12, setLayoutProperty 11, addSource 10, project 6, addControl 5, setGlobalStateProperty 4, fitBounds 3, easeTo 3, unproject 2, setTerrain 2, removeLayer 2, queryRenderedFeatures 2 (`track-edit.js:245`, `main.js:1678`), moveLayer 2 (`layer-engine.js:217,333`), getStyle 2, flyTo 2, setStyle 1, removeSource 1, jumpTo 1, getBounds 1; `GeoJSONSource.setData` throughout tracks/profile.
- Events: load, style.load, data, error, click, dblclick, mousedown/move/up/out, touchend, dragstart, move, moveend, zoomend, rotateend, pitchend (list with lines: `main.js:629,653,659,1567,1583-1601,1626,1654-1703,1758`; `tracks.js:1237,1324`; `track-edit.js:479-837`; `selection-tools.js:233`).
- Not used: `transformRequest`, `setWorkerUrl`/CSP worker, `queryTerrainElevation`, feature-state, images, custom layers.
- Opacity helper maps layer type → `raster-opacity|fill-opacity|line-opacity|text-opacity+icon-opacity|background-opacity|circle-opacity` (`layer-engine.js:57-65`); terrain-analysis handled separately via `opacity`.

## 3. Docs / plans: features depending on blending

- `FEATURES.md:11` "Multiply blend — optional compositing mode for the DEM analysis overlay on the basemap"; `:192-193` terrain-analysis supports slope/aspect/elevation expressions, ramps, blend modes, switched with `setPaintProperty`; `:202` settings list includes "Multiply blend".
- `FEATURES.md:171-178` z-order doc says OpenSkiMap sits above analysis "so blend mode composites cleanly", but the code now moves **all** catalog layers (basemaps and overlays, raster and vector) below `dem-loader` (`layer-engine.js:177,213-217,324-333`), so the stack is: catalog basemaps/overlays → hillshade (`dem-loader`) → `analysis` → `analysis-relief` → contours → tracks. The terrain-analysis layer therefore multiplies over everything beneath it, including vector overlays and hillshade.
- Layer types that get a blend mode today: **only terrain-analysis**. Basemap rasters: none. Raster slope overlays (`pubmap-eslo-walps`, `ign-slopes`, `swisstopo-slope`): none yet. Vector overlays (OpenSkiMap fill/line/symbol, `layer-registry.js:271-309`): none.
- `plans/20260410-PLAN-BLEND-MODES-AND-LAYER-ORDER.md` (from `prompts/20260410-blend-raster.md`) — **planned, not implemented**: catalog-level default `blend: 'multiply'|'normal'` for raster overlays, per-layer override in `state.layerSettings[catalogId].blend`, resolution order user → catalog → normal; candidates `pubmap-eslo-walps` (likely multiply), `ign-slopes`/`swisstopo-slope` (maybe), ski rasters and basemaps normal; `openskimap` explicitly "not relevant … because it is vector, not raster"; a `×` marker in the debug panel for multiply layers (current code prints the raw blend string instead, `main.js:609`); unified layer-order model with system layers. This plan is where fork raster `blend-mode` → upstream `raster-layer-blend` matters, and where `fill-layer-blend`/`line-layer-blend` would newly allow vector overlays (OpenSkiMap pistes) to take part.
- **No workaround exists for vector-layer blending** (no canvas/CSS `mix-blend-mode`, no pre-rasterising); grep for `blend|multiply` in `app/css`, `app/js` shows only the items in 2b. So there is nothing to remove, only a new capability.
- `prompts/20260326-FEAT.md:633`: user note that terrain-analysis looks "a bit darker than I expect (both in multiply or normal blend)" — a visual baseline to re-check after the rebase.
- `app/user-guide/content/map-and-visualization.md` describes modes; no blend documentation in the user guide.

Conventions (`AGENTS.md`):
- "when asked to plan, generate a file in plan/ folder" (actual dir is `plans/`); "when implementing a complex feature (only), create a report in reports/ folder"; update `app/user-guide` when adding features; commit often with `nit/feat/chore` prefixes; e2e should use `#test_mode=true`; transversal concerns: persistence & bookmarks, caches; see `tests/README.md`.
- Naming: `plans/YYYYMMDD-PLAN-<TOPIC>.md` (also `YYYYMMDD-IMPL-PLAN-…`, a few `YYYYMMDD_…`); `prompts/YYYYMMDD-<topic>.md` (verbatim user prompts, later ones lower-case with optional sequence number, e.g. `20260408-3-prompts-bug-custom-source.md`); `reports/YYYYMMDD-<TOPIC>-REPORT.md`.

## 4. Test state

### Suites and how they run
| Suite | Location | Command | Notes |
|---|---|---|---|
| Vitest unit | `tests/unit/*.test.mjs` (19 files) | `npm run test:unit` / `just test-unit`; coverage `npm run test:coverage` | node env, mocks only, no MapLibre |
| Playwright e2e | `tests/e2e/*.spec.js` (11 files, 89 tests) | `npm test` / `just test-e2e`; `npm run test:e2e:coverage` | `playwright.config.js`: Chromium + SwiftShader, 1 worker, 10 s timeout, `retries: CI ? 1 : 0`, python http.server :8089 |
| Screenshot tests | inside Playwright: `dem-loading.spec.js` (3× `toHaveScreenshot`, `maxDiffPixelRatio 0.08`, full page) and `tile-serving.spec.js` (2× centre-crop `toMatchSnapshot`, 0.02) | — | committed baselines are **`*-chromium-darwin.png` only** |
| Rust | `src-tauri` (48 tests per README) | `cargo test` / `just test-rust` | not run here: `webkit2gtk-4.1` dev libs absent |
| Tauri WebDriver e2e | `tests/tauri-e2e` (wdio, 27 tests, pixelmatch baselines in `tests/tauri-e2e/snapshots/`) | `just test-tauri-e2e` | needs built desktop binary; not run here; **not in CI** |

Docs are stale on counts: README/CONTRIBUTING say 80 unit / 51 e2e, `tests/README.md` says 14 files / 73 e2e.

### Results in this environment (node 22.23.4, npm 10.9.9)

- `npm ci` **fails**: lockfile out of sync (`Missing: @emnapi/core@1.11.3`, `@emnapi/runtime@1.11.3`; `@emnapi/wasi-threads@1.2.1` does not satisfy 1.2.3). CI uses `npm ci`, so this may also break CI depending on npm version. Used `npm install` instead and reverted `package-lock.json` (copy of the regenerated lock: `/tmp/claude-0/-home-user/ce0ea5fe-3ab0-5ba9-ae43-3190e5bd5477/scratchpad/package-lock.installed.json`).
- **Unit**: 19 files, **179/179 passed**, 2.1 s (2.8 s wall).
- **Playwright** (project pins 1.58.2 → wants chromium rev 1208; ran with `executablePath: /opt/pw-browsers/chromium` via wrapper config `/tmp/claude-0/-home-user/ce0ea5fe-3ab0-5ba9-ae43-3190e5bd5477/scratchpad/pw.config.js`, `--trace off`): **82 passed / 7 failed, 3.6 min**. Network is irrelevant: `installOfflineRouteGuard` (`tests/e2e/helpers.js:19-31`) fulfils every non-local request with 404 and DEM tiles are served from fixtures by route interception.

| Failing test | Reason | Class |
|---|---|---|
| `dem-loading.spec.js:154` color-relief | "A snapshot doesn't exist … -chromium-linux.png, writing actual" | environment/baseline (darwin-only baselines) |
| `dem-loading.spec.js:185` slope+relief | same | same |
| `dem-loading.spec.js:206` slope | same | same |
| `tile-serving.spec.js:110` MBTiles | same | same |
| `tile-serving.spec.js:143` PMTiles | same | same |
| `track-desktop.spec.js:321` Redo restores midpoint insert | undo left 4 points instead of 3 | **flaky** — passed 2/2 on re-run |
| `track-import.spec.js:223` Multi-segment collapse/expand | after re-expand 1 row instead of 4 | **genuine, deterministic** (failed 3/3), unrelated to MapLibre |

Re-run of the 5 screenshot tests ×2 against the linux baselines generated by the first run: all pass (rendering is deterministic under SwiftShader). Those generated baselines were moved out of the repo to `/tmp/claude-0/-home-user/ce0ea5fe-3ab0-5ba9-ae43-3190e5bd5477/scratchpad/linux-baselines-5.21.9/` — usable as the pre-upgrade 5.21.9 reference. Full log: `/tmp/claude-0/-home-user/ce0ea5fe-3ab0-5ba9-ae43-3190e5bd5477/scratchpad/e2e.log`.

Consequence for CI: on ubuntu with `retries: 1`, a missing linux baseline is written on attempt 1 and compared against itself on attempt 2, so the screenshot assertions are **vacuous in CI** (reported as flaky-pass, never a regression).

### Coverage of blend / terrain-analysis rendering (the safety net)

Covered:
- `tests/e2e/dem-loading.spec.js:154-225` — real terrain-analysis rendering in three modes with synthetic Terrarium DEM fixtures (`tests/fixtures/tiles/dem`), `basemap=none&opacity=1`: asserts layer existence/visibility, `gl.readPixels` non-white ratio > 5 %, and a full-page screenshot at 8 % tolerance. Because opacity = 1 and `multiplyBlend` defaults to true, the `slope` and `color-relief` shots are rendered with **`blend-mode: 'multiply'` over hillshade + white background** (hillshade is visibly multiplied through in the linux actual), so `multiply` is exercised implicitly.
- `tests/e2e/bookmark.spec.js:43-136` — mode/slopeOpacity/layerOrder state round-trip with `_analysis` (state only, `test_mode`, no rendering).
- `tests/unit/constants.test.mjs` (ramp parsing/legend), `tests/unit/layer-engine.test.mjs` (mock map: ordering, opacity properties), `tests/unit/ui-url-state.test.mjs`, `startup-state.test.mjs` (mode parsing).
- `tests/tauri-e2e/tests/dem-tile-serving.spec.mjs` — desktop DEM render baseline (`01-dem-tile-cache-working.png`), not runnable in CI/here.

Not covered (gaps to close before upgrading):
1. No assertion that names `blend-mode` at all — no test toggles `#multiplyBlend`, calls `getPaintProperty(..., 'blend-mode')`, or checks `applyModeState` emits the right value (no unit test of `applyModeState`).
2. `soft-multiply` (opacity ≤ 0.7, the **default** app state: slopeOpacity 0.45) and `normal` are never rendered; `screen` unused.
3. No pixel-level blend assertion (e.g. known basemap colour × known ramp colour = expected RGB); only "non-white ratio" plus an 8 % whole-page diff, which a blend regression limited to tone could pass. Full-page shots include UI chrome.
4. No blend over a raster basemap or vector overlay (tests use `basemap=none`); no test of future raster/fill/line blend.
5. `aspect` mode never rendered; slope+relief crossfade opacity expressions only at one zoom (12).
6. 3D terrain (`setTerrain`) + terrain-analysis (the known fragile path) has no test.
7. `map.style.tileManagers` cursor-elevation path has no e2e test.
8. Linux baselines missing; CI screenshot checks vacuous (above).
9. `setStyle` swap to SwissTopo vector style and back (re-adding terrain-analysis layers) is not render-tested.
10. No console-error/style-validation assertion (a renamed/removed paint property would surface only as a MapLibre `error` event → `main.js:1758`).

## 5. CI (`.github/workflows`)

- `ci.yml` (push/PR to main), three jobs on ubuntu-latest: `js-tests` (`npm ci`, `npm run test:coverage`); `rust-tests` (webkit deps, `cargo clippy -- -D warnings`, `cargo llvm-cov --text` in `src-tauri`, `RUSTFLAGS=-D warnings`); `e2e` (`npm ci`, `npx playwright install --with-deps chromium`, `npm run test:e2e:coverage`, uploads `coverage/e2e-summary.txt` always and `test-results/`+`playwright-report/` on failure). No Tauri WebDriver e2e, no `vendor:check`, no lint.
- `deps-update.yml`: cron 1st of every other month 08:00 UTC + manual; vendor update + cargo/npm cooldown updates; PR via `peter-evans/create-pull-request@v7`.
- `release.yml`: on `v*` tags/manual; `tauri-apps/tauri-action@v0` matrix macOS arm64/x64, ubuntu-22.04, windows; draft GitHub release.

## 6. Housekeeping

- `node_modules/` installed (git-ignored). `package-lock.json` restored with `git checkout`. Generated `*-chromium-linux.png` snapshots moved out of the repo. Final `git status --short`: empty.
