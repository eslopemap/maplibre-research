# Report: RTT Multiply Blend Fix — v5.21.5

**Date**: 2025-03-27  
**Version**: 5.21.5  
**Published**: npm `@eslopemap/maplibre-gl@5.21.5`

## Problem

`terrain-analysis` layers with `blend-mode: multiply` rendered blank (invisible) in 3D terrain (RTT) mode.

## Root Causes

### 1. Test tile coverage gap

The original test used `tiles/{z}-{x}-{y}.terrain.png` which only has z12 tiles. 3D terrain needs a full zoom pyramid (z4–z12) for the terrain mesh. All tiles were 404ing → nothing rendered.

**Fix**: Switched to `tiles/terrain-shading/{z}-{x}-{y}.terrain.png` (Alps area, center `[11.52517, 47.34487]`, z4–z12).

### 2. Same-stack multiplyDrape against empty screen

When `background` + `terrain-analysis(multiply)` shared an RTT stack, the entire stack (including the white background) was draped with `multiplyDrape`. Multiply against the cleared screen (`rgba(0,0,0,0)`) = invisible.

**Fix**: Added `rttDeferMultiply` flag to `RenderOptions`. The RTT system only defers multiply to drape time when the terrain-analysis layer is the **first** layer in its stack (isolated — FBO starts transparent black). When preceded by other layers (e.g., background), multiply happens inside the FBO against that content.

## Files Changed

- `src/render/painter.ts` — added `rttDeferMultiply?: boolean` to `RenderOptions`
- `src/render/render_to_texture.ts` — only set `currentStack.blendMode` when layer is first in stack; pass `rttDeferMultiply` in options
- `src/render/draw_terrain_analysis.ts` — use `rttDeferMultiply` instead of `isRenderingToTexture` for blend override

## New Tests

| Test | Description | Validates |
|------|-------------|-----------|
| `terrain-analysis/3d-terrain-basic` | Normal blend + 3D terrain | terrain-analysis renders at all in RTT |
| `terrain-analysis/multiply-3d-terrain` | Multiply + 3D terrain (same stack as background) | same-stack multiply works |
| `terrain-analysis/multiply-3d-stack-isolated` | Multiply + 3D terrain with stack break | isolated-stack multiplyDrape works |

## Test Results

All 14 terrain-related render tests pass: 6 terrain-analysis + 4 color-relief + 3 resampling + 1 terrain/default.

## Next

- Investigate `screen` blend mode in 3D terrain — likely broken by same logic
- Consider per-layer blend mode in RTT stack
