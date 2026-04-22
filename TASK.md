# TASK.md

## Current task

**Calendar visualisation — visual design pass.** The three calendar markers (month, day, date) are fully implemented and working. This task is a visual design refinement of those markers. No new features; no logic changes. Only geometry and proportions.

---

## Project state

### File structure

```
github-pages/
├── index.html          ← active working file — all new work goes here
├── old.html            ← deprecated, ignore
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
<g id="ticks"/>             ← 12 hour ticks + calendar markers
<g id="minTicks"/>          ← 48 minute ticks (toggleable, hidden by default)
<g id="hHand"/>             ← hour hand
<g id="mHand"/>             ← minute hand
<g id="sHand"/>             ← second hand + lollipop circle
```

### Blend mode

All `.tick`, `.hand`, `.pip` use `mix-blend-mode: difference; fill: white; stroke: white`.
**Do not animate `fill` or `opacity`** — that interacts badly with blend mode. Only animate `transform`.
To override CSS `fill: white` on individual SVG elements, use **inline style** (`element.style.fill = 'none'`), not a presentation attribute — CSS specificity overrides SVG attributes.

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

### `buildTicks()` structure

Called inside `build()` on load and on slider change. Clears and regenerates both `#ticks` and `#minTicks`. All calendar markers live inside `buildTicks()` and are regenerated on every call.

---

## Calendar markers — current implementation

Three values are encoded geometrically. Terminology agreed: `month` = month of year, `day` = day of week, `date` = date of month.

### Month

The current month's hour tick is drawn with stroke-width `stem × φ³` instead of the normal `mW = stem × φ²`. All other 11 hour ticks are normal weight.

Mapping: January → 1:00 position (`i = 5`), February → 2:00 (`i = 10`), …, December → 12:00 (`i = 0`).

```js
const monthTickIdx = ((curMonth + 1) % 12) * 5;
// sw = i === monthTickIdx ? stem * phi3 : mW
```

### Day of week

Seven small circles on an inner perimeter at radius `0.780`, at the seven hour positions spanning 9→3 (top arc). All render hollow (`fill: none`); current day is solid filled.

Anchor: **Monday = 9**, Tuesday = 10, Wednesday = 11, Thursday = 12, Friday = 1, Saturday = 2, **Sunday = 3**.

```js
const DAY_TO_IDX    = [15, 45, 50, 55, 0, 5, 10]; // indexed by getDay() (0=Sun)
const DAY_POSITIONS = [45, 50, 55, 0, 5, 10, 15]; // Mon→Sun, renders all 7
// circle r = stem * phi3;  stroke-width = sW
// hollow override: circle.style.fill = 'none'
```

### Date of month

A single short tick on an outer perimeter at the minute-marker angle for today's date (date × 6°). Covers positions 1–31 (just past 12:00 → just past 6:00).

```js
const dateInner = 0.955;
const dateOuter = 0.985;   // length ≈ 0.030; stroke-width = sW
```

### Dimension table (default `stem = 0.005`)

| Element | Property | Value | ~default |
|---|---|---|---|
| Day circles (×7) | center radius | `0.780` | — |
| Day circles (×7) | circle radius | `stem × φ³` | `≈ 0.021` |
| Day circles (×7) | stroke-width | `stem` | `0.005` |
| Date tick (×1) | inner radius | `0.955` | — |
| Date tick (×1) | outer radius | `0.985` | — |
| Date tick (×1) | stroke-width | `stem` | `0.005` |
| Month tick | stroke-width | `stem × φ³` | `≈ 0.021` |

---

## The visual design task

The functional implementation is complete and correct. The next session focuses purely on visual design:

- Do the proportions feel right at default `stem`? Are the day circles too large / too small?
- Does the month tick read clearly as distinct without being jarring?
- Is the date tick too close to / too far from the outer tick ring?
- Any spacing, sizing, or perimeter radius adjustments needed.

**Ask the user to open the preview and walk through their impressions before proposing changes.**
