# TASK.md

## Current task

**Calendar visualisation — redesign the Date and Month markers.**

This new task starts after a focused visual-design pass on the calendar markers.

- The **Day** marker treatment is now in a good place and should be treated as the accepted baseline.
- The **Date** marker is still the weak point: it is functionally correct, but **not scannable enough by human eye**.
- In the next task, we will try **another approach** for representing the **Date** and likely reconsider the **Month** marker in tandem so they feel like a coherent pair.

This is still a **visual design task**, not a feature-expansion task.

---

## Work scope

### File structure

```text
github-pages/
├── index.html          ← active working file — all new work goes here
├── TASK.md             ← this handoff note
├── old.html            ← deprecated, ignore
├── favicon.png
└── docs/
    ├── sitemap.md
    └── architecture-analog.md
```

**Work only in `index.html`.**

### SVG coordinate system

- `viewBox="-1 -1 2 2"`
- Origin at centre
- All geometry is in normalised `-1 → +1` space
- Clock element is `min(70vw, 70vh)` with `border-radius: 50%`

### Layer order

```text
<rect class="bg"/>          ← background fill
<circle class="pip"/>       ← centre pip r=0.35
<g id="ticks"/>             ← 12 hour ticks + always-visible calendar markers
<g id="minTicks"/>          ← 48 minute ticks (toggleable, hidden by default)
<g id="hHand"/>             ← hour hand
<g id="mHand"/>             ← minute hand
<g id="sHand"/>             ← second hand + lollipop circle
```

### Blend mode rules

All `.tick`, `.hand`, `.pip` use:

```css
mix-blend-mode: difference;
fill: white;
stroke: white;
```

Important:

- Do **not** animate `fill` or `opacity`; only animate `transform`
- To make a day circle hollow, use inline style:

```js
circle.style.fill = 'none';
```

not a presentation attribute, because CSS specificity wins

---

## Design constants and geometry

### Width system

```js
const PHI  = (1 + Math.sqrt(5)) / 2;   // φ ≈ 1.618
let   stem = 0.005;                    // slider-controlled base dimension

// derived widths
// sW = stem
// mW = stem × φ²
// hW = stem × φ⁴
```

### Tick geometry

```js
const tickOuter  = 0.940;
const majorInner = 0.860;
const majorLen   = tickOuter - majorInner;                  // 0.080
const minorInner = tickOuter - majorLen / (PHI * PHI);     // ≈ 0.909
const minorLen   = tickOuter - minorInner;                 // ≈ 0.031
```

---

## Accepted current state

### Day of week

The day markers are the most settled part of the system right now.

- Seven circles on an inner perimeter across the top arc
- Positions span `9 → 3`
- Mapping:

```js
const DAY_TO_IDX    = [15, 45, 50, 55, 0, 5, 10]; // getDay(): 0=Sun
const DAY_POSITIONS = [45, 50, 55, 0, 5, 10, 15]; // Mon→Sun
```

- The perimeter radius for the day-marker centres is:

```js
const dayCircleRad = 0.650;
```

- Day-circle size is **fixed**, not stem-scaled
- The day-circle diameter matches the seconds hand lollipop diameter
- Shared constant:

```js
const SECOND_LOLLIPOP_R = 0.040;
```

- To keep the perceived outer size stable while stroke changes with `stem`, the stroke is made to grow **inward**:

```js
const dayCircleOuterR = SECOND_LOLLIPOP_R;
const dayCircleR      = Math.max(dayCircleOuterR - (sW / 2), 0);
```

- Non-current days are hollow
- Current day is solid filled

This day treatment should be preserved unless the user explicitly wants it revisited.

### Month

Current accepted implementation before the next redesign pass:

- Month uses the **hour tick** at the month position
- Width stays the normal hour-marker width (`mW`)
- Geometry change is by **length only**, extending inward
- Current length multiplier is `PHI^1`

```js
const monthInner = tickOuter - (majorLen * PHI);
const monthTickIdx = ((curMonth + 1) % 12) * 5;
```

Month mapping:

- January → `1:00` position (`i = 5`)
- February → `2:00` (`i = 10`)
- ...
- December → `12:00` (`i = 0`)

### Date

Current implementation is still live, but it is the part the user wants to rethink next.

- Date uses a **minute-width** marker
- Current date tick is derived from the minute-marker geometry
- It is always drawn in `#ticks`, so it remains visible even when `#minTicks` is hidden
- Current inward length multiplier is `PHI^1`

```js
const dateInner  = tickOuter - (minorLen * PHI);
const dateTickIdx = curDate;   // 1–31
```

Current behaviour:

- If the date lands on a normal minute position, draw the minute-style date marker in `#ticks`
- If the date lands on an hour position, draw the hour tick normally, then draw the minute-style date marker on top of it in `#ticks`

This overlap case matters for:

- January 5
- February 10
- March 15
- April 20
- May 25
- June 30

These are the literal month/date shared-angle cases.

---

## Testing aids already added

There is now a **randomize button** in the bottom-left corner above the stem slider.

Purpose:

- Quickly generate a random valid date/time snapshot
- Useful for testing month/date/day combinations and hand positions without waiting for real time

Implementation notes:

- Button writes a coherent random `Date` into:

```js
let testNow = null;
```

- Shared helper:

```js
function getDisplayDate() {
  return testNow ? new Date(testNow.getTime()) : new Date();
}
```

- Both `buildTicks()` and hand rendering read from `getDisplayDate()`
- This keeps Day / Date / Month / Hour / Minute / Second aligned to the same randomized state

This testing control is very useful for the next task and should remain available.

---

## Recently tried and discarded

One alternative date treatment was tried and then discarded:

- **Date as a through-tick crossing the outer ring**

Why it was discarded:

- It still did not feel sufficiently scannable / satisfying
- The user explicitly discarded that change and wants to continue with another approach in a new task

Assume that approach is **not** the direction to continue from.

---

## Next-task objective

Start the next task from this point:

- Keep the accepted **Day** treatment
- Keep the randomize test control
- Revisit the **Date** marker first, because it is the current readability problem
- Consider whether **Month** should also change so the Date/Month pair feels more intentional and legible together

Key design question for the next task:

- What geometric treatment makes the Date immediately readable without making the clock feel noisy or overly symbolic?

Good framing for the next session:

- Open the preview
- Use the randomize button heavily
- Compare candidate Date/Month treatments across many randomized states
- Pay extra attention to the shared-angle month/date cases listed above

