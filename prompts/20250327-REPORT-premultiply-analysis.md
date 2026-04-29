# Analysis: Could `premultiply: false` work as a blend mode?

**Date**: 2025-03-27  
**Context**: User request in `20250327-premultiply-github-issue.md` proposes a raster source option `premultiply: false`. Question: could this be a layer-level `blend-mode` instead?

## Short Answer

**No. `premultiply: false` is a source/decode concern, not a blend mode.** They solve different problems at different pipeline stages. However, they *interact*, and our existing `blend-mode` infrastructure already handles part of the consequence.

## Detailed Analysis

### What `premultiply: false` does

It prevents two things in the tile decode/upload pipeline:

1. **JS-side premultiplication** (`premultiplyAlpha()` in `src/util/image.ts`): For raw `Uint8Array` data (e.g., `RGBAImage`), MapLibre explicitly multiplies `R,G,B` by `A/255` before uploading to the GPU.

2. **GL-side premultiplication** (`UNPACK_PREMULTIPLY_ALPHA_WEBGL` in `src/render/texture.ts`): For DOM images (`HTMLImageElement`, `ImageBitmap`), the GPU driver premultiplies during `texImage2D`.

Both are controlled by the `premultiply` option in `Texture.update()`. The raster tile source (`src/source/raster_tile_source.ts` line 201-203) currently uploads with `{useMipmap: true}` and no explicit `premultiply` setting — which defaults to `true` (see `texture.ts` line 67: `const wantPremultiply = this.format === gl.RGBA && (!options || options.premultiply !== false)`).

### Why it's not a blend mode

A blend mode controls **how pixels combine with the framebuffer destination** at draw time:
- `normal`: `src + dst × (1 - src_alpha)` — standard alpha compositing
- `multiply`: `src × dst` — darkens
- `screen`: `src + dst - src × dst` — lightens

Premultiplication controls **how the source pixel data is encoded** before it even reaches the blend stage. It's a data representation choice, not a compositing operation.

If you have packed RGBA payload data (e.g., weather bands encoded as `[wind_u, wind_v, temperature, humidity]`), premultiplication corrupts the data at decode time — long before any blend mode runs. No blend mode can undo `R = R × A / 255` after the fact.

### Interaction with our blend-mode infrastructure

That said, `premultiply: false` does affect how blend modes work:

| Source premultiply | Blend mode | Shader output | GL blend needed |
|---|---|---|---|
| `true` (default) | normal | premultiplied RGBA | `[ONE, ONE_MINUS_SRC_ALPHA]` (standard) |
| `true` | multiply | premultiplied RGBA | `[DST_COLOR, ZERO]` with shader adjustments |
| `false` | normal | straight RGBA | `[SRC_ALPHA, ONE_MINUS_SRC_ALPHA]` (classic) |
| `false` | multiply | straight RGBA | `[DST_COLOR, ZERO]` (simpler — no PMA correction) |

Our `terrain-analysis` shader already tracks this via `u_is_premultiplied` and `u_blend_neutral` uniforms, and `draw_terrain_analysis.ts` sets `isPremultiplied: 0/1` based on blend mode. The same mechanism would work for a non-premultiplied source.

### What about `raster` layer type?

The GitHub issue targets raster sources (ordinary image tiles), not `terrain-analysis`. For raster layers, the decode pipeline is different:
- Raster tiles go through `ImageRequest.getImage()` → DOM `Image` → `Texture` upload
- The premultiplication happens in the `texImage2D` call via `UNPACK_PREMULTIPLY_ALPHA_WEBGL`
- The raster fragment shader (`src/shaders/raster.fragment.glsl`) assumes premultiplied input

So the source-level `premultiply: false` option would need to:
1. Skip `UNPACK_PREMULTIPLY_ALPHA_WEBGL` during texture upload
2. Use `createImageBitmap({premultiplyAlpha: 'none'})` in the image request path
3. Adjust the raster fragment shader or blend func to handle straight-alpha output

This is fundamentally a **source pipeline change**, not something any blend mode can address.

### Could we add a `raw` or `passthrough` blend mode?

Hypothetically, a `blend-mode: "passthrough"` could skip premultiplication at the shader output stage and use `[ONE, ZERO]` (replace) blending. But this still wouldn't solve the problem because:

1. The corruption happens at **image decode**, before the shader runs
2. A blend mode would only affect the final compositing, not the texture data
3. It would need to be paired with the source-level `premultiply: false` anyway

### Recommendation

The GitHub issue's design is correct:

- **`premultiply: false`** belongs on the **source** — it controls data encoding at decode/upload time
- Our existing **`blend-mode`** infrastructure is orthogonal — it controls compositing at draw time
- They can be combined: a `premultiply: false` source with `blend-mode: multiply` would work, using the straight-alpha multiply path

The two features complement each other but serve different purposes. Implementing `premultiply` as a blend mode would be architecturally wrong and wouldn't solve the stated problem.

### Implementation notes for `premultiply: false` on raster sources

Key touchpoints in the codebase:

1. **`src/source/raster_tile_source.ts:201-203`** — Pass `{premultiply: false}` to `Texture` constructor
2. **`src/render/texture.ts:67`** — Already supports `premultiply: false` option
3. **`src/util/image_request.ts`** — Need to use `createImageBitmap({premultiplyAlpha: 'none'})` path
4. **`src/shaders/raster.fragment.glsl`** — May need a `u_is_premultiplied` uniform (like terrain-analysis already has) to handle straight-alpha output correctly
5. **`src/render/draw_raster.ts`** — Adjust blend func for straight-alpha if needed

The terrain-analysis layer already handles both premultiplied and straight-alpha paths correctly, so it serves as a reference implementation.
