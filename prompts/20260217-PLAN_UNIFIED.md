# Plan: Unified DEM Prepare Pass & Terrain Analysis Architecture

## Context

Currently, `hillshade` uses a two-pass rendering pipeline:
1. **Prepare pass** (once per tile): samples 9 DEM texels (3×3 neighborhood), computes x/y derivatives via Horn's algorithm, stores them in an FBO (R/G channels).
2. **Render pass** (every frame): reads cached derivatives from FBO, applies illumination model.

Meanwhile, `terrain-analysis` does everything in a **single pass**: samples 9 DEM texels, computes derivatives, derives the selected attribute (slope/aspect), and applies the color ramp — all every frame.

This means terrain-analysis pays the 9-tap DEM sampling cost on every frame, while hillshade pays it only once per tile load.

## Phase 1: Shared Prepare Pass (rendering optimization)

### Goal
Factor out a shared `demPrepare` pass that both `hillshade` and `terrain-analysis` can consume. This:
- Gives terrain-analysis the same per-tile caching benefit hillshade already has.
- Enables efficient layer stacking: if a user has both hillshade + terrain-analysis on the same DEM source, the prepare pass runs **once** and both render passes read from the same cached FBO.

### Architecture

```
                  ┌──────────────────────┐
                  │   DEM Tile Data      │
                  │  (raster-dem source) │
                  └──────────┬───────────┘
                             │
                    ┌────────▼─────────┐
                    │  demPrepare      │  ← runs once per tile
                    │  (offscreen FBO) │
                    │                  │
                    │  R: dz/dx        │
                    │  G: dz/dy        │
                    │  B: elevation    │  ← NEW: store raw elevation too
                    │  A: (reserved)   │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼───────┐ ┌────▼────────┐ ┌───▼──────────┐
     │  hillshade     │ │ terrain-    │ │ future DEM   │
     │  render pass   │ │ analysis    │ │ layers...    │
     │                │ │ render pass │ │              │
     │ illumination   │ │ scalar →    │ │              │
     │ model          │ │ color ramp  │ │              │
     └────────────────┘ └─────────────┘ └──────────────┘
```

### FBO Layout

The current hillshade prepare pass stores derivatives in R/G and wastes B/A. We can extend it:

| Channel | Content | Used by |
|---------|---------|---------|
| R | dz/dx (normalized) | hillshade, slope, aspect, curvature |
| G | dz/dy (normalized) | hillshade, slope, aspect, curvature |
| B | elevation (normalized) | color-relief / elevation attribute |
| A | reserved (1.0) | future use |

### Implementation Steps

1. **Rename `hillshadePrepare` → `demPrepare`** (shader + program)
   - Extend the prepare shader to also output elevation in the B channel.
   - Keep the same 3×3 Horn's algorithm for derivatives.

2. **Move FBO ownership from hillshade tile state to shared tile state**
   - Currently `tile.fbo` is set by `prepareHillshade`. Make it a shared DEM cache.
   - Add `tile.needsDemPrepare` flag (replacing `tile.needsHillshadePrepare`).

3. **Update hillshade render pass** to read from the shared FBO (no functional change, just the FBO source).

4. **Update terrain-analysis render pass** to read from the shared FBO instead of raw DEM:
   - Remove the 9-tap sampling from `terrain_analysis.fragment.glsl`.
   - Read derivatives from R/G, elevation from B.
   - Compute slope/aspect from derivatives as before (just cheaper input).

5. **Ensure prepare pass runs for any DEM-consuming layer**
   - In `painter.ts`, the offscreen pass should trigger `demPrepare` if any layer (hillshade or terrain-analysis) references a raster-dem source.

### Style API: No Change
This is purely a rendering optimization. The style spec stays as-is:
- `hillshade` layer type with its existing paint properties.
- `terrain-analysis` layer type with its existing paint properties.
- `color-relief` as backward-compatible alias for terrain-analysis with attribute=elevation.

---

## Limits of the dx/dy Prepare Pass

### What dx/dy derivatives support

The first-order partial derivatives (dz/dx, dz/dy) — computed by Horn's algorithm from a 3×3 window — are sufficient for:

| Attribute | Formula | Supported? |
|-----------|---------|------------|
| **Slope** | `atan(sqrt(dx² + dy²))` | ✅ Yes |
| **Aspect** | `atan2(dy, -dx)` | ✅ Yes |
| **Hillshade** | illumination model on (dx, dy) | ✅ Yes |

### What requires second-order derivatives (zxx, zxy, zyy)

**Curvatures** are the second derivative of elevation. They describe convexity/concavity of terrain. There are many variants (profile, tangential, planform, flowline, maximal, minimal), but they all require second-order partial derivatives:
- zxx (∂²z/∂x²)
- zxy (∂²z/∂x∂y)
- zyy (∂²z/∂y²)

These **cannot** be computed from first-order derivatives alone. They require either:
- **Zevenbergen & Thorne (1987)**: partial quartic polynomial fit to a 3×3 window → gives zx, zy, zxx, zxy, zyy
- **Florinsky (2009)**: third-order polynomial fit to a 5×5 window → more robust, same derivatives

Horn's algorithm (what we currently use) only computes zx and zy. It **cannot** produce curvatures.

**Impact on the prepare pass:** To support curvatures in the future, we would need to either:
1. Switch from Horn to Zevenbergen-Thorne in the prepare pass (still 3×3, but different math, and store zxx/zxy/zyy in additional channels — would need a second FBO or RGBA16F).
2. Use Florinsky (5×5 window = 25 taps instead of 9) for higher quality.
3. Compute curvatures in a separate prepare pass that stores them directly as scalars.

Option (3) is simplest: add a `curvaturePrepare` pass that runs alongside `demPrepare` when a curvature attribute is requested. The cost is one extra FBO per tile, but curvature is a niche use case.

### What requires windowed statistics (not derivatives at all)

These attributes operate on the **raw elevation values** in a neighborhood window, not on derivatives:

| Attribute | Definition | Window | Computable from dx/dy? |
|-----------|-----------|--------|----------------------|
| **TRI** (Terrain Ruggedness Index) | Mean or RMS difference between center pixel and neighbors | 3×3 (default, up to NxN) | ❌ No — needs raw elevation differences |
| **TPI** (Topographic Position Index) | Center elevation minus mean of neighbors | 3×3 (default, up to NxN) | ❌ No — needs raw elevations |
| **Roughness** | Max elevation difference in window | 3×3 (default, up to NxN) | ❌ No — needs raw elevations |
| **Rugosity** | Ratio of real 3D surface area to planimetric area | 3×3 only | ❌ No — needs 3D triangle areas |
| **Fractal roughness** | Local fractal dimension via box-counting | ≥5×5 (default 13×13) | ❌ No — needs large window, iterative |

These cannot use the shared prepare pass at all. They would need their own computation, either:
- A dedicated prepare pass per attribute type, or
- Computed in the single-pass fragment shader (feasible for 3×3 window attributes like TRI/TPI/roughness, expensive for larger windows).

For 3×3 windowed attributes (TRI, TPI, roughness), the cost is similar to the current terrain-analysis single-pass approach: 9 DEM taps per pixel. These could share the same "raw elevation" data if we stored the full 3×3 neighborhood, but that's 9 floats — impractical for an FBO.

**Practical recommendation:** TRI, TPI, and roughness can be implemented as single-pass shaders reading the raw DEM (like terrain-analysis does today). They don't benefit from the derivative cache. Rugosity and fractal roughness are too expensive for real-time GPU rendering and are better suited to offline/CPU computation.


### Summary: attribute taxonomy

```
DEM-derived attributes
├── First-order derivatives (dx, dy) ← shared demPrepare FBO
│   ├── slope
│   ├── aspect
│   └── hillshade (+ illumination parameters)
│
├── Second-order derivatives (zxx, zxy, zyy) ← needs extended prepare or separate pass
│   ├── profile curvature
│   ├── tangential curvature
│   ├── planform curvature
│   ├── flowline curvature
│   ├── maximal curvature
│   └── minimal curvature
│
├── Windowed statistics (raw elevations) ← single-pass from DEM, no caching benefit
│   ├── TRI (terrain ruggedness index)
│   ├── TPI (topographic position index)
│   ├── roughness
│   └── rugosity (3×3 only, 3D surface area)
│
└─ Frequency domain ← requires FFT, not feasible in real-time
    └── texture shading (fractional Laplacian)

```

---

## Phase 2 (future): Curvature Support

If curvature attributes are desired:

1. Add `curvature-profile`, `curvature-tangential`, etc. as `terrain-analysis-attribute` enum values.
2. Add a `curvaturePrepare` pass using Zevenbergen-Thorne (3×3) that stores zxx, zxy, zyy in an additional FBO.
3. The terrain-analysis render pass reads from the curvature FBO when a curvature attribute is selected.
4. The prepare pass only runs when a curvature attribute is actually used.

## Phase 3 (future): Windowed Attributes

TRI, TPI, and roughness can be added as single-pass terrain-analysis attributes. They read the raw DEM (9 taps for 3×3 window) and don't benefit from the derivative cache. This is acceptable performance for these simpler computations.
