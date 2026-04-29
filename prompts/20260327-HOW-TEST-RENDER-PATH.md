# How to Test the Render-to-Texture (RTT/FBO) Path

## Overview

MapLibre GL JS render tests use **Puppeteer** to render a map in a headless browser, extract pixels via `gl.readPixels()`, and compare against reference images using **pixelmatch**.

## Directory structure

```
test/integration/render/
├── run_render_tests.ts            ← Main runner script
└── tests/
    ├── terrain-analysis/          ← terrain-analysis layer tests (2D only today)
    │   ├── multiply/style.json    ← blend-mode: multiply WITHOUT 3D terrain
    │   ├── opacity/style.json
    │   └── rainbow/style.json
    └── terrain/                   ← 3D terrain tests (raster/hillshade, no terrain-analysis)
        ├── default/style.json
        └── ...
```

## Running render tests

```bash
# All render tests
npm run test-render

# Specific test by ID substring
node --no-warnings --loader ts-node/esm test/integration/render/run_render_tests.ts terrain-analysis/multiply

# Update expected images
UPDATE=true node --no-warnings --loader ts-node/esm test/integration/render/run_render_tests.ts terrain-analysis/multiply-3d
```

Results are written to `test/integration/render/results.html`.

## Style.json anatomy

Each test is a `style.json` file in its own folder under `tests/`:

```json
{
  "version": 8,
  "metadata": {
    "test": {
      "height": 256,
      "width": 256
    }
  },
  "center": [-113.26903, 35.9654],
  "zoom": 11,
  "sources": {
    "source": {
      "type": "raster-dem",
      "tiles": ["local://tiles/{z}-{x}-{y}.terrain.png"],
      "maxzoom": 15,
      "tileSize": 256
    }
  },
  "layers": [ ... ],
  "terrain": {                          ← THIS enables the RTT/FBO path
    "source": "source",
    "exaggeration": 1
  }
}
```

## How 3D terrain (RTT) activates

1. The `"terrain"` top-level property triggers `RenderToTexture` in the painter.
2. Layers in `LAYERS_TO_TEXTURES` (background, fill, line, raster, hillshade, color-relief, terrain-analysis) are rendered into FBO textures instead of directly to the screen.
3. These FBOs are then "draped" onto the 3D terrain mesh via `drawTerrain()`.

## The RTT stack model

Layers are grouped into contiguous **stacks**. A new stack is created whenever a non-drapeable layer (e.g., symbol, circle) appears between drapeable layers.

```
Stack 0: [background, raster, terrain-analysis]  ← contiguous drapeable
         symbol layer breaks the stack
Stack 1: [fill, line]                              ← new stack
```

Each stack is rendered into its own FBO, cleared to transparent black `(0,0,0,0)`.

## The multiply-in-FBO bug

When `blend-mode: multiply` is applied inside an FBO cleared to transparent black:
- `multiply` blend: `result = src × dst`
- `src × (0,0,0,0) = (0,0,0,0)` → **invisible output**

The fix (commit `35a606f`):
1. Inside FBO: render terrain-analysis with **normal alpha blending** (not multiply)
2. At drape time: use `multiplyDrape` color mode to blend the FBO texture against actual screen content

## DEM tile fixtures

Two sets of DEM tiles exist in `test/integration/assets/tiles/`:

| Path | Tiles | Used by |
|------|-------|---------|
| `tiles/{z}-{x}-{y}.terrain.png` | 4 tiles at z12 | terrain-analysis tests (center: -113.27, 35.97) |
| `tiles/terrain-shading/{z}-{x}-{y}.terrain.png` | ~40 tiles | terrain/ 3D tests (center: 11.53, 47.34) |
| `tiles/terrain/{z}-{x}-{y}.terrain.png` | ~16 tiles | additional terrain tests |
| `tiles/number/{z}.png` | z0..z22 | raster basemap pattern tiles |

## What's NOT tested today

| Gap | Why it matters |
|-----|---------------|
| **terrain-analysis + multiply + 3D terrain** | The FBO fix in `35a606f` has zero render-test coverage |
| **Stack-isolated multiply** (non-drapeable layer forces separate stack) | The worst-case scenario that originally caused the white/invisible bug |
| **Raster + multiply + 3D terrain** | `draw_raster.ts` doesn't have RTT-aware blend handling at all |

## Key source files

| File | Role |
|------|------|
| `src/render/render_to_texture.ts` | Stack grouping, FBO management, drape dispatch |
| `src/render/draw_terrain_analysis.ts` | `getBlendModeState()` — RTT-aware blend routing |
| `src/render/draw_terrain.ts` | `drawTerrain()` — drapes FBO onto 3D mesh, accepts `drapeColorMode` |
| `src/gl/color_mode.ts` | `multiplyDrape` blend factors |
| `src/render/draw_raster.ts` | Raster drawing — NO RTT blend override |
