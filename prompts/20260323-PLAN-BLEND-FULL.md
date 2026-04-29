# Plan: Add `blend-mode` to `terrain-analysis` layers

## Goal
Support `blend-mode` (normal | multiply | screen) on `terrain-analysis` layers,
identical to the existing raster-layer implementation.

## Current state
- `blend-mode` is defined in `paint_raster` in the style-spec (`v8.json`),
  generated into `raster_style_layer_properties.g.ts`, and wired through
  `draw_raster.ts` → `raster_program.ts` → `raster.fragment.glsl`.
- The raster implementation uses:
  - **GL blend state**: `ColorMode.multiply` / `ColorMode.screen` (separate
    blend funcs for RGB and alpha).
  - **Shader output**: when `u_is_premultiplied == 0`, the fragment shader
    writes `mix(blend_neutral, rgb, alpha)` instead of `rgb * alpha`, so
    transparent pixels blend to the neutral element (white for multiply,
    black for screen) rather than black.
  - **Uniforms**: `u_is_premultiplied` and `u_blend_neutral` are passed from
    `draw_raster.ts` through `raster_program.ts`.

## Files to change

### 1. Style-spec: `maplibre-style-spec/src/reference/v8.json`
Add `"blend-mode"` to `paint_terrain-analysis` (copy from `paint_raster`).

### 2. Style-spec: rebuild types & tarball
```bash
cd maplibre-style-spec
npm run generate-style-spec   # regenerates src/types.g.ts
npm pack                      # produces .tgz
```
Then update root `package.json` to point to new tarball and `npm install`.

### 3. Root codegen: `npm run codegen`
Regenerates `terrain_analysis_style_layer_properties.g.ts` to include
the new `blend-mode` DataConstantProperty.

### 4. Shader: `src/shaders/terrain_analysis.fragment.glsl`
Add `uniform float u_is_premultiplied;` and `uniform float u_blend_neutral;`.
Change final `fragColor` line to match the raster shader pattern:
```glsl
vec4 rampColor = u_opacity * texture(u_color_stops, vec2(x, 0));
if (u_is_premultiplied > 0.5) {
    fragColor = vec4(rampColor.rgb * rampColor.a, rampColor.a);
} else {
    fragColor = vec4(mix(vec3(u_blend_neutral), rampColor.rgb, rampColor.a), rampColor.a);
}
```
Wait — the terrain-analysis shader currently outputs pre-multiplied alpha
(`u_opacity * texture(...)`) and the default color mode is `alphaBlended`
which expects premultiplied. The rampColor from the texture is already
straight-alpha RGBA, and multiplying by `u_opacity` gives premultiplied.

For blend modes, we need to output **straight alpha** with neutral fill
for transparent areas, same pattern as raster.

### 5. Uniforms: `src/render/program/terrain_analysis_program.ts`
Add `u_is_premultiplied` and `u_blend_neutral` uniforms + their values.

### 6. Drawing: `src/render/draw_terrain_analysis.ts`
Add `getBlendModeState()` (same as in `draw_raster.ts`) and wire it into
the `program.draw(...)` call, replacing the plain `colorMode`.

### 7. Demonstrator: `slope-builtin.html`
Move `blend-mode` from basemap layer to analysis layer (where it belongs).

### 8. Tests
- Run `npm run codegen` and verify generated file.
- Run style-spec tests: `cd maplibre-style-spec && npm test`
- Run root tests: `npm test`

## Execution order
1 → 2 → 3 → 4+5+6 (parallel) → 7 → 8
