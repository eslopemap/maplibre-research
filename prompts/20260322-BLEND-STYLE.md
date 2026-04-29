# blend-mode Style Spec Integration — Debug Notes

## What was done

1. **Added `blend-mode` to `paint_raster` in `maplibre-style-spec/src/reference/v8.json`**
   - Enum property with values: `"normal"` (default), `"multiply"`, `"screen"`
   - Data-constant, zoom-only expression, no transition

2. **Rebuilt the style-spec package**
   - `cd maplibre-style-spec && npm install && npm run build && npm pack`
   - Confirmed `blend-mode` is present in:
     - `maplibre-style-spec/src/reference/v8.json` ✅
     - `maplibre-style-spec/dist/latest.json` ✅
     - The tarball `maplibre-maplibre-gl-style-spec-24.7.0.tgz` ✅

3. **Regenerated root repo style code**
   - `npm run generate-style-code` → `raster_style_layer_properties.g.ts` now contains `blend-mode` ✅

4. **Attempted `npm install` in the root repo** — ❌ this is where it fails

## Why it fails

The root `package-lock.json` has:
```json
"node_modules/@maplibre/maplibre-gl-style-spec": {
  "resolved": "maplibre-style-spec",
  "link": true
}
```

But `node_modules/@maplibre/maplibre-gl-style-spec` is **NOT a symlink** — it's a regular
directory containing a stale extracted copy from the original `.tgz` (timestamps: 15:25,
vs the rebuilt dist at 18:48).

`npm install` sees the lockfile says `"link": true` and the directory already exists, so it
skips it. The `package.json` references `"file:./maplibre-style-spec/maplibre-maplibre-gl-style-spec-24.7.0.tgz"`
but the lockfile was already resolved to a directory link by a previous `npm install` that saw `./maplibre-style-spec/`
as a workspace-like directory. The stale copy never gets updated.

**User also tried** `npm link @eslopemap/maplibre-gl-style-spec` which failed because:
- `npm link` expects the package name from the directory's `package.json` (`@maplibre/maplibre-gl-style-spec`)
- The command used `@eslopemap/...` which doesn't match

## Options to fix

### Option A: vendor the built dist (simplest, most reliable)
Copy `maplibre-style-spec/dist/` directly into `node_modules/@maplibre/maplibre-gl-style-spec/dist/`:
```bash
cp maplibre-style-spec/dist/latest.json node_modules/@maplibre/maplibre-gl-style-spec/dist/latest.json
cp maplibre-style-spec/dist/index.* node_modules/@maplibre/maplibre-gl-style-spec/dist/
cp maplibre-style-spec/src/reference/v8.json node_modules/@maplibre/maplibre-gl-style-spec/src/reference/v8.json
```

### Option B: change package.json to `"file:./maplibre-style-spec"` (directory, not tarball)
Then delete `node_modules/@maplibre/maplibre-gl-style-spec` and re-run `npm install` so npm
creates an actual symlink.

### Option C: nuke lockfile + node_modules (nuclear)
```bash
rm -rf node_modules package-lock.json && npm install
```

## Next step
Try Option A (vendor copy) first as it's zero-risk and immediately testable, then fall back
to Option B if we want a permanent fix.

## Resolution — Option A worked

**Root cause**: `package-lock.json` says `"link": true` for the style-spec entry but
`node_modules/@maplibre/maplibre-gl-style-spec` is a **regular directory** (stale extracted
copy from the original tarball), not a symlink. `npm install` sees the lockfile entry and
skips updating, so the rebuilt tarball with `blend-mode` is never picked up.

**Fix applied**:
1. Fixed v8.json `blend-mode` definition to include `"normal"` as a value with `"default": "normal"`
2. Rebuilt style-spec: `cd maplibre-style-spec && npm run build`
3. Vendored the built artifacts directly into `node_modules`:
   ```bash
   cp maplibre-style-spec/dist/latest.json  node_modules/@maplibre/maplibre-gl-style-spec/dist/latest.json
   cp maplibre-style-spec/dist/index.cjs    node_modules/@maplibre/maplibre-gl-style-spec/dist/index.cjs
   cp maplibre-style-spec/dist/index.mjs    node_modules/@maplibre/maplibre-gl-style-spec/dist/index.mjs
   cp maplibre-style-spec/src/reference/v8.json node_modules/@maplibre/maplibre-gl-style-spec/src/reference/v8.json
   ```
4. Regenerated style code: `npm run generate-style-code`
5. All 37 tests pass with `--config vitest.config.unit.ts`

**Note**: Tests must be run with `npx vitest run --config vitest.config.unit.ts` (or `npm run test-unit`)
to get the jsdom environment. Running bare `npx vitest run` uses node environment which lacks `document`.

**Note**: After any `npm install`, the vendored copy will be overwritten by the stale
tarball again. To make this permanent, either:
- Delete lockfile entry and `node_modules/@maplibre/maplibre-gl-style-spec`, then
  `npm install` so npm creates a proper symlink
- Or change `package.json` dep to `"file:./maplibre-style-spec"` (directory reference)
