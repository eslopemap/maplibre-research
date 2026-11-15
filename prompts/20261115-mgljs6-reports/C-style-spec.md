# C — style-spec fork vs upstream v26.8.0

Repo `/home/user/maplibre-style-spec`, branch `blend` (HEAD b19f5ebd), merge-base 9b304f7a (upstream 24.7.0), target `v26.8.0` (352 upstream commits ahead).
All work was read-only; the trial worktree was removed, nothing pushed.

## 0. Which upstream version has what

| Commit | PR | First tag |
|---|---|---|
| 10b247c7 `{fill,line}-layer-blend` | #1798 | **v26.5.0** |
| e285fb6c layer-blend test cases | #1950 | v26.5.0 |
| 749083c9 `raster-layer-blend` | #1964 | **v26.6.0** |

So GL JS v6.18.0's `^26.6.0` is exactly the first version with `raster-layer-blend`.

## 1. Fork inventory (`git log 9b304f7a..HEAD`, 6 commits)

| # | Commit | Date | Content |
|---|---|---|---|
| 1 | 20fad69e | 2026-02-09 | new `slope` layer type (`layout_slope`, `paint_slope` with `slope-opacity`, `slope-color`), `slope` expression, validate_layer allows raster-dem |
| 2 | 55d1eabe | 2026-02-09 | tests: `expression/tests/slope/basic/test.json`, `slope-valid` layer in `layers.input.json`, `layers.output.json` |
| 3 | 103a3156 | 2026-02-17 | **renames `slope` layer to `terrain-analysis`**, adds `terrain-analysis-attribute`, `aspect` expression; regenerates 37 `*.output.json` |
| 4 | 040c485d | 2026-03-23 | `blend-mode` on `paint_raster` and `paint_terrain-analysis` (normal/multiply/screen), type tests |
| 5 | 4b7dc8ba | 2026-03-29 | drops the `terrain-analysis-` prefix (`attribute`, `opacity`, `color`), adds `soft-multiply` |
| 6 | b19f5ebd | 2026-03-29 | package rename `@eslopemap/maplibre-gl-style-spec`, 24.7.0 -> 24.7.1, `publishConfig.access=public`, lockfile churn (1202 lines) |

Net result: there is **no `slope` layer type any more** — only `terrain-analysis`. Leftovers of the old name are stale (see "fork defects").

### 1.1 `src/reference/v8.json` (net +243 lines)

- `layer.type.values["terrain-analysis"]` (doc: "Client-side terrain analysis visualization based on DEM data…", sdk-support js `5.17.0`, native issue 3408).
- `layout` list gains `layout_terrain-analysis`; `paint` list gains `paint_terrain-analysis`.
- `layout_terrain-analysis`: only `visibility` (enum visible/none, default `visible`, expression params `["global-state"]`, `data-constant`) — copy of color-relief's.
- `paint_terrain-analysis` (all names unprefixed):

| Property | Type | Default | Expression | property-type | transition |
|---|---|---|---|---|---|
| `attribute` | enum `elevation` / `slope` / `aspect` | `slope` | not interpolated, `["zoom"]` | data-constant | (unset) |
| `opacity` | number 0..1 | 1 | interpolated, `["zoom"]` | data-constant | true |
| `color` | color | none | interpolated, `["elevation","slope","aspect"]` | color-ramp | false |
| `blend-mode` | enum `normal` / `multiply` / `soft-multiply` / `screen` | `normal` | not interpolated, `["zoom"]` | data-constant | false |

- `paint_raster["blend-mode"]`: identical definition, inserted before `resampling`:

```json
"blend-mode": {
  "type": "enum", "default": "normal",
  "doc": "Controls how this raster layer is blended with layers beneath it.",
  "values": {
    "normal":        {"doc": "Standard alpha blending (default)."},
    "multiply":      {"doc": "Multiplies the layer colors with the colors beneath, darkening the result."},
    "soft-multiply": {"doc": "Softer, less darkening version of multiply that does not crush midtones."},
    "screen":        {"doc": "Inverts, multiplies, and inverts again, lightening the result."}
  },
  "transition": false,
  "sdk-support": {"basic functionality": {"js": "5.21.0", "android": ".../2963", "ios": ".../2963"}},
  "expression": {"interpolated": false, "parameters": ["zoom"]},
  "property-type": "data-constant"
}
```

- `expression_name.values`: new `slope` (0-90 deg) and `aspect` (0-360 deg, 0 = north), both zero-arg -> number, group `"Terrain Analysis"`, js `5.17.0`.
- `elevation` expression entry: **not changed** by the fork. It is only newly listed as an allowed parameter of terrain-analysis `color`.
- `$root.layers.doc`: list of types extended with `` `slope` `` (stale; should say `terrain-analysis`).

### 1.2 TS sources

- `src/expression/compound_expression.ts`: registers `slope: [NumberType, [], ctx => ctx.globals.slope || 0]` and same for `aspect`; adds both to the non-constant list in `isExpressionConstant`.
- `src/expression/index.ts`: `GlobalProperties` gains `slope?: number; aspect?: number`.
- `src/index.ts`: `ExpressionParameters` union gains `'slope'` (not `'aspect'`).
- `src/validate/validate_layer.ts`: raster-dem source allowed for `terrain-analysis`; message becomes "…'hillshade', 'color-relief', or 'terrain-analysis'."
- `build/generate-style-spec.ts` (type generation): **not touched**. `TerrainAnalysisLayerSpecification` and `blend-mode` types come out of the generator automatically from v8.json.
- `src/index.test-d.ts`: raster paint accepts `blend-mode: 'multiply' | 'screen'`, rejects `blend-mode-transition`.

### 1.3 Tests and the one-line `*.output.json` changes

- New: `test/integration/expression/tests/slope/basic/test.json` (interpolate over `["slope"]`, input `{slope: 45}`).
- `layers.input.json`: `slope-valid` layer (type `slope`, `slope-opacity`, `slope-color`) and `"blend-mode": "multiply"` on the `redundant-resampling` raster layer.
- 37 `*.output.json` files changed in commit 103a3156; 36 of them differ only by `\ No newline at end of file`. Cause: they were regenerated with `UPDATE=1`, and `validate_spec.test.ts` writes `JSON.stringify(result, null, 2)` with no trailing newline. Pure noise, no semantic change; only `layers.output.json` has real content changes.

### 1.4 Fork defects worth fixing during the rebase

1. `slope-valid` fixture still uses the removed `slope` layer type, so the expected output now *asserts two errors* for it (`type: expected one of […] "slope" found`, and the raster-dem error). There is no positive validation test of a `terrain-analysis` layer.
2. `$root.layers.doc` mentions `slope` instead of `terrain-analysis`.
3. `'aspect'` missing from `ExpressionParameters` in `src/index.ts`.
4. `addGlobalState()` in `src/expression/index.ts` rebuilds globals from a fixed list (`zoom, heatmapDensity, elevation, lineProgress, isSupportedScript, accumulated`) — `slope`/`aspect` are **dropped whenever a global state is attached**. Present in both the fork and v26.8.0 (line ~842); must be extended.
5. No expression test for `aspect`; no CHANGELOG entry.

## 2. Upstream `*-layer-blend` at v26.8.0

Three **paint** properties: `paint_fill["fill-layer-blend"]`, `paint_line["line-layer-blend"]`, `paint_raster["raster-layer-blend"]`. Identical value sets (enforced by a test).

```json
"raster-layer-blend": {
  "type": "enum",
  "values": {
    "normal":       {"doc": "[Normal](https://www.w3.org/TR/compositing-1/#blendingnormal) blending. The layer is drawn on top of existing content using standard alpha compositing."},
    "multiply":     {"doc": "[Multiply](…#blendingmultiply) blending. The layer color is multiplied with the colors underneath, producing darker results. Useful for textures, hillshades, and combining raster and vector content."},
    "color":        {"doc": "[Color](…#blendingcolor) blending. Creates a color with the hue and saturation of the layer and the luminosity of the content underneath, preserving its gray levels. Useful for tinting monochrome content such as a hillshade."},
    "luminosity":   {"doc": "[Luminosity](…#blendingluminosity) blending. Creates a color with the luminosity of the layer and the hue and saturation of the content underneath. The inverse of `color`."},
    "plus-lighter": {"doc": "[Plus-lighter](…#porterduffcompositingoperators_plus) blending. Additive blending. The layer color is added to the colors underneath, producing brighter results. Useful for glow and heat accumulation effects."},
    "screen":       {"doc": "[Screen](…#blendingscreen) blending. The inverse of the colors is multiplied, producing lighter results. Useful for glow, heatmap, and light-based effects."},
    "overlay":      {"doc": "[Overlay](…#blendingoverlay) blending. Multiplies or screens colors depending on the colors underneath, preserving highlights and shadows while adding contrast."}
  },
  "default": "normal",
  "doc": "Controls how the layer is composited with the content already rendered beneath it. Unlike [`raster-opacity`](#raster-opacity), which only fades the layer, this property determines how its colors interact with the existing map contents.",
  "sdk-support": {"basic functionality": {
    "js": "https://github.com/maplibre/maplibre-gl-js/issues/48",
    "android": "https://github.com/maplibre/maplibre-native/issues/2963",
    "ios": "https://github.com/maplibre/maplibre-native/issues/2963"}},
  "property-type": "constant",
  "transition": false
}
```

Key points:

- **No `expression` key and `property-type: "constant"`** — a property type that exists in v8.json's `property-type` section ("Property is constant across all zoom levels and property values") and in the `ExpressionType` union in `src/index.ts`. No zoom, no global-state, no expressions at all. Generated type is a bare literal union, not `PropertyValueSpecification<…>`:
  `"raster-layer-blend"?: "normal" | "multiply" | "color" | "luminosity" | "plus-lighter" | "screen" | "overlay"`.
- sdk-support in the spec still points at issues (js issue #48) rather than a version number, even at v26.8.0. Whether GL JS 6.18 actually renders it must be checked on the GL JS side.
- fill/line docs reference `assets/{fill,line}-layer-blend.svg`; raster has no image.
- **Validation code**: none specific. Generic enum validation applies; an expression value fails as a non-enum value. The only dedicated code is a spec self-test in `test/integration/style-spec/validate_spec.test.ts`:
  - collects every paint/layout property whose name ends in `-layer-blend`,
  - asserts the list equals exactly `['fill-layer-blend','line-layer-blend','raster-layer-blend']`,
  - asserts all have identical value keys.
  Plus fixture `tests/layer-blend.input.json` (expected output `[]`).
- Related: `fill-layer-opacity` / `line-layer-opacity` (24.10.0, global-state allowed since 25.0.0) established the `<type>-layer-*` naming for whole-layer compositing.

### Fork `blend-mode` vs upstream enum

| Fork `blend-mode` | Upstream `*-layer-blend` | Note |
|---|---|---|
| `normal` | `normal` | 1:1 |
| `multiply` | `multiply` | 1:1 |
| `screen` | `screen` | 1:1 |
| `soft-multiply` | — | **no equivalent**; not a W3C compositing mode |
| — | `color`, `luminosity`, `plus-lighter`, `overlay` | new upstream |

Other differences: the fork allows zoom expressions (`data-constant`, `parameters:["zoom"]`); upstream is strictly constant. Fork applies it to `raster` + `terrain-analysis`; upstream to `fill`, `line`, `raster` (not hillshade, color-relief).

## 3. Upstream changes 9b304f7a..v26.8.0 that matter

Breaking / major:

- **25.0.0** — legacy filter validation changed to add suggestions on invalid expressions (different error messages).
- **26.0.0** — `ValidationError` gains a `severity` field (every expected `*.output.json` entry now has `"severity": "error"`); runtime warnings carry a location, which changed signatures: `createExpression(expression, rootKey, propertySpec?, globalState?)` and `createPropertyExpression(expressionInput, rootKey, propertySpec, …)` now take a **`rootKey` second argument**. Expression runtime errors are prefixed with the key (`"layers[0].paint.some-property: Expected value…"`). Any GL JS fork code calling these must be updated.
- **26.1.0** — `src/reference/latest.ts` no longer `export default latest as any`; it exports a typed `StyleSpecificationReference = typeof v8`. The reference JSON is now typed from the JSON itself, so fork additions are visible/checked in types.
- **26.4.3** — `DiffCommand` is a discriminated union; legacy class arg dropped from `setPaintProperty`/`setLayoutProperty` diff commands.
- **26.4.1** — ESM build is published as one module per source file rather than a single bundle.
- **26.4.1 / 26.1** — `MLFontFace` / `FontFacesSpecification` types changed.

Build / tooling:

- 24.8.2: rollup -> **rolldown**, TS 6 with `tsgo` (`@typescript/native-preview`) for typecheck; `rollup.config.ts` deleted, `rolldown.config.ts` added; `dts-bundle-generator` replaced by `rolldown-plugin-dts`; `ts-node` removed (scripts run with plain `node build/*.ts`, imports use `.ts` extensions); `rw` replaced by `node:fs`; vitest 4 -> 5; `.nvmrc` now 24.20 (unit + integration tests still passed on Node 22 in the trial).
- `npm run build` now runs `generate-style-spec` first; `npm test` = unit + integration + build tests.
- 24.8.1: generator emits `AllPaintProperties` / `AllLayoutProperties` (object-merge over all layer types — unprefixed names shared by several layer types, like the fork's `blend-mode`, `color`, `opacity`, collapse to the last definition).

Touching the fork's areas:

- **`elevation` expression** (26.5.0, #1851): now also allowed in `fill-extrusion-color`; doc reworded ("For a `raster-dem` source this is the elevation of a pixel…; for a fill extrusion it is the height above the ground"), new sdk-support sub-entry. Runtime definition unchanged (`ctx.globals.elevation || 0`), so the fork's `slope`/`aspect` lines apply next to it cleanly.
- **color-relief / hillshade**: no paint property changes (still `color-relief-opacity`, `color-relief-color`, `resampling`; hillshade unchanged); only doc link and sdk-support updates. Neither got a `*-layer-blend`.
- **Layer type registration**: unchanged mechanism (`layer.type.values`, `layout`/`paint` lists, `layout_*`/`paint_*` sections); `validate_layer.ts` raster-dem check unchanged.
- **Type generation**: `build/generate-style-spec.ts` changed (new expression types, `AllPaint/LayoutProperties`) but the fork never touched it.
- Experimental `emptyResponse` (26.5.1) on sources including raster-dem, replaced in 26.7.0 by `sparse` on vector sources only.
- Other additions: `semiliteral`, `symbol-height-offset/anchor`, `fill-extrusion-rounded-corner-distance`, negative extrusion heights, `oklch`/`oklab`/`display-p3` colors and `interpolate-oklch/oklab`, optional image source `url`.

## 4. Trial rebase (cherry-pick of the 6 commits onto v26.8.0)

| Commit | Result |
|---|---|
| 20fad69e slope layer | conflict: `layers.output.json` (3 hunks) only; v8.json and all TS applied cleanly |
| 55d1eabe tests | conflict: `layers.output.json` (1 hunk) |
| 103a3156 terrain-analysis | conflict: `layers.output.json` (3 hunks); the 36 newline-only output files merged cleanly |
| 040c485d blend-mode | clean |
| 4b7dc8ba rename props | clean |
| b19f5ebd package | conflict: `package.json` (1 hunk: version line) and `package-lock.json` (94 hunks) |

Difficulty: **low**. Every conflict is in a regenerable file (`UPDATE=1` for the fixture, `npm install` for the lockfile) or a one-line version choice. No conflict in `v8.json` or any TS source.

Verification on the 5 content commits applied to v26.8.0 (conflicted fixture left at upstream's version), after `npm ci`:

- `npm run generate-style-spec`: OK; `TerrainAnalysisLayerSpecification` generated; raster has both `raster-layer-blend` (literal union) and `blend-mode` (`PropertyValueSpecification<…>`).
- unit tests: 710/710 pass, no type errors.
- integration: 2344/2345 pass; the single failure is `layers.output.json` needing regeneration.
- build tests not run.

Recommendation: do not replay the history; squash into 2-3 fresh commits on top of v26.8.0 (spec + TS, tests, package), which also removes the slope -> terrain-analysis churn and the newline noise.

## 5. Recommended target design

### Raster
Drop the fork's `blend-mode` on raster and use upstream **`raster-layer-blend`** as is. `normal`/`multiply`/`screen` map 1:1. This removes one fork delta entirely.

### terrain-analysis
Add **`terrain-analysis-layer-blend`** following upstream: paint, enum, `default: "normal"`, `property-type: "constant"`, `transition: false`, no `expression`, same 7 values and docs. Consequences:

- upstream's self-test must be edited: add the name to the expected list (one line). If the value set is identical, the "same values" test passes unchanged.
- loses zoom-expression support for blend (the fork allowed `["zoom"]`); check whether any style uses it — unlikely for an enum.

### soft-multiply — options

| Option | Pros | Cons |
|---|---|---|
| A. Add `soft-multiply` as an 8th value to `terrain-analysis-layer-blend` only | minimal, explicit | breaks upstream's "all layer-blend properties have the same values" test (needs a fork exception); permanent enum divergence |
| B. Add `soft-multiply` to all four `*-layer-blend` enums | keeps the self-test green; raster keeps soft-multiply too | fork must implement it for fill/line in GL JS or document as unsupported; diverges 3 upstream properties; conflict-prone on every upstream enum edit |
| C. Drop `soft-multiply` from the spec; express it as `multiply` plus lower `opacity`, or a lighter color ramp | zero enum divergence, smallest fork | only valid if the shader is equivalent to "multiply with a lerp toward white"; needs checking against the GL JS implementation; for raster needs `raster-opacity` to act as that lerp |
| D. Separate fork-only numeric paint property (e.g. `terrain-analysis-blend-strength`, 0..1) | upstream-shaped enum, tunable, clean semantics | new property to implement; not applicable to raster unless duplicated |

Suggested: **C if the math allows it, otherwise A** (scoped to the fork-only layer, with a two-line test exception). Avoid B.

### Keep `blend-mode` as a deprecated alias?
- Pro: existing styles using `blend-mode` keep validating.
- Con: two properties for one thing, needs a "redundantly specifies" validation rule like `resampling`/`raster-resampling`, more permanent fork delta, and unprefixed names collide in `AllPaintProperties`.
- Suggested: **no alias in the spec**. The package jumps 24 -> 26 anyway (two majors); migrate styles once (`blend-mode` -> `raster-layer-blend` / `terrain-analysis-layer-blend`), optionally with a small rewrite in the application layer for old saved styles.

### Property naming
The fork's unprefixed `attribute` / `opacity` / `color` deviate from upstream convention (only the shared `resampling` is unprefixed upstream) and merge ambiguously in `AllPaintProperties`. Renaming back to `terrain-analysis-*` is cleaner for upstreaming but costs a GL JS + style migration; decide together with the blend rename since styles must be touched anyway.

### Minimal fork delta that must remain
1. `v8.json`: `terrain-analysis` layer type, `layout_terrain-analysis`, `paint_terrain-analysis` (attribute, opacity, color, layer-blend), entries in the `layout`/`paint` lists, `slope` and `aspect` expressions, corrected `$root.layers.doc`.
2. `compound_expression.ts`: register `slope`, `aspect`; add to `isExpressionConstant` list.
3. `expression/index.ts`: `GlobalProperties.slope/aspect` **and** pass them through `addGlobalState`.
4. `src/index.ts`: `'slope' | 'aspect'` in `ExpressionParameters`.
5. `validate_layer.ts`: allow raster-dem for `terrain-analysis`.
6. Tests: slope + aspect expression tests, a valid `terrain-analysis` fixture layer, regenerated `layers.output.json`, updated layer-blend self-test, type tests for the new property.
7. `package.json`: name `@eslopemap/maplibre-gl-style-spec`, `publishConfig`, a version scheme that tracks upstream (e.g. `26.8.0-eslope.1` or `26.8.1`); regenerate lockfile.

Open point outside this repo: GL JS 6.18 will import `@maplibre/maplibre-gl-style-spec`; the fork package needs an alias/override there, and the GL JS call sites must match the 26.x `createExpression(…, rootKey, …)` signature.
