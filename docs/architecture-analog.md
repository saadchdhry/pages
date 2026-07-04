# Architecture — Analog Clock (`index.html`)

## Technical Architecture

### Stack and dependencies

Single self-contained HTML file; no external dependencies, no build step. Markup, styles, and logic live in one file; rendering is via inline SVG.

### Coordinate system

The SVG uses `viewBox="-1 -1 2 2"`, placing the origin at the canvas centre. All geometry is in normalised `−1 → +1` space. Sized `min(50vw, 50vh)`, the clock scales to any viewport without coordinate changes.

Two radial reference values govern the dial:

```js
const BASE_TICK_OUTER = 0.940;              // legacy outer tick radius
const DIAL_OUTER_R    = 0.800;              // current outer tick radius
const DIAL_SCALE      = DIAL_OUTER_R / BASE_TICK_OUTER;
```

A `dialR(value)` helper rescales any legacy radial value into the current, tighter dial (`return value * DIAL_SCALE`). Radial positions pass through `dialR()`; stroke widths do **not** — they derive from `stem` and `PHI` and stay in absolute normalised units.

### SVG layer order

DOM order determines paint order (later = on top):

```
<rect class="bg"/>        ← 1. Background fill (known, stable base for blend modes)
<circle id="pip"/>        ← 2. Centre pip — below hands so it blends only against bg
<g id="minTicks"/>        ← 3. Minute tick marks (toggleable, hidden by default)
<g id="ticks"/>           ← 4. Hour ticks + calendar markers
<g id="hHand"/>           ← 5. Hour hand
<g id="mHand"/>           ← 6. Minute hand
<g id="sHand"/>           ← 7. Second hand (includes lollipop circle)
```

Minute ticks sit **below** the hour/calendar tick layer intentionally, so hour ticks and calendar markers always paint over the finer minute ring.

The pip sits just above the background so it blends only against the background fill, making it reliably visible in both light and dark modes. Placing it above the hands caused it to cancel out under three stacked `difference` layers.

### Blend mode mechanics

The base clock elements — `.tick`, `.hand`, `.pip` — carry:

```css
mix-blend-mode: difference;
fill: white;
stroke: white;
```

`difference` computes `|source − destination|` per channel. On a near-black background, white reads as near-white; on a near-white background, it reads as near-black. Where elements overlap, each successive layer inverts the rendered colour of whatever lies below it.

Two calendar markers are deliberate exceptions that opt out of blending and paint in a fixed theme colour:

```css
.hour-tick { mix-blend-mode: normal; stroke: var(--ui-color); }
.month-dot { mix-blend-mode: normal; fill:   var(--ui-color); stroke: none; }
```

The date marker keeps the base blend behaviour:

```css
.date-dot  { mix-blend-mode: difference; fill: white; stroke: none; }
```

`--ui-color` is a CSS custom property that flips with the active theme (see *Colour and theme*).

### Design constants

```js
const PHI  = (1 + Math.sqrt(5)) / 2;   // φ ≈ 1.618 — golden ratio
let   stem = 0.005;                     // universal base dimension, slider-controlled

sW = stem;                              // second hand / minor tick width
mW = stem * PHI * PHI;                 // minute hand / major (hour) tick width ≈ 0.013
hW = stem * PHI * PHI * PHI * PHI;    // hour hand width                       ≈ 0.034

const SECOND_LOLLIPOP_R = 0.040;       // second-hand counterweight + day-circle radius
```

All stroke widths and many proportions derive from `stem` and `PHI`, so the whole clock rescales harmonically when the stem slider moves.

### Tick mark generation

Generated at page load (and on any rebuild) inside `buildTicks()`. 60 positions total; `major = (i % 5 === 0)` selects the 12 hour markers.

```js
const tickOuter  = DIAL_OUTER_R;                         // 0.800
const majorInner = dialR(0.860);
const majorLen   = tickOuter - majorInner;
const minorInner = tickOuter - majorLen / (PHI * PHI);   // minor tick length ÷ φ²
```

Endpoint formula — `−cos` puts position 0 at 12 o'clock:

```js
x =  Math.sin(a) * radius
y = −Math.cos(a) * radius
```

Every minute position gets a minor tick appended to `minTicksEl` (`stroke-width = sW`). Hour ticks are appended to `ticksEl` only when `showHourTicks` is on, carrying `class="tick hour-tick"` at width `mW`.

### Calendar visualisation

Calendar values are encoded geometrically inside `buildTicks()`; no text is used. Marker radii are derived from `stem` and `PHI`:

```js
const monthDotR    = stem * (PHI ** 3) / 2;
const dateDotR     = stem * PHI / 2;
const markerOrbitR = tickOuter + monthDotR + (mW * PHI);  // orbit outside the tick ring
```

**Terminology:** `month` = month of year, `day` = day of week, `date` = date of month.

#### Month (12 values) — `.month-dot`

The current month is a solid dot (`month-dot`, radius `monthDotR`) orbiting **outside** the tick ring at `markerOrbitR`, placed at the month's hour angle. It paints in `var(--ui-color)` with `mix-blend-mode: normal`, so it is not inverted by the hands.

Mapping: January → hour-1 position (`i = 5`), February → hour-2 (`i = 10`), …, December → hour-12 (`i = 0`).

```js
const monthTickIdx = ((curMonth + 1) % 12) * 5;
```

#### Date of month (1–31) — `.date-dot`

The current date is a small dot (`date-dot`, radius `dateDotR`) on the same `markerOrbitR`, positioned at the date's minute angle (`dateTickIdx × 6°`), covering dates 1–31 from just past 12 o'clock round to just past 6 o'clock. It keeps `mix-blend-mode: difference`.

```js
const dateTickIdx        = curDate;
const dateOnHourBoundary = (dateTickIdx % 5 === 0);
```

When minute ticks are hidden and the date does not fall on a 5-minute (hour) boundary, `buildTicks()` renders the four minor ticks of the date's own 5-minute segment as a locating aid, so the isolated date dot has nearby graduations to read against:

```js
if (showDate && !showMinuteTicks && showHourTicks && !dateOnHourBoundary) {
  const segmentStart = Math.floor(dateTickIdx / 5) * 5;
  for (let i = 1; i <= 4; i++) { /* append minor tick at segmentStart + i */ }
}
```

#### Day of week (7 values)

Seven circles sit on an inner perimeter at radius `dialR(0.650)`, at the seven hour positions spanning 9 o'clock → 3 o'clock (top arc). Each renders as a hollow ring (`fill: none`); today's is filled solid. The ring radius is derived from the lollipop radius less half the stem, so it tracks stroke weight:

```js
const DAY_POSITIONS   = [45, 50, 55, 0, 5, 10, 15]; // Mon→Sun
const dayCircleRad    = dialR(0.650);
const dayCircleOuterR = SECOND_LOLLIPOP_R;
const dayCircleR      = Math.max(dayCircleOuterR - (sW / 2), 0);
```

Anchor: Monday = position 9 → Sunday = position 3. `getDay()` (0=Sun…6=Sat) is mapped to a tick index via `DAY_TO_IDX = [15, 45, 50, 55, 0, 5, 10]` to determine which ring is today. These circles carry `class="tick"` and blend as `difference`.

### Hand geometry

Hands are SVG `<rect>` elements centred on the origin, tip toward `−y`. Radial extents pass through `dialR()`; widths are absolute:

```js
hHandEl: mkRect(-hW/2, dialR(-0.500), hW, dialR(0.570), hW/2);
mHandEl: mkRect(-mW/2, dialR(-0.755), mW, dialR(0.845), mW/2);
sHandEl: mkRect(-sW/2, dialR(-0.862), sW, dialR(0.952), sW/2);
sHandEl: mkCircle(0, dialR(0.210), SECOND_LOLLIPOP_R);   // lollipop counterweight
```

The `y` origin is the rect start (`dialR(-0.500)` for the hour hand) and `height` (`dialR(0.570)`) carries it back past the centre, giving each hand a short tail. The second hand also carries a circle (`r = SECOND_LOLLIPOP_R = 0.040`) at `dialR(0.210)` as its counterweight.

### Animation and clock modes

A single `requestAnimationFrame` loop (`draw`) drives all rendering, throttled by a timestamp delta against `FRAME_MS`. The `dt` is capped at 0.1 s so a hidden-then-restored tab cannot skip the animation. Mode is toggled by `#fpsToggle` and persisted to `localStorage` key `'fps'`.

**Continuous (30 FPS):** fractional seconds (`getSeconds() + ms/1000`) cascade through minute and hour, giving every hand a perfectly smooth sweep.

```js
const s = targetFps === 30
            ? now.getSeconds() + ms / 1000   // continuous
            : now.getSeconds();              // ticking — snaps to markers
const m = now.getMinutes() + s / 60;
const h = (now.getHours() % 12) + m / 60;
```

**Ticking (1 FPS):** `getSeconds()` (integer) snaps the second hand to each marker. On load in ticking mode a **startup sweep** runs: `FRAME_MS` is forced back to 30 FPS and all hands animate from 12:00 to their correct positions at `ANIM_SPEED = 240` deg/sec (~1.5 s worst-case), after which the loop drops to `1000 / targetFps` (1 FPS). The sweep is cancelled if the mode is switched mid-animation.

Each frame writes `setAttribute('transform', 'rotate(deg)')` on the hand `<g>` elements, rotating around the SVG origin (clock centre).

---

## Visual UI Architecture

### Design language

Pure geometric primitives, strict functional reduction, no ornamentation. Every element carries information; the aesthetic emerges from geometry and blend-mode behaviour.

### Colour and theme

Colours are driven by three CSS custom properties on `:root` — `--bg-color`, `--ui-color`, `--accent-color` — and the clock body is monotone: base elements paint `white` (before blending) and their perceived colour comes from `difference` blending against `--bg-color`. The opt-out markers (`.hour-tick`, `.month-dot`) instead paint `--ui-color` directly.

Theming is **no longer media-query-only**. A three-mode toggle (`system` / `light` / `dark`) is exposed via `#themeToggle`:

- **`system`** — no `data-theme` attribute; colours follow `@media (prefers-color-scheme)`.
- **`light`** / **`dark`** — an explicit `data-theme` attribute on `<html>` pins the palette via `:root[data-theme="…"]` rules.

The selected mode is persisted to `localStorage` key `themeMode` and applied by `applyThemeMode()`. A small inline `<head>` script reads `themeMode` and sets `data-theme` **before first paint**, preventing a flash of the wrong theme on load.

### Visual hierarchy

Levels of visual weight, heaviest to lightest:

1. **Hour ticks** — 12 medium-length marks (`mW`) define hour positions; drawn in `--ui-color`.
2. **Hour and minute hands** — thick and medium rectangles; their mass communicates time at a glance.
3. **Second hand** — thin rod with lollipop counterweight; most active but perceptually lightest.
4. **Minute ticks, calendar markers, and centre pip** — fine detail that resolves on closer inspection.

### Blend mode as aesthetic layer

`mix-blend-mode: difference` is the primary visual signature of the base clock. Three effects arise from it:

**Hand-on-hand inversion.** Where two hands cross, the upper inverts the lower's colour — a crisp negative-space line.

**Hand-on-tick inversion.** As hands sweep over ticks, they invert them momentarily; a white hand over a white tick briefly cancels it (white − white = black). The face itself reacts to the hands.

**Centre accumulation.** All three hands converge near the pip, which blends against the accumulated inversion of the overlapping hands — a small region of high-contrast complexity that anchors the face.

The two `normal`-blend calendar markers (hour ticks, month dot) sit deliberately outside this system so they remain stable, readable reference points regardless of what sweeps over them.

### UI controls

Controls live in a collapsible rail fixed to the bottom-right (`.controls-shell`). An always-visible dock holds `#controlsToggle` (opens/closes the panel) and `#themeToggle`. The expandable `#controlsPanel` (hidden by default) contains, in order:

| Control | Purpose | Persistence |
|---|---|---|
| `#stemSlider` | Stroke weight (`stem`, 2–20 → `/1000`) | `localStorage['stem']` |
| `#minToggle` | Show/hide 60 minute ticks | `localStorage['minTicks']` |
| `#hourToggle` | Show/hide hour ticks | `localStorage['showHourTicks']` |
| `#dayToggle` | Show/hide day-of-week circles | `localStorage['showDayMarker']` |
| `#dateToggle` | Show/hide date dot | `localStorage['showDateMarker']` |
| `#monthToggle` | Show/hide month dot | `localStorage['showMonthMarker']` |
| `#fpsToggle` | Continuous (30 FPS) / ticking (1 FPS) mode | `localStorage['fps']` |
| `#randomizeBtn` | Dev-only: randomise calendar + time | — |

`#controlsToggle` and `#themeToggle` sit in the always-visible dock; the rest live inside `#controlsPanel`. Pressing **Escape** or clicking outside the shell closes the panel (an outside-click guard tracks whether the interaction started inside the controls). Buttons render at low opacity (~0.26), rising on hover/focus and when `.active`, so they fade into the background when idle.

`#randomizeBtn` is a **dev-mode** control, hidden by default. Dev mode is disabled in production (`setDevMode(false)`); it can be toggled at runtime via `window.setClockDevMode(true)`, which unhides the button. Randomising sets a `testNow` date, rebuilds the ticks, and renders the hands at that instant.
