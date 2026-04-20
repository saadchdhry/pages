# TASK.md

## Current task

**Fix the centre pip** — it is invisible on screen. Understand why, and implement a working solution.

---

## What was completed in the previous session

1. **Persistent settings** ✅ — `stem` and `minTicks` are now saved to `localStorage` and restored on load.
2. **Consolidate to one page** ✅ — `analog.html` renamed to `index.html`; old digital clock `index.html` deleted; `.tab` nav dot and all its CSS removed; `#minToggle` repositioned from `bottom: 70px` to `bottom: 24px` now that the tab is gone.

---

## The pip problem

### What the pip is

`<circle cx="0" cy="0" r="0.036" class="pip" />` — painted last (topmost layer) in the SVG. Its purpose is to cap the hand roots at the centre with a clean, readable circle — a standard watchmaking detail that hides the mechanical pivot.

### Why it is invisible — root cause

The entire clock uses `mix-blend-mode: difference; fill: white; stroke: white` on `.tick`, `.hand`, and `.pip`. This works well for ticks and hands because they blend against the background rect.

The pip is painted on top of all three hands. All three hands always pass through the origin (their rotation pivot), so the backdrop at the pip's location is the composited result of three stacked difference-blended white elements. Stacked difference operations at the centre cycle the colour:

```
bg (dark ≈ 12)  →  hand 1: 255−12 = 243  →  hand 2: 255−243 = 12  →  hand 3: 255−12 = 243
pip: 255 − 243 = 12  ≈  background colour  →  invisible
```

The pip cancels itself out against the composited hand stack beneath it. This is true regardless of `r` value — changing the radius does not fix it.

### What was tried (did not work, changes discarded)

- **`stroke: none` on `.pip`** — the pip was previously also suffering from SVG's default `stroke-width: 1` (= 1 full user unit in this coordinate space, enormous). Adding `stroke: none` fixed the stroke bloat but did not fix the blend-mode cancellation. The pip remained invisible.
- **Explicit background-matching fill, no blend mode** — attempted removing `.pip` from the difference system and giving it `fill: #0c0c0c` (dark) / `fill: #f5f5f3` (light) via media queries, with `stroke: none`. This was discarded; the user wants a different approach.

### What is known and agreed

- The pip's `stroke` must be `none` (or an explicit small value) — the default SVG `stroke-width: 1` in this coordinate space is half the clock width, which is wrong.
- The pip needs to be **visible** and render as a distinct circle at the centre.
- The pip size target: `r = lollipop_r × (1 + PHI²) = 0.040 × 3.618 ≈ 0.145` — large enough to visually cap and extend beyond the lollipop circle on the second hand.
- The fix must work in both dark and light colour schemes.

---

## Project state (current `index.html`)

### File structure

```
github-pages/
├── index.html          ← the clock (was analog.html)
├── favicon.png
└── docs/
    ├── sitemap.md
    └── architecture-analog.md
```

### SVG coordinate system

`viewBox="-1 -1 2 2"`. Origin at centre. All geometry in normalised `−1 → +1` space. Clock element is `min(70vw, 70vh)` with `border-radius: 50%`.

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

`.tick`, `.hand` use `mix-blend-mode: difference; fill: white; stroke: white`. `.hand` additionally has `stroke: none`. The `<rect class="bg">` provides a stable base.

### Design constants (JS)

```js
const PHI  = (1 + Math.sqrt(5)) / 2;   // φ ≈ 1.618
let   stem = 0.005;                     // universal base dimension — slider-controlled
let   sW, mW, hW;

function updateWidths() {
  sW = stem;
  mW = stem * PHI * PHI;              // stem × φ²
  hW = stem * PHI * PHI * PHI * PHI;  // stem × φ⁴
}
```

### Dimension table (at default `stem = 0.005`)

| Element | Property | Value / Formula |
|---|---|---|
| Clock size | CSS | `min(70vw, 70vh)` |
| Hour ticks (×12) | stroke-width | `mW` |
| Minute ticks (×48) | stroke-width | `sW` |
| Hour hand | width | `hW = stem × φ⁴ ≈ 0.0343` |
| Hour hand | tip / tail | `−0.500` / `+0.070` |
| Minute hand | width | `mW = stem × φ² ≈ 0.0131` |
| Minute hand | tip / tail | `−0.755` / `+0.090` |
| Second hand | width | `sW = stem = 0.005` |
| Second hand | tip / tail | `−0.862` / `+0.090` |
| Lollipop circle | cy / r | `0.210` / `0.040` |
| Centre pip | r | `0.036` (hardcoded — **target: 0.145**) |

### UI controls (current state)

| Control | Position | Behaviour |
|---|---|---|
| `#stemSlider` | `fixed; bottom: 32px; left: 24px` | min=2 max=20 step=1; `stem = value/1000`; persisted to `localStorage` |
| `#minToggle` | `fixed; bottom: 24px; right: 24px` | Toggles `#minTicks` visibility; persisted to `localStorage` |
