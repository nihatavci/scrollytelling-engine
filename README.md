# Scrollytelling Engine — Technical Documentation

> How the "Preston Curve" scrollytelling demo works, what shortcuts were taken, and what can break.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [The Coordinate Space Problem (and Solution)](#the-coordinate-space-problem)
3. [How Each State Transition Works](#state-transitions)
4. [Libraries & Why Each Was Chosen](#libraries)
5. [Shortcuts & Trade-offs](#shortcuts)
6. [Known Bugs & Edge Cases](#known-bugs)
7. [Performance Considerations](#performance)
8. [How to Extend This](#extending)

---

## Architecture Overview

The entire demo is a single HTML file with no build step. It renders 10 scroll-triggered states using the same 15 `<path>` SVG elements that are never destroyed or recreated.

```
┌─────────────────────────────────────────────────────────┐
│                    SINGLE SVG (900×500 viewBox)          │
│                                                          │
│  g (translated by margins)                               │
│  ├── bgG ........... background world map countries       │
│  ├── gridG ......... axis grid lines                      │
│  ├── axXG .......... x-axis                               │
│  ├── axYG .......... y-axis                               │
│  ├── fxG ........... effects (pulse rings, annotations)   │
│  ├── dataG ......... THE 15 <path> elements (morphable)   │
│  ├── lblG .......... 15 <text> labels per country         │
│  └── annotG ........ temporary annotations (center text)  │
│                                                          │
│  Layer order matters: dataG is above gridG so paths       │
│  render on top of grid lines. fxG is below dataG so       │
│  pulse rings appear behind the country shapes.            │
└─────────────────────────────────────────────────────────┘
```

### Scroll Mechanism

Uses vanilla `IntersectionObserver` (not Scrollama.js — zero dependency for scroll detection).

```
rootMargin: "-50% 0px -50% 0px"
```

This creates a trigger line at exactly 50% of the viewport height. When a `.step` element's top edge crosses this line, the corresponding state function fires.

The graphic panel uses `position: sticky; top: 0; height: 100vh` to stay fixed while step text scrolls over it on the right side.

### Data Flow

```
User scrolls
  → IntersectionObserver fires
    → State function called (s0, s1, ... s9)
      → For each of 15 paths:
        1. Read current "d" attribute (the SVG path string)
        2. Generate target path string for new state
        3. Create flubber interpolator between current → target
        4. Apply D3 transition with attrTween("d", interpolator)
      → Update title, subtitle, axes, labels, effects
```

---

## The Coordinate Space Problem

This is the single most important architectural decision in the project.

### The Problem

D3's `d3.arc()` generates path strings centered at (0,0). For a donut chart, you'd normally do:

```js
// Standard approach — BREAKS flubber transitions
selection
  .attr("transform", `translate(${cx}, ${cy})`)  // shift origin
  .attr("d", arcGenerator(slice));                // path at 0,0
```

But all other states (map, scatter, bars, bubbles) generate paths in the SVG's own coordinate space (after the margin `g` transform). When flubber tries to interpolate between a bar at `x=200` and an arc at `x=0` (with a translate shifting it to `x=400`), it sees a massive jump and produces garbage intermediate shapes.

### The Solution: `arcPathAt()`

Instead of using `translate()`, we bake the center offset directly into the path coordinates:

```js
function arcPathAt(innerR, outerR, startA, endA, padA, centerX, centerY) {
  // 1. Generate arc path at origin (0,0)
  const arcAtOrigin = d3.arc()({
    innerRadius: innerR, outerRadius: outerR,
    startAngle: startA, endAngle: endA, padAngle: padA
  });
  // 2. Sample points along the path and shift them by (centerX, centerY)
  return shiftPath(arcAtOrigin, centerX, centerY);
}
```

`shiftPath()` creates a temporary invisible SVG `<path>`, samples ~60 points along it using `getPointAtLength()`, offsets each point, and rebuilds as a polygon string.

**Result**: Every state — map, circle, rect, arc — produces paths in the same coordinate space. Flubber always interpolates between paths that live in the same world. No jumps.

**Trade-off**: The sampled arc paths are polygonal approximations (60 line segments) rather than true SVG arc commands. At the rendered size, this is invisible. At extreme zoom, you'd see faceting.

---

## State Transitions

| # | State | Target Shape | Path Generator | Key Technique |
|---|-------|-------------|----------------|---------------|
| 0 | World Map | Country polygons | `d3.geoPath(projection)(feature)` | TopoJSON → GeoJSON → SVG path |
| 1 | Highlight | Same as 0 | Same | Only opacity/stroke changes |
| 2 | Circles (grid) | Perfect circles | `cP(cx, cy, r)` — arc-based circle path | flubber: polygon → circle |
| 3 | Scatter | Circles at data positions | `cP(xScale(gdp), yScale(life), rScale(pop))` | flubber: circle → circle (just moves) |
| 4 | US Outlier | Same as 3 | Same positions | Opacity dimming + pulse effect |
| 5 | Bars | Rounded rectangles | `rP(x, y, w, h, cornerRadius)` | flubber: circle → rectangle |
| 6 | Ranked Bars | Same shape, same positions | Same as 5 | Identical — only labels update |
| 7 | Packed Bubbles | Circles from d3.pack() | `cP(node.x, node.y, node.r)` | flubber: rectangle → circle, elastic easing |
| 8 | Donut | Arc segments | `arcPathAt(inner, outer, start, end, pad, cx, cy)` | flubber: circle → arc (baked center) |
| 9 | Split Donuts | Smaller arc segments | `arcPathAt(inner, outer, start, end, pad, regionCx, regionCy)` | flubber: arc → arc (different centers) |

### Helper Path Generators

**`cP(cx, cy, r)`** — Circle as SVG path:
```
M(cx-r),cy A r,r 0 1,1 (cx+r),cy A r,r 0 1,1 (cx-r),cy Z
```
Two semicircular arcs. This is important — a circle must be a closed path for flubber to work, not a `<circle>` element.

**`rP(x, y, w, h, cr)`** — Rounded rectangle as SVG path:
```
M(x+cr),y L(x+w-cr),y Q(x+w),y (x+w),(y+cr) L(x+w),(y+h-cr) ...
```
Uses quadratic Bézier curves for corners. `cr` is corner radius, clamped to half the smallest dimension.

**`arcPathAt()`** — Arc with baked-in center offset:
Generates d3.arc path at origin, samples it to polyline, shifts all points by (cx,cy), returns as `M...L...L...Z`.

---

## Libraries

| Library | Version | Size | Purpose | Why This One |
|---------|---------|------|---------|-------------|
| **D3.js** | 7.9.0 | ~280KB | Scales, axes, geo projection, transitions, data binding, pack layout, pie layout | Industry standard. Flourish uses it internally. |
| **TopoJSON Client** | 3.x | ~4KB | Convert TopoJSON → GeoJSON features | Required to decode world-atlas data |
| **Flubber** | 0.4.2 | ~14KB | Shape-to-shape path interpolation | The only library that smoothly morphs arbitrary SVG paths. D3's built-in `d3.interpolate` fails when paths have different numbers of points. |
| **World Atlas** | 2.x | ~100KB (fetched) | Country boundary data (110m resolution) | Smallest available world TopoJSON. 110m = low resolution but tiny file. |

### Why Not Scrollama.js?

Scrollama is excellent but adds a dependency for something achievable in ~8 lines with `IntersectionObserver`. The scroll detection here is trivial — no progress tracking, no container enter/exit. Just "which step is at 50%?"

For a production app, Scrollama adds value (resize handling, progress callbacks, debug mode). For a single-file demo, native IntersectionObserver is sufficient.

---

## Shortcuts & Trade-offs

### 1. Arc Path Sampling (not true arcs)

`arcPathAt()` converts smooth SVG arc commands into ~60-segment polylines. This is a shortcut to avoid writing a proper SVG path parser + transformer.

**Impact**: Invisible at normal zoom. Slightly heavier path strings (~2KB vs ~200B per arc).

**Proper solution**: Parse SVG path commands, identify coordinate values, apply affine transform, serialize back. Libraries like `svg-path-parser` + `svg-path-builder` do this.

### 2. Flubber's `maxSegmentLength`

Set to 10px throughout. Lower = smoother morphs but more intermediate points = slower transitions. The value 10 is a balance point for this data size.

**If morphs look jagged**: Lower to 5 or 3.
**If transitions lag**: Raise to 15 or 20.

### 3. Country ID Matching

TopoJSON uses ISO 3166-1 numeric codes (e.g., USA = "840"). These are hardcoded in `DATA[]`. If the world-atlas file changes its ID scheme, all country matching breaks silently (paths just won't appear).

### 4. Elastic Easing on Bubble State

`d3.easeElasticOut.amplitude(1).period(.5)` creates a bouncy effect. This can cause paths to momentarily extend outside the SVG viewBox. The SVG clips by default, so it's invisible, but it's technically incorrect geometry during the bounce.

### 5. Static Pack Layout

`d3.pack()` is computed once at load time. If data changes, the layout must be recomputed. This is fine for a static demo but won't work for dynamic data.

### 6. Pie Sort Order

`d3.pie().sort(null)` preserves input order. The donut slices appear in DATA array order, not sorted by size. This is intentional (maintains data-index consistency for flubber matching) but means small slices can end up between large ones.

### 7. Single-File Architecture

Everything is in one HTML file. This is great for distribution but terrible for maintenance. A production version should split into: `index.html`, `styles.css`, `data.js`, `states.js`, `scroll.js`, `helpers.js`.

---

## Known Bugs & Edge Cases

### Bug 1: Rapid Scrolling

**Symptom**: If you scroll very fast through multiple states, transitions can queue up and overlap, causing jittery intermediate shapes.

**Cause**: D3 transitions on the same element are supposed to cancel previous ones, but flubber's `attrTween` closures can capture stale path strings.

**Workaround**: Each state function reads the *current* `d` attribute at call time (`getCur(el)`), so it always interpolates from wherever the path actually is. But during a running transition, the "current" value is the in-progress interpolated value, which may not be a valid endpoint for flubber.

**Fix**: Add `.interrupt()` before each transition to force-complete the previous one. This snaps to the target immediately, which can feel jarring but prevents overlap:
```js
d3.select(this).interrupt().transition()...
```

### Bug 2: Flubber Failure on Complex Paths

**Symptom**: Occasionally, flubber throws on very complex country shapes (e.g., Indonesia's archipelago). The `si()` wrapper catches this and falls back to an instant swap at t=0.5.

**Cause**: Flubber can't always find a valid interpolation for multi-polygon paths or self-intersecting shapes.

**Affected countries**: Indonesia (archipelago), USA (Alaska + Hawaii as separate polygons in some projections).

**Fix**: Pre-simplify complex country paths using `topojson.simplify()` or use the 110m resolution data (already done — this is the lowest resolution available).

### Bug 3: Font Loading Race

**Symptom**: On slow connections, labels may render in a fallback font for the first ~1-2 seconds, then shift when Google Fonts loads.

**Cause**: `@import url(...)` in CSS is render-blocking for styles but not for the font file download.

**Fix**: Use `<link rel="preload">` for font files, or use `document.fonts.ready.then(...)` to delay first render.

### Bug 4: Mobile Layout Overlap

**Symptom**: On narrow screens (<900px), the step cards stack below the graphic, but the `margin-top: -100vh` trick for overlay positioning doesn't apply. The graphic height is reduced to 55vh.

**Edge case**: On very short phones (e.g., iPhone SE in landscape), the 55vh graphic can be too small to read labels.

**Fix**: Add a media query for `max-height: 500px` that further reduces font sizes and circle radii.

### Bug 5: Safari IntersectionObserver Timing

**Symptom**: Safari sometimes fires IntersectionObserver callbacks in a different order than Chrome/Firefox, causing steps to occasionally skip or fire out of sequence.

**Fix**: Add a check that the new step index is adjacent to the current one (±1). If it jumps by more than 1, sequentially trigger intermediate states:
```js
if (Math.abs(idx - cur) > 1) {
  const dir = idx > cur ? 1 : -1;
  for (let i = cur + dir; i !== idx; i += dir) fns[i]();
}
```

### Bug 6: Donut Label Collision

**Symptom**: Small donut slices (e.g., S. Korea at 1%) have labels that overlap with adjacent slice labels.

**Cause**: Label positions are computed from arc midpoints. Adjacent small slices have midpoints very close together.

**Fix**: Implement a collision detection pass that nudges overlapping labels apart:
```js
// After computing all label positions, sort by angle and
// enforce minimum vertical distance between consecutive labels
```

---

## Performance Considerations

| Concern | Current State | Optimization Path |
|---------|--------------|-------------------|
| **Path complexity** | ~60 segments per arc, ~200+ per country | Use `topojson.presimplify()` to reduce country point count |
| **Transition count** | 15 paths + 15 labels = 30 simultaneous transitions | Acceptable. D3 handles this fine. |
| **TopoJSON fetch** | ~100KB, fetched on page load | Could embed inline as base64, or use 50m for more detail |
| **Flubber computation** | Runs on every state change for all 15 paths | Could pre-compute all interpolators at load time |
| **SVG vs Canvas** | SVG (DOM-based) | For >50 elements, Canvas would perform better. 15 is fine for SVG. |
| **Font loading** | Google Fonts external fetch | Self-host or use `font-display: swap` |

### Pre-computing Interpolators

The biggest performance win would be pre-computing all flubber interpolators at load time:

```js
// At load time, compute all transitions:
const interpolators = {};
DATA.forEach((d, i) => {
  interpolators[d.name] = {
    mapToCircle: flubber.interpolate(mapPath, circlePath, opts),
    circleToScatter: flubber.interpolate(circlePath, scatterPath, opts),
    scatterToBar: flubber.interpolate(scatterPath, barPath, opts),
    // ... etc
  };
});
```

This front-loads ~200ms of computation but makes every subsequent transition instant.

---

## How to Extend This

### Adding a New State

1. Define the state function (`s10`, `s11`, etc.)
2. Generate target path strings for all 15 elements
3. Use `morphTo(element, targetPath, duration, delay)` for each
4. Update title, subtitle, axes, effects
5. Add to `fns[]` array
6. Add a new `.step` div in the HTML
7. Ensure the new path is in the **same coordinate space** (no `translate()` on path elements)

### Adding More Countries

1. Add to `DATA[]` with correct ISO 3166-1 numeric ID
2. Recompute `sorted`, `packLayout`, `donutSlices`, `xBand` domain
3. The 5-column grid layout in state 2 (`col = i % 5`) will need adjustment
4. More countries = more flubber computations = watch performance

### Replacing D3 Charts with Real Chart Library

If you want to use Chart.js, Recharts, etc. instead of raw D3:
- You lose the ability to morph between chart types (these libraries own their DOM)
- You'd need to crossfade between two chart instances instead of morphing
- The "same element morphs" magic is only possible with direct SVG path manipulation

### Making It Data-Driven (CMS Integration)

To turn this into a reusable tool:
1. Extract `DATA[]` into a JSON endpoint
2. Extract step text into a content array
3. Extract state functions into a generic `transitionTo(stateType, data)` dispatcher
4. Build a visual editor that outputs the JSON config
5. The renderer becomes a generic scrollytelling engine

This is essentially what Flourish built — but they spent 7+ years and a 44-person team on it.

---

## File Inventory

| File | What It Is |
|------|-----------|
| `scrollytelling_world_map_morph.html` | First version — with white card wrapper |
| `scrollytelling_no_card.html` | Card removed, glassmorphic steps |
| `preston_curve_scrollytelling.html` | Real data + real narrative, still has card |
| `preston_curve_fullwidth.html` | Full-width layout, overlay steps |
| `preston_curve_ultimate.html` | Added fit curve + bubbles + donut (has coordinate bug) |
| `preston_curve_freytag.html` | Freytag narrative arc (has coordinate bug) |
| **`preston_curve_seamless.html`** | **Final version — coordinate space fix, all transitions seamless** |

Use `preston_curve_seamless.html` — all others are iterations.

---

*Last updated: April 2026*
*Single-file HTML, no build step, CDN dependencies only*
