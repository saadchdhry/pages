# TASK.md

## Current task

**Calendar visualisation** — encode month, day-of-week, and date-of-month onto the clock face using purely visual/geometric cues. No text or labels.

The month implementation from the previous session was discarded and is **not present in the current code**. Start fresh on all three.

---

## Project state

### File structure

```
github-pages/
├── index.html          ← original clock (to be deprecated)
├── index.html      ← active working file — all new work goes here
├── favicon.png
└── docs/
    ├── sitemap.md
    └── architecture-analog.md
```

**Work only in `index.html`.**

### SVG coordinate system

`viewBox="-1 -1 2 2"`. Origin at centre. All geometry in normalised `−1 → +1` space. Clock element is `min(70vw, 70vh)` with `border-radius: 50%`.

### Layer order (paint order in SVG)

```
<rect class="bg"/>          ← background fill
<circle class="pip"/>       ← centre pip r=0.35 — below everything
<g id="ticks"/>             ← 12 hour ticks + any calendar markers
<g id="minTicks"/>          ← 48 minute ticks (toggleable, hidden by default)
<g id="hHand"/>             ← hour hand
<g id="mHand"/>             ← minute hand
<g id="sHand"/>             ← second hand + lollipop circle
```

### Blend mode

All `.tick`, `.hand`, `.pip` use `mix-blend-mode: difference; fill: white; stroke: white`.
**Do not animate `fill` or `opacity`** — that interacts badly with blend mode. Only animate `transform`.

### Design constants (JS)

```js
const PHI  = (1 + Math.sqrt(5)) / 2;   // φ ≈ 1.618
let   stem = 0.005;                     // slider-controlled base dimension
// sW = stem, mW = stem×φ², hW = stem×φ⁴
```

### Tick geometry

```js
const tickOuter  = 0.940;
const majorInner = 0.860;
const majorLen   = 0.080;               // tickOuter − majorInner
const minorInner = tickOuter - majorLen / (PHI * PHI);  // ≈ 0.909
```

Major tick midpoint radius = `(tickOuter + majorInner) / 2 = 0.900`.

### Dimension table (default `stem = 0.005`)

| Element | Property | Value |
|---|---|---|
| Hour ticks (×12) | stroke-width | `mW = stem × φ² ≈ 0.013` |
| Minute ticks (×48) | stroke-width | `sW = stem = 0.005` |
| Major tick length | geometry | `0.080` |
| Minor tick length | geometry | `majorLen ÷ φ² ≈ 0.031` |
| Hour hand tip/tail | y | `−0.500 / +0.070` |
| Minute hand tip/tail | y | `−0.755 / +0.090` |
| Second hand tip/tail | y | `−0.862 / +0.090` |
| Lollipop circle | cy / r | `0.210 / 0.040` |
| Centre pip | r | `0.35` |

### Clock modes

Two modes, persisted to `localStorage` key `'fps'`:

| Mode | FPS | `targetFps` |
|---|---|---|
| Continuous | 30 | `30` |
| Ticking | 1 | `1` |

**Startup sweep** (ticking mode only): on page load, all hands animate from 12:00 to the correct time at 240 deg/sec (30 FPS), then drop to 1 FPS. Governed by `startupAnim` flag in `draw()`.

### UI controls

| Control | Position | Behaviour |
|---|---|---|
| `#stemSlider` | `fixed; bottom: 32px; left: 24px` | `stem = value/1000`; persisted |
| `#minToggle` | `fixed; bottom: 24px; right: 24px` | Toggles `#minTicks`; persisted |
| `#fpsToggle` | `fixed; bottom: 72px; right: 24px` | Toggles continuous/ticking mode; full circle icon vs broken circle icon; persisted |

### `buildTicks()` structure

Called inside `build()` on load and on slider change. Clears and regenerates both `#ticks` and `#minTicks`. The loop runs `i = 0…59`; `major = (i % 5 === 0)`. Any calendar markers added here must survive `build()` being re-called (i.e. they must be regenerated each call, not appended once).

---

## The calendar task

### Design principle

Represent month, day-of-week, and date-of-month **visually and geometrically** — no text, no labels. Each piece of information maps onto the existing clock geometry.

### Month (12 positions)

- The 12 hour markers map to the 12 months: position 0 (12:00) = January, position 1 (01:00) = February, … position 11 (11:00) = December.
- `new Date().getMonth()` returns `0`–`11`.
- The current month's hour marker is visually distinguished from the other 11. **Implementation is open** — the previous approach (filled dot replacing the line) was tried and discarded. Approach this fresh. Ask clarifying questions before implementing.

### Day of week (7 values)

- Not yet designed. Ask clarifying questions before implementing.

### Date of month (1–31)

- Not yet designed. Ask clarifying questions before implementing.

### Constraints for all three

- Use only SVG geometry (`<line>`, `<circle>`, `<rect>`, `<path>`) inside `#ticks` (or a new dedicated `<g>` layer added to the SVG).
- All new elements must carry `class="tick"` (or equivalent) so blend mode is inherited.
- Do not add text nodes to the SVG.
- Do not break the existing tick geometry or hand behaviour.
- Ask clarifying questions with numbered items and multi-choice options before implementing any of the three.
