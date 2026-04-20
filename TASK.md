# TASK.md

## Next tasks

Two items, in order:

1. **Persistent settings** — save user-adjusted values to `localStorage` so they survive page reload. Settings to persist:
   - `stem` (slider value)
   - `minTicks` visibility (toggle state)

2. **Consolidate to one page** — delete `index.html` (the digital clock) and rename `analog.html` → `index.html`. Remove the tab link (the `•` dot at bottom-right that navigated between the two pages) since there is no longer a second page.

---

## Project

Minimal static site on GitHub Pages. No build system, no dependencies. Single HTML files.

```
github-pages/
├── analog.html         ← becomes index.html (rename, delete old index)
├── index.html          ← digital clock — DELETE
├── favicon.png
└── docs/
    ├── sitemap.md
    └── architecture-analog.md
```

---

## `analog.html` — current state (as of last session)

### SVG coordinate system

`viewBox="-1 -1 2 2"`. Origin at centre. All geometry in normalised `−1 → +1` space. Clock element is sized `min(70vw, 70vh)` with `border-radius: 50%` (clips the square SVG to a circle — eliminates the square bounding box artefact caused by isolated blend mode compositing).

### Layer order (paint order)

```
<rect class="bg"/>          ← background fill — stable blend mode base
<g id="ticks"/>             ← 12 hour tick marks (generated in JS)
<g id="minTicks"/>          ← 48 minute tick marks (toggleable, hidden by default)
<g id="hHand"/>             ← hour hand
<g id="mHand"/>             ← minute hand
<g id="sHand"/>             ← second hand + lollipop circle
<circle id="pip"/>          ← centre pip, painted last
```

### Blend mode

All `.tick`, `.hand`, `.pip` use `mix-blend-mode: difference; fill: white; stroke: white`. Hands additionally have `stroke: none` to prevent SVG's default `stroke-width: 1` (= enormous in this coordinate space) from dominating. The `<rect class="bg">` provides a stable base for all blend calculations.

### Design constants (JS)

```js
const PHI  = (1 + Math.sqrt(5)) / 2;   // φ ≈ 1.618
let   stem = 0.005;                     // universal base dimension — slider-controlled
let   sW, mW, hW;                       // derived widths, recomputed in updateWidths()

function updateWidths() {
  sW = stem;
  mW = stem * PHI * PHI;              // stem × φ²
  hW = stem * PHI * PHI * PHI * PHI;  // stem × φ⁴
}
```

`stem` is the single master dimension. `sW/mW/hW` are the second/minute/hour hand widths, cascading by φ² each step (not φ — this was a deliberate design choice for more pronounced separation).

### Dimension table (at default `stem = 0.005`)

| Element | Property | Value / Formula |
|---|---|---|
| Clock size | CSS | `min(70vw, 70vh)` |
| Hour ticks (×12) | inner r | `0.860` |
| Hour ticks (×12) | outer r | `0.940` |
| Hour ticks (×12) | length | `0.080` |
| Hour ticks (×12) | stroke-width | `mW` |
| Minute ticks (×48) | inner r | `0.868` |
| Minute ticks (×48) | outer r | `0.940` |
| Minute ticks (×48) | stroke-width | `sW` |
| Hour hand | width | `hW` = `stem × φ⁴ ≈ 0.0343` |
| Hour hand | tip | `−0.500` |
| Hour hand | tail | `+0.070` |
| Minute hand | width | `mW` = `stem × φ² ≈ 0.0131` |
| Minute hand | tip | `−0.755` |
| Minute hand | tail | `+0.090` |
| Second hand | width | `sW` = `stem = 0.005` |
| Second hand | tip | `−0.862` |
| Second hand | tail | `+0.090` |
| Lollipop circle | cy | `0.210` |
| Lollipop circle | r | `0.040` |
| Centre pip | r | `0.036` (hardcoded — fixed, does not scale with stem) |

All ticks and hands use `stroke-linecap: round`. Hand rects use `rx = width/2` (pill/capsule ends).

### UI controls

| Control | Position | Behaviour |
|---|---|---|
| `#stemSlider` | `fixed; bottom: 32px; left: 24px` | `input[type=range]` min=2 max=20 value=5 step=1; maps to `stem = value/1000`; calls `build()` on input |
| `#minToggle` | `fixed; bottom: 70px; right: 24px` | Button; toggles `#minTicks` visibility; adds `.active` class when on |
| `.tab` | `fixed; bottom: 24px; right: 24px` | Link to `index.html` — **DELETE when renaming analog → index** |

All controls: `opacity: 0.18` idle, `0.55` on hover. `.active` state for `#minToggle`: `0.40`.

### `build()` architecture

Changing `stem` (via slider) calls `build()`, which:
1. `updateWidths()` — recomputes `sW/mW/hW`
2. `buildTicks()` — clears and regenerates both tick groups
3. `buildHands()` — clears and regenerates hand children (rotation `transform` on `<g>` is preserved)

Animation runs independently via `requestAnimationFrame`, updating only `transform="rotate(...)"` on the hand groups each frame.

### Settings not yet persistent

`stem` and `minTicks` toggle state reset on every page load. **This is task 1.**
