## MapLibre 3D terrain rendering pipeline — background

### Key concepts

**Framebuffer Object (FBO)**: A GPU-side "off-screen canvas." Instead of drawing directly to the screen, you draw into an FBO's texture. Later, you can use that texture as an image — e.g., paste it onto a 3D mesh. Think of it as rendering to a photo, then sticking the photo onto a surface.

**Render-to-Texture (RTT)**: The technique of rendering 2D map layers (fills, lines, raster, hillshade, terrain-analysis) into FBO textures, then draping those textures onto the 3D terrain mesh. Without RTT, these layers would render flat (ignoring terrain elevation).

**The render pool**: MapLibre keeps a pool of 30 reusable FBOs (each 1024×1024). For each terrain tile visible on screen, it borrows an FBO from the pool, renders layers into it, then drapes that texture onto the terrain, and returns the FBO to the pool.

### How layers become stacks

Not all layers can be rendered to texture. The `LAYERS_TO_TEXTURES` table:
- **Texture-able**: background, fill, line, raster, hillshade, color-relief, terrain-analysis
- **Not texture-able ("live")**: circle, symbol, heatmap, fill-extrusion, custom

MapLibre processes layers bottom-to-top. Consecutive texture-able layers are grouped into a **stack** — they all render into the same FBO, compositing on top of each other. When a non-texture layer appears (e.g., circle), it **breaks** the stack.

```
hillshade     ─┐
fill          ─┤ Stack 0 (all render into one FBO)
line          ─┘
circle         ← rendered live (not in FBO), BREAKS the stack
line          ─┐
terrain-analysis ─┤ Stack 1 (new FBO, starts empty)
              ─┘
```

Each stack gets its own freshly-cleared FBO (`rgba(0, 0, 0, 0)` — transparent black).

### The draping step

After rendering all layers in a stack into FBOs (one per terrain tile), `drawTerrain()` drapes them:
1. Binds the screen framebuffer
2. For each terrain tile, binds the FBO texture
3. Draws the terrain 3D mesh with that texture using **alpha blending** (`ONE, ONE_MINUS_SRC_ALPHA`)

This composites the stack's content onto whatever is already on screen. Then the pool is freed, and the next stack starts.

### The multiply problem, precisely

**Within the FBO**, terrain-analysis with multiply blend does:
```
result_rgb = src_rgb × dst_rgb     (DST_COLOR, ZERO)
result_alpha = dst_alpha           (ZERO, ONE)
```

In Stack 0 (with hillshade below it), `dst` has hillshade content → multiply works.

In Stack 1 (terrain-analysis alone), the FBO was just cleared to `(0,0,0,0)`. Multiplying `src × 0 = 0`. The texture is empty. When draped, nothing appears.

**The fix we need**: multiply should happen between the terrain-analysis output and the **basemap** (what's already on screen), not between terrain-analysis and whatever happens to be in the same FBO.

### Why screen "works" by accident

Screen blend: `result = src + dst × (1 - src)`. When `dst = 0`, result = src — the source colors survive. The alpha is still 0, but when draped with `(ONE, ONE_MINUS_SRC_ALPHA)` and alpha=0, the draping becomes additive (`screen_rgb + existing_rgb`), which is visually close to screen blending. Lucky accident.

---

## Fix strategy for upstream

The key insight: **blend mode is the wrong abstraction level inside the FBO**. Inside the FBO, you're compositing layers-within-a-stack (hillshade + terrain-analysis). The blend mode (multiply) is meant to describe how the layer interacts with **the final scene** (basemap + everything below) — not with the FBO.

### The upstream-friendly approach: apply blend mode at drape time

This aligns with how the existing pipeline already works conceptually:

1. **Inside the FBO**: Layers render with normal alpha blending. The FBO texture accumulates the correct colors and alpha of the terrain-analysis layer.

2. **At drape time** (`drawTerrain`): The blend mode is applied when compositing the FBO texture onto the 3D terrain mesh. This is where the layer meets the basemap — where multiply should happen.

The change touches two layers of the pipeline:

**A) `render_to_texture.ts`** — Track the blend mode for each stack:
```
// _stacks changes from Array<Array<string>> to:
_stacks: Array<{layers: Array<string>, blendMode?: string}>
```

When building a stack, if any layer has a non-standard blend mode, record it. Inside the FBO rendering, always use `ColorMode.alphaBlended` regardless of the layer's blend-mode property.

**B) draw_terrain.ts / `drawTerrain()`** — Accept and apply the blend mode:
```
// Currently:
const colorMode = painter.colorModeForRenderPass();  // always alphaBlended

// With fix:
const colorMode = stackBlendMode === 'multiply'
  ? ColorMode.multiply
  : painter.colorModeForRenderPass();
```

Now multiply happens against the **screen content** (basemap + lower stacks) — exactly what we want.

### Why this is upstream-friendly

- **No hack**: It's a genuine architectural correction. Blend modes semantically belong at the composition boundary (FBO → screen), not inside the FBO.
- **No new GPU features**: Uses the same WebGL blend equations, just at the right stage.
- **Backward-compatible**: Layers without blend-mode are unaffected. The default path through `drawTerrain` is identical.
- **Stack-position-independent**: Works whether terrain-analysis is in its own stack or shares one with hillshade. The FBO always has correct colors; the drape step always applies the right blend.
- **Generalizable**: Same pattern works for screen, darken, lighten, and any future blend modes.

### The subtlety: multiply + hillshade in same stack

When terrain-analysis is in the **same stack** as hillshade (no circle layers between them), the current code gives: `multiply(terrain-analysis, hillshade)` inside the FBO, then `alphaBlend(result, basemap)` at drape.

With the fix: `alphaBlend(terrain-analysis, hillshade)` inside the FBO, then `multiply(result, basemap)` at drape.

These give different visual results: currently you get `multiply(slope, hillshade)` composited normally onto the basemap. With the fix you get `multiply(alphaBlend(slope, hillshade), basemap)`. The second is arguably more correct — multiply the combined terrain layers against the basemap — but it's a visual change.

If you want **identical behavior** when same-stack, you'd apply multiply inside the FBO only when there's content to multiply against, and **also** at drape time. But that's over-engineered. The drape-time approach gives the correct semantic: "this layer's output multiplies against everything below."

Want me to implement this?

# Alternative Fix strategies considered

The core problem: when a blend mode layer (multiply/screen) ends up in its own RTT stack, it renders into a fresh FBO cleared to `rgba(0,0,0,0)`. Multiplying against zero = zero.

### Strategy 1: Two-pass rendering in the FBO (chosen)

When a layer uses a non-standard blend mode, render it in **two passes** within the FBO:
1. **Pass 1**: Draw the layer with `ColorMode.alphaBlended` (normal alpha) into the FBO — this writes the slope/elevation colors into the texture correctly.
2. The blend mode (multiply/screen) is then applied **only at terrain drape time**, in `drawTerrain()`.

This requires `drawTerrain` to know the blend mode of the stack. You'd store the blend mode alongside the stack in `_stacks`, and pass it to `drawTerrain` which would use it instead of the default `colorModeForRenderPass()`.

**Pros**: Architecturally clean, correct regardless of stack position.  
**Cons**: Requires plumbing blend mode info from the RTT stacks to `drawTerrain`. Mixed blend modes within one stack (unlikely but possible) would need care.

### Strategy 2: Prime the FBO with a white fill before multiply layers

Before rendering a multiply-blend layer into a fresh FBO, fill it with `rgba(1,1,1,1)` (the multiply identity). For screen, fill with `rgba(0,0,0,1)` (the screen identity).

In `render_to_texture.ts`, after `context.clear({color: Color.transparent})`, check if the first layer in the stack uses a non-standard blend mode and draw a full-screen white/black quad first.

**Pros**: Minimal code change — just a conditional fill.  
**Cons**: Semantically wrong: the "background" of the FBO becomes opaque white, which composites incorrectly when `drawTerrain` drapes it with alpha blending on top of existing screen content (you'd see a white rectangle behind the terrain).

### Strategy 3: Skip RTT for blend-mode layers (render directly)

Remove `'terrain-analysis'` from `LAYERS_TO_TEXTURES` when it uses a blend mode, letting it render directly to the screen framebuffer like symbols/circles.

**Pros**: Simple — the blend mode works because the screen already has content.  
**Cons**: Breaks 3D terrain draping entirely — the layer would render flat, ignoring the terrain mesh. Not viable for a DEM-draped layer.

### Strategy 4: Force terrain-analysis into the same stack as hillshade

In `renderLayer()`, detect when a terrain-analysis layer has a non-standard blend mode and force it into the most recent RTT stack instead of letting circle layers break the stacking.

**Pros**: Preserves the existing multiply-against-hillshade behavior.  
**Cons**: Fragile — depends on hillshade being present. Doesn't solve the general problem for arbitrary stack orderings.

### Strategy 5: Accumulate all stacks into one texture per tile

Instead of rendering each stack into the pool, flushing to screen, and freeing — render **all** stacks' layers into a single persistent texture per tile. This is an architectural refactor of the RTT pipeline to use one FBO per tile with all layers composited sequentially.

**Pros**: Eliminates the multi-stack problem entirely. Better performance (one `drawTerrain` call instead of N).  
**Cons**: Largest code change. Changes how non-RTT layers (symbols) interleave with RTT layers visually.

---