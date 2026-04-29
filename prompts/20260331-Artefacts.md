User:
read FORK.md for context
there are artifacts (wrong slope, lower than expected) at the edges between some kind of tiles (not sure if source DEM tiles or some internal maplibre tiling), I would guess some slope shader is not computed properly at its edges, is it possible to fix without modifying maplibre internals too much ? I'm still hoping to upstream at some point

---
Agent:

Let me focus on understanding the root cause by comparing texture handling between hillshade and terrain-analysis more carefully.

Starting: *Identify root cause of edge artifacts* (4/5)

Searched for text `update` (`**/src/render/texture.ts`), 6 results

Read [](src/render/texture.ts#49-49), lines 49 to 130

Good - `update` with same size calls `_updateRawData` → `gl.texSubImage2D`, which re-uploads the texture data every frame. So the DEM data IS refreshed. The backfilled borders should be in the texture.

Now let me carefully trace through the key difference. The critical issue is the texture filter mode:

Read [](src/render/draw_terrain_analysis.ts#80-80), lines 80 to 105

Now let me look at the `getElevation` function in the terrain_analysis shader vs hillshade_prepare:

**Hillshade** (in hillshade_prepare.fragment.glsl):
- Binds DEM texture with `gl.NEAREST`
- Directly calls `texture(u_image, coord)` to get exact texel values
- Renders to a dim×dim FBO where each output pixel maps 1:1 to a DEM data texel
- Sample positions at texel centers (e.g., 1.5/stride, 2.5/stride, ...)

**Terrain-analysis** (in terrain_analysis.fragment.glsl):
- Binds DEM texture with `gl.LINEAR`
- Uses `getDecodedElevation()` which calls `texture()` — with LINEAR, hardware may blend encoded RGBA values
- The comment on `getElevation` says: *"Manually bilinearly filter the decoded elevation to avoid 8-bit hardware filtering precision errors on the encoded RGB values"*
- But the manual bilinear filter calls `getDecodedElevation` at computed texel-center positions — with `gl.LINEAR`, these should return exact values at centers... **but floating point imprecision can cause slight off-center samples, blending encoded values incorrectly**

The fix is straightforward: change `gl.LINEAR` to `gl.NEAREST` for the DEM texture binding in draw_terrain_analysis.ts, matching what hillshade does. The manual bilinear filter in `getElevation` already handles proper interpolation of decoded elevation values.

But there's potentially a second issue. Let me verify by looking at the vertex shader mapping more carefully and how fragments map to texture coordinates:
Now I have the full picture. Here's the analysis:

1. **`gl.LINEAR` vs `gl.NEAREST` for DEM texture**: The `getElevation` function uses manual bilinear filtering specifically because hardware filtering of encoded DEM values introduces precision errors. It snaps to texel centers via `(floor(tc) + 0.5) / size`. But with `gl.LINEAR`, any floating-point imprecision in the texel-center calculation causes the hardware to blend encoded RGBA channels before decoding. At tile edges, where border pixels have very different encoded values (neighbor data), this creates visible artifacts. Hillshade uses `gl.NEAREST` which avoids this entirely.

2. **The vertex shader maps tile edges to between-texel positions**: At v_pos = 1/stride (left edge), the texel-space position is 1.0, which falls between texel 0 center (0.5) and texel 1 center (1.5). So the manual bilinear in `getElevation` blends the border pixel (texel 0) and first data pixel (texel 1) 50/50. This is mathematically correct for continuous elevation, but when computing Horn derivatives with `computeNormalDeriv`, the 3×3 kernel at the edge samples positions that span the border/data boundary, potentially reducing gradients.

The **primary fix** is changing `gl.LINEAR` to `gl.NEAREST`:

Starting: *Fix tile edge artifacts* (5/5)

Replacing 10 lines with 10 lines in [](src/render/draw_terrain_analysis.ts)


Now let me run the existing render tests to make sure this change doesn't break anything:

Ran terminal command:  kill $(lsof -ti:2900) 2>/dev/null; sleep 1

Ran terminal command:  node --no-warnings --loader ts-node/esm test/integration/render/run_render_tests.ts terrain-analysis 2>&1; echo "DONE: $?"

Checked background terminal output

Ran terminal command:  echo "waiting for previous result"

Let me check the remaining output:

Ran terminal command:  cat test/integration/render/results-errored-caseIds.txt 2>/dev/null; echo "---"; cat results.txt 2>/dev/null

All 10 existing tests pass. The only "error" is the slope-edges test which has no expected PNG (it's the new test I created). Let me clean that up and also update the expected image:

Ran terminal command:  kill $(lsof -ti:2900) 2>/dev/null; sleep 1 && UPDATE=true node --no-warnings --loader ts-node/esm test/integration/render/run_render_tests.ts terrain-analysis/slope-edges 2>&1; echo "DONE: $?"
Completed with input: {
  "command": "kill $(lsof -ti:2900) 2>/dev/null; sleep 1 && UPDATE=true node --no-warnings --loader ts-node/esm test/integration/render/run_render_tests.ts terrain-analysis/slope-edges 2>&1; echo \"DONE: $?\"",
  "explanation": "Generate expected image for slope-edges test",
  "goal": "Create expected test image",
  "isBackground": false,
  "timeout": 120000
}

Made changes.
