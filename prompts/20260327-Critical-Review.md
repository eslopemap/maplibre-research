## Critical Review

### `multiplyDrape` math verification

`multiplyDrape = [DST_COLOR, ONE_MINUS_SRC_ALPHA, ZERO, ONE]`:
`out.rgb = dst × src.rgb + (1 - src.a) × dst.rgb = dst × (src.rgb + 1 - src.a)`

With premultiplied FBO: `src.rgb = color × opacity`, `src.a = opacity`:
`out.rgb = dst × (color × opacity + 1 - opacity)` ✓ correct (passthrough when `opacity=0`, full multiply when `opacity=1`)


---

### 🔴 Issue 1: The existing `multiply` render test does NOT exercise the FBO path

```
test/integration/render/tests/terrain-analysis/multiply/style.json
  layers: [background, terrain-analysis/multiply]
```

No 3D terrain source, no `terrain` property → `isRenderingToTexture = false` throughout. The `getBlendModeState` branch `if (isRenderingToTexture && blendMode === 'multiply')` is **never hit** by any render test. The critical fix in `35a606f` is untested by integration tests.

---

### 🟡 Issue 2: Multi-stack blend mode leakage (edge case, but real)

In render_to_texture.ts, `currentStack.blendMode = 'multiply'` is set for the **whole stack** when ANY `terrain-analysis` layer within it uses multiply. If a stack contains:
- `terrain-analysis` (normal/screen blend)
- `terrain-analysis` (multiply)

…then `multiplyDrape` is applied to the entire draped FBO texture, incorrectly multiply-compositing the non-multiply layers. This isn't triggered by current demos but will surface when mixing blend modes across layers in the same stack.

---

### ✅ What IS working correctly

- The RTT stack-isolation fix (`35a606f`) is properly structured: `isRenderingToTexture && multiply` → render normally into FBO → drape with `multiplyDrape`
- `terrain-analysis` is in `LAYERS_TO_TEXTURES` ✓
- `_getDrapeColorMode` wiring through `drawTerrain` ✓
- The double-premultiply fix in the shader (`759cfea`) is correct
- `screen` blend is unaffected by FBO contents and needs no special RTT handling ✓

---

Now creating the multi-layer demo that demonstrates the FBO fix in a visible, testable way:

Starting: *Create multi-layer FBO demo HTML* (2/2)

Created [](local_demos/slope-blend-fbo-demo.html)

Now fix the slope-builtin-published.html to reference the correct version (5.21.3):

Read [](slope-builtin-published.html#1-1), lines 1 to 10

Replacing 2 lines with 3 lines in [](slope-builtin-published.html)


```

```


Completed: *Create multi-layer FBO demo HTML* (2/2)

---

## Summary

### Critical Review findings

| # | Severity | Issue |
|---|----------|-------|
| 1 | 🔴 | **Package 5.21.3 not published** — slope-builtin-published.html was pinned to 5.21.2 (now fixed to 5.21.3, but still needs `npm publish` to run) |
| 2 | 🔴 | **FBO path has zero render test coverage** — the `multiply` render test has no 3D terrain source so `isRenderingToTexture` is never `true`; the key fix in `35a606f` is unvalidated |
| 3 | 🟡 | **Blend-mode leakage across layers in same RTT stack** — if a future style mixes a `multiply` and a `normal` terrain-analysis layer in the same RTT stack, the entire FBO gets `multiplyDrape`; per-layer drape is not supported |
| 4 | ✅ | `multiplyDrape` math is correct for premultiplied FBO textures |
| 5 | ✅ | `getBlendModeState` RTT override correctly routes multiply to normal alpha blend inside FBO |

### New demo: slope-blend-fbo-demo.html

This deliberately tests the FBO scenario:

- **"Stack break" checkbox** (on by default): inserts an empty `geojson/symbol` layer between the basemap and the terrain-analysis layers — exactly the condition that was rendering multiply as **invisible before the fix** (symbol layer flushes the RTT stack, terrain-analysis goes into its own FBO cleared to transparent black)
- **Two terrain-analysis layers** (slope with multiply, elevation with normal) in the isolated stack
- **3D terrain toggle**: activates the RTT/FBO path; the stack diagram updates to show `Stack A` and `Stack B` and labels when `multiplyDrape` is in effect
- A live **stack diagram** in the panel shows the exact render order, stack groupings, and drape mode so you can reason about what the GPU is doing

### What to do next

1. **`npm publish`** from the `blend` branch — 5.21.3 is working but not on npm
2. Add a **3D render test** for multiply: a style with `terrain` source + `terrain-analysis/multiply` + a non-RTT layer forcing stack isolation
3. Consider whether to track blend mode **per-layer within a stack** (currently whole-stack granularity) to fix the leakage edge case

Made changes.
