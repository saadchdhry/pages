# TASK.md

## Current task

**Hour tick animation** — when the second hand sweeps past an hour marker tick, that tick briefly rotates around its own axis (a "nudge" or "spin" effect).

---

## Project state

### File structure

```
github-pages/
├── index.html          ← the clock (single page)
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
<circle class="pip"/>       ← centre pip, r=0.35 — rendered BELOW everything else
<g id="ticks"/>             ← 12 hour tick marks (generated in JS)
<g id="minTicks"/>          ← 48 minute tick marks (toggleable, hidden by default)
<g id="hHand"/>             ← hour hand
<g id="mHand"/>             ← minute hand
<g id="sHand"/>             ← second hand + lollipop circle
```

**Pip is at the bottom** — this was a deliberate fix. The pip was previously topmost and invisible due to `mix-blend-mode: difference` cancellation from three stacked hands. Moving it just above `<rect class="bg">` means it blends only against the background, making it reliably visible in both dark and light modes.

### Blend mode

`.tick`, `.hand`, `.pip` all use `mix-blend-mode: difference; fill: white; stroke: white`.
`.hand` and `.pip` additionally have `stroke: none` (prevents default SVG stroke-width: 1, which equals half the clock width in this coordinate space).

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

### Tick geometry (current state)

```js
const tickOuter  = 0.940;
const majorInner = 0.860;
const majorLen   = tickOuter - majorInner;              // 0.080
const minorInner = tickOuter - majorLen / (PHI * PHI);  // majorLen ÷ φ² ≈ 0.909
```

Minor tick length = major tick length ÷ φ² — this is a deliberate PHI relationship introduced in the last session.

### Dimension table (at default `stem = 0.005`)

| Element | Property | Value / Formula |
|---|---|---|
| Clock size | CSS | `min(70vw, 70vh)` |
| Hour ticks (×12) | stroke-width | `mW = stem × φ²` |
| Minute ticks (×48) | stroke-width | `sW = stem` |
| Major tick length | geometry | `0.080` (hardcoded) |
| Minor tick length | geometry | `majorLen ÷ φ²` ≈ 0.031 |
| Hour hand | width | `hW = stem × φ⁴ ≈ 0.034` |
| Hour hand | tip / tail | `−0.500` / `+0.070` |
| Minute hand | width | `mW = stem × φ² ≈ 0.013` |
| Minute hand | tip / tail | `−0.755` / `+0.090` |
| Second hand | width | `sW = stem = 0.005` |
| Second hand | tip / tail | `−0.862` / `+0.090` |
| Lollipop circle | cy / r | `0.210` / `0.040` |
| Centre pip | r | `0.35` |

### UI controls

| Control | Position | Behaviour |
|---|---|---|
| `#stemSlider` | `fixed; bottom: 32px; left: 24px` | min=2 max=20 step=1; `stem = value/1000`; persisted to `localStorage` |
| `#minToggle` | `fixed; bottom: 24px; right: 24px` | Toggles `#minTicks` visibility; persisted to `localStorage` |

---

## The animation task

### Effect description

Every time the second hand sweeps past an hour marker tick, that tick briefly **rotates around its own radial axis** — a subtle mechanical "nudge" as if the hand physically disturbed it.

- The second hand completes one rotation per 60 seconds
- There are 12 hour markers at 0°, 30°, 60°, 90°, 120°, 150°, 180°, 210°, 240°, 270°, 300°, 330°
- The second hand passes each hour marker once every 5 seconds

### What "rotates around its own axis" means

The tick is a radial line segment. Rotating it around its own axis means it **spins around its own midpoint** — the line stays in place but flips/rotates, like a needle spinning on its centre pin.

### Implementation considerations

1. **Trigger detection** — in the `draw()` RAF loop, compare the second hand's current angle against each hour tick's angle. Fire when they coincide (within a small threshold, or on the exact integer-second crossing).

2. **Tick element identity** — hour ticks are currently generated anonymously in `buildTicks()`. Each needs a stable reference (e.g. `data-hour="N"` or individual `id`) so the animation can be targeted per tick.

3. **Animation mechanism** — SVG `transform` rotate on the tick's own centre point. Options:
   - CSS `@keyframes` class toggled via JS (clean, GPU-accelerated)
   - JS-driven `setAttribute('transform', ...)` in the RAF loop (more control but more code)

4. **Blend mode** — the tick uses `mix-blend-mode: difference`. The animation must not break this. A CSS animation on `transform` is safe; avoid animating `fill` or `opacity` which would interact with the blend mode unexpectedly.

5. **One-shot trigger** — the animation should fire once per pass, not loop continuously while the hand overlaps. Use a "fired" flag or compare with the previous frame's angle to detect the crossing edge.
