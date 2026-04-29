Context: prompts/20260329-Soft-multiply-research.md

User: based on slope-custom-6, give me the emulated css 'darken'  'soft-light' additional blend modes, and another luma-compensated one if useful. put it in a new file slope-custom-6-soft.html

I've added the extra blend modes in a new file, `local_demos/slope-custom-6-soft.html`, because they needed a substantial amount of new code. `slope-custom-6.html` is unchanged.

The Blend dropdown now has two groups:
- **Native:** the 7 modes MapLibre 6.18 supports, unchanged.
- **Emulated:** `darken`, `soft-light`, `multiply (linear light)` and `multiply (luma-compensated)`.

## Why the emulated modes need a custom layer
`raster-layer-blend` only offers those 7 modes, and `darken` and `soft-light` can't be built by stacking them. Both need the pixel already drawn underneath, and only a custom layer can read it. So for the emulated modes the raster layer is hidden and a custom layer draws instead. Each frame it:
1. Copies the framebuffer into a backdrop texture once (`copyTexSubImage2D`), which is the same trick 6.18 uses internally.
2. Draws the same slope/aspect tiles as the raster layer, through a shader that computes the blend from the W3C Compositing spec, including opacity and transparent areas underneath.

The tiles come from `map.coveringTiles()` plus the same tile-rendering function the raster layer's custom protocol uses, so there's no hand-written tile bookkeeping this time. While a tile loads, the matching part of an already-loaded lower-zoom tile is shown instead. At most 300 tile textures are kept, dropping the least recently used.

## The two multiply variants
- **`multiply (linear light)`:** multiplies and applies opacity in linear light instead of sRGB. This targets the "too dark around 50% opacity" problem from your soft-multiply notes, and at 100% it's close to normal multiply.
- **`multiply (luma-compensated)`:** takes the colour from multiply but keeps the basemap's original brightness (W3C `SetLum`). The result tints without darkening. On a mostly white basemap like OSM it gives pastel colours with labels fully readable. It's useful when you want the slope colours as a hint rather than a dominant layer.

## Testing (headless Chrome)
I compared the screen output with the formulas computed in JavaScript from two reference captures: basemap only, and the slope colours alone.
- **Native `multiply`:** matches `Cb·mix(1, Cs, α)` to within 1 level out of 255 at opacities 1, 0.5 and 0.25. So native multiply in 6.18 is just as dark at mid opacity as before; linear light is the fix for that.
- **`darken` and `soft-light`:** exact (error 0) in flat colour areas. The remaining differences sit only on the 1-pixel boundaries between colour bands. The native raster and my tile quads sample the tile slightly differently there, and shifting by a whole pixel doesn't remove it. It's barely noticeable on screen.
- I checked the linear-light and luma-compensated modes visually only. There were no page errors.

## Limitations
- **Main-thread cost:** the emulated modes add one full-screen copy per frame. The tile rendering still runs on the main thread, as in `-6`.
- **2D Mercator only:** the custom layer uses the plain Mercator matrix and doesn't implement `renderToTerrainTile`. In 3D terrain or globe view, only the native modes would display correctly.
- **Tile seams:** the tile-edge artefacts from earlier are unchanged in both versions. The padded-neighbour fix would apply to both, since they share the tile-rendering function.
