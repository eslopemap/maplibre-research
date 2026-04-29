# Report: RTT Screen Blend Fix — v5.21.6

**Date**: 2025-03-27  
**Version**: 5.21.6  
**Published**: npm `@eslopemap/maplibre-gl@5.21.6`

## Problem

`terrain-analysis` layers with `blend-mode: screen` lost their screen effect in 3D terrain (RTT) isolated stacks. Screen against the cleared FBO (`rgba(0,0,0,0)`) is an identity operation: `src + 0×(1-src) = src`, so the layer rendered with normal alpha blending — the screen lightening effect was lost.

## Fix

Generalized the `rttDeferMultiply` mechanism from 5.21.5 to cover both blend modes:

- **Renamed** `rttDeferMultiply` → `rttDeferBlend` in `RenderOptions`, `render_to_texture.ts`, and `draw_terrain_analysis.ts`
- **Added** `ColorMode.screenDrape` — `[ONE, ONE_MINUS_SRC_COLOR, ZERO, ONE]` for draping premultiplied-alpha FBO textures with screen blend onto the screen
- **Extended** `_getDrapeColorMode()` to return `screenDrape` for stacks with `blendMode === 'screen'`
- **Extended** the isolated-stack detection to also capture `screen` blend (not just `multiply`)

The same two-case logic applies:
- **Isolated stack** (terrain-analysis first/only in stack): defer screen to drape time via `screenDrape`
- **Same stack** (preceded by background etc.): apply screen inside FBO against existing content

## Files Changed

- `src/gl/color_mode.ts` — added `screenDrape`
- `src/render/painter.ts` — renamed `rttDeferMultiply` → `rttDeferBlend`
- `src/render/render_to_texture.ts` — handle `screen` in stack blendMode capture and drape mode selection
- `src/render/draw_terrain_analysis.ts` — use `rttDeferBlend`, handle screen in deferred path

## New Tests

| Test | Description |
|------|-------------|
| `terrain-analysis/screen` | 2D screen blend baseline (dark #333 background) |
| `terrain-analysis/screen-3d-terrain` | Screen + 3D terrain, same stack as background |
| `terrain-analysis/screen-3d-stack-isolated` | Screen + 3D terrain, isolated stack |
| `terrain-analysis/3d-terrain-dark-bg` | Normal blend + dark bg baseline for comparison |

## Verification

- `screen-3d-stack-isolated` avg RGB 404.7 vs `3d-terrain-dark-bg` avg RGB 345.0 — screen lightening clearly visible
- `screen-3d-terrain` (same stack, screen inside FBO against #333) avg RGB 354.6 — slight lightening
- All 18 terrain-related render tests pass with no regressions
