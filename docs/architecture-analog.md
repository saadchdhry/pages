# Architecture — Analog Clock (`analog.html`)

## Technical Architecture

### Stack and dependencies

The page is a single self-contained HTML file with no external dependencies, no build step, and no JavaScript libraries. Everything — markup, styles, and logic — lives in one file. Rendering is done entirely via inline SVG.

### Coordinate system

The SVG uses `viewBox="-1 -1 2 2"`, placing the origin `(0, 0)` at the exact centre of the canvas. All geometry is expressed in a normalised `−1 → +1` space where the outer ring of ticks sits at radius `0.94`. This makes the clock inherently resolution-independent: the SVG element is sized with `min(70vw, 70vh)` in CSS and scales smoothly to any viewport without any coordinate changes.

### SVG layer order

Elements are written top-to-bottom in the DOM, which determines their paint order (later = painted on top):

```
<rect class="bg"/>        ← 1. Background fill (known, stable base for blend modes)
<g id="ticks"/>           ← 2. Tick marks
<g id="hHand"/>           ← 3. Hour hand
<g id="mHand"/>           ← 4. Minute hand
<g id="sHand"/>           ← 5. Second hand  (includes lollipop circle)
<circle class="pip"/>     ← 6. Centre pip
```

This ordering is intentional: each layer blends against the full composited stack below it, which is what produces the inversion effects described in the blend mode section.

### Blend mode mechanics

Every clock element (`.tick`, `.hand`, `.pip`) carries:

```css
mix-blend-mode: difference;
fill: white;
stroke: white;
```

`difference` blending computes `|source − destination|` per channel. The behaviour across the two colour themes:

| Context | Source (white = 1.0) | Backdrop | Result |
|---|---|---|---|
| Dark bg, no overlap | 1.0 | ~0.05 (`#0c0c0c`) | ~0.95 → near-white |
| Light bg, no overlap | 1.0 | ~0.96 (`#f5f5f3`) | ~0.04 → near-black |
| Hand over hand (dark) | 1.0 | ~0.95 (prior hand) | ~0.05 → inverts to black |
| Hand over hand (light) | 1.0 | ~0.04 (prior hand) | ~0.96 → re-inverts to white |

The background `<rect>` is critical: it gives every blend operation a known, stable base. Without it, the blend would target whatever the HTML `body` background happens to be, which can vary across browsers and compositing implementations.

### Tick mark generation

Tick marks are created in JavaScript at page load via `createElementNS`. For each of the 60 positions, the angle `a = i × 6°` is converted to radians and the line endpoints are computed using unit-circle trigonometry:

```javascript
x = Math.sin(a) * radius
y = −Math.cos(a) * radius   // −cos because SVG y-axis points down;
                             // at a=0 the line points toward −y (12 o'clock)
```

The 12 major ticks (at `i % 5 === 0`) are drawn from `r = 0.780` to `r = 0.940` at `stroke-width 0.026`. The 48 minor ticks run from `r = 0.868` to `r = 0.940` at `stroke-width 0.009`. All ends use `stroke-linecap: butt` for flat, square terminations.

### Hand geometry

Hands are plain SVG `<rect>` elements (and one `<circle>` for the second-hand lollipop). Each hand rectangle is drawn in its own local coordinate system centred on the origin, with its tip pointing toward `−y` (12 o'clock) and a short tail in `+y` as a counterweight:

| Hand | Width | Tip (`y`) | Tail (`+y`) |
|---|---|---|---|
| Hour | 0.060 | −0.500 | +0.070 |
| Minute | 0.036 | −0.755 | +0.090 |
| Second | 0.014 | −0.862 | +0.090 |

The second hand also has a circle (`r = 0.040`) centred at `cy = 0.210` — the Bauhaus lollipop counterweight.

### Animation

Time is read with `new Date()` on every frame. Fractional positions are derived by cascading sub-unit remainders so that every hand moves continuously rather than jumping:

```javascript
const ms = now.getMilliseconds();
const s  = now.getSeconds()       + ms / 1000;   // seconds + ms fraction
const m  = now.getMinutes()       + s  / 60;     // minutes + seconds fraction
const h  = (now.getHours() % 12) + m  / 60;     // hours   + minutes fraction
```

Angles are then `s × 6°`, `m × 6°`, `h × 30°`. Each hand's `<g>` receives a `transform="rotate(deg)"` attribute on every frame, which SVG applies around the origin `(0, 0)` — coinciding with the clock centre. The loop is driven by `requestAnimationFrame`, which ties updates to the display refresh rate (~60 fps) and automatically pauses when the tab is hidden.

---

## Visual UI Architecture

### Design language

The clock draws on Bauhaus principles: pure geometric primitives, strict functional reduction, and no ornamentation. There are no numerals, no decorative borders, no drop shadows. Every element exists because it carries information; nothing exists for aesthetics alone. The aesthetic emerges from the geometry and the blend mode behaviour.

### Colour and theme

The palette is strictly monotone. A single foreground value (`white`, before blending) is used for every element. The perceived colour in the browser is entirely determined by `mix-blend-mode: difference` acting against the background:

- **Dark mode** (`#0c0c0c`): elements read as bright, near-white forms on a very dark field.
- **Light mode** (`#f5f5f3`): elements read as near-black forms on a warm off-white field.

The theme switch requires no JavaScript — it is handled entirely by `@media (prefers-color-scheme)` on the `body` background and the `.bg` SVG fill.

### Visual hierarchy

The clock face has four levels of visual weight, from heaviest to lightest:

1. **Major tick marks** — 12 wide, long marks define the hour positions and bound the clock face. They are the heaviest stroke and serve as the structural skeleton.
2. **Hour and minute hands** — thick and medium-width rectangles respectively. Their mass communicates time at a glance and dominates the centre of the face.
3. **Second hand** — a very thin rod with a lollipop circle counterweight. It is the most active element but visually the lightest, sitting below the heavier hands in perceptual weight.
4. **Minor tick marks and centre pip** — fine detail that resolves on closer inspection but does not compete with the hands.

### Blend mode as aesthetic layer

`mix-blend-mode: difference` is not just a technical mechanism — it is the primary visual signature of the clock. Three effects arise from it:

**Hand-on-hand inversion.** Where two hands cross, the upper hand inverts the colour of the lower. In dark mode a white hand crossing another white hand produces a black stripe at the intersection — a crisp, graphic negative-space line. The effect cascades: the third hand crossing a crossing point re-inverts back toward white, creating a layered depth that no flat rendering could produce.

**Hand-on-tick inversion.** As hands sweep over tick marks, they invert them momentarily. In dark mode, a white hand passing over a white tick mark causes the tick to briefly disappear (white − white = black, reading as the background). This makes the clock subtly animate beyond just the hand movement — the face itself reacts.

**Centre accumulation.** All three hands converge at the centre pip. The pip is the last element painted, so it blends against the full accumulated inversion of three overlapping hands. The result is a small region of high-contrast complexity that anchors the face visually.

### Proportional decisions

The clock occupies `min(70vw, 70vh)` — 70% of the shortest viewport dimension. This leaves 15% breathing room on each side, which is deliberate: Bauhaus design treats negative space as a compositional element. The clock does not fill the screen; it sits in it.

Hand proportions follow a strict diminishing ratio: the hour hand is the widest (0.060 units), the minute hand is 60% as wide (0.036), and the second hand is 23% as wide (0.014). Length follows the inverse: each successive hand reaches closer to the tick ring, reinforcing that the faster hands govern finer detail.
