# Slope Protocol Plan (MapLibre-Contour vNext, Backward-Compatible)

## 0) Decision and scope

The goal is to display an overlay with colors based on slope classes (eg yellow for slopes between 30 and 35 degrees, etc.]

The data source would be the same as supported for contour and hillshading, ie a DEM in eg Terarrium RGB format.

This plan switches to **extending `maplibre-contour`** with an additive slope/aspect/... feature set.
The enclosing maplibre project is provided as context but all code should occur in ./maplibre-contour

### Why this direction

1. `maplibre-contour` already has the right architecture for protocol-backed derived tiles.
2. It already solves hard problems we need (worker offload, DEM caching, seam-safe neighbor fetch).
3. We can keep a clean API for users and preserve backward compatibility.

Key existing anchors:

- Public API pattern (`DemSource`, `setupMaplibre`) in @maplibre-contour/README.md#21-33
- Shared DEM protocol URL in @maplibre-contour/README.md#96-106
- Protocol registration in @maplibre-contour/src/dem-source.ts#170-175
- Worker-dispatch architecture in @maplibre-contour/src/worker-dispatch.ts#18-76

---

## 1) Direct answers to product questions

### 1.1 Will this fix the 3D draping bug?
**Yes, expected.**

By rendering slope as a protocol-backed native `raster` source/layer, MapLibre handles:

- tile visibility/covering
- terrain draping
- depth ordering

This avoids custom-layer divergence that caused “disappearing near camera” regressions.

### 1.2 Can we still hook into common DEM / tile cache?
**Yes, with high overlap.**

Use one shared DEM manager for:

- contour vector tile generation
- slope raster tile generation
- optional `raster-dem` shared source (`sharedDemProtocolUrl`)

This is cleaner than touching MapLibre private caches and remains stable across MapLibre versions.

### 1.3 Inefficiencies and mitigation
Main costs:

1. Slope CPU compute per tile
2. Raster encoding cost
3. Risk of duplicate processing across derived products
4. Seam-neighbor fetch overhead

Mitigation:

- Shared raw/parsed DEM caches
- Shared padded “virtual tile” cache
- Dedicated slope output cache keyed by z/x/y + style params
- Worker path default for slope processing
- Quantized style params to improve cache hit rate

### 1.4 Backward compatibility viability
**Strong yes** if we keep `DemSource` API additive.

No breaking changes required for current contour users.

---

## 2) Backward-compatible API proposal (user-facing)

## 2.1 Existing API kept unchanged

Must remain stable:

1. `new DemSource(options)`
2. `demSource.setupMaplibre(maplibregl)`
3. `demSource.contourProtocolUrl(options)`
4. `demSource.sharedDemProtocolUrl`

## 2.2 Additive slope API

Add to `DemSource`:

```ts
type GlobalAnalysisTileOptions = {
  mode?: 'slope' | 'aspect';
  units?: 'degrees' | 'percent';
  ramp?: string; // named ramp or encoded custom ramp
  alpha?: number;
  smooth?: boolean;
  exaggeration?: number;
};

class DemSource {
  // existing fields/methods ...
  analysisProtocolUrl(options: GlobalAnalysisTileOptions): string;
}
```

Expected usage:

```ts
const demSource = new mlcontour.DemSource({
  url: 'https://tiles.mapterhorn.com/{z}/{x}/{y}.webp',
  encoding: 'terrarium',
  maxzoom: 14,
  worker: true,
  cacheSize: 256
});

demSource.setupMaplibre(maplibregl);

map.addSource('slope', {
  type: 'raster',
  tiles: [demSource.analysisProtocolUrl({mode: 'slope', units: 'degrees', ramp: 'default'})],
  tileSize: 512,
  maxzoom: 14
});
```

## 2.3 Optional future API (not required for v1)

- `analysisProtocolUrl({type: 'contour' | 'slope', ...})`

Keep as future consolidation; do not block initial delivery.

## 2.4 Color-ramp syntax strategy (clarification)

Users write style expressions like:

```json
["step", ["slope"], "#ffffff", 30, "#ff8800", 45, "#ff0000"]
or
[
    "interpolate",
    ["linear"],
    ["slope"],
    0, "transparent",
    90, "black"
]
```

- Keep protocol slope output as colored raster tiles for portability and API consistency with contour.
- Add a **MapLibre-like ramp parser subset** in maplibre-contour for slope options:
  - support `step` and `interpolate`
  - support inputs tied to scalar domain (`slope`, `aspect`, optional `elevation`)
  - no zoom-dependent expressions in protocol parser v1.

This preserves familiar syntax while keeping plugin runtime independent of MapLibre internals.

---

## 3) Internal architecture changes in maplibre-contour

## 3.1 `DemSource` extension

In @maplibre-contour/src/dem-source.ts:

1. Add `analysisProtocolId`, `analysisProtocolUrlBase`
2. Register `maplibre.addProtocol(this.analysisProtocolId, this.analysisProtocol)`
3. Implement `analysisProtocolV4` analogous to `contourProtocolV4`
4. Add `analysisProtocolUrl(options)` encoder/decoder path

Keep `shared` + `contour` protocol behavior untouched.

## 3.2 DemManager interface extension

In @maplibre-contour/src/types.ts and implementations:

Add:

```ts
fetchSlopeTile(
  z: number,
  x: number,
  y: number,
  options: IndividualAnalysisTileOptions,
  abortController: AbortController,
  timer?: Timer
): Promise<SlopeTile>;
```

Implement in:

- `LocalDemManager`
- `RemoteDemManager`
- worker dispatch bridge

## 3.3 Worker-dispatch changes

In @maplibre-contour/src/worker-dispatch.ts add handler similar to `fetchContourTile`:

- `fetchSlopeTile(...)`
- transfer ArrayBuffer efficiently

## 3.4 New slope pipeline modules

Add modules (example layout):

- `src/slope/slope-kernel.ts` (Horn + texel-center)
- `src/slope/slope-color.ts` (ramp LUT)
- `src/slope/slope-encode.ts` (PNG/WebP bytes)
- `src/slope/slope-options.ts` (option encode/decode)

Reuse existing `HeightTile` and neighbor-combine path where possible.

---

## 4) Carry-over of previous slope optimizations

From @SLOPE-SHADER.md and slope-hybrid-border.html:

1. Texel-center derivative evaluation
2. Seam-safe padded neighbors/backfill
3. Horn consistency (no endpoint-phase drift)
4. Deterministic ramp interpolation
5. Support both `step` and `interpolate` color ramps

These become **non-negotiable correctness requirements** in the new CPU/worker slope pipeline.

---

## 5) Versioning and compatibility policy

Target: `maplibre-contour` minor release (e.g., `0.2.x`) with additive API.

Compatibility commitments:

1. Existing contour integrations require no changes.
2. Existing generated URLs keep their semantics.
3. Existing worker and non-worker modes remain supported.
4. MapLibre v3/v4 interop path remains intact (current protocol compatibility path preserved).

If a behavior change is unavoidable, gate behind explicit option flag and default to legacy behavior.

---

## 6) Detailed implementation roadmap (junior engineer)

## Phase A — API + type scaffolding (Day 1)

1. Add slope option types in `src/types.ts`.
2. Add URL encode/decode for slope options.
3. Add `DemSource.analysisProtocolUrl` and protocol IDs.

Deliverable: compile passes, no runtime logic yet.

## Phase B — Local slope generation (Days 1–2)

1. Implement slope scalar generation from `HeightTile`.
2. Apply seam-safe neighbor composition.
3. Implement raster encoding to tile bytes.
4. Add slope cache in `LocalDemManager`.

Deliverable: `fetchSlopeTile` works in main-thread mode.

## Phase C — Worker pipeline (Days 2–3)

1. Add worker dispatch methods and response transferables.
2. Extend `RemoteDemManager` calls.
3. Ensure parity with local mode.

Deliverable: worker mode returns slope tiles and matches local results.

## Phase D — MapLibre integration demo (Day 3)

1. Update `test-project` with slope source/layer example.
2. Demonstrate terrain + hillshade + slope overlay.
3. Keep contour example unchanged in same demo.

Deliverable: side-by-side contour + slope from one `DemSource`.

## Phase E — Hardening and docs (Days 4–5)

1. Document new API in README with backward compatibility note.
2. Add migration FAQ (“nothing breaks; slope is additive”).
3. Add perf metrics hooks and sample outputs.

Deliverable: releasable package docs.

---

## 7) Test plan (must-pass)

## 7.1 Unit tests

1. Slope kernel correctness on synthetic DEM fixtures
2. Texel-center regression tests
3. Neighbor seam continuity tests
4. Slope option encode/decode and cache-key stability
5. Ramp interpolation/gamma tests
6. Ramp parser parity tests for `step` and `interpolate` semantics

## 7.2 Integration tests (package level)

1. `DemSource` registers all protocols (`shared`, `contour`, `slope`)
2. `analysisProtocolUrl` returns valid tile bytes
3. Cache hit behavior under repeated requests
4. Abort behavior under cancellation
5. Worker/local parity

## 7.3 Compatibility tests

Reuse and extend existing build tests:

- `test-build` (MapLibre v4 path)
- `test-build-v3` (legacy path)

Acceptance: existing contour tests stay green unchanged.

## 7.4 E2E map behavior tests

Playwright scenarios:

1. Terrain ON/OFF slope visibility
2. High pitch/high zoom stability
3. Rapid pan/zoom churn
4. No custom-layer disappearance behavior

---

## 8) Risks and mitigations

1. **Encoding overhead**
   - mitigate with output cache + efficient encoder settings
2. **Parameter explosion reducing cache hit rate**
   - mitigate with option normalization/quantization
3. **Worker serialization overhead**
   - transfer buffers, avoid extra copies
4. **Contour regressions from shared internals**
   - isolate slope path and preserve contour defaults

---

## 9) Migration strategy from current custom slope layer

1. Add runtime flag: `engine=custom|protocol` in example app.
2. Integrate protocol-backed slope first without removing custom path.
3. Run visual + perf comparisons on known problematic views.
4. Flip default to protocol after two stable iterations.
5. Remove custom path only after sign-off.

---

## 10) Definition of done

1. `maplibre-contour` exposes additive `analysisProtocolUrl` API.
2. Existing contour API usage remains valid without changes.
3. Slope drapes correctly with terrain in native pipeline.
4. Previously failing high pitch/high zoom scenarios no longer disappear.
5. Seam/artifact behavior follows constraints from @SLOPE-SHADER.md.
6. Tests pass for unit, integration, v3/v4 compatibility.
