# slope-builtin — Built-in Terrain Analysis Demonstrator

A single-page demo of the new **`terrain-analysis`** layer type and **`blend-mode`** raster paint property.
Unlike `slope-custom.html` which uses a hand-written WebGL custom layer, this version uses only
built-in MapLibre GL JS features — **no custom shaders required**.

**File:** [`slope-builtin.html`](slope-builtin.html)

---

## Key features

### Built-in `terrain-analysis` layer
The `terrain-analysis` layer type replaces the need for a custom WebGL layer.
It reads a `raster-dem` source directly and computes the selected scalar attribute
(slope, aspect, or elevation) on the GPU via built-in shaders:

```json
{
  "id": "analysis",
  "type": "terrain-analysis",
  "source": "dem",
  "paint": {
    "attribute": "slope",
    "opacity": 0.6,
    "color": [
      "interpolate", ["linear"], ["slope"],
      0,  "#00ff00",
      15, "#ffff00",
      30, "#ff8800",
      45, "#ff0000",
      60, "#aa0000",
      90, "#550000"
    ]
  }
}
```

Paint properties:

| Property | Type | Description |
|---|---|---|
| `attribute` | `"slope"` \| `"aspect"` \| `"elevation"` | Which scalar to compute from DEM |
| `opacity` | `number` (0–1) | Layer opacity |
| `color` | `interpolate` or `step` expression | Color ramp keyed on `["slope"]`, `["aspect"]`, or `["elevation"]` |

### Blend mode on raster layers
The basemap is a standard `raster` layer with the new **`blend-mode`** paint property:

```json
{
  "id": "basemap",
  "type": "raster",
  "source": "osm",
  "paint": {
    "blend-mode": "multiply"
  }
}
```

Available values:

| Value | Effect |
|---|---|
| `"normal"` | Standard premultiplied-alpha blending (default) |
| `"multiply"` | Multiplies colors with the layer beneath — darkens, preserves detail |
| `"screen"` | Inverted multiply — lightens the result |

### Mode toggle (combobox)
A single `<select>` switches between **Slope**, **Aspect**, and **Color Relief** (elevation).
All three modes use the same `terrain-analysis` layer — only the `attribute`
and `color` paint properties are updated at runtime via `setPaintProperty`:

```js
map.setPaintProperty('analysis', 'attribute', 'elevation');
map.setPaintProperty('analysis', 'color', colorRampExpression);
```

### Dynamic legend
The legend auto-updates its gradient and labels when the mode changes.

---

## Comparison with `slope-custom.html`

| | `slope-custom.html` | `slope-builtin.html` |
|---|---|---|
| **Layer type** | `custom` (WebGL) | `terrain-analysis` (built-in) |
| **Shader code** | ~80 lines GLSL in JS | None — handled by MapLibre |
| **DEM tile management** | Manual fetch, cache, eviction | Automatic via `raster-dem` source |
| **Blend compositing** | Manual `gl.blendFunc` | `blend-mode: "multiply"` paint property |
| **Color ramp** | Uniform arrays, manual parsing | Standard style expressions (`interpolate` / `step`) |
| **Elevation mode** | Not supported | Built-in via `attribute: "elevation"` |
| **Tile loading** | Re-implements tile logic | Uses MapLibre's tile pipeline |
| **Lines of JS** | ~400 | ~60 |

---

## Running

```bash
# Build the local dev bundle
npm run build-dev

# Serve from the repo root
npx http-server -p 8080

# Open in browser
open http://localhost:8080/slope-builtin.html
```
