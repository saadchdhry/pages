# TASK.md

## Current task

**Clock marker blend pass — improve Month marker behavior against minute ticks.**

This task starts after a controls-focused pass on the analog/calendar clock in `index.html`. The next expected work is **not** a controls redesign; it is a visual/geometry pass on marker blending, especially the Month marker interacting with the minute tick ring.

The user has the in-app browser open to:

```text
file:///Users/saad/Library/Mobile%20Documents/iCloud~md~obsidian/Documents/Saad/github-pages/index.html
```

Use that preview for quick visual checks.

---

## Work scope

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

## Current UI controls

The controls are now a flat horizontal rail in the bottom-right.

Code names:

- `#controlsToggle`: one visible control button; reveals/hides `#controlsPanel`.
- `#controlsPanel` / `.controls-panel`: the expanded flat control row.
- `#themeToggle`: always-visible sibling button beside the control toggle; cycles `system / light / dark`.
- `#stemSlider`: controls base stroke width and persists to `localStorage`.
- `#minToggle`: shows/hides minute ticks; persists to `localStorage`.
- `#hourToggle`: shows/hides hour ticks; persists to `localStorage`.
- `#dayToggle`: shows/hides day marker; persists to `localStorage`.
- `#dateToggle`: shows/hides date marker; persists to `localStorage`.
- `#monthToggle`: shows/hides month marker; persists to `localStorage`.
- `#fpsToggle`: toggles continuous/ticking seconds mode.
- `#randomizeBtn`: visible because Dev mode is currently enabled; randomizes a coherent test `Date` into `testNow`.

Current control order:

```text
Slider | Minute Hour | Day Date Month | Ticking Dev | Controls Theme
```

Recent visual choices:

- Expanded controls open horizontally to the left of the control icon.
- `#controlsPanel` has no panel fill, border, blur, or shadow.
- Slider was made visually subtle with a thin custom track/thumb.
- Control icons were redesigned:
  - Minute: 12 thin ticks.
  - Hour: 8 thick ticks.
  - Day: 5 dots on a curve.
  - Date: right half-circle with tiny ticks.
  - Month: circle with 12 tiny ticks.
  - Ticking mode: solid circle for continuous, dotted circle for ticking.

---

## Clock geometry context

The SVG still uses:

```html
<svg id="clock" viewBox="-1 -1 2 2">
```

Coordinate system:

- Internal coordinates run from `-1` to `+1`.
- The origin is the clock center.
- Full SVG radius is `1.000`.
- Visible dial outer radius is `DIAL_OUTER_R = 0.800`.

Current constants:

```js
const PHI  = (1 + Math.sqrt(5)) / 2;
let   stem = 0.005;
let   sW, mW, hW;
const BASE_TICK_OUTER = 0.940;
const DIAL_OUTER_R    = 0.800;
const DIAL_SCALE      = DIAL_OUTER_R / BASE_TICK_OUTER;
const SECOND_LOLLIPOP_R = 0.040;
```

`dialR(value)` scales old accepted radial values by `DIAL_SCALE`. Stroke widths are not scaled by `dialR`.

Important derived values:

- `tickOuter = 0.800`
- `majorInner = dialR(0.860) ~= 0.73191`
- `minorInner ~= 0.77399`
- `monthInner = tickOuter - (majorLen * PHI) ~= 0.68985`
- Date marker center currently uses `dateDotRad = minorInner - 3 * sW`

With stem slider value `11`, the current Date marker radius is:

```text
minorInner - 3 * 0.011 = 0.74099
```

---

## SVG Z order

Current SVG order from bottom to top:

1. `rect.bg`
2. `circle#pip`
3. `g#minTicks`
4. `g#ticks`
5. `g#hHand`
6. `g#mHand`
7. `g#sHand`

`#minTicks` was intentionally moved below `#ticks` so hour/calendar ticks can sit above the minute ring.

---

## Current marker behavior

### Minute ticks

- `#minTicks` now draws ticks at all 60 positions, including hour positions.
- This removed the visual gaps when hour markers are hidden.
- Minute ticks still use `.tick`, so they use `mix-blend-mode: difference`.

### Hour ticks

- Normal hour ticks now receive an additional `hour-tick` class.
- `.hour-tick` uses:

```css
.hour-tick {
  mix-blend-mode: normal;
  stroke: var(--ui-color);
}
```

This was added because minute ticks underneath hour ticks were visually subtracting from the hour ticks. Removing blend mode from normal hour ticks fixed that issue.

### Day marker

Day marker was restored to the accepted seven-circle arc:

- Seven circles on an inner top arc.
- Positions span `9 -> 3`.
- Non-current days are hollow via inline `circle.style.fill = 'none'`.
- Current day is filled.
- Circle size remains based on `SECOND_LOLLIPOP_R`, not stem-scaled.

### Date marker

Date marker is now a tiny filled dot, not an extended tick.

Current code:

```js
const dateDotA   = dateTickIdx * 6 * Math.PI / 180;
const dateDotRad = minorInner - 3 * sW;
const dateDotR   = sW / 2;
```

Interpretation:

- Dot diameter equals the minute marker stroke width (`sW`).
- Dot sits just inside the minute tick ring, moved inward by `3 * sW`.
- Date marker remains visible even when minute ticks are hidden.

### Month marker

Month marker currently replaces the normal hour tick at the month position:

```js
if (showMonth && isMonthTick) {
  appendTick(ticksEl, a, monthInner, tickOuter, mW);
} else if (showHourTicks) {
  appendTick(ticksEl, a, majorInner, tickOuter, mW, 'hour-tick');
}
```

This avoids stacking Month directly over a normal hour tick.

Known issue for next task:

- Month still uses `.tick`, therefore `mix-blend-mode: difference`.
- Because minute ticks now exist at all 60 positions underneath `#ticks`, Month can visually interfere/subtract against the underlying minute tick at the same angle.
- The next task should improve this Month-vs-minute blend issue.

Likely solution direction:

- Consider giving Month its own class, similar to `.hour-tick`, and rendering it with `mix-blend-mode: normal` and `stroke: var(--ui-color)`.
- Be careful: Month is currently a special calendar marker and may need to preserve contrast over the center/pip/hands differently than normal hour ticks.
- Another possible direction is splitting minute ticks at major positions into a separate group and hiding the major-position minute tick whenever Month is visible at that position.
- The user has already accepted removing blend mode from normal Hour markers because it solved the hour/minute subtraction issue.

---

## Blend mode rules and caveats

Base CSS still applies to most clock geometry:

```css
.tick,
.hand,
.pip {
  mix-blend-mode: difference;
  fill: white;
  stroke: white;
}
```

Current exception:

```css
.hour-tick {
  mix-blend-mode: normal;
  stroke: var(--ui-color);
}
```

Important:

- Do not animate `fill` or `opacity` on clock geometry.
- Hand animation should remain transform-based.
- UI controls can use opacity/hover behavior normally.
- Blend-mode bugs often appear only visually in the browser, so refresh and inspect the in-app browser after changes.

---

## Suggested starting point

1. Refresh the in-app browser preview.
2. Turn on Dev mode/randomize if needed; currently it is enabled by default via `setDevMode(true)`.
3. Ensure minute ticks and Month marker are visible.
4. Randomize dates/months until Month lands somewhere easy to inspect.
5. Compare Month marker behavior to normal Hour marker behavior.
6. Try a focused fix, probably a dedicated Month class or major-minute suppression at Month position.
7. Verify in both light and dark/system themes.
