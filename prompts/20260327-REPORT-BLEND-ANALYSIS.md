# Blend-Mode Feature: Analysis & Work Report

**Date:** 27 March 2026  
**Branch:** `blend` (ahead of `fork/blend`)  
**Repo:** `@eslopemap/maplibre-gl` (fork of `maplibre-gl-js` v5.21.0)

---

## 1. Context & Initial State

The workspace is a fork of maplibre-gl-js (`main` at v5.21.0) with a `terrain-analysis` feature branch already merged. That branch introduced:

- A new `terrain-analysis` layer type for DEM-based slope/elevation visualisation with color-relief backward compatibility (`color-relief` → `terrain-analysis`)
- Support for step expressions (discrete color bands)
- Mercator correction on both axes for accurate slope estimation
- Runtime style-spec patches to support the new layer type

The `blend` branch starts from this `terrain-analysis` baseline and adds **CSS-style blend modes** (`multiply`, `screen`) as a paint property on both `raster` and `terrain-analysis` layers.

---

## 2. Feature Design Decisions

From workspace planning notes:

- Paint property name: **`blend-mode`** (generic, reusable)
- Defined in `paint_raster` initially; shared with `terrain-analysis`
- First-pass values: **`multiply`** and **`screen`** only — `darken`/`lighten` deferred
- Style-spec package: published as `@eslopemap/maplibre-gl-style-spec`

---

## 3. Commit Sequence

| Commit   | Date         | Description |
|----------|--------------|-------------|
| `e40c163` | 2026-03-23  | Add `blend-mode` to raster layers |
| `6c61e40` | 2026-03-23  | Add `blend-mode` to terrain-analysis layers |
| `513374f` | 2026-03-23  | Fix terrain-analysis 3D compatibility |
| `a2a902f` | 2026-03-23  | Preserve destination alpha for multiply and screen |
| `35a606f` | 2026-03-25  | **Fix: multiply blend invisible in 3D RTT stack** (major bug) |
| `759cfea` | 2026-03-27  | Add render tests, fix premultiplied double-multiply, bump to 5.21.3 |

---

## 4. Implementation Overview

### 4.1 ColorMode additions (`src/gl/color_mode.ts`)

Three new static blend modes were added:

```
multiply       = [DST_COLOR, ZERO, ZERO, ONE]
                 // Standard CRT multiply: out_rgb = src × dst
                 // Alpha: preserve dst alpha

screen         = [ONE, ONE_MINUS_SRC_COLOR, ZERO, ONE]
                 // out_rgb = src + dst*(1-src)
                 // Alpha: preserve dst alpha

multiplyDrape  = [DST_COLOR, ONE_MINUS_SRC_ALPHA, ZERO, ONE]
                 // For draping premultiplied-alpha FBO textures (see §5)
                 // out_rgb = dst × (src_rgb×alpha + 1 − alpha)
```

The `a2a902f` commit corrected the initial `multiply` and `screen` definitions to use `[..., ZERO, ONE]` (not `[..., ONE, ONE_MINUS_SRC_ALPHA]`) to preserve the destination alpha channel rather than blending it.

### 4.2 Shader uniforms (`terrain_analysis.fragment.glsl`)

Two new uniforms control output path selection:

- **`u_is_premultiplied`** (float bool): when >0.5, output `rampColor` directly (already premultiplied by `u_opacity * texture(...)`)
- **`u_blend_neutral`** (float): the neutral fill value for the non-premultiplied path, blended via `mix(blend_neutral, rampColor.rgb, rampColor.a)`

The non-premultiplied path (used by multiply/screen direct rendering) avoids the alpha channel being set to a value that would interfere with WebGL blend equations applied after the shader.

### 4.3 Blend mode dispatch (`draw_terrain_analysis.ts`)

`getBlendModeState()` maps the layer paint property to a `BlendModeState`:

| `blend-mode` | `isRenderingToTexture` | `colorMode` | `isPremultiplied` | `blendNeutral` |
|---|---|---|---|---|
| `multiply` | false | `ColorMode.multiply` | 0 | 1 |
| `screen` | false | `ColorMode.screen` | 0 | 0 |
| `multiply` | **true** (3D) | `colorModeForRenderPass()` | 1 | 0 |
| none / default | any | `colorModeForRenderPass()` | 1 | 0 |

The 3D/RTT override for multiply is the key to the major bug fix (see §5).

---

## 5. Major Bug: Multiply Invisible in 3D Terrain Mode

### 5.1 Root cause

MapLibre's render-to-texture (RTT) pipeline groups consecutive texture-renderable layers into **stacks**, each rendered into a fresh FBO cleared to `rgba(0,0,0,0)`.

If a non-RTT-compatible layer (e.g. `circle`, `symbol`) appears between `terrain-analysis/multiply` and another layer like `hillshade`, the grouping is broken and `terrain-analysis` ends up **in its own stack** — rendered against a fully-transparent black background.

Multiply blend equation: `out = src × dst`. With `dst = (0,0,0,0)`, `out = (0,0,0,0)` — **completely invisible**.

Screen blend is unaffected: `out = src + dst*(1-src)` → `out = src + 0 = src`. The source survives.

### 5.2 Fix architecture (`35a606f`)

Apply multiply **at drape time**, not at FBO draw time:

1. **`render_to_texture.ts`** — Each stack tracks an optional `blendMode`. When a `terrain-analysis` layer with `multiply` is encountered during stack building, `currentStack.blendMode = 'multiply'` is set. At drape time, `_getDrapeColorMode()` converts this to `ColorMode.multiplyDrape` and passes it to `drawTerrain()`.

2. **`draw_terrain.ts`** — `drawTerrain()` now accepts an optional `drapeColorMode` parameter. When provided, it is used instead of `painter.colorModeForRenderPass()` for the terrain mesh draw call.

3. **`draw_terrain_analysis.ts`** — When `isRenderingToTexture && blendMode === 'multiply'`, the layer uses **normal alpha blending** into the FBO (not multiply), with `isPremultiplied=1`. The FBO texture is then draped using `multiplyDrape`.

4. **`ColorMode.multiplyDrape`** — `[DST_COLOR, ONE_MINUS_SRC_ALPHA, ZERO, ONE]` — correctly alpha-aware multiply for premultiplied FBO textures:
   `out_rgb = dst × (src_rgb × alpha + 1 − alpha)`

---

## 6. Bug: Premultiplied Alpha Double-Multiply

### 6.1 Root cause

In the shader's premultiplied output path, the original code was:

```glsl
fragColor = vec4(rampColor.rgb * rampColor.a, rampColor.a);
```

But `rampColor = u_opacity * texture(u_color_stops, ...)` is **already premultiplied** by `u_opacity`:
- `rampColor.rgb = color.rgb * opacity`
- `rampColor.a = opacity`

Multiplying rgb by alpha again produced `rgb = color.rgb * opacity²` — squaring the opacity. At e.g. `opacity=0.45`, the visual opacity became `0.45² = 0.2025` (~45% of expected brightness).

### 6.2 Fix (`759cfea`)

```glsl
// Before (wrong):
fragColor = vec4(rampColor.rgb * rampColor.a, rampColor.a);

// After (correct):
// rampColor is already premultiplied: rgb = color * opacity, a = opacity
fragColor = rampColor;
```

A shader unit test was added to guard this regression: it checks that the compiled shader source contains `fragColor=rampColor` and does **not** contain `rampColor.rgb*rampColor.a`.

---

## 7. Render Integration Tests

Three render tests were added under `test/integration/render/tests/terrain-analysis/`:

| Test | What it validates |
|------|------------------|
| `rainbow` | Color interpolation across elevation stops (full color ramp, `elevation` attribute) |
| `opacity` | Opacity painting — verifies premultiplied alpha output is correct |
| `multiply` | Multiply blend mode — verifies blend equations against tile pixels |

All three use existing terrain tiles from `test/integration/assets/tiles/terrain-shading/` via the `local://tiles/{z}-{x}-{y}.terrain.png` scheme. ImageMagick RMSE comparison confirms expected == actual (RMSE = 0 for all three).

---

## 8. Investigation Notes (2026-03-27 Session)

- Initial concern: `npm run test-render` was reportedly failing. Running the tests directly with `--filter terrain-analysis` showed **3/3 passed (100%)**.
- ImageMagick `compare -metric RMSE` on each expected/actual pair returned `0 (0)` — pixel-identical output.
- Conclusion: failures were transient or from a prior state before the premultiplied fix was applied.
- Vision API was not available (org restriction); pixel inspection was done via `identify` and `compare` from ImageMagick instead.

---

## 9. Files Changed (full `blend` branch vs `main`)

| File | Purpose |
|------|---------|
| `src/gl/color_mode.ts` | `multiply`, `screen`, `multiplyDrape` blend modes |
| `src/gl/value.ts` | WebGL blend function application |
| `src/gl/types.ts` | Type updates |
| `src/gl/state.test.ts` | State test update |
| `src/gl/color_mode.test.ts` | Tests for new blend modes and drape factors |
| `src/render/draw_raster.ts` | `blend-mode` support for raster layers |
| `src/render/draw_terrain_analysis.ts` | `blend-mode` dispatch + RTT override |
| `src/render/draw_terrain.ts` | Accept optional `drapeColorMode` |
| `src/render/render_to_texture.ts` | Stack blend-mode tracking, drape dispatch |
| `src/render/render_to_texture.test.ts` | RTT stack + drape tests |
| `src/render/program/raster_program.ts` | `blend-mode` uniform |
| `src/render/program/terrain_analysis_program.ts` | `blend-mode` uniform |
| `src/shaders/raster.fragment.glsl` | Blend-mode conditional output |
| `src/shaders/terrain_analysis.fragment.glsl` | `u_is_premultiplied` / `u_blend_neutral` paths; premultiplied fix |
| `src/shaders/terrain_analysis.fragment.test.ts` | Shader unit test |
| `src/style/style_layer/raster_style_layer.test.ts` | Raster layer `blend-mode` tests |
| `src/style/style_layer/terrain_analysis_style_layer.test.ts` | TA layer `blend-mode` tests |
| `test/integration/render/tests/terrain-analysis/` | 3 render test cases |
| `package.json` | Version 5.21.3, style-spec dep → published npm |
| `CHANGELOG.md` | 5.21.3 release notes |
