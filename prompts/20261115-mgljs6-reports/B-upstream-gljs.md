# Upstream maplibre-gl-js 0cac8ee50 (~v5.21) -> v6.18.0: findings for the `blend` fork rebase

Repo `/home/user/maplibre-gl-js`, all refs read with `git show v6.18.0:<path>` (no working-tree changes).
1155 commits between merge-base `0cac8ee50` and `v6.18.0` (`7ff68855d`, 2026-11-11).
Style-spec facts read from `/home/user/maplibre-style-spec` (remote `upstream`, tags v26.5.0..v26.8.0).

---------------------------------------------------------------------------------------------------

## 1. Upstream layer-blend feature

### 1.1 Commits

| commit | date | what |
|---|---|---|
| `64482eb1b` (#7570) | 6.0.0 | `{line,fill}-layer-opacity`: introduced the scratch-FBO-then-composite mechanism (`draw_layer_opacity.ts`, `layerOpacity` program) |
| `b9b42711c` (#8073) | 2026-11-09 | `{line,fill}-layer-blend`; renames `draw_layer_opacity.ts` -> `draw_layer_composite.ts`, `layerOpacity` program -> `layerComposite`, `painter.layerOpacityFbo` -> `painter.layerCompositeFbo`, adds `painter.layerCompositeBackdrop`, `ColorMode.plusLighter`, `ColorMode.screen` |
| `fd6830726` (#8986) | 2026-11-10 | `raster-layer-blend` (15 lines in `draw_raster.ts`, 9 in `draw_layer_composite.ts`, 1 in `painter.ts`) |
| `6ce65922d` (#9006) | 2026-11-11 | `gl.flush()` before clearing the scratch FBO when it was already sampled this frame (Mesa 24.3-25.3.1 / Intel Gen12, fixes #9003) |

No follow-ups: `git log v6.18.0..upstream/main` (16 commits to `23a5cbd47`) contains nothing touching blend.
All three shipped in **6.18.0** (CHANGELOG lines 12, 13, 20). Commit bodies of #8073/#8986 are empty; no limitations
text in `docs/`. Only example: `test/examples/interactive-layer-blending.html`.

### 1.2 Style-spec definition (style-spec v26.5.0 adds fill/line, v26.6.0 adds raster; gl-js 6.18.0 depends on `^26.6.0`, lockfile 26.6.0)

- Property names: `fill-layer-blend`, `line-layer-blend`, `raster-layer-blend`.
- Group: **paint** (`paint_fill`, `paint_line`, `paint_raster`).
- Type `enum`, default `"normal"`, `"property-type": "constant"`, `"transition": false`, **no `expression` block**
  => no zoom expressions, no data-driven, no `global-state`, no transitions. Plain constant only.
- Values (7, identical for the three layer types):
  `normal`, `multiply`, `color`, `luminosity`, `plus-lighter`, `screen`, `overlay`
  (all linked to W3C compositing-1 definitions).
- `sdk-support` still points at issues (js #48, native #2963) - not implemented in maplibre-native.
- Related: `fill-layer-opacity`, `line-layer-opacity` (number 0..1, default 1, transition true, zoom + global-state
  expressions, `data-constant`), js 6.0.0. **There is no `raster-layer-opacity`** and no `*-layer-*` property on
  hillshade, color-relief, background, circle, symbol, heatmap, fill-extrusion.

### 1.3 Architecture (file: `src/webgl/draw/draw_layer_composite.ts`, 263 lines)

Hybrid: **one shared offscreen FBO + a fullscreen composite pass**; the composite uses either fixed-function
`blendFunc` or a shader that reads a **copy of the backdrop**.

Call pattern in each draw function (`draw_fill.ts`, `draw_line.ts`, `draw_raster.ts`):

```ts
// src/webgl/draw/draw_raster.ts @ v6.18.0
const layerBlend = layer.paint.get('raster-layer-blend');
if (layerBlend === 'normal') { drawRasterTiles(...); return; }
const results = prepareDrawLayerComposite(painter, tileManager, layer, tileIDs, frameRenderContext);
drawRasterTiles(tileManager, layer, tileIDs, frameRenderContext);
drawLayerComposite(painter, 1, layerBlend, results, layer, frameRenderContext);
```
```ts
// src/webgl/draw/draw_fill.ts
if (layerOpacity < 1 || layerBlend !== 'normal') {
    if (frameRenderContext.currentPass !== 'translucent') return;
    const results = prepareDrawLayerComposite(painter, tileManager, layer, coords, frameRenderContext);
    drawFillAndOutline(...);
    drawLayerComposite(painter, layerOpacity, layerBlend, results, layer, frameRenderContext);
```

`prepareDrawLayerComposite`:
1. remembers the currently bound framebuffer + viewport (`compositeTarget`, `compositeViewport`) - the canvas, or
   the per-terrain-tile RTT FBO;
2. `gl.flush()` if `frameRenderContext.layerCompositeSampled` (#9006);
3. binds `painter.layerCompositeFbo` (single RGBA8 texture, NEAREST, + DEPTH_STENCIL renderbuffer, **not
   multisampled**, lazily created, resized in place to the target size);
4. clears it `{color: transparent, depth: 1, stencil: 0}`, invalidates + re-renders tile clipping masks;
5. computes scissor `bounds`: for fill/line the screen bbox of tiles that have a bucket for the layer
   (`tileManager.getTile(coord)?.getBucket(layer)`), whole viewport when terrain or `useSubdivision` (globe) or a
   corner is behind the camera; **for raster always the whole viewport**.

The layer then draws itself normally (premultiplied alpha, `ColorMode.alphaBlended`) into the scratch FBO.

`drawLayerComposite(painter, opacity, blend, prepareResult, layer, frameRenderContext)`:
- rebinds the target, sets scissor to `bounds`;
- mode table:

```ts
const shaderBlendDefines = {
    multiply:   ['#define LAYER_BLEND;', '#define LAYER_BLEND_MULTIPLY;'],
    overlay:    ['#define LAYER_BLEND;', '#define LAYER_BLEND_OVERLAY;'],
    color:      ['#define LAYER_BLEND;', '#define LAYER_BLEND_NONSEPARABLE;', '#define LAYER_BLEND_COLOR;'],
    luminosity: ['#define LAYER_BLEND;', '#define LAYER_BLEND_NONSEPARABLE;']
};
const fixedFunctionBlends = { 'plus-lighter': ColorMode.plusLighter, screen: ColorMode.screen };
```
  - `normal` (opacity only): `colorModeForRenderPass()` (= `[ONE, ONE_MINUS_SRC_ALPHA]`).
  - `screen`: `ColorMode.screen = [ONE, ONE_MINUS_SRC_COLOR]`; `plus-lighter`: `[ONE, ONE]`. No backdrop copy.
  - `multiply|overlay|color|luminosity`: `copyBackdrop()` does `gl.copyTexSubImage2D` of the scissor box of the
    target into `painter.layerCompositeBackdrop` (grow-only RGBA texture, TEXTURE1; this also resolves MSAA),
    the shader computes the final pixel, drawn with `ColorMode.unblended` (replace).
  - overdraw inspector: forces `plusLighter`.
- draws `painter.viewportBuffer` quad with program `layerComposite`
  (`useProgram('layerComposite', null, true /*forceSimpleProjection*/, defines)`), depth
  `getDepthModeForSublayer(0, ReadOnly)`, stencil disabled;
- sets `frameRenderContext.layerCompositeSampled = true`.

Shader `src/shaders/glsl/layer_composite.fragment.glsl` (88 lines): `texelFetch` both textures by
`gl_FragCoord`, `src = texel * u_opacity`, `if (src.a == 0.0) discard;`, then W3C formulas **in premultiplied
space with proper destination alpha**:

```glsl
// multiply
co = invDstA * src.rgb + invSrcA * dst.rgb + src.rgb * dst.rgb;
...
return vec4(co, src.a + dst.a * invSrcA);
```
Non-separable modes un-premultiply (`cb = dst.a > 0.0 ? dst.rgb / dst.a : vec3(0.0)`), use `lum/clipColor/setLum`.

Other files:
- `src/webgl/program/layer_composite_program.ts` (uniforms `u_image`, `u_backdrop`, `u_backdrop_offset`, `u_opacity`)
- `src/webgl/program/program_uniforms.ts`, `src/shaders/shaders.ts` (`layerComposite` entries)
- `src/webgl/color_mode.ts` (+`ONE_MINUS_SRC_COLOR`, `plusLighter`, `screen`). Note upstream `BlendFuncType` is
  still a 2-tuple with `gl.blendFunc`; the fork's 4-tuple / `blendFuncSeparate` change in `src/gl/value.ts`,
  `src/gl/types.ts` has no upstream counterpart.
- `src/render/painter.ts`: fields `layerCompositeFbo`, `layerCompositeBackdrop` (destroyed in `destroy()`), and

```ts
frameRenderContext.opaquePassCutoff = Infinity;
for (let i = 0; i < layerIds.length; i++) {
    const layer = style._layers[layerIds[i]];
    if (layer.is3D() || layerBlendsInShader(layer)) { frameRenderContext.opaquePassCutoff = i; break; }
}
```
- `src/render/frame_render_context.ts`: `layerCompositeSampled`.

### 1.4 Interactions

- **`*-layer-opacity`**: same pass. `drawLayerComposite(painter, layerOpacity, layerBlend, ...)`; opacity scales the
  premultiplied source before blending. Tests `fill-layer-blend/fill-layer-opacity`, `line-layer-blend/line-layer-opacity`.
  Raster passes opacity `1` (`raster-opacity` is applied per tile inside the scratch FBO).
- **3D terrain (draped layers)**: under terrain `compositeTarget` is the terrain tile's RTT FBO (rttSize x rttSize),
  so the blend happens **inside the RTT stack texture**, against the layers below *in the same render stack*. The
  stack texture is then draped with ordinary alpha blending (`drawTerrain` uses `colorModeForRenderPass()`, no
  blend-mode parameter). Consequences:
  - Works when backdrop and blended layer are in one stack (tests `{fill,line,raster}-layer-blend/terrain`:
    background + hillshade/fill + blended layer).
  - If the blending layer is the **first layer of a stack** (a symbol/circle/heatmap/fill-extrusion/non-draped
    custom layer sits between it and the content below) the backdrop is transparent: `multiply` then degenerates to
    the plain source colour (formula with `dst.a = 0` gives `co = src.rgb`), i.e. behaves as `normal`. Upstream has
    **no drape-time blending** - this is exactly the case the fork solved with `multiplyDrape/screenDrape` +
    `rttDeferBlend` (fork tests `terrain-analysis/multiply-3d-stack-isolated`, `screen-3d-stack-isolated`).
  - Cost is paid only when a terrain tile's stack is re-rendered (RTT textures are cached per tile/stack with
    `RTTFingerprint`), not every frame.
  - Bounds fall back to the whole RTT viewport.
- **Globe**: supported; `getCoordsViewportBounds` returns the whole viewport when `useSubdivision`. Raster's two-pass
  stencil (`stencilConfigForOverlapTwoPass`, which clears stencil) runs inside the scratch FBO, which has its own
  depth-stencil. No dedicated `*-layer-blend/globe` render test (there are `{fill,line}-layer-opacity/globe`).
- **Premultiplied alpha / destination alpha**: everything premultiplied; result alpha is source-over
  (`src.a + dst.a*(1-src.a)`). Over a transparent backdrop the layer shows as normal (tests
  `line-layer-blend/transparent-backdrop`, `fill-layer-blend/translucent-backdrop`). This differs from the fork,
  whose `multiply = [DST_COLOR, ZERO, ZERO, ONE]` preserves destination alpha (multiply over transparent = invisible).
  Over an opaque backdrop the two give the same RGB: `d*(1 - a + c*a)`.
- **Opaque pass / depth**: a shader-blend layer ends the opaque pass at its index (layers at or above it lose the
  opaque-pass optimisation and are drawn painter's-algorithm). Fixed-function modes (`screen`, `plus-lighter`) do not.
  Tests `under-opaque-fill`, `under-opaque-fill-antialiased`.
- **MSAA**: scratch FBO is not multisampled, so a blended/layer-opacity layer is not antialiased by canvas MSAA
  (`antialias: true`); the backdrop copy resolves MSAA. Render test runner got an `antialias` option for this.

### 1.5 Perf cost per blended layer (per frame on a flat map; per re-rendered terrain tile under terrain)

FBO switch x2, full clear of colour+depth+stencil at target size, re-render of tile clipping masks (and
invalidation for the next layer), one fullscreen-quad draw; for shader modes additionally one `copyTexSubImage2D`
of the scissor box (**whole viewport for raster**, and for fill/line under globe/terrain); `gl.flush()` for the
2nd+ composited layer in a frame; loss of the opaque pass for layers above a shader-blend layer. One extra
viewport-sized RGBA texture + depth-stencil renderbuffer, plus one backdrop texture. `screen`/`plus-lighter` are
the cheap modes; `normal` costs nothing (fast path).

### 1.6 Render tests (`test/integration/render/tests/`)

- `fill-layer-blend/`: color, fill-layer-opacity, luminosity, multiply, no-features, overdraw-inspector, overlap,
  overlay, plus-lighter, screen, terrain, translucent-backdrop, under-opaque-fill, under-opaque-fill-antialiased (14)
- `line-layer-blend/`: color, line-layer-opacity, luminosity, multiply, overlap, overlay, plus-lighter, screen,
  terrain, transparent-backdrop (10)
- `raster-layer-blend/`: image-beyond-tile, multiply, terrain, under-opaque-fill-antialiased (4)
- plus `fill-layer-opacity/*` (16) and `line-layer-opacity/*` (17) which exercise the same composite path.

### 1.7 Is it reusable for a custom DEM layer? Yes - it is layer-type-agnostic except for three hard-coded spots.

`raster-layer-blend` (#8986) is the template: 27 lines of src. For `terrain-analysis` (or color-relief/hillshade):

1. **Style spec** (fork of style-spec, rebased to >= 26.6): add `"terrain-analysis-layer-blend"` to
   `paint_terrain-analysis` in `src/reference/v8.json`, copying the `raster-layer-blend` block verbatim
   (`type: enum`, 7 values, `default: normal`, `property-type: constant`, `transition: false`).
   `build/generate-style-code.ts` iterates `Object.keys(latest.layer.type.values)` so
   `terrain_analysis_style_layer_properties.g.ts` is generated automatically.
2. **`src/webgl/draw/draw_layer_composite.ts`** (3 edits):
   - `layerBlendsInShader()`: add
     `if (isTerrainAnalysisStyleLayer(layer)) return isShaderBlend(layer.paint.get('terrain-analysis-layer-blend'));`
     (this is what makes `painter.render` cut the opaque pass - no painter change needed for blending).
   - widen the `layer` parameter union of `prepareDrawLayerComposite` and `drawLayerComposite`
     (`LineStyleLayer | FillStyleLayer | RasterStyleLayer | TerrainAnalysisStyleLayer`).
   - bounds: `const bounds = isRasterStyleLayer(layer) ? [0,0,width,height] : getCoordsViewportBounds(drawnCoords, ...)`
     must also take the whole-viewport branch for the DEM layer. **Gotcha**: DEM tiles have no bucket, so
     `drawnCoords` would be empty, bounds `[0,0,0,0]`, and `drawLayerComposite` returns early - nothing drawn.
3. **`src/webgl/draw/draw_terrain_analysis.ts`**: wrap the tile loop exactly like `drawRaster` (prepare -> draw
   tiles with the normal `colorModeForRenderPass()` -> `drawLayerComposite(painter, 1, layerBlend, results, layer, ctx)`).
   The draw function already receives `painter` (as `drawColorRelief` does).
4. Render tests under `terrain-analysis-layer-blend/`.

Everything fork-specific for blending can then be deleted: `ColorMode.multiply/softMultiply/screen*Drape`, 4-tuple
`BlendFuncType`/`blendFuncSeparate`, `u_is_premultiplied`/`u_blend_neutral` uniforms in raster and terrain-analysis
shaders, `rttDeferBlend`, `_stacks[].blendMode`, `drawTerrain(..., drapeColorMode)`.

Not covered by upstream, would stay fork-only if wanted:
- **`soft-multiply`** (fork's alpha-squared / Modulate-2X mode): no upstream enum value. Adding it = one enum value
  in the spec + one entry in `shaderBlendDefines` + one `#ifdef` branch in `layer_composite.fragment.glsl`.
- **Blend across RTT stacks under terrain** (blend layer first in its stack): needs the fork's drape-time logic
  re-implemented on top of the new `src/webgl/render_to_texture.ts` (stack type is `string[][]`, per-stack release,
  fingerprints) and `drawTerrain` - or accept the limitation / keep layers in one stack.
- Fork property name is `blend-mode` (un-prefixed) with values `normal|multiply|soft-multiply|screen`; upstream
  naming is `<type>-layer-blend`. Styles need migrating (`multiply`, `screen` map 1:1).

---------------------------------------------------------------------------------------------------

## 2. Blend on hillshade / color-relief; slope / aspect upstream

- **No** `hillshade-layer-blend` / `color-relief-layer-blend` in gl-js v6.18.0, upstream/main, or style-spec
  v26.8.0 / upstream/main (`git grep` for `hillshade-layer|color-relief-layer` in `v8.json`, CHANGELOG, docs: no hits).
  `layerBlendsInShader` only knows fill, line, raster. No plan/issue is referenced in CHANGELOG or commit text beyond
  the umbrella issues gl-js #48 and native #2963. The spec doc strings do mention the use case
  ("multiply ... Useful for textures, hillshades", "color ... tinting monochrome content such as a hillshade") -
  hillshade is treated as the *backdrop*, blended over by a fill/raster.
- **No slope / aspect / terrain-analysis layer** upstream. `git grep -iE "slope|aspect|terrain-analysis"` in `src`
  at v6.18.0 only hits hillshade shader internals (`hillshade.fragment.glsl`, `hillshade_prepare.fragment.glsl`),
  `dem_data.ts` comments, globe maths, aspect-ratio uniforms. CHANGELOG 5.21..6.18 has no slope/aspect entry.
  Style-spec upstream/main: no `slope`/`aspect` property or layer type.
- color-relief / hillshade changes upstream in the range are fixes/perf only (see section 3).

---------------------------------------------------------------------------------------------------

## 3. Architectural changes 5.21 -> 6.18 relevant to porting

### 3.1 Big-ticket items (CHANGELOG 6.0.0 section, lines 409-470, and later)

- **Directory move** (`9adb85f5b` #7422 "webgl draw/program isolation", `cb8c8ecfc` #7418):
  `src/gl/*` -> `src/webgl/*`; `src/render/draw_*.ts` -> `src/webgl/draw/draw_*.ts`;
  `src/render/program/*` -> `src/webgl/program/*`; `src/render/program.ts`, `texture.ts`, `uniform_binding.ts`,
  `vertex_array_object.ts` -> `src/webgl/`; `src/render/render_to_texture.ts` -> `src/webgl/render_to_texture.ts`
  (+ `src/render/render_to_texture_interface.ts`, `src/webgl/rtt_fingerprint.ts`);
  `src/shaders/*.glsl` -> `src/shaders/glsl/*.glsl`. `src/gl/render_pool.ts` is gone.
  Every fork file path changes; git rename detection will not carry the fork's new files.
- **Imports need `.ts` extensions** (`allowImportingTsExtensions`, Node native TS for build scripts #7565),
  `isolatedDeclarations: true` (#7566) => exported functions/methods need explicit return types; type-only imports
  must be in a separate `import type` section (lint #8460); `Array<T>` -> `T[]` lint.
- **WebGL1 removed** (`900b2c8da` #7453). `Painter` takes `WebGL2RenderingContext`. Shaders compiled with
  `#version 300 es` unconditionally.
- **Layout qualifiers** (`655467108` #7644): static vertex attributes must be `layout(location = N) in ...`;
  `bindAttribLocation` is gone (`shaders.ts` assigns locations for `#pragma` attributes starting after the static count).
  The fork's `terrain_analysis.vertex.glsl` has `in vec2 a_pos;` -> must become `layout(location = 0) in vec2 a_pos;`.
- **`#pragma mapbox` -> `#pragma maplibre`** (#7761).
- **UBOs** (`df77f517a` #8290 projection, `5c0a6079b` #8306 terrain, `1429ce66a` #8313 per-frame, `694105870` #8304):
  `Program.draw` binds `context.terrainUniformBuffer`, `context.frameUniformBuffer`,
  `bindProjectionUniformBuffer(context, projectionData)`. `Program.draw` signature is unchanged in shape
  (`uniformValues, terrain, projectionData, layerID, ...`), so a layer that only uses `projectTile()` and its own
  fixed uniforms (as color-relief does) needs no UBO work. Do not declare uniforms that now live in the prelude UBOs.
- **`FrameRenderContext`** replaces `RenderOptions` and most painter helpers (`f5d5553e0` #8318, `e3bbcabc3` #8568,
  `96f64557f` #8603, `d86af301d` #8606, `94aab1f3e` #8561, `c2031bfb2` #8545). Mapping for a draw function:

  | 5.21 | 6.18 |
  |---|---|
  | `painter.renderPass` | `frameRenderContext.currentPass` |
  | `renderOptions.isRenderingToTexture` | `frameRenderContext.isRenderingToTexture` (rarely needed) |
  | `painter.style.projection.useSubdivision` | `frameRenderContext.data.useSubdivision` |
  | `painter.getDepthModeForSublayer`, `colorModeForRenderPass`, `useProgram` | same names on `frameRenderContext` |
  | `painter.stencilConfigForOverlapTwoPass`, `getStencilConfigForOverlapAndUpdateStencilID` | on `frameRenderContext` |
  | `projection.getMeshFromTileID(context, ...)` | `frameRenderContext.getMeshFromTileID(canonical, useBorder, true, 'raster')` |
  | `painter.style.map.terrain?.getTerrainData(coord)` | `frameRenderContext.getTerrainDataForTile(coord)` (null in RTT) |
  | `transform.getProjectionData({overscaledTileID, aligned, applyGlobeMatrix: !isRTT, applyTerrainMatrix: true})` | `frameRenderContext.getProjectionDataForTile(coord, {aligned})` |
  | `painter.options.moving` | `frameRenderContext.data.moving` |
  | `painter.transform` | `frameRenderContext.transform` |
  | `painter.context` | `frameRenderContext.context` |

- **Draw-function registry**: `src/webgl/draw/index.ts` exports `DrawFunctions` type + `webglDrawFunctions` table;
  `Painter.renderLayer(painter, tileManager, layer, coords, frameRenderContext)` dispatches via
  `this.drawFunctions.colorRelief(...)` etc. A new layer type needs: an entry in `DrawFunctions` +
  `webglDrawFunctions`, and an `else if (isXStyleLayer(layer)) draw.x(...)` branch in `painter.ts`.
- **DEM border is now 2 px** (`e380b587f` #8302, 2026-09-05): `DEMData.dim = data.height - 4`, `_idx` offset `+2`.
  color-relief vertex shader changed accordingly:

  ```glsl
  -    highp vec2 epsilon = 1.0 / u_dimension;
  -    float scale = (u_dimension.x - 2.0) / u_dimension.x;
  +    highp vec2 epsilon = 2.0 / u_dimension;
  +    float scale = (u_dimension.x - 4.0) / u_dimension.x;
  ```
  The fork's `terrain_analysis.vertex.glsl` is an unmodified copy of the old shader and
  `terrain_analysis.fragment.glsl` hard-codes `float tileSize = u_dimension.x - 2.0;` and 1-texel neighbour offsets:
  **both must be updated or slopes at tile edges/scale will be wrong**. Upside: with a 2 px border a 3x3 Sobel
  kernel plus bilinear sampling now has valid neighbours at tile edges.
  Related: `dc767a362` #8420 (terrain DEM cell centres aligned with hillshade/color-relief), `dda75ad45` #8431
  (`highp` hillshade DEM texcoords), `7a897d0b0` #8551 (hillshade pole gradient removed), `4a10f85f0` #8145
  (`texelFetch` for exact lookups).
- **DEM texture upload** (`23703cb2e` #8209): `tile.needsColorReliefPrepare` flag (set in
  `src/source/raster_dem_tile_source.ts:103`, `src/tile/tile_manager_raster_dem.ts:27`); upload once per tile, not
  every frame. A fork layer should either reuse this flag or add `needsTerrainAnalysisPrepare` in those two places
  + `src/tile/tile.ts`. `Texture` constructor changed (`new Texture(context, {width, height, data: null}, gl.RGBA)`
  then `.update(pixels, {premultiply: false})`), textures are immutable (`texStorage2D`, #7643).
  `gl.getParameter(MAX_TEXTURE_SIZE)` must be cached (GPU stall).
- **Empty DEM tiles**: `if (!dem?.data) continue;` must precede first-tile setup (204 responses).
- **RTT rewrite** (`1454a123f` #7549, #7637 shared FBO, `4b4f0d6f4` #8368, `3b43c0911` #8250, `542e21807` #8400,
  `b4910e6cf` #8328 mipmaps, `a6dd56577` #8782 per-stack invalidation, `84d2534d4` #8588 custom layers on terrain):
  `RenderPool` removed; per-tile cached `RTTObject`s (`tile.getRTT/acquireRTT/releaseRTT/releaseRTTStack`),
  one shared FBO (`painter.bindRTT`), `RTTFingerprint` (tile keys, source revision, zoom, visible layer ids),
  budgeted re-render (`needsFollowUpFrame`), `obj.texture.generateMipmap()` after each stack render,
  `setFrameUniformWorldSize` around RTT. `_stacks` is `string[][]`. `LAYERS_TO_TEXTURES` still exists and still
  lists `background, fill, line, raster, hillshade, 'color-relief'` - the fork's one-line `'terrain-analysis': true`
  ports trivially; the fork's stack-blendMode / drape-colour-mode logic does not (rewrite or drop, see 1.7).
- **Style layer registration**: `src/style/create_style_layer.ts` is now a typed `switch` returning `AnyStyleLayer`
  (union type to extend) with constructor `(layer, globalState)`; `src/style/style_layer/typed_style_layer.ts` union
  unchanged in shape. `hasOffscreenPass(): boolean` etc. need return types.
- **`build/generate-style-code.ts`**: rewritten (368 lines changed), imports `latest` (not `v8`) from the style-spec,
  typed by `StylePropertySpecification['property-type']`; still driven by the spec's `layer.type.values`, so a new
  layer type only needs the spec. A new `property-type` would be a compile error.
- **Style-spec 24.7 -> 26.6**: v25 makes legacy-expression problems validation errors (warning severity);
  the fork's style-spec fork (`@eslopemap/maplibre-gl-style-spec` 24.7.1, HEAD `b19f5ebd`) must be rebased onto
  >= v26.6.0 (v26.8.0 is latest tag) to get the `*-layer-blend` definitions.
- **Tests**: render runner is now `test/integration/render/render.test.ts` (vitest 5; old `run_render_tests.ts`
  deleted), run with `npm run test-render`; new test-data option `antialias`. `test/build/bundle_size.json` deleted
  (`b11c70538` #8905, CI reports size instead) - the fork's edit there just drops. `es2020-compat.test.ts` ->
  `es2022-compat.test.ts`; new `esm.test.ts`, `examples.test.ts`, `node-version.test.ts`. Typecheck is `tsgo`
  (`@typescript/native-preview`), TypeScript ^6.0.3, eslint 10, `eslint-plugin-import-x`.

### 3.2 Per-file diff summary (`git diff -M 0cac8ee50 v6.18.0`)

```
 src/render/painter.ts                                 | 837 +++----  (heavy rewrite)
 src/render/render_to_texture.ts                       | 207 ---    (moved to src/webgl/, rewritten)
 src/shaders/{ => glsl}/color_relief.fragment.glsl     |   3 +-
 src/shaders/{ => glsl}/color_relief.vertex.glsl       |   6 +-
 src/shaders/{ => glsl}/raster.fragment.glsl           |   8 +-
 src/style/style_layer/color_relief_style_layer.ts     |  25 +-
 src/{gl => webgl}/color_mode.ts                       |   8 +-
 src/{render => webgl/draw}/draw_color_relief.ts       | 105 +-
 src/{render => webgl/draw}/draw_raster.ts             | 101 +-
 src/{render => webgl}/program/color_relief_program.ts |  10 +-
```

- `draw_color_relief.ts`: `RenderOptions` -> `FrameRenderContext` (table above); upload DEM texture only when
  `!tile.demTexture || tile.needsColorReliefPrepare`; cached `MAX_TEXTURE_SIZE`; empty-DEM check moved first.
  No blending change (still `colorModeForRenderPass()`).
- `color_relief.fragment.glsl`: `getElevationStop` uses `texelFetch(u_elevation_stops, ivec2(stop, 0), 0)`.
  (`d274f3db3` #8554 fixed zero-factor custom encodings in the style layer / `packDEMData`.)
- `color_relief.vertex.glsl`: `layout(location = 0)`, 2 px border maths.
- `color_relief_program.ts`: import paths only.
- `color_relief_style_layer.ts`: imports (`../../webgl/texture.ts`, `.ts` extensions), `T[]` types, return types;
  API (`getColorRampTextures(context, maxLength, unpackVector)`) unchanged.
- `render_to_texture.ts`: see RTT rewrite above.
- `draw_raster.ts`: `FrameRenderContext`; `drawRaster(painter, tileManager, layer, tileIDs, ctx)` now wraps
  `drawRasterTiles` with the composite for `raster-layer-blend`; projective image-source quads (`88672376a` #7887);
  `RasterTileSource#setPremultiplyAlpha(false)` (#7235).
- `raster.fragment.glsl`: `v_pos0/v_pos1` are `vec3`, sampled as `v_pos0.xy / v_pos0.z` (projective). No blend
  uniforms - the fork's `u_is_premultiplied`/`u_blend_neutral` additions conflict textually and should be dropped.
- `color_mode.ts`: +`ONE_MINUS_SRC_COLOR`, `plusLighter`, `screen` (2-tuple blend funcs). Fork's version conflicts
  on `screen` (fork: `[ONE, ONE_MINUS_SRC_COLOR, ZERO, ONE]`).
- `painter.ts`: no `RenderOptions`/`renderPass`/`style`/`transform` state on the painter; `render(style, transform,
  data: FrameRenderData)` builds a `FrameRenderContext`; `drawFunctions` table; RTT pool + shared FBO
  (`acquireRTT/releaseRTT/bindRTT/clearRTTPool`); `layerCompositeFbo`/`layerCompositeBackdrop`; opaque cutoff also on
  `layerBlendsInShader`; `renderToTexture: IRenderToTexture` (constructed in `src/ui/map.ts:3094`).

Fork diff (`git diff 0cac8ee50 HEAD --stat`: 67 files, +4133/-366) for reference: the fork **deleted**
`draw_color_relief.ts`, `color_relief_program.ts`, `color_relief.fragment.glsl` and renamed the vertex shader
(terrain-analysis replaced color-relief, "with color-relief backward compat"). Upstream kept evolving those files,
so a rebase will produce modify/delete conflicts; simplest is to keep upstream color-relief intact and add
terrain-analysis as a sibling (re-copied from the v6.18 color-relief files).

---------------------------------------------------------------------------------------------------

## 4. Toolchain / distribution changes

`package.json` @ v6.18.0 vs 0cac8ee50:

- **ESM-only** (`abd79a382` #6254, 6.0.0): `"main": "dist/maplibre-gl.js"` removed; now
  `"module": "dist/maplibre-gl.mjs"`, `"exports": {".": {"types": "./dist/maplibre-gl.d.ts", "import":
  "./dist/maplibre-gl.mjs"}, "./dist/*": "./dist/*", "./package.json": ...}`, `"sideEffects": ["*.css", "src/**/*.ts"]`.
- **Dist files**: `dist/maplibre-gl.mjs`, `dist/maplibre-gl-worker.mjs` (separate worker file, must be deployed next
  to the main bundle), `-dev` variants (`maplibre-gl-dev.mjs`, `maplibre-gl-worker-dev.mjs`), empty placeholder
  `maplibre-gl-shared(.dev).mjs`, `maplibre-gl.d.ts`, `maplibre-gl.css`. **`dist/maplibre-gl.js`,
  `maplibre-gl-dev.js`, `maplibre-gl-csp.js`, `maplibre-gl-csp-worker.js` no longer exist** (`rollup.config.csp.ts`
  deleted). No UMD, no CommonJS `require()` (`ERR_PACKAGE_PATH_NOT_EXPORTED`).
- Downstream app impact (from `docs/guides/v5-to-v6-migration-guide.md`):
  - `<script src=".../maplibre-gl.js">` + global `maplibregl` -> `<script type="module"> import * as maplibregl from
    '.../dist/maplibre-gl.mjs'`. No default export: `import maplibregl from` -> `import * as maplibregl from` or named.
  - Worker: with direct browser ESM the worker URL is auto-derived from `import.meta.url`
    (`src/util/web_worker.ts:39-43`) - the worker `.mjs` must sit beside the main file; cross-origin CDN loads go
    through a same-origin Blob (`worker-src blob:` needed then). With a bundler, call
    `setWorkerUrl(...)` once (Vite: `import workerUrl from 'maplibre-gl/dist/maplibre-gl-worker.mjs?worker&url'`).
  - CSP build gone; self-hosted worker needs only `worker-src 'self'`.
  - Output target ES2022. WebGL2 required; constructor throws `GPUInitializationError` otherwise.
  - `map.transform` removed (Map composes Camera); events are classes; `styleimagemissing` notify-only;
    `GeoJSONSource.setData` 2nd param removed; nested GeoJSON props are real objects; `zoomLevelsToOverscale`
    default 4; RTL plugin deprecated in favour of built-in bidi (`ce10d2208` #8343).
- **Build**: Rolldown 1.3 (`rolldown.config.ts`) replaces Rollup; `rolldown-plugin-dts` (oxc) for typings; build
  scripts run as `node build/*.ts` (native TS stripping, ts-node removed) => **Node >= 24 needed to build**;
  `.nvmrc` is `26` (was `24.11`). `engines` field is unchanged and stale (`node >=16.14.0`, `npm >=8.1.0`).
  Scripts: `build-dist` = css + unicode data + shaders + `build-dev` + `build-prod`; `codegen` unchanged in name;
  `typecheck` = `tsgo --noEmit && tsgo --project tsconfig.dist.json`.
- **Dependencies**: `@maplibre/maplibre-gl-style-spec ^26.6.0` (lock 26.6.0; was ^24.7.0);
  `@mapbox/vector-tile ^3`, `pbf ^5`, `@maplibre/mlt ^1.3`, `@maplibre/geojson-vt ^6.3`, new `bidi-js`;
  removed `@mapbox/jsonlint-lines-primitives`, `@mapbox/whoots-js`. Dev: vitest 5.x, TypeScript ^6.0.3 +
  `@typescript/native-preview`, eslint 10, puppeteer 25.
- Fork packaging (`@eslopemap/maplibre-gl` 5.21.9, style-spec aliased as
  `npm:@eslopemap/maplibre-gl-style-spec@24.7.1` + `file:maplibre-style-spec`, `"main": "dist/maplibre-gl.js"`)
  must be redone on the new `exports` map, and the style-spec fork re-published from a >= 26.6 base.
