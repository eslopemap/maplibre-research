Motivation
----------

This is a backport/upstreaming of functionality we actually already use to ship multi-band weather data to a custom layer client.

Raster source pipelines currently assume that the alpha channel represents opacity. As a result, image decode and texture upload paths may premultiply `RGB` by `A` before application code reads the tile data.

That behavior is correct for ordinary imagery, but it breaks raster tiles that use the alpha channel as a payload channel rather than transparency. This is increasingly relevant for packed RGBA tiles used for scientific, weather, and other multi-band data products, where each channel carries an independent value.

When premultiplication is applied to such tiles, the red, green, and blue channels are silently corrupted. This causes incorrect colors, incorrect decoded values, and incorrect downstream rendering. The core problem is that there is currently no source-level way to preserve exact RGBA bytes for raster tiles.

Proposed Change
---------------

Add a raster source option that controls whether raster tiles should be alpha-premultiplied during decode and texture upload.

Proposed source option:

`premultiply: boolean`

Default:\
`true`

Behavior:

-   `premultiply: true`\
    Preserve current behavior. Raster tiles may be decoded and uploaded using alpha-premultiplied semantics.
-   `premultiply: false`\
    Decode and upload raster tiles without alpha premultiplication, so `RGBA` bytes remain unchanged.

This option should be defined on raster sources because the problem occurs before any layer consumes the raster data. A layer-level option would be too late.

### Behavior Details

When `premultiply` is `false`, the implementation should:

-   avoid image-loading paths that implicitly premultiply alpha
-   prefer fetch/ArrayBuffer decoding paths when needed
-   use `createImageBitmap(..., { premultiplyAlpha: 'none' })` where supported
-   upload textures with unpack premultiplication disabled

When `premultiply` is `true`, the implementation should preserve the current code path and behavior.

### Why This Design

This is the smallest change that solves the actual problem:

-   it is explicit
-   it is source-scoped, which matches where decode/upload decisions are made
-   it is backward compatible by default
-   it supports packed RGBA data without requiring a custom source or custom rendering stack

API Modifications
-----------------

Add a new optional field to raster source definitions:

```source-json
{
  "type": "raster",
  "tiles": ["https://example.com/{z}/{x}/{y}.webp"],
  "premultiply": false
}
```

Expected API impact:

-   style spec: add `premultiply` to raster source specification
-   validation: accept only boolean values
-   generated style types: include `premultiply?: boolean`
-   documentation: describe when to use `premultiply: false`

No change is required to raster layer APIs.

Migration Plan and Compatibility
--------------------------------

No migration is required for existing users.

Compatibility statement:

-   Existing styles remain fully compatible because the default is `premultiply: true`.
-   Existing raster imagery workflows continue to behave exactly as before unless users opt into `premultiply: false`.
-   Users with packed RGBA raster sources can adopt the new option incrementally on affected sources only.

If a user is currently relying on custom workarounds to preserve RGBA values, those workarounds can be removed once the source is configured with `premultiply: false`.

Rejected Alternatives
---------------------

### Do Nothing

Rejected because it leaves packed RGBA raster sources fundamentally unsupported.

### Add a New Raw-Raster Source Type

Rejected because the problem is a decode/upload behavior switch, not a wholly different source model. A new source type would add unnecessary API and implementation surface.

### Add a Per-Layer Option

Rejected because the corruption happens before layer rendering. By the time a layer sees the tile, premultiplication may already have modified the data.

### Require Custom Layers or Custom Tile Pipelines

Rejected because it pushes a low-level implementation burden onto users. Applications should not need to reimplement tile fetching, caching, decode, and texture upload just to preserve RGBA bytes.

### Infer the Correct Behavior Automatically

Rejected because the renderer cannot reliably know whether alpha is opacity or payload. This must be an explicit user-controlled choice.
