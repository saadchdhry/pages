# TASK.md

## Current task

**Clock controls — improve and refine the UI controls.**

This new task starts after a short setup pass on the clock size and internal SVG geometry. The immediate focus is no longer the Date/Month marker redesign. Instead, the next task should focus on the control surface around the clock: the existing fixed-position buttons and stem slider.

The user has the in-app browser open to:

```text
file:///Users/saad/Library/Mobile%20Documents/iCloud~md~obsidian/Documents/Saad/github-pages/index.html
```

Use that preview for quick visual checks as the controls change.

---

## Work scope

### File structure

```text
github-pages/
├── index.html          <- active working file; all new work goes here
├── TASK.md             <- this handoff note
├── old.html            <- deprecated, ignore
├── favicon.png
└── docs/
    ├── sitemap.md
    └── architecture-analog.md
```

**Work only in `index.html`.**

---

## Current controls

The active controls are all in `index.html`:

- `#fpsToggle`: bottom-right, toggles continuous vs ticking mode.
- `#minToggle`: bottom-right, toggles minute markers.
- `#themeToggle`: bottom-right, cycles system/light/dark.
- `#randomizeBtn`: bottom-left, randomizes a coherent test date/time.
- `#stemSlider`: bottom-left, controls the base stroke width.

Current controls are functional but visually basic. The next task is expected to improve them.

Important behavior to preserve:

- `fpsToggle` changes between 30 FPS continuous seconds and 1 FPS ticking seconds.
- `minToggle` persists minute-marker visibility to `localStorage`.
- `themeToggle` persists theme mode to `localStorage`.
- `randomizeBtn` writes a coherent random `Date` into `testNow`.
- `stemSlider` persists the selected stem value to `localStorage` and rebuilds the clock.

The randomize control is especially useful during visual work and should remain available.

---

## Recent changes to remember

### Clock CSS size

The clock element was changed from 70% to 50% of the limiting viewport dimension:

```css
#clock {
  width: min(50vw, 50vh);
  height: min(50vw, 50vh);
  display: block;
  border-radius: 50%;
}
```

This makes the whole SVG smaller on the page.

### SVG coordinate system

The SVG still uses:

```html
<svg id="clock" viewBox="-1 -1 2 2">
```

Meaning:

- Internal coordinates run from `-1` to `+1` on both axes.
- The origin is the center of the clock at `(0, 0)`.
- The full SVG radius is `1.000`.
- Geometry values are normalized SVG units, not pixels.

### Dial geometry was moved inward

The visible dial geometry now fits inside an outer radius of `0.800`, leaving outer SVG room for future marker ideas.

Current constants:

```js
const BASE_TICK_OUTER = 0.940;
const DIAL_OUTER_R    = 0.800;
const DIAL_SCALE      = DIAL_OUTER_R / BASE_TICK_OUTER;
```

Helper:

```js
function dialR(value) {
  return value * DIAL_SCALE;
}
```

Interpretation:

- Old accepted radial positions were based on outer radius `0.940`.
- New radial positions are scaled by `0.800 / 0.940`.
- Stroke widths are **not** scaled by this helper.
- The dial keeps its visual line weight while the geometry moves inward.

This was intentionally chosen over a parent `<g transform="scale(0.8)">`, because the user wanted real outer space for future markers while preserving stroke presence.

---

## Current geometry reference

All values below are in normalized SVG units.

| Element | Code Value | Effective Geometry | Meaning |
|---|---:|---:|---|
| SVG drawing space | `-1 -1 2 2` | `x/y = -1..+1` | Full 2-by-2 coordinate square centered at `(0, 0)`. |
| Clock CSS size | `min(50vw, 50vh)` | 50% of shorter viewport side | Size of the whole SVG on the page. |
| Full SVG radius | `1.000` | 100% | Distance from center to SVG edge. |
| Dial outer radius | `DIAL_OUTER_R = 0.800` | 80% of SVG radius | Outermost tick endpoints sit here. |
| Previous tick outer | `BASE_TICK_OUTER = 0.940` | reference only | Used to scale the old accepted geometry inward. |
| Dial scale | `0.800 / 0.940` | `0.85106` | Converts old radial positions into the new smaller dial. |
| Hour tick outer | `tickOuter` | `0.800` | Outer endpoint for hour, month, date, and minute ticks. |
| Normal hour tick inner | `dialR(0.860)` | `0.73191` | Inner endpoint of normal hour ticks. |
| Minute tick inner | derived | `~0.77399` | Inner endpoint of normal minute ticks. |
| Month marker inner | derived from hour length x `PHI` | `~0.68985` | Month marker extends farther inward than a normal hour tick. |
| Date marker inner | derived from minute length x `PHI` | `~0.75792` | Date marker extends farther inward than a normal minute tick. |
| Day marker ring | `dialR(0.650)` | `0.55319` | Centers of the seven day circles sit on this inner arc. |
| Day circle outer radius | `SECOND_LOLLIPOP_R` | `0.040` | Day circles keep the same visual size as the seconds lollipop. |
| Center pip radius | `dialR(0.350)` | `0.29787` | Center circle scaled inward with the dial geometry. |
| Hour hand forward reach | `dialR(-0.500)` | `-0.42553` | Hour hand extends upward from center. |
| Minute hand forward reach | `dialR(-0.755)` | `-0.64255` | Minute hand extends farther toward the tick ring. |
| Second hand forward reach | `dialR(-0.862)` | `-0.73362` | Second hand extends closest to the tick ring. |
| Second lollipop center | `dialR(0.210)` | `0.17872` | Lollipop sits below center on the second hand. |

---

## Marker state to preserve unless explicitly revisited

The next task is controls-focused, but this marker context may help avoid accidental drift.

### Day of week

The Day marker treatment is accepted:

- Seven circles on an inner perimeter across the top arc.
- Positions span `9 -> 3`.
- Non-current days are hollow.
- Current day is solid filled.
- Day-circle outer size matches the seconds lollipop.
- Day-circle size is fixed, not stem-scaled.

Important implementation detail:

```js
circle.style.fill = 'none';
```

Use inline style for hollow day circles because CSS specificity would override a presentation attribute.

### Date and Month

Date is still the weaker calendar marker visually, but it is not the focus of the next task unless the user redirects back to marker design.

Current behavior:

- Month uses the hour tick at the month position and extends inward.
- Date uses a minute-width tick at `curDate` and extends inward.
- Date is always drawn in `#ticks`, so it remains visible even when minute ticks are hidden.
- If Date lands on an hour position, the normal hour tick is drawn first, then the date marker is drawn on top.

---

## Blend mode and animation rules

All `.tick`, `.hand`, and `.pip` use:

```css
mix-blend-mode: difference;
fill: white;
stroke: white;
```

Important:

- Do not animate `fill` or `opacity` on clock geometry.
- Hand animation should remain transform-based.
- Controls can use normal UI opacity/hover behavior, but avoid interfering with the clock geometry blend-mode system.

---

## Suggested starting point for the next task

Start by reviewing the controls as a small interface system, not as separate one-off buttons.

Likely questions:

- Should the controls remain split bottom-left and bottom-right, or become a more coherent toolbar?
- Should the stem slider be more discoverable or visually integrated?
- Should randomize feel like a testing/dev control, or part of the clock experience?
- How should hover/active states read against both light and dark themes?
- How much should controls disappear when idle versus remain available?

Good first step:

1. Open or refresh the existing in-app browser preview.
2. Inspect the current control layout at desktop and narrow/mobile sizes.
3. Improve the controls in `index.html` without changing clock behavior.
