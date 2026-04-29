# Hillshade vs Terrain-Analysis Shader Comparison

## Key Finding
Both shaders use **identical coordinate mapping and border handling**:
- Same scaling formula: `scale = (u_dimension.x - 2.0) / u_dimension.x`
- Same epsilon adjustment: `v_pos = (texture_coord / 8192.0) * scale + epsilon`
- Same `u_dimension` setup: `[stride, stride]`

## Critical Differences

### 1. Vertex Shader Differences

**Hillshade Prepare:**
- Uses `a_texture_pos` input attribute
- Simple matrix transform: `gl_Position = u_matrix * vec4(a_pos, 0, 1);`
- No pole handling

**Terrain Analysis:**
- Uses `a_pos` input attribute (custom mesh coordinates)
- Uses custom `projectTile()` function for projection
- Has special pole handling:
  ```glsl
  if (a_pos.y < -32767.5) v_pos.y = 0.0;    // North pole
  if (a_pos.y > 32766.5) v_pos.y = 1.0;    // South pole
  ```

### 2. Fragment Shader Differences

**Hillshade Prepare (`hillshade_prepare.fragment.glsl`):**
- Simple elevation sampling: `vec4 data = texture(u_image, coord) * 255.0`
- Direct 3x3 derivative computation (Horn algorithm)
- Computes raw slopes: `(c + f + f + i) - (a + d + d + g)`
- No latitude distortion correction
- 2-pass pipeline (prepare → render with methods like BASIC, COMBINED, IGOR, STANDARD)

**Terrain Analysis (`terrain_analysis.fragment.glsl`):**
- **Manual bilinear filtering** to avoid 8-bit precision errors:
  ```glsl
  vec2 tc = coord * size - 0.5;
  vec2 f = fract(tc);
  // Blend between 4 decoded values instead of hardware-filtering encoded values
  ```
- Identical Horn derivative computation
- **Latitude distortion correction**: `deriv /= cos(radians(lat))`
- Single-pass pipeline (elevation → slope/aspect → color ramp)
- Aspect calculation with proper compass bearing conversion

### 3. Program Uniforms

**Hillshade Prepare** (`hillshadeUniformPrepareValues`):
```typescript
{
  'u_matrix': matrix,           // Orthographic [stride, stride]
  'u_image': 1,                 // DEM texture unit
  'u_dimension': [stride, stride],
  'u_zoom': tileID.overscaledZ,
  'u_unpack': dem.getUnpackVector()
}
```

**Terrain Analysis** (`terrainAnalysisUniformValues`):
```typescript
{
  'u_image': 0,
  'u_unpack': dem.getUnpackVector(),
  'u_dimension': [dem.stride, dem.stride],
  'u_zoom': zoom,
  'u_scalar_stops': 1,
  'u_color_stops': 4,
  'u_color_ramp_size': colorRampSize,
  'u_opacity': layer.getOpacity(),
  'u_latrange': getTileLatRange(tileID),     // Added for Mercator correction
  'u_attribute': attributeToInt(...),         // elevation/slope/aspect
  'u_step_mode': layer.isStepMode ? 1 : 0
}
```

## Conclusion

**They handle tile edges identically** via the same border mapping strategy.

The differences are:
1. **Pipeline design**: Hillshade is 2-pass (prepare slopes → apply shading), terrain-analysis is 1-pass (DEM → attribute → color)
2. **Precision**: Terrain analysis uses manual bilinear filtering to avoid 8-bit quantization artifacts
3. **Lat correction**: Only terrain-analysis applies Mercator distortion correction
4. **Mesh setup**: Hillshade uses standard raster bounds, terrain-analysis uses custom projections with pole handling
