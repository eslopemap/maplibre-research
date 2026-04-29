# Slope Shader Artifact Investigation and Fix

This document explains the artifact investigation for `slope-hybrid-border.html`, why the initial strategy was insufficient, and the final shader fix.

## Context

At `z=14` (DEM `tileSize=512`), around:

- `lng=7.408525`
- `lat=44.097279`

we observed two visual issues:

1. localized slope artifacts (sharp/unnatural patches)
2. slope behavior near tile edges that did not match expectations

The setup uses Terrarium-encoded DEM tiles (`.webp`) and computes slope/aspect in a custom fragment shader.

---

## Initial strategy

The first pass focused on the most likely causes:

1. **Tile seam handling**
   - use padded DEM (1px border)
   - backfill border from neighbor DEM tiles when available
2. **Derivative scaling and projection correction**
   - Horn derivatives in tile space
   - per-fragment latitude correction for Mercator (`1 / cos(lat)`)
   - dynamic `u_tile_size` instead of hardcoded constants
3. **Noise mitigation**
   - optional `smooth-before-slope` (3x3 average before derivative)

### Targeted shader sample (initial derivative path)

```glsl
vec2 hornDeriv(vec2 uvCore) {
  vec2 uv = paddedUV(uvCore);
  vec2 e = u_texel;

  float a = elevationForDeriv(uv + vec2(-e.x, -e.y));
  float b = elevationForDeriv(uv + vec2( 0.0, -e.y));
  float c = elevationForDeriv(uv + vec2( e.x, -e.y));
  float d = elevationForDeriv(uv + vec2(-e.x,  0.0));
  float f = elevationForDeriv(uv + vec2( e.x,  0.0));
  float g = elevationForDeriv(uv + vec2(-e.x,  e.y));
  float h = elevationForDeriv(uv + vec2( 0.0,  e.y));
  float i = elevationForDeriv(uv + vec2( e.x,  e.y));

  float dzdx = (c + 2.0 * f + i) - (a + 2.0 * d + g);
  float dzdy = (g + 2.0 * h + i) - (a + 2.0 * b + c);

  vec2 deriv = vec2(dzdx, dzdy) * u_tile_size / pow(2.0, 28.2562 - u_zoom);
  float lat = (u_latrange.x - u_latrange.y) * (1.0 - uvCore.y) + u_latrange.y;
  deriv.x = deriv.x / max(cos(radians(lat)), 0.0001);
  return deriv;
}
```

---

## Why this was not enough

Even with correct border backfill and projection scaling, a sampling-phase mismatch can still produce artifacts:

- the derivative kernel is conceptually a **discrete DEM texel stencil**
- but runtime shader sampling can become **interpolated between texel centers** depending on the interpolated `v_uv` phase
- at max zoom and high contrast terrain, this phase drift is visible

In short: the algorithm was right, but the sampling coordinates were not always aligned with DEM texel centers.

---

## Reproduction and proof

A dedicated probe was used:

- `test/artifacts/dem_slope_artifact_probe.py`

It compares:

1. discrete Horn reference
2. shader-like Horn with **pixel-center UVs**
3. shader-like Horn with **endpoint UVs**

Observed at the problematic tile:

- pixel-center emulation vs discrete reference: `mean=0.000`, `p95=0.000`, `max=0.000`
- endpoint emulation vs discrete reference: `mean=1.280`, `p95=4.409`, `max=32.570`

Interpretation:

- when sample phase is texel-centered, the shader matches the reference exactly
- when sample phase drifts (endpoint-like), large errors appear

This isolates sampling alignment as a root cause for the remaining artifact behavior.

---

## Final fix

### Principle

Always snap derivative evaluation UV to DEM texel centers before running Horn sampling.

### Targeted shader sample (final)

```glsl
vec2 coreUVForDeriv(vec2 uvCore) {
  vec2 tileCoord = floor(clamp(uvCore, vec2(0.0), vec2(1.0)) * u_tile_size);
  tileCoord = clamp(tileCoord, vec2(0.0), vec2(u_tile_size - 1.0));
  return (tileCoord + vec2(0.5)) / u_tile_size;
}

void main() {
  vec2 d = hornDeriv(coreUVForDeriv(v_uv));
  float gradient = length(d);
  float slopeDeg = clamp(degrees(atan(gradient)), 0.0, 90.0);
  // ... aspect + color ramp
}
```

### Why this works

- forces derivative stencil evaluation on the same discrete lattice as DEM texels
- removes phase-dependent interpolation bias
- keeps border backfill/projection logic unchanged
- preserves optional smoothing and blending controls

---

## 3D draping mismatch: why slope looked slightly "under" terrain

After enabling 3D for the custom slope layer, the overlay followed terrain shape but still sat slightly below built-in layers (hillshade) in some areas.

### Root cause (from MapLibre internals)

The initial custom drape mesh used:

1. a coarse grid (`16x16`), and
2. nearest-neighbor DEM sampling at mesh vertices.

MapLibre terrain internals use a denser mesh and bilinear elevation lookup:

- `Terrain.meshSize = 128` (`src/render/terrain.ts`)
- bilinear interpolation in elevation lookup (`getDEMElevation`) (`src/render/terrain.ts`)
- terrain shaders rely on interpolated DEM elevation (`get_elevation`) (`src/shaders/_prelude.vertex.glsl`)

So our custom layer geometry and MapLibre terrain geometry were not sampling exactly the same surface. That sub-texel difference is enough to show a slight vertical offset.

### Fix applied

The custom slope drape was aligned to the same strategy:

- switch custom terrain mesh subdivisions from `16` to `128`
- switch DEM sampling for mesh elevation from nearest-neighbor to bilinear interpolation

Targeted sample for bilinear elevation in the custom mesh path:

```js
const cx = Math.max(0, Math.min(core, uvx * core));
const cy = Math.max(0, Math.min(core, uvy * core));

const x0 = Math.floor(cx), y0 = Math.floor(cy);
const x1 = Math.min(x0 + 1, core), y1 = Math.min(y0 + 1, core);
const tx = cx - x0, ty = cy - y0;

// e00/e10/e01/e11 sampled from padded DEM
const e = (1 - tx) * (1 - ty) * e00 + tx * (1 - ty) * e10 + (1 - tx) * ty * e01 + tx * ty * e11;
```

With this change, slope draping visually matches hillshade much more closely in 3D.

---

## Additional notes

- DEM seam mismatch still exists in source data (measured in probe), so some hard edges can still be data-driven.
- The fix addresses shader-introduced amplification/attenuation from sampling misalignment.
- This behavior is most visible at max DEM zoom, where each texel contributes directly to high-frequency slope response.

---

## Files impacted

- Runtime shader integration (custom layer):
  - `slope-hybrid-border.html`
- Investigation and reproducibility:
  - `test/artifacts/dem_slope_artifact_probe.py`

---

## Validation commands

```bash
npm run test-build -- test/build/dev.test.ts
python3 test/artifacts/dem_slope_artifact_probe.py --lon 7.408525 --lat 44.097279 --zoom 14
```

Both passed after the final fix.
