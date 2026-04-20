# Architecture — Analog Clock (`analog.html`)

## Technical Architecture

### Stack and dependencies

Single self-contained HTML file; no external dependencies, no build step. Markup, styles, and logic in one file; rendering via inline SVG.

### Coordinate system

The SVG uses `viewBox="-1 -1 2 2"`, placing the origin at the canvas centre. All geometry is in normalised `−1 → +1` space; the outer tick ring sits at radius `0.94`. Sized `min(70vw, 70vh)`, the clock scales to any viewport without coordinate changes.

### SVG layer order

DOM order determines paint order (later = on top):

```
<rect class="bg"/>        ← 1. Background fill (known, stable base for blend modes)
<g id="ticks"/>           ← 2. Tick marks
<g id="hHand"/>           ← 3. Hour hand
<g id="mHand"/>           ← 4. Minute hand
<g id="sHand"/>           ← 5. Second hand  (includes lollipop circle)
<circle class="pip"/>     ← 6. Centre pip
```

Each layer blends against the full composited stack below it. The background `<rect>` gives every blend operation a known, stable base; without it the blend targets the `body` background, which varies across browsers.

### Blend mode mechanics

Every clock element (`.tick`, `.hand`, `.pip`) carries:

```css
mix-blend-mode: difference;
fill: white;
stroke: white;
```

`difference` computes `|source − destination|` per channel. On a near-black background (~`0.05`), white reads as near-white; on a near-white background (~`0.96`), it reads as near-black. Where hands overlap, each successive layer inverts the rendered colour of whatever lies below it.

### Tick mark generation

60 tick marks are generated at page load. For each position, `a = i × 6°` is converted to radians and endpoints computed:

```javascript
x = Math.sin(a) * radius
y = −Math.cos(a) * radius   // −cos: at a=0 the line points toward −y (12 o'clock)
```

Major ticks (12, at `i % 5 === 0`): `r 0.780 → 0.940`, `stroke-width 0.026`. Minor ticks (48): `r 0.868 → 0.940`, `stroke-width 0.009`. All ends use `stroke-linecap: butt`.

### Hand geometry

Hands are SVG `<rect>` elements, each centred on the origin with tip toward `−y`:

| Hand | Width | Tip (`y`) | Tail (`+y`) |
|---|---|---|---|
| Hour | 0.060 | −0.500 | +0.070 |
| Minute | 0.036 | −0.755 | +0.090 |
| Second | 0.014 | −0.862 | +0.090 |

The second hand also carries a circle (`r = 0.040`) at `cy = 0.210` — the Bauhaus lollipop counterweight.

### Animation

`requestAnimationFrame` calls `new Date()` every frame. Sub-unit remainders cascade so every hand moves continuously:

```javascript
const s  = now.getSeconds()      + ms / 1000;
const m  = now.getMinutes()      + s  / 60;
const h  = (now.getHours() % 12) + m  / 60;
```

Angles: `s × 6°`, `m × 6°`, `h × 30°`. Each hand's `<g>` gets `transform="rotate(deg)"` around the origin every frame. The loop pauses automatically when the tab is hidden.

---

## Visual UI Architecture

### Design language

Pure geometric primitives, strict functional reduction, no ornamentation. Every element carries information; the aesthetic emerges from geometry and blend mode behaviour.

### Colour and theme

Strictly monotone. A single foreground value (`white`, before blending) is used for every element; perceived colour is determined entirely by `difference` blending against the background:

- **Dark mode** (`#0c0c0c`): elements read as near-white on a very dark field.
- **Light mode** (`#f5f5f3`): elements read as near-black on warm off-white.

Theme switching requires no JavaScript — handled entirely by `@media (prefers-color-scheme)` on `body` and `.bg`.

### Visual hierarchy

Four levels of visual weight, heaviest to lightest:

1. **Major tick marks** — 12 wide, long marks define hour positions and bound the face.
2. **Hour and minute hands** — thick and medium-width rectangles; their mass communicates time at a glance.
3. **Second hand** — thin rod with lollipop counterweight; most active but perceptually lightest.
4. **Minor ticks and centre pip** — fine detail that resolves on closer inspection.

### Blend mode as aesthetic layer

`mix-blend-mode: difference` is the primary visual signature of the clock. Three effects arise from it:

**Hand-on-hand inversion.** Where two hands cross, the upper inverts the lower's colour. In dark mode a white hand crossing another produces a black stripe — a crisp negative-space line. The effect cascades: the third hand crossing that point re-inverts toward white, creating layered depth that no flat rendering could produce.

**Hand-on-tick inversion.** As hands sweep over tick marks, they invert them momentarily. In dark mode a white hand over a white tick causes the tick to briefly disappear (white − white = black). The face itself reacts, not just the hands.

**Centre accumulation.** All three hands converge at the pip, which is painted last. It blends against the full accumulated inversion of three overlapping hands — a small region of high-contrast complexity that anchors the face.

### Proportional decisions

The clock occupies 70% of the shortest viewport dimension, leaving 15% breathing room on each side. Negative space is a compositional element; the clock sits in the screen rather than filling it.

Hand width diminishes by a strict ratio (hour → minute → second: 0.060, 0.036, 0.014). Length inverts this: each successive hand reaches closer to the tick ring, reinforcing that faster hands govern finer detail.
