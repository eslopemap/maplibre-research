user journey: I wanted to make a gpx track editor.
the goal was to show big circle for the active (selected) track but hide them for the inactive tracks, with special case for start/end:

<img width="402" height="228" alt="Image" src="https://github.com/user-attachments/assets/3706513b-a65c-4873-801a-ffb5b3b6a096" />

of course I used AI to navigate the filter spec 😆  and when it failed repeatedly, I asked it for several solutions, and it highlighted one that it believed should work but did not.

Why? it mixed legacy and new syntax. The training data is certainly full of it. https://maplibre.org/maplibre-style-spec/deprecations/ has more context on this legacy syntax and says explicitly **Expression syntax and the deprecated syntax below cannot be mixed in a single filter definition.** but does not enforce it.

Who reads docs these days ? in this case, not me, and not AIs. so I'm kind of "holding it wrong" but the result is silent failure, no web developer console logs, so it's tricky to debug.

Apparently, in some of those mixed cases `isExpressionFilter` classifies the whole filter as **legacy**.

**maplibre-gl-js version**: 5.19.0 / master

**browser**:  Chrome 134

### Steps to Trigger Behavior
 1. Go to https://gistpreview.github.io/?1fa00de0a5f06bfcc784566f1c4714ec/track-bug.html
 2. Click 'Track A' to see expected behaviour
 3. Click 'Track A2' and observe nothing happens

Warning:  lightly edited AI-assisted investigation below, I'm out of my depth but I did correct some hallucinations... it's not a 'legacy-looking' filter being mis-classified, it's really a 'legacy' filter ! anyway, the fix works for my case 🤷

----

### Link to Demonstration
Here is an even smaller, contrived example

https://gistpreview.github.io/?1fa00de0a5f06bfcc784566f1c4714ec/track-issue-minimal-reproduction.html

1. Open the HTML file above
2. Click "Activate" — 3 rows should have same number of points
3. **Top row (green)** and bottom (blue) appear
4. **Middle row (red)** never appears — `global-state` in filter is ignored

source of both examples above: https://gist.github.com/eddy-geek/1fa00de0a5f06bfcc784566f1c4714ec

key piece:

```js
  // WORKING LEGACY: global-state in paint only
  map.addSource('working', { type: 'geojson', data: geojson });
  map.addLayer({
    id: 'working-pts', type: 'circle', source: 'working',
    filter: ['==', '$type', 'Point'],
    paint: {
      // Active: all points. Inactive: only start/end.
      'circle-radius': ['case', isActive, 6, isStartOrEnd, 6, 0],
      'circle-color': 'green'
    }
  });

  // BROKEN MIXED: global-state in filter
  map.addSource('broken', { type: 'geojson',
    data: { type: 'FeatureCollection', features: geojson.features.map(f => ({
      ...f, geometry: { ...f.geometry, coordinates: [f.geometry.coordinates[0], -20] }
    }))}
  });
  map.addLayer({
    id: 'broken-pts', type: 'circle', source: 'broken',
    // This mixed filter should behave like working-pts semantics,
    // but currently fails due to expression/legacy classification.
    filter: ['all',
      ['==', '$type', 'Point'],
      ['case', isActive, true, isStartOrEnd]
    ],
    paint: {
      'circle-radius': 6,
      'circle-color': 'red'
    }
  });

  // WORKING MODERN: what I should have done instead
  map.addSource('fixed', { type: 'geojson',
    data: { type: 'FeatureCollection', features: geojson.features.map(f => ({
      ...f, geometry: { ...f.geometry, coordinates: [f.geometry.coordinates[0], -40] }
    }))}
  });
  map.addLayer({
    id: 'fixed-pts', type: 'circle', source: 'fixed',
    filter: ['all',
      ['==', ['geometry-type'], 'Point'],
      ['case', isActive, true, isStartOrEnd]
    ],
    paint: {
      'circle-radius': 6,
      'circle-color': 'blue'
    }
  });
```

### Root cause (confirmed)

The issue is with `isExpressionFilter()` in `maplibre-style-spec/src/feature_filter/index.ts`.

When a filter mixes legacy syntax with expression operators — e.g.:
```js
['all', ['==', '$type', 'Point'], ['case', ['==', ['get', 'id'], ['global-state', 'activeId']], true, false]]
```
`isExpressionFilter` classifies the entire filter as **legacy** because `['==', '$type', 'Point']` (3 args, no arrays) is a legacy filter. The `all`/`any` case required ALL children to be expression filters, so one ambiguous/legacy-looking child caused the whole filter to be sent through `convertFilter()`, which silently mangles expression-only operators like `case` and `global-state`.

Even without `$type`, a pure `['case', isActive, true, false]` filter works fine in isolation — the bug only manifests when combined with legacy siblings in `all`/`any`.

### Fix (2 parts in `feature_filter/index.ts`)

1. **`isExpressionFilter` `all`/`any` logic**: Changed from "all children must be expressions" to "if ANY child is definitely an expression, treat the whole filter as expression". A legacy `all` can only contain legacy children.

2. **`normalizeExpressionFilter`**: New function called after `isExpressionFilter` returns `true`, which converts legacy `$type`/`$id` shorthands to proper expression syntax (`['geometry-type']`/`['id']`) within an otherwise-expression filter.

### Tests added

- `feature_filter.test.ts`: 5 new tests for `global-state` in filters (basic, reactive mutation, case expression, isExpressionFilter mixed detection, featureFilter with $type + global-state)
- `worker_tile.test.ts`: 2 new tests for global-state filter in tile parsing (matching features, state mutation + re-parse)

## Environment

- **maplibre-gl-js**: 5.18.0
- **Browser**: Chrome 134 / Safari 18 (same behavior)
- **OS**: macOS

## Related

- `global-state` spec: https://maplibre.org/maplibre-style-spec/expressions/#global-state
- Filter example: https://maplibre.org/maplibre-gl-js/docs/examples/filter-layer-symbols-using-global-state/
- `setGlobalStateProperty` API: https://maplibre.org/maplibre-gl-js/docs/API/classes/Map/#setglobalstateproperty
