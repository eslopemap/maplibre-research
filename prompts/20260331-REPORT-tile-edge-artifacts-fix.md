# Fix: Tile Edge Artifacts in terrain-analysis Layer

**Date:** 2026-03-31
**Commit:** `04429fe` on branch `blend`

## Problem

Visible artifacts (reduced/incorrect slope values) appeared at boundaries between DEM tiles in the terrain-analysis layer. The slopes appeared lower than expected at these edges, creating a grid-like pattern matching the tile layout.

## Root Cause

The DEM texture was bound with `gl.LINEAR` filtering in `draw_terrain_analysis.ts`. The terrain-analysis shader's `getElevation()` function performs **manual bilinear filtering** of decoded elevation values — it snaps to texel centers via `(floor(tc) + 0.5) / size`, fetches 4 exact texels, decodes each from RGBA to elevation, then blends the decoded values.

With `gl.LINEAR`, any floating-point imprecision in the texel-center calculation caused the GPU hardware to blend the **encoded RGBA values** before decoding. At tile boundaries, where border pixels contain neighbor tile data with potentially very different encoded values, this hardware blending of encoded bytes produced incorrect decoded elevations. The Horn algorithm's 3×3 derivative kernel then computed reduced gradients from these incorrect values.

Hillshade's `hillshade_prepare` shader uses `gl.NEAREST` for the same reason — it needs exact texel values for its own manual filtering.

## Fix

**`src/render/draw_terrain_analysis.ts`**: Changed `gl.LINEAR` → `gl.NEAREST` for DEM texture binding (2 lines).

This ensures the shader's manual bilinear filter receives exact texel values, eliminating the edge artifacts.

## Additional Fixes

Fixed two stale unit test expectations in `src/render/render_to_texture.test.ts`:
- `blendMode` is only tracked when terrain-analysis is the **first** layer in a stack (isolated FBO scenario); the test expected it to be set when terrain-analysis was the second layer
- `_getDrapeColorMode` now returns `ColorMode.screenDrape` for `screen` blend mode (was added after the test was written)

## New Test

Added `test/integration/render/tests/terrain-analysis/slope-edges/` render test that validates slope rendering at a zoom level where tile boundaries are visible, ensuring the fix prevents regression.

## Validation

- **11/11 render tests pass** (including new `slope-edges`)
- **2796/2796 unit tests pass** (including fixed RTT tests)
