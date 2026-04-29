User: I am getting a slightly too dark rendering of the multiply blend, when I set alpha of the upper layer to ~50%. the results are good closer to 0 and 100% opacity

---

### Current ImplementationContext
Your `multiply` is currently a mathematically true sRGB multiply.
In raster.fragment.glsl and `terrain_analysis.fragment.glsl`, the shader does this for multiply:
`fragColor = vec4(mix(vec3(1.0), rgb, opacity), opacity)`
Combined with `glBlendFunc(gl.DST_COLOR, gl.ZERO)`, the destination pixel becomes:
`Dst * (1.0 - a + Src * a)`
This equals **`Dst * (1.0 - a) + (Dst * Src) * a`**. This perfectly matches the Photoshop/CSS definition of alpha-composited multiply.

**Why does it look too dark at 50% opacity?**
Because this mathematical interpolation is happening in **sRGB** color space. In sRGB, a value of `0.5` represents significantly less than 50% physical light (due to the 2.2 gamma curve). So cross-fading down to `0.5` opacity causes the midpoints to dip aggressively into darker luminosity bands.

Here is how the recommendations map:

### 1. Multiply in linear space
* **Mapping**: The industry-standard solution to the "dark midpoints" problem.
* **Feasibility in MapLibre**: WebGL handles this automatically **if** the framebuffers use an sRGB internal format (like `gl.SRGB8_ALPHA8`).  However, MapLibre GL JS currently draws into standard `gl.RGBA` FBOs and assumes an sRGB pipeline. Shifting the core FBO architecture to SRGB textures would be deeply invasive, potentially breaking `normal` alpha blending for dozens of other layer types.
* **Verdict**: Too disruptive to implement directly just for one blend mode.

### 2. Soft multiply (if constrained to blend funcs)
* **What it is**: Algorithms like "Pegtop Blend" or specific variants of multiply that soften the absolute black point to mimic light absorption better than mathematical multiplication.
* **Feasibility in MapLibre**: **High**. Because MapLibre currently applies multiply in a single pass using built-in `gl.blendFunc` hardware (avoiding expensive FBO readbacks), any formula we can build out of `vec4 shader_output` + `glBlendMode(src_factor, dst_factor)` is basically "free" in terms of performance.
* **Verdict**: If the user provides a soft-multiply algebra that fits within the source output and WebGL's blend factors, this is excellent.

### 3. Luma‑compensated multiply (shader path)
* **What it is**: Computing the luminance of the backdrop and adjusting the intensity of the multiply to prevent crushing shadows.
* **Feasibility in MapLibre**: **Low for direct layers, High for 3D drapes**. In WebGL, a shader can't read the pixel it's currently drawing on top of without doing a costly frame buffer copy (ping-ponging FBOs).
* **However**, your fork notes that for 3D terrain, you already have a `multiplyDrape` stage where the analysis layer is on a texture and being applied over the main map FBO. *At that specific step*, an FBO read/shader path is possible. But it wouldn't work easily for standard 2D stacked layers.
* **Verdict**: Best visual quality, but breaks the single-pass direct render architecture for 2D.

### What Could Be a Good Default?
**Standard `multiply` should remain the default.**
It exactly matches what designers expect from CSS `mix-blend-mode: multiply` and Adobe tools. The fact that sRGB gradients look muddy at 50% is a well-known quirk of digital color, but overriding it by default might confuse users porting existing QGIS or Photoshop map stacks where they are already accustomed to accommodating the sRGB darkening.

### Should several multiply variants be available?
Yes, creating distinct enum values (e.g., `blend-mode: "multiply" | "soft-multiply"`) is the cleanest architectural path.

**Next Steps Strategy:**
If you want to try option 2 (Soft multiply func), you can alter your shader output to pass a modified color instead of `mix(1.0, rgb, alpha)` while keeping `[DST_COLOR, ZERO]`, or adjust the `ColorMode.multiply` blend factors in color_mode.ts. If you have the specific math for "soft multiply", I can help translate it into the `glBlendFunc` logic!

---

# Research soft-multiply WebGL blend formulae

User: implement 'soft-multiply'. research formulae that work well for map overlays and choose the best one to implement.

---

Based on the research and your requirement to use native `glBlendFunc` math with no added shaders, I've successfully implemented **`soft-multiply`** using the industry-standard composite technique known as **Modulate 2X**.

### Why Modulate 2X instead of the older formulas?

Modulate 2X resolves this via pure `glBlendFunc` settings:
*   **Formula:** $Out = 2 \times Dst \times mix(0.5, Src, opacity)$
*   **How it feels:** A 50% shading map linearly maps to `0.5`, which yields `2.0 * Dst * 0.5 = Dst`. This means a mid-gray slope does not darken the map. Only shadows (< 50%) will organically darken, and highlights (> 50%) actively brighten the sunlit slopes up to 2x without blowing them out.

### What I did in the project:
1.  **Updated Style Specification:**
    *   Added `"soft-multiply"` to v8.json.
    *   Rebuilt the style spec and ran the script to regenerate TypeScript property types and values (`raster_style_layer_properties.g.ts`).
2.  **Configured `ColorMode`:**
    *   Added standard 2X blending math to color_mode.ts: `[DST_COLOR, SRC_COLOR, ZERO, ONE]`.
3.  **Hooks in MapLibre render stack:**
    *   Linked `soft-multiply` in draw_terrain_analysis.ts and draw_raster.ts.
    *   We leverage the existing `u_blend_neutral` argument by passing `0.5` (instead of `1.0` used by standard multiply) so the shader outputs the correct midpoint without modifying *any* GLSL files.
    *   Handled the equivalent 3D draped pass (`softMultiplyDrape`) inside render_to_texture.ts.
4.  **Rebuilt Maps:**
    *   Run `npm run build-prod` to generate everything into maplibre-gl.js.
5.  **Demos modified:**
    *   Added Modulate 2X to your local mockup tool demo-multiply-too-dark.html for you to inspect the pure Javascript math.
    *   Added the `soft-multiply` option to the comboboxes inside both slope-builtin.html and slope-blend-fbo-demo.html.

You can now open slope-builtin.html in your browser and switch between "multiply" and "soft-multiply".

Made changes.

---

User:
i'm ok with 'Mod 2x' soft-multiply aspect up to an overaly opacity of around 0.5 or 0.6, but above, it gets too bright and the base color starts to disappear to become almost fully white at 1.0

( I am using http://localhost:9966/local_demos/slope-builtin.html to visualize )

ideally I would like to see the 'normal' multiply at 1.0
instead of C_out=C_dst×(1.0−α+C_src×α)
could we try:
C_out=C_dst×α×C_src
?
or is there another better way ?

---


Your proposal `C_out = C_dst × α × C_src` won't work (goes to black at α=0). Instead, I'll use an **α² power curve** with standard multiply blend:

$$C_{out} = C_{dst} \times (1 - \alpha^2 + \alpha^2 \times C_{src})$$

- At α=0: $C_{dst}$ (identity)
- At α=0.5: $C_{dst} \times (0.75 + 0.25 \times C_{src})$ — softer than standard multiply
- At α=1.0: $C_{dst} \times C_{src}$ — exact standard multiply

This reuses `[DST_COLOR, ZERO]` blend factors (same as multiply) and signals the α² curve via `u_blend_neutral = 2.0`.

**Approach: α² power curve** — instead of Mod 2X, soft-multiply now uses the same `[DST_COLOR, ZERO]` blend factors as standard multiply, but squares the alpha in the shader's mix interpolation:

```glsl
float effective_alpha = u_blend_neutral > 1.5 ? alpha * alpha : alpha;
fragColor = vec4(mix(vec3(1.0), rgb, effective_alpha), alpha);
```

**Behavior comparison at α=0.5 (mid-gray source):**

| Mode | Formula | Result |
|------|---------|--------|
| Standard multiply | $C_{dst}(0.5 + 0.5 \times C_{src})$ | 75% of $C_{dst}$ — too dark |
| Old Mod 2X | $2 \times C_{dst} \times C_{src}$ | 100% of $C_{dst}$ — correct for mid-gray but 200% at full opacity |
| **New soft-multiply** | $C_{dst}(0.75 + 0.25 \times C_{src})$ | 87.5% of $C_{dst}$ — gentle darkening |

**Key properties:**
- α=0 → identity (no change)
- α=0.5 → subtle, soft darkening (the "soft" part)
- α=1.0 → **exact standard multiply** $C_{dst} \times C_{src}$ (your requirement)

Files changed: raster.fragment.glsl, terrain_analysis.fragment.glsl, draw_raster.ts, draw_terrain_analysis.ts, render_to_texture.ts. The `u_blend_neutral = 2.0` signals the shader to use the α² curve (clamped to neutral=1.0 for the mix).

Made changes.

---

# Comparison with upstream MapLibre GL JS v6.18.0 layer blending

_Research date: 2026-11-15. Sources: maplibre-gl-js PRs [#8073](https://github.com/maplibre/maplibre-gl-js/pull/8073) (`{line,fill}-layer-blend`, merged 2026-11-09) and [#8986](https://github.com/maplibre/maplibre-gl-js/pull/8986) (`raster-layer-blend`, merged 2026-11-10), style-spec PRs [#1798](https://github.com/maplibre/maplibre-style-spec/pull/1798) and [#1964](https://github.com/maplibre/maplibre-style-spec/pull/1964) (style-spec 26.6.0), and the v6.18.0 tag (released 2026-11-11). The 5.x line (v5.24.1) does **not** have it._

## TL;DR

| | Fork (`blend` branch, based on 5.21) | Upstream v6.18 |
|---|---|---|
| Property | one shared `blend-mode` (raster, terrain-analysis) | one property per layer type: `fill-layer-blend`, `line-layer-blend`, `raster-layer-blend` |
| Values | `normal`, `multiply`, `soft-multiply`, `screen` | `normal`, `multiply`, `screen`, `overlay`, `plus-lighter`, `color`, `luminosity` (W3C Compositing-1 names) |
| Expressions | `data-constant`, zoom expressions allowed | `constant`, no expressions, no transition |
| Technique | single pass, **fixed-function `glBlendFunc`** + shader emits a "neutral" color (`u_blend_neutral`) | **offscreen layer FBO + backdrop copy + composite shader** (W3C formulas, premultiplied) |
| Cost | ~free (no extra pass) | per blended layer: clear FBO, draw layer, `copyTexSubImage2D` of backdrop, full-screen composite quad. Raster layers always use the full viewport. |
| Transparent backdrop | multiply against transparent black gives transparent black (the "invisible in 3D" bug), worked around with `multiplyDrape` / `rttDeferBlend` | handled by the W3C formula: `co = (1-αb)·Cs + (1-αs)·Cb + Cs·Cb`, so over transparent it acts like `normal` |
| 3D terrain | blend deferred to drape time (`_stacks[].blendMode`, `multiplyDrape` ColorMode) | no special handling: the composite runs *inside* each RTT tile texture, against the layers already in that stack |
| Color space | sRGB, RGBA8 | sRGB, RGBA8 (no linearization either) |
| Layer types | raster, terrain-analysis | fill, line, raster (no hillshade, color-relief, background or terrain-analysis yet) |

## How upstream works (6.18)

1. `drawRaster` / `drawFill` / `drawLine` check `*-layer-blend`. If it isn't `normal` (or if `*-layer-opacity < 1` for fill/line), they call `prepareDrawLayerComposite()`, which:
   - binds a shared scratch FBO `painter.layerCompositeFbo` (RGBA8 color plus depth-stencil, resized to the current target),
   - clears it to transparent and re-renders the tile clipping masks,
   - computes a scissor box from the projected tile corners (fill/line). Raster always uses the whole viewport because image sources can draw outside their tile.
2. The layer is drawn normally into the scratch FBO. Raster tile cross-fades and parent/child overlaps are flattened there first, so the layer is blended as one surface.
3. `drawLayerComposite()` rebinds the original target (canvas, or the per-tile RTT texture under terrain), then:
   - **`screen`, `plus-lighter`**: fixed-function, `[ONE, ONE_MINUS_SRC_COLOR]` and `[ONE, ONE]`. These are exact for premultiplied input, so no backdrop copy is needed.
   - **`multiply`, `overlay`, `color`, `luminosity`**: `copyTexSubImage2D` of the target's scissor box into `painter.layerCompositeBackdrop`. The `layer_composite.fragment.glsl` shader then `texelFetch`es both the layer and the backdrop and writes the W3C premultiplied result with `ColorMode.unblended` (replace).
4. `painter.ts` ends the opaque pass before the first shader-blended layer (`layerBlendsInShader`). This guarantees the backdrop is complete when it's copied.
5. Follow-up fix #9006 (also in 6.18) works around an Intel Gen12/Mesa glitch between consecutive composite layers.

Upstream's multiply, in premultiplied space:
```glsl
co = (1.0 - dst.a) * src.rgb + (1.0 - src.a) * dst.rgb + src.rgb * dst.rgb;
alpha = src.a + dst.a * (1.0 - src.a);
```

## Key finding 1: upstream `multiply` is just as dark at 50% as the fork's

For raster, `drawLayerComposite` is called with opacity `1`. `raster-opacity` is applied earlier, inside `raster.fragment.glsl`, which outputs premultiplied `(rgb·α, α)`. With an opaque backdrop (`dst.a = 1`) the formula above becomes:

$$C_{out} = (1-\alpha)\,C_b + \alpha\,C_s\,C_b = C_b\,(1 - \alpha + \alpha C_s)$$

This is **the same equation as the fork's `multiply`** (`DST_COLOR, ZERO` with `mix(1, Cs, α)`). Both run in sRGB on RGBA8, so the "too dark around 50% opacity" complaint carries over unchanged. Upstream doesn't add a soft or linear variant.

## Key finding 2: the fork's current `soft-multiply` is upstream `multiply` with opacity squared

The fork's α² soft-multiply is `C_b·(1 - α² + α²·C_s)`. That is upstream multiply with an effective opacity of α². So on stock 6.18 you get the same result with:
```json
"raster-layer-blend": "multiply",
"raster-opacity": 0.25   // = 0.5², or any zoom expression producing α²
```
The RGB output is exactly the same. Alpha differs: the fork keeps destination alpha (`[ZERO, ONE]` on the alpha channel), while upstream computes source-over alpha. That makes no difference over an opaque basemap. So **`soft-multiply` doesn't need its own enum value**; it's a styling choice. The older "Modulate 2X" variant has no upstream equivalent. `overlay` is the closest, but it switches on the *backdrop*, not the source.

## Key finding 3: upstream's architecture makes the "better" options cheap

The first section of this file rejected "linear-space multiply" and "luma-compensated multiply" because 2D layers can't read the destination. Upstream now *always* has the destination available (the `u_backdrop` texture) for shader-blended modes. Variants like these become small, local additions in `layer_composite.fragment.glsl`:
- `multiply` in linear light: `pow(c, 2.2)` (or the exact sRGB EOTF) on both inputs, multiply, then encode back. No sRGB framebuffers needed.
- luma-compensated / soft-light (W3C `soft-light` is not in the spec yet) using `dst.rgb` directly.

The extra cost is already paid by `multiply` itself, so these would add no new passes.

## Behavioural differences worth knowing

- **3D terrain, stack boundaries.** Upstream blends against what's already in the *same* RTT stack's tile texture. If a blended layer is the first one in a stack (for example, right after a symbol, circle or fill-extrusion layer), its backdrop is transparent. Multiply then falls back to `normal` and is draped with alpha. Nothing disappears (unlike the fork's original bug), but the multiply effect is silently lost. The fork's drape-time `multiplyDrape` does blend against the real screen in that case, so it's better there.
- **Overlapping raster tiles.** Upstream flattens tiles first and blends once. The fork blends every tile draw directly, so during fades, or wherever a parent and child tile overlap, the layer can be multiplied twice.
- **Performance.** The fork is single pass. Upstream adds roughly two full-viewport passes plus a framebuffer copy per blended raster layer. Under terrain this happens per RTT tile, but RTT tiles are cached between frames.
- **Precision.** Both use 8-bit. Upstream un-premultiplies only for `color` and `luminosity`.
- **Spec shape.** Upstream uses `constant` with no zoom expressions. The fork's `blend-mode` accepts zoom expressions. The names also differ (`raster-layer-blend` vs `blend-mode`).

## Recommendation for the fork

1. When rebasing onto 6.x, **drop the fork's fixed-function blend machinery**: `ColorMode.multiply`, `softMultiply`, `screen`, the `*Drape` modes, `u_blend_neutral`, the premultiplied flag, `rttDeferBlend` and `_stacks[].blendMode`. Adopt upstream's composite pipeline instead.
2. **Port `terrain-analysis` to the upstream pattern**: add a `terrain-analysis-layer-blend` property with the same enum as the other `*-layer-blend` properties, and wrap `drawTerrainAnalysis` in `prepareDrawLayerComposite` / `drawLayerComposite`. Also add the type to `layerBlendsInShader`. Both functions only need a union-type change, as #8986 did for raster.
3. **Retire `blend-mode` / `soft-multiply`** in favour of `*-layer-blend: multiply` plus `*-opacity` set to α². Document this in FORK.md, or keep `soft-multiply` only as a deprecated alias that squares the opacity.
4. If the 50% darkening is still unacceptable, **implement a linear-light multiply** as a new `#define` branch in `layer_composite.fragment.glsl`. Ideally propose it upstream as a style-spec value rather than adding fork-only enums again.
5. Keep a regression demo for the terrain stack-boundary case described above. It's the one scenario where the fork's current behaviour is better than upstream's.
