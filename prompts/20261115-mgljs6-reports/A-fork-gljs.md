# A — Fork inventory: maplibre-gl-js `blend` (@eslopemap/maplibre-gl@5.21.9)

Repo `/home/user/maplibre-gl-js`, branch `blend`, merge-base `0cac8ee50` (= `v5.21.0` + 1 commit), 26 fork commits.
Read-only audit; all line numbers are for `HEAD` (`04429fe27`). Source diff (excluding `dist/`, `package-lock.json`): 65 files, +3068/−300.

Companion spec fork: `/home/user/maplibre-style-spec` (branch `blend`, published as `@eslopemap/maplibre-gl-style-spec@24.7.1`). It defines the properties consumed here.

---

## 0. Headline findings (things that differ from what FORK.md / CHANGELOG say)

1. **`soft-multiply` at HEAD is NOT "Modulate 2X"** as FORK.md and CHANGELOG state. The last-but-one commit `c5606cf2e "attempt at better soft-multiply"` replaced it with *plain multiply GL blend + squared alpha in the shader*. FORK.md, CHANGELOG, and the two `ColorMode.softMultiply*` presets are now stale / dead code.
2. **The committed `dist/maplibre-gl.js` (the 5.21.9 build) still contains the OLD Modulate-2X soft-multiply** (`mix(vec3(u_blend_neutral),blendedRgb,color.a)`, no `effective_alpha`). So the two newest commits (`c5606cf2e` α² soft-multiply, `04429fe27` NEAREST DEM) are **after the 5.21.9 version bump and not in the committed bundle**. Which soft-multiply behaviour is "the one to preserve" needs a decision by the owner.
3. `soft-multiply` in an isolated 3D RTT stack silently degrades to plain `multiply` (`render_to_texture.ts:240`).
4. **Raster `blend-mode` has no RTT isolation fix** — only `terrain-analysis` triggers the drape-time deferral. A raster layer with `multiply` that is first in its RTT stack will be invisible in 3D terrain.
5. terrain-analysis non-normal blend paths feed a **premultiplied** ramp colour into a `mix()` that expects straight colour, so the colour term gets an extra alpha factor (details §2.4). Raster path is correct. Render-test `expected.png`s bake this in.
6. The legacy `ColorReliefStyleLayer` class file is orphaned (kept on disk, nothing imports it); the legacy `slope` layer type is **gone** (no mapping), but its two render tests and example HTML are still in the tree and are stale/invalid.
7. `build/generate-style-code.ts` reads `../maplibre-style-spec/src/reference/v8.json` relative to `build/`, i.e. a nested checkout `maplibre-gl-js/maplibre-style-spec/` which is **not in git and not present** in this checkout (the spec fork lives at the sibling `/home/user/maplibre-style-spec`). `npm run codegen` fails without a symlink/nested clone.

---

## 1. Feature inventory

### 1.1 `terrain-analysis` layer type

Commits: `8a9c9c132` (initial `slope` layer), `9fb0053a0` (runtime spec patch; later removed), `b216fabb9` (unified `terrain-analysis`), `86a654cb4` (step), `5b43f7a3a` (type cleanup), `0b5e7b745` (mercator), `9520e3b90` (manual bilinear), `a73bac829` (prop rename + delete color-relief renderer), `44fae0cdb` (last-stop fix), `04429fe27` (NEAREST).

Files (all new unless noted):
- `src/style/style_layer/terrain_analysis_style_layer.ts` (180 lines)
- `src/render/draw_terrain_analysis.ts` (163)
- `src/render/program/terrain_analysis_program.ts` (94)
- `src/shaders/terrain_analysis.fragment.glsl` (182), `terrain_analysis.vertex.glsl` (pure rename of `color_relief.vertex.glsl`, 0 changes)
- wiring edits: `src/render/painter.ts:28,60,684-685`; `src/shaders/shaders.ts`; `src/render/program/program_uniforms.ts`; `src/style/create_style_layer.ts`; `src/style/style_layer/typed_style_layer.ts`; `src/render/render_to_texture.ts:23-24`
- **deleted**: `src/render/draw_color_relief.ts`, `src/render/program/color_relief_program.ts`, `src/shaders/color_relief.{fragment,vertex}.glsl`

Style-spec (from `/home/user/maplibre-style-spec/src/reference/v8.json`, `paint_terrain-analysis`), exact names/defaults:

| paint property | type | default | notes |
|---|---|---|---|
| `attribute` | enum `elevation` \| `slope` \| `aspect` | `slope` | data-constant, zoom expr, not interpolated |
| `opacity` | number 0..1 | `1` | transition: true |
| `color` | color, `property-type: color-ramp` | none | expression parameters `elevation`, `slope`, `aspect` (three new expression globals in the spec fork) |
| `blend-mode` | enum `normal` \| `multiply` \| `soft-multiply` \| `screen` | `normal` | transition: false |

Layout: only `visibility`. Source must be `raster-dem` (spec `validate_layer.ts:122-128` extended).
Names were originally `terrain-analysis-attribute/-opacity/-color` and renamed to bare `attribute/opacity/color` in `a73bac829` (spec commit `4b7dc8ba`). `blend-mode` was always unprefixed.

Behaviour:
- **Attributes**: `u_attribute` 0=elevation, 1=slope, 2=aspect (`terrain_analysis_program.ts:56-63`). Slope = `atan(length(deriv))` in degrees clamped 0..90; aspect = compass bearing `mod(90 - degrees(atan(dzdy, -dzdx)), 360)`, flat (`length(deriv) < 1e-4`) → 0 (`terrain_analysis.fragment.glsl:104-130`). Derivatives via Horn 3x3 (`:73-102`).
- **Mercator correction** (`0b5e7b745`): both axes divided by `cos(lat)`; `u_latrange` = tile N/S latitude, lat linearly interpolated across the tile (`fragment.glsl:96-99`, `terrain_analysis_program.ts:47-54`). Metres scale: `deriv * tileSize / pow(2.0, 28.2562 - u_zoom)` with `tileSize = u_dimension.x - 2`.
- **Ramp encoding** (`terrain_analysis_style_layer.ts:119-151`): two 1×N RGBA textures. Scalar stops: elevation packed with the DEM unpack vector (`packDEMData`), aspect in R(hi)/G(lo) of `value/360*256`, slope `value/90` in R only (8-bit → 0.35° stop resolution). Scalar texture bound NEAREST on unit 1, colour texture LINEAR on unit 4 (`draw_terrain_analysis.ts:69-72`). Ramp truncated to `MAX_TEXTURE_SIZE` with `warnOnce`.
- **step vs interpolate** (`86a654cb4`, `terrain_analysis_style_layer.ts:70-90`): accepts `ZoomConstantExpression` whose inner is `Interpolate` or `Step`. For `Step`, `labels[0]` (−Infinity) is replaced by `-500` (elevation) or `0` (slope/aspect); sets `isStepMode` → `u_step_mode`. Shader binary-searches the stops, then snaps to texel `l` in step mode or lerps in interpolate mode (`fragment.glsl:132-167`). Anything else (e.g. `interpolate-hcl`? any non-Interpolate/Step root, zoom-dependent expr) falls back to a transparent 2-stop ramp. `44fae0cdb` added "if scalar >= last stop, promote l to r" so the last colour is reachable (`:153-158`).
- **DEM sampling**: `9520e3b90` added manual bilinear on *decoded* elevations (`getElevation`, `fragment.glsl:36-52`, needs `u_dimension`), 4 taps × 9 Horn samples = 36 texture fetches per fragment for slope/aspect. `04429fe27` then switched the DEM texture bind from LINEAR to `gl.NEAREST` (`draw_terrain_analysis.ts:87,90`) so the manual filter sees exact texels (fixes tile-edge artifacts). DEM texture uploaded with `premultiply:false` and reuses `tile.demTexture` (shared with hillshade's slot).
- **Per-frame cost**: no prepare pass; everything recomputed per frame (FORK.md mentions a `PLAN_UNIFIED.md` shared `demPrepare` idea — that file is not in the repo).
- `hasOffscreenPass()` returns true once ramp textures exist (copied from color-relief; no offscreen pass actually exists).

**Legacy types**
- `color-relief`: still in the spec; in GL JS `create_style_layer.ts:24-25` routes it to `createTerrainAnalysisFromColorRelief()` (`terrain_analysis_style_layer.ts:163-180`) which copies `color-relief-opacity`→`opacity`, `color-relief-color`→`color`, forces `attribute:'elevation'`, builds a `TerrainAnalysisStyleLayer` with `_attributeOverride='elevation'` and then sets `styleLayer.type = 'color-relief'`. Consequences / hacks:
  - `color-relief` paint `resampling` (upstream 5.20) is silently dropped (always NEAREST + manual bilinear).
  - The layer's property table is terrain-analysis's, so runtime `setPaintProperty(id,'color-relief-opacity',…)` / `getPaintProperty('color-relief-color')` and `serialize()` no longer speak the color-relief names (serialises `opacity`/`color`/`attribute` under `type:'color-relief'`). Only construction-time translation exists.
  - `src/style/style_layer/color_relief_style_layer.ts` is left in the tree, unreferenced; `color_relief_style_layer.test.ts` was rewritten to test the compat shim.
  - `_opacityProperty`, `_detectExpressionGlobal`, unused `unpackVector` param of `_createScalarRamp` are leftover scaffolding.
- `slope`: initial layer type (`slope-opacity`, `slope-color`) from `8a9c9c132`; fully removed in `b216fabb9` (no alias). `test/integration/render/tests/slope/*` and `test/examples/dem-slope-overlay.html` still use `type:'slope'` → invalid against the current spec.

### 1.2 `blend-mode` on raster and terrain-analysis

Commits: `e40c16308` (raster), `6c61e4074` (terrain-analysis), `a2a902f20` (dst alpha), `7200e4bb4` (soft-multiply v1), `c5606cf2e` (soft-multiply v2). Spec commit `040c485d`.

Property: `blend-mode` (same unprefixed name on `paint_raster` and `paint_terrain-analysis`), enum `normal` (default) | `multiply` | `soft-multiply` | `screen`, data-constant, no transition.

Implementation = fixed-function GL blend + shader output shaping. **No FBO / backdrop copy** in 2D.

GL state plumbing (`e40c16308`):
- `src/gl/types.ts:3` `BlendFuncType` widened to 2- or 4-tuple.
- `src/gl/value.ts:238-256` `BlendFunc.set` now always calls `gl.blendFuncSeparate(v0, v1, srcA, dstA)` (2-tuples reuse RGB factors); default `[ONE, ZERO, ONE, ZERO]`. `state.test.ts` adjusted.
- `src/gl/color_mode.ts:49-59`:

```ts
ColorMode.Replace = [ONE, ZERO, ONE, ZERO];
ColorMode.alphaBlended      = [ONE, ONE_MINUS_SRC_ALPHA, ONE, ONE_MINUS_SRC_ALPHA]
ColorMode.multiply          = [DST_COLOR, ZERO,                ZERO, ONE]
ColorMode.softMultiply      = [DST_COLOR, SRC_COLOR,           ZERO, ONE]   // DEAD since c5606cf2e
ColorMode.screen            = [ONE,       ONE_MINUS_SRC_COLOR, ZERO, ONE]
ColorMode.multiplyDrape     = [DST_COLOR, ONE_MINUS_SRC_ALPHA, ZERO, ONE]
ColorMode.softMultiplyDrape = [DST_COLOR, SRC_COLOR,           ZERO, ONE]   // DEAD since c5606cf2e
ColorMode.screenDrape       = [ONE,       ONE_MINUS_SRC_COLOR, ZERO, ONE]   // identical to screen
```
  (`SRC_ALPHA` constant at `:10` is unused.) No `blendEquation` changes — always FUNC_ADD.
- **Destination-alpha preservation** (`a2a902f20`): alpha factors `(ZERO, ONE)` on all non-normal modes ⇒ `A_out = A_dst`. Originally the 2-tuple reused colour factors for alpha.

Shader side — two new uniforms on both programs, `u_is_premultiplied` and `u_blend_neutral` (`raster_program.ts`, `terrain_analysis_program.ts`):

`src/render/draw_raster.ts:163-193` and `src/render/draw_terrain_analysis.ts:115-163`:

| blend-mode | ColorMode | isPremultiplied | blendNeutral |
|---|---|---|---|
| normal | `painter.colorModeForRenderPass()` | 1 | 0 |
| multiply | `ColorMode.multiply` | 0 | 1 |
| soft-multiply | `ColorMode.multiply` | 0 | **2.0 (sentinel)** |
| screen | `ColorMode.screen` | 0 | 0 |

`src/shaders/raster.fragment.glsl:50-58` (terrain-analysis is the same shape at `terrain_analysis.fragment.glsl:168-177`):

```glsl
vec3 blendedRgb = mix(u_high_vec, u_low_vec, rgb);
if (u_is_premultiplied > 0.5) {
    fragColor = vec4(blendedRgb * color.a, color.a);
} else {
    // u_blend_neutral > 1.5 signals soft-multiply: use α² curve with neutral=1.0
    float neutral = min(u_blend_neutral, 1.0);
    float effective_alpha = u_blend_neutral > 1.5 ? color.a * color.a : color.a;
    fragColor = vec4(mix(vec3(neutral), blendedRgb, effective_alpha), color.a);
}
```

Resulting maths (C = straight source colour, α = source alpha × opacity, D = destination):
- multiply: src = `1 − α + αC`; `D_out = D · (1 − α + αC)` — W3C multiply with source-over alpha, assuming opaque backdrop.
- screen: src = `αC`; `D_out = αC + D(1 − αC)`.
- soft-multiply: see §2.
- In all three `A_out = A_dst`.

In `draw_raster.ts:156` the per-tile draw uses `blendModeState.colorMode ?? colorMode`. Raster cross-fade: both parent/child fade draws use the multiply blend, so during fades tiles are multiplied twice (not addressed).

### 1.3 3D-terrain render-to-texture changes

Commits: `513374ff2`, `35a606f2a`, `89524eb8e` (release commit carrying real code), `adf072f38` (release commit carrying real code), `7200e4bb4`, `c5606cf2e`.

Problem: RTT groups consecutive drapable layers into a stack rendered into an FBO cleared to (0,0,0,0). If a non-RTT layer (symbol/circle…) precedes the blended layer, it starts a fresh stack: multiply against 0 = invisible; screen against 0 = identity (effect lost).

Fix = "defer blend to drape time" for a stack whose **first** layer is a blended terrain-analysis layer:

`src/render/render_to_texture.ts`
- `:23-24` `'terrain-analysis': true` added to `LAYERS_TO_TEXTURES` (`513374ff2`).
- `:53` `_stacks: Array<{layers: Array<string>; blendMode?: string}>` (was `Array<Array<string>>`).
- `:153-172`:
```ts
if (!this._prevType || !LAYERS_TO_TEXTURES[this._prevType]) this._stacks.push({layers: []});
...
if (type === 'terrain-analysis') {
    const blendMode = (layer as any).paint.get('blend-mode');
    if ((blendMode === 'multiply' || blendMode === 'soft-multiply' || blendMode === 'screen') && currentStack.layers.length === 1) {
        currentStack.blendMode = blendMode;
    }
}
```
- `:180-186` options object is now built per stack: `{...renderOptions, isRenderingToTexture: true, rttDeferBlend: !!stackInfo?.blendMode}`.
- `:190,221` `drawTerrain(painter, terrain, rttTiles, options, drapeColorMode)`.
- `:237-243`:
```ts
private _getDrapeColorMode(stackInfo) {
    if (!stackInfo?.blendMode) return undefined;
    if (stackInfo.blendMode === 'multiply') return ColorMode.multiplyDrape;
    if (stackInfo.blendMode === 'soft-multiply') return ColorMode.multiplyDrape;   // soft == multiply here
    if (stackInfo.blendMode === 'screen') return ColorMode.screenDrape;
    return undefined;
}
```

`src/render/draw_terrain.ts:72,77`: `drawTerrain(..., drapeColorMode?)`; `const colorMode = drapeColorMode ?? painter.colorModeForRenderPass();`

`src/render/painter.ts:81-88`: `RenderOptions.rttDeferBlend?: boolean`.

`src/render/draw_terrain_analysis.ts:24-25,126-132`: when `rttDeferBlend` and mode is multiply/soft-multiply/screen, draw into the FBO with normal premultiplied alpha blending (`isPremultiplied: 1`), and let the drape apply the blend. `multiplyDrape` on a premultiplied texture gives `D·(αC) + D·(1−α) = D(1−α+αC)` (correct multiply); `screenDrape` gives `αC + D(1−αC)`.

Limitations / hacks of this design:
- The drape ColorMode applies to the **whole stack**: any later drapable layers in the same stack (e.g. a line layer after the terrain-analysis layer) get multiplied/screened too.
- Only the first layer in the stack is considered; a second blended terrain-analysis layer in the same stack blends inside the FBO against whatever is there (including transparent areas).
- Only `terrain-analysis` is handled; `draw_raster.ts` ignores `rttDeferBlend` (raster multiply as first-in-stack is invisible in 3D).
- Same-stack case (e.g. `background` + terrain-analysis in one stack) blends inside the FBO against the stack content only, not against layers in previous stacks.
- `(layer as any).paint.get(...)` cast; `blendMode` typed as `string`.
- Globe / non-terrain path unaffected.

### 1.4 Build / packaging

Commits: `9fb0053a0`, `0e4547a8c`, `759cfea0b`, `00c6f5301`, plus version bumps in `45f9d7c76`, `89524eb8e`, `adf072f38`, `6c6d2be63`, `44fae0cdb`.

- `package.json`: name `@eslopemap/maplibre-gl`, version `5.21.9`, `publishConfig.access: public`, and **two** spec deps:
  ```json
  "@eslopemap/maplibre-gl-style-spec": "file:maplibre-style-spec",
  "@maplibre/maplibre-gl-style-spec": "npm:@eslopemap/maplibre-gl-style-spec@24.7.1",
  ```
  The second is an npm alias so every `import '@maplibre/maplibre-gl-style-spec'` resolves to the fork. The first (`file:maplibre-style-spec`, added in `00c6f5301`) points at a nested directory that does not exist in git — looks like a leftover of local dev and would break a clean `npm install`; it also means published consumers get a `file:` dep. History: `file:./maplibre-style-spec/…-24.7.0.tgz` (`0e4547a8c`) → npm alias (`759cfea0b`) → alias + file dep (`00c6f5301`).
- `build/generate-style-code.ts:4-11`: replaces `import {v8} from '@maplibre/maplibre-gl-style-spec'` with `JSON.parse(fs.readFileSync(path.join(__dirname, '../maplibre-style-spec/src/reference/v8.json')))` (comment still says "to support slope layer type"). Unnecessary once the aliased package is installed; upstream v6.18.0 uses `import {latest, …}` and has a rewritten generator.
- Runtime style-spec patches: `src/style/slope_spec_patch.ts` and a `validateStyle` wrapper were added in `9fb0053a0` and **removed** in `b216fabb9`. Net residue: only a cosmetic paren change in `src/style/validate_style.ts:28` and a cast tweak in `src/style/style_layer.ts:132`. No runtime patching remains.
- `test/build/bundle_size.json`: 1048557 → 1055330.
- `test/build/sourcemaps.test.ts:85,87`: tolerances loosened `<5` → `<10` and `<16` → `<25` (to tolerate the aliased/nested style-spec paths in the sourcemap).
- `dist/maplibre-gl.js` is committed (only that file; +59 lines vs base, built at `44fae0cdb`), `package-lock.json` +1072 lines. Both should be regenerated, not rebased.
- `*.g.ts` generated files are gitignored (`.gitignore:31`), so `terrain_analysis_style_layer_properties.g.ts` must come from codegen against the forked spec.
- Fork tags: `v5.21.1, v5.21.5, v5.21.6, v5.21.7, v5.21.9`.
- Docs: `FORK.md` (159 lines, added by `7aa8e441a`), `PUBLISH.md` (112, `e8abcbfd8`; paths in it predate the move of demos into `local_demos/`), CHANGELOG entries for 5.21.3 / 5.21.5 / 5.21.6 + a `main` line for soft-multiply.

---

## 2. soft-multiply — exact maths

### 2.1 v1 (commit `7200e4bb4`, shipped in 5.21.7–5.21.9 and in the committed `dist/`), "Modulate 2X"
- GL: `blendFuncSeparate(DST_COLOR, SRC_COLOR, ZERO, ONE)` ⇒ `D_out = S·D + D·S = 2·S·D`.
- Shader: `S = mix(0.5, C, α)` (`blendNeutral = 0.5`).
- Net: `D_out = D · (1 − α + 2αC)`, clamped to 1. Mid-grey (C=0.5) is neutral, C<0.5 darkens, C>0.5 **brightens** up to 2× (so it is really a linear "overlay/modulate", not a multiply).
- RTT isolated stack: `softMultiplyDrape` = same factors applied to a premultiplied FBO texture `(αC, α)` ⇒ `2·αC·D` — transparent pixels → black. This was wrong, which is presumably part of why v2 happened.

### 2.2 v2 (commit `c5606cf2e`, HEAD, unreleased per committed dist)
- GL: `ColorMode.multiply` = `blendFuncSeparate(DST_COLOR, ZERO, ZERO, ONE)`.
- Shader (sentinel `u_blend_neutral = 2.0` → neutral clamps to 1, α squared):
  `S = mix(1, C, α²)`, alpha out = α (ignored by blend).
- Net (raster, straight C): **`D_out = D · (1 − α² + α²·C)`**, `A_out = A_dst`.
  I.e. ordinary multiply with the opacity curve `α → α²`. At α=1 identical to multiply; at α=0.5 it has the strength of multiply at 0.25. It never brightens.
- RTT isolated stack: falls back to `multiplyDrape` with un-squared α ⇒ plain multiply `D(1 − α + αC)` — inconsistent with 2D.
- No test covers it (no render test, no unit test; the shader unit test file `terrain_analysis.fragment.test.ts` added in `759cfea0b` was deleted in `7200e4bb4`).

### 2.3 Portability to upstream v6.18.0
Upstream enum for `raster-layer-blend` / `{fill,line}-layer-blend` (from `src/webgl/draw/draw_layer_composite.ts` @ v6.18.0): `normal`, `multiply`, `overlay`, `color`, `luminosity` (shader-evaluated with a backdrop copy) and `plus-lighter`, `screen` (fixed-function). No `soft-multiply`.
- v2 soft-multiply with a uniform layer alpha (opaque texels) is exactly upstream `multiply` at `raster-opacity' = raster-opacity²` — can be done in the application/style with no engine change. For textures with per-pixel alpha it is not exactly equivalent (texel alpha is squared too).
- v1 (Modulate 2X) has no upstream equivalent; closest is `overlay` (which is conditional on backdrop, not source) — not equivalent.

### 2.4 Latent premultiplication inconsistency in terrain-analysis blend paths
`colorTexture` is created with the default `Texture` option (premultiply = true, `terrain_analysis_style_layer.ts:148`), and the shader does `rampColor = u_opacity * texture(u_color_stops, …)` (`fragment.glsl:168`), so `rampColor.rgb = C·a_c·op` and `rampColor.a = a_c·op` (premultiplied). The non-premultiplied branch then does `mix(neutral, rampColor.rgb, effective_alpha)`:
- multiply: `1 − α + α·(Cα)` (for opaque ramp colours: `1 − op + C·op²`, vs the intended `1 − op + C·op`). E.g. a white ramp colour at opacity 0.7 darkens the backdrop to 0.79.
- screen: source `C·op²` instead of `C·op`.
- soft-multiply v2: `1 − op² + C·op³`.
- RTT-deferred path uses the premultiplied branch and the drape modes, giving the *correct* `D(1 − α + αC)` — so isolated-stack 3D output differs from 2D / same-stack output at opacity < 1.
The raster shader un-premultiplies first (`raster.fragment.glsl:23-28`), so raster is correct. (Derived by reading code, not by running it; `expected.png` for `terrain-analysis/multiply` at opacity 0.7 and `screen` at 0.8 bake the current behaviour in.)

---

## 3. Test inventory

### 3.1 Unit tests
- `src/gl/color_mode.test.ts` (new, 42 lines; `a2a902f20`, `35a606f2a`): asserts blend factor tuples of `multiply` `[DST_COLOR, ZERO, ZERO, ONE]`, `screen` `[ONE, ONE_MINUS_SRC_COLOR, ZERO, ONE]`, `multiplyDrape` `[DST_COLOR, ONE_MINUS_SRC_ALPHA, ZERO, ONE]` + full mask. One test's title/comment ("screen works against cleared FBO, no drape override needed") is stale after `adf072f38`. Nothing on softMultiply/screenDrape.
- `src/gl/state.test.ts` (1-line change): BlendFunc value test uses a 4-tuple.
- `src/render/render_to_texture.test.ts` (+119): (a) fill + terrain-analysis(multiply) in one stack → 1 stack, `blendMode` undefined; (b) same with screen → undefined (comment stale); (c) normal → undefined; (d) fill, circle, terrain-analysis(multiply) → 2 stacks, second has `blendMode: 'multiply'`; (e-g) `_getDrapeColorMode` → `multiplyDrape` / `screenDrape` / undefined. No soft-multiply case. Uses `rtt._stacks[i].layers`.
- `src/style/style_layer/terrain_analysis_style_layer.test.ts` (new, 257): interpolate ramp `[0,30,45]`, `isStepMode` false; step ramps (slope `[0,15,30,45]`, elevation min `-500`, aspect min `0`), discrete colours; default transparent ramp `[0,1]`; step ramp remapped to `maxLength`; a **JS re-implementation** of the shader binary search (`shaderBinarySearchIndex`) asserting last-stop reachability in step/interpolate modes (does not exercise GLSL); `blend-mode` paint property undefined by default, accepts `multiply`/`screen`, coexists with `opacity`. No `soft-multiply`, no `attribute` default test.
- `src/style/style_layer/color_relief_style_layer.test.ts` (rewritten, 5 tests): `createStyleLayer({type:'color-relief'})` returns a `TerrainAnalysisStyleLayer` with `type === 'color-relief'`, attribute `elevation`, opacity translation, ramp creation/remapping.
- `src/style/style_layer/raster_style_layer.test.ts` (+2 tests): `blend-mode` undefined by default; accepts `multiply`, `screen`.
- Deleted along the way: `src/shaders/terrain_analysis.fragment.test.ts` (added `759cfea0b`, removed `7200e4bb4`) — FORK.md still claims it exists.
- No unit test for `draw_raster` / `draw_terrain_analysis` blend state selection, none for `BlendFunc.set` separate-alpha caching.

### 3.2 Render tests (`test/integration/render/tests/`), all 256×256
`slope/` (from `8a9c9c132`, **stale: `type: "slope"`, `slope-color` over `["elevation"]`**):
- `slope/opacity` — black background + slope layer, `slope-opacity` 0.5, green→yellow→red→dark-red ramp 0–90, z11 Grand Canyon tiles.
- `slope/rainbow` — slope layer only, opacity 1, 6-stop ramp.

`terrain-analysis/` (z11 `local://tiles/{z}-{x}-{y}.terrain.png` for 2D; z13 pitch 60 `tiles/terrain-shading/…` with `terrain: {source: dem, exaggeration: 2}` for 3D):
- `rainbow` — elevation attribute, opacity 1, 7-stop interpolate ramp 400–2000 m, no background.
- `opacity` — same ramp, opacity 0.5 over black background.
- `multiply` — same ramp, opacity 0.7, `blend-mode: multiply` over white background (2D).
- `screen` — elevation ramp, opacity 0.8, `blend-mode: screen` over `#333` background (2D, z12).
- `slope-edges` — `attribute: slope`, interpolate over `["slope"]` 0–60°, opacity 1; regression for tile-edge artifacts (NEAREST fix). The only render test of the slope attribute.
- `3d-terrain-basic` — 3D, `#888` background + elevation ramp opacity 0.8, normal blend (baseline).
- `3d-terrain-dark-bg` — same with `#333` background (baseline for screen comparisons).
- `multiply-3d-terrain` — 3D, white background + multiply in the **same** RTT stack (in-FBO blend).
- `multiply-3d-stack-isolated` — 3D, white background, empty-geojson `symbol` layer "stack-break", then multiply layer alone in its stack (drape-time `multiplyDrape`).
- `screen-3d-terrain` — 3D, `#333` background + screen, same stack.
- `screen-3d-stack-isolated` — 3D, `#333` background, symbol stack-break, screen layer isolated (`screenDrape`).

Gaps: no render test for `aspect`, `step` ramps, `soft-multiply` (2D or 3D), raster `blend-mode` (any mode, 2D or 3D), `color-relief` compat beyond the pre-existing upstream `color-relief/*` tests (which now run through the terrain-analysis shader), globe projection.

### 3.3 Examples / demos
- `test/examples/dem-slope-overlay.html` (`8a9c9c132`) — stale, uses `type:'slope'`, `slope-opacity/-color`.
- `local_demos/slope-builtin-published.html` — loads `unpkg.com/@eslopemap/maplibre-gl@5.21.9`; hillshade + one `terrain-analysis` layer switching slope/aspect/elevation with a `blend-mode` selector; uses current prop names.
- `local_demos/slope-blend-fbo-demo.html` — 3D RTT stack isolation debugging demo (toggle a symbol layer to split stacks); **stale**: still uses `terrain-analysis-attribute/-opacity/-color`.

---

## 4. Hacks, known bugs, TODOs

From FORK.md / commit messages:
- Style-spec dependency drift: stale artifacts picked by root install, symlink trouble with Rollup/TS, vendoring rebuilt spec into `node_modules` as the "working short-term fix" — explicitly called not durable. (FORK.md "Main issues" §1; numbering there skips §2.)
- `9fb0053a0`: "Temporary workarounds until style-spec is updated" — the `generate-style-code.ts` local-JSON read is the surviving part.
- `c5606cf2e "attempt at better soft-multiply"`: **experimental/unfinished** — one-line message, no tests, no docs/CHANGELOG update, sentinel-value hack (`blendNeutral 2.0`), leaves `softMultiply`/`softMultiplyDrape` dead, drape path degrades to multiply, not in the committed dist.
- FORK.md commit table stops at `45f9d7c`; describes soft-multiply as Modulate 2X; references a shader unit test and `PLAN_UNIFIED.md` that no longer / do not exist.
- CHANGELOG has no entries for 5.21.4, 5.21.7 (only a `main` bullet), 5.21.8, 5.21.9.

Found in code:
- §2.4 premultiplied-colour inconsistency in terrain-analysis blend branches.
- Raster blend-mode not RTT-safe (§1.3); raster cross-fade double-multiply.
- Stack-level drape blend affects every layer in the stack (§1.3).
- `screenDrape` ≡ `screen`; `softMultiplyDrape` ≡ `softMultiply`; unused `SRC_ALPHA` const.
- `blendFuncSeparate` is now used for every draw in the engine (global change to `BlendFunc`), including custom layers' state restore.
- color-relief compat is construction-time only (§1.1); `resampling` dropped; orphaned `color_relief_style_layer.ts`.
- Step-mode default lower bound hard-coded (`-500` m elevation) — elevations below −500 m still get the first colour by clamp, fine, but the constant is arbitrary; slope stop precision is 8-bit (90/255°).
- Ramp expression must be exactly `interpolate` or `step` at the root, zoom-constant; otherwise silently transparent.
- 36 texture fetches/fragment for slope/aspect, recomputed every frame.
- `hasOffscreenPass()` true without an offscreen pass.
- Stale tests/demos: `slope/*` render tests, `dem-slope-overlay.html`, `slope-blend-fbo-demo.html`; stale comments in `color_mode.test.ts` and `render_to_texture.test.ts`.
- `package.json` `file:maplibre-style-spec` dependency on a non-existent path; `generate-style-code.ts` same.

---

## 5. Commit classification and proposed re-port series

| # | commit | kind | disposition |
|---|---|---|---|
| 1 | `8a9c9c132` slope layer | feature, superseded | squash into TA; drop `slope/*` tests + example |
| 2 | `9fb0053a0` runtime spec patches | temp hack, mostly reverted by #3 | drop |
| 3 | `b216fabb9` unified terrain-analysis | feature | core of commit A |
| 4 | `86a654cb4` step expressions | feature | squash into A (or A2) |
| 5 | `5b43f7a3a` type cleanup | fixup | squash |
| 6 | `0b5e7b745` mercator both axes | fix | squash into A |
| 7 | `0e4547a8c` publish as @eslopemap | chore/packaging | redo as one packaging commit |
| 8 | `e40c16308` raster blend-mode | feature | **drop in favour of upstream `raster-layer-blend`** (except soft-multiply) |
| 9 | `6c61e4074` TA blend-mode | feature | re-port on upstream mechanism |
| 10 | `513374ff2` RTT: terrain-analysis drapable | fix (1 line) | squash into A |
| 11 | `a2a902f20` preserve dst alpha | fix | squash into blend commit / drop if upstream ColorModes used |
| 12 | `35a606f2a` RTT multiply drape | fix | RTT commit (re-evaluate vs upstream composite) |
| 13 | `759cfea0b` render tests + premult fix + version + dep | mixed | split: tests → A, shader fix → A, version/dep → packaging |
| 14 | `45f9d7c76` bump 5.21.4 + demos | chore | demos → demo commit; bump dropped |
| 15 | `e8abcbfd8` PUBLISH.md | docs | docs commit |
| 16 | `89524eb8e` "release 5.21.5" | **release commit containing real code** (`rttDeferMultiply`, painter option, 3 render tests) | code → RTT commit; bump/dist dropped |
| 17 | `adf072f38` "release 5.21.6" | **release commit containing real code** (`screenDrape`, rename to `rttDeferBlend`, 4 render tests) | code → RTT commit; bump/dist dropped |
| 18 | `7aa8e441a` "(split: selected paths)" | FORK.md only | docs commit |
| 19 | `7200e4bb4` soft-multiply v1 "(split remainder)" | feature, superseded by #25 | squash with #25 |
| 20 | `6c6d2be63` release 5.21.7 | pure chore (version, demo URL, dist) | drop |
| 21 | `9520e3b90` manual bilinear | fix (also touched since-deleted color_relief shader) | squash into A |
| 22 | `00c6f5301` "(split: selected paths)" | package.json only (5.21.8 + `file:` dep) | drop / packaging |
| 23 | `a73bac829` rename props + delete color-relief renderer "(split remainder)" | refactor | squash into A (never introduce prefixed names) |
| 24 | `44fae0cdb` last-stop fix + bump 5.21.9 | fix + chore | fix → A; bump dropped |
| 25 | `c5606cf2e` attempt at better soft-multiply | experimental | soft-multiply commit, after deciding v1 vs v2 |
| 26 | `04429fe27` NEAREST DEM + test fixes | fix | squash into A; test fixes into RTT commit |

"split" pairs: (#18,#19) and (#22,#23) are each one logical change split by path; #18 and #22 are the trivial halves.
Pure release/chore: #7, #14, #20, #22, and the version/dist/lock parts of #13, #16, #17, #24. `dist/maplibre-gl.js` and `package-lock.json` changes should never be replayed.

### Proposed clean series on top of v6.18.0 (note upstream moved `src/gl` + `src/render/draw_*` + programs to `src/webgl/…`, shaders to `src/shaders/glsl/`, and uses `.ts` import suffixes)

1. **build: consume forked style-spec** — package name/`publishConfig`, single alias dep to a spec fork rebased on spec ^26.6 (no `file:` dep, no `generate-style-code.ts` hack), bundle-size + sourcemap tolerance updates last.
2. **feat: `terrain-analysis` layer (elevation/slope/aspect)** — style layer, program, shaders (Horn, mercator both axes, manual bilinear + NEAREST DEM, last-stop fix, premultiplied output fix), interpolate + step ramps, painter/shaders/uniform wiring, `LAYERS_TO_TEXTURES`, unit tests, render tests `rainbow`, `opacity`, `slope-edges`, `3d-terrain-basic`, `3d-terrain-dark-bg` (+ add aspect/step tests). Squashes #1,3,4,5,6,10,13(part),21,23,24,26.
3. **feat: `color-relief` handling** — decision point: either keep upstream's native color-relief untouched (recommended; zero diff, keeps `resampling`) or re-add the compat shim. If the shim is dropped this commit disappears and the 4 deleted upstream files stay.
4. **feat: layer blend for terrain-analysis** — expose upstream-style property (e.g. `terrain-analysis-layer-blend` or keep `blend-mode` as alias) wired through upstream `drawLayerComposite`; raster `blend-mode` code (#8, #11) dropped in favour of `raster-layer-blend`. Tests: `multiply`, `screen`.
5. **fix/feat: blend in 3D terrain RTT** — only if upstream's composite path does not already handle isolated stacks; otherwise just port the four `*-3d-*` render tests as regression tests. Squashes #12,16,17 code.
6. **feat: `soft-multiply`** — the only blend with no upstream equivalent; one isolated commit (enum value + shader define in `layer_composite.fragment.glsl`, or documented as `multiply` with squared opacity), with a render test. Squashes #19,#25.
7. **docs/demos** — FORK.md (rewritten to match), PUBLISH.md, `local_demos/*` (fix stale prop names), CHANGELOG. Drop `test/examples/dem-slope-overlay.html` and `slope/*` render tests.
