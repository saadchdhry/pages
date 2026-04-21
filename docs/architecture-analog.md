# Architecture — Analog Clock (`index.html`)

## Technical Architecture

### Stack and dependencies

Single self-contained HTML file; no external dependencies, no build step. Markup, styles, and logic in one file; rendering via inline SVG.

### Coordinate system

The SVG uses `viewBox="-1 -1 2 2"`, placing the origin at the canvas centre. All geometry is in normalised `−1 → +1` space; the outer tick ring sits at radius `0.940`. Sized `min(70vw, 70vh)`, the clock scales to any viewport without coordinate changes.

### SVG layer order

DOM order determines paint order (later = on top):

```
<rect class="bg"/>        ← 1. Background fill (known, stable base for blend modes)
<circle class="pip"/>     ← 2. Centre pip — below hands so it blends only against bg
<g id="ticks"/>           ← 3. Hour tick marks (+ calendar markers)
<g id="minTicks"/>        ← 4. Minute tick marks (toggleable, hidden by default)
<g id="hHand"/>           ← 5. Hour hand
<g id="mHand"/>           ← 6. Minute hand
<g id="sHand"/>           ← 7. Second hand (includes lollipop circle)
```

The pip sits just above the background so it blends only against the background fill, making it reliably visible in both light and dark modes. Placing it above the hands caused it to cancel out under three stacked `difference` layers.

### Blend mode mechanics

Every clock element (`.tick`, `.hand`, `.pip`) carries:

```css
mix-blend-mode: difference;
fill: white;
stroke: white;
```

`difference` computes `|source − destination|` per channel. On a near-black background (`#0c0c0c`), white reads as near-white; on a near-white background (`#f5f5f3`), it reads as near-black. Where hands overlap, each successive layer inverts the rendered colour of whatever lies below it.

### Design constants

```js
const PHI  = (1 + Math.sqrt(5)) / 2;   // φ ≈ 1.618 — golden ratio
let   stem = 0.005;                     // universal base dimension, slider-controlled

sW = stem;                              // second hand / minor tick width
mW = stem * PHI * PHI;                 // minute hand / major tick width  ≈ 0.013
hW = stem * PHI * PHI * PHI * PHI;    // hour hand width                 ≈ 0.034
```

All stroke widths and many proportions derive from `stem` and `PHI`, so the entire clock rescales harmonically when the slider is moved.

### Tick mark generation

Generated at page load (and on slider change) inside `buildTicks()`. 60 positions total; `major = (i % 5 === 0)` selects the 12 hour markers.

```js
const tickOuter  = 0.940;
const majorInner = 0.860;   // major tick length = 0.080
const minorInner = tickOuter - majorLen / (PHI * PHI);  // minor tick length ÷ φ²
```

Endpoint formula — `−cos` puts position 0 at 12 o'clock:

```js
x = Math.sin(a) * radius
y = −Math.cos(a) * radius
```

### Hand geometry

Hands are SVG `<rect>` elements centred on the origin, tip toward `−y`:

| Hand | Width | Tip (`y`) | Tail (`+y`) |
|---|---|---|---|
| Hour | `hW ≈ 0.034` | −0.500 | +0.070 |
| Minute | `mW ≈ 0.013` | −0.755 | +0.090 |
| Second | `sW = 0.005` | −0.862 | +0.090 |

The second hand also carries a circle (`r = 0.040`) at `cy = 0.210` — the lollipop counterweight.

### Animation and clock modes

Two modes, toggled by the `#fpsToggle` button and persisted to `localStorage` key `'fps'`:

**Continuous (30 FPS):** fractional seconds (`getSeconds() + ms/1000`) cascade through minute and hour, giving every hand a perfectly smooth sweep. `requestAnimationFrame` is throttled to 30 FPS via a timestamp delta.

**Ticking (1 FPS):** `getSeconds()` (integer) snaps the second hand to each marker position. On page load, a startup sweep animates all hands from 12:00 to their correct positions at 240 deg/sec before dropping to 1 FPS.

```js
const s = targetFps === 30
            ? now.getSeconds() + ms / 1000   // continuous
            : now.getSeconds();              // ticking — snaps to markers
```

The RAF loop uses `setAttribute('transform', 'rotate(deg)')` on each hand `<g>`, which rotates around the SVG origin (clock centre).

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

**Hand-on-hand inversion.** Where two hands cross, the upper inverts the lower's colour. In dark mode a white hand crossing another produces a black stripe — a crisp negative-space line.

**Hand-on-tick inversion.** As hands sweep over tick marks, they invert them momentarily. In dark mode a white hand over a white tick causes the tick to briefly disappear (white − white = black). The face itself reacts to the hands.

**Centre accumulation.** All three hands converge near the pip, which blends against the full accumulated inversion of overlapping hands — a small region of high-contrast complexity that anchors the face.

### UI controls

| Control | Position | Behaviour |
|---|---|---|
| `#stemSlider` | `fixed; bottom: 32px; left: 24px` | Adjusts stroke weight; persisted to `localStorage` |
| `#minToggle` | `fixed; bottom: 24px; right: 24px` | Shows/hides 48 minute ticks; persisted |
| `#fpsToggle` | `fixed; bottom: 72px; right: 24px` | Toggles continuous / ticking mode; full-circle vs broken-circle icon; persisted |

All three controls render at 18% opacity, rising to 55% on hover, so they fade into the background when not in use.
