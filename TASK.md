# TASK.md

## Next task

Two items, in order:

1. **Edit `docs/architecture-analog.md`** — evaluate and rewrite for:
   - Redundancy / verbosity: cut repeated ideas, over-explained trivialities
   - Information-density: more signal per sentence
   - Efficiency: same information, fewer words

2. **Enhance `analog.html`** — aesthetic improvements to the analog clock (specifics TBD with Saad at task start)

---

## Project

A minimal two-page static site hosted on GitHub Pages. No build system. No dependencies. Single HTML files.

```
github-pages/
├── index.html          — Digital clock: 24h, HH:MM + SS, date. setInterval 1s.
├── analog.html         — Bauhaus analog clock (see below)
├── favicon.png         — 32×32 PNG
└── docs/
    ├── sitemap.md
    └── architecture-analog.md   ← target of task 1
```

Navigation: a `position: fixed` tab at bottom-right links the two pages. `○` on digital → analog. `•` on analog → digital. Opacity 0.18, 0.55 on hover.

---

## `analog.html` — current state

**Coordinate system:** SVG `viewBox="-1 -1 2 2"`. Origin at centre. All geometry in normalised `−1 → +1` space. Sized `min(70vw, 70vh)`.

**Layer order (z-order / paint order):**
1. `<rect class="bg">` — background fill, gives blend modes a known base
2. `<g id="ticks">` — 60 tick marks (12 major, 48 minor), generated in JS
3. `<g id="hHand">` — hour hand
4. `<g id="mHand">` — minute hand
5. `<g id="sHand">` — second hand + lollipop circle
6. `<circle class="pip">` — centre dot, painted last

**Blend mode:** All elements (`.tick`, `.hand`, `.pip`) use `mix-blend-mode: difference; fill: white; stroke: white`. On dark bg (~0.05): white renders white. On light bg (~0.96): white renders near-black. Where hands cross: upper hand inverts lower's rendered colour. Cascades through all three hands.

**Hands — geometry (SVG rect, tip toward −y):**

| Hand | Width | Tip (y) | Tail (+y) | Extra |
|---|---|---|---|---|
| Hour | 0.060 | −0.500 | +0.070 | — |
| Minute | 0.036 | −0.755 | +0.090 | — |
| Second | 0.014 | −0.862 | +0.090 | circle at cy=0.210, r=0.040 |

**Ticks:**
- Major (×12, `i % 5 === 0`): `r 0.780 → 0.940`, `stroke-width 0.026`
- Minor (×48): `r 0.868 → 0.940`, `stroke-width 0.009`
- Endpoint math: `x = sin(a) * r`, `y = −cos(a) * r` (−cos so 0° = 12 o'clock)

**Animation:** `requestAnimationFrame`. Fractional time cascade:
```js
const s = seconds + ms/1000
const m = minutes + s/60
const h = (hours % 12) + m/60
// angles: h×30°, m×6°, s×6°
```
Each hand group gets `transform="rotate(deg)"` around origin every frame.

**Theme:** `@media (prefers-color-scheme)` on `body` background + `.bg` SVG fill. No JS needed for theming.

---

## `architecture-analog.md` — critique starting points

Read the file before editing. Some things to look for:
- The blend mode table re-explains what `difference` does after already stating the formula — likely redundant
- The "Stack and dependencies" section may over-explain the obvious (no build system etc.)
- "Design language" section opens with a definition of Bauhaus that may be unnecessary context
- Hand proportions section at the end partially restates the geometry table from the Technical section
- The three named blend effects (hand-on-hand, hand-on-tick, centre accumulation) are the most original content — preserve these
