# Sitemap

```
pages/
├── index.html                    — Analog clock (active page)
├── README.md                     — Project overview
├── LICENSE                       — Licence
├── .gitignore
├── favicon.png                   — Site favicon
├── og-image.svg                  — Open Graph / social preview image
├── robots.txt                    — Crawler directives
├── sitemap.xml                   — XML sitemap
├── llms.txt                      — LLM summary
├── llms-full.txt                 — LLM full description
└── docs/
    ├── sitemap.md                — This file
    └── architecture-analog.md    — Technical architecture reference
```

## Pages

### `/` · `index.html`

Minimal SVG analog clock, self-contained (HTML + CSS + JS, no build step). Two clock modes:

- **Continuous** (30 FPS): smooth sweeping hands
- **Ticking** (1 FPS): second hand snaps to markers, with a startup sweep animation on load

Three geometric hands (hour, minute, second + lollipop counterweight). Base elements use `mix-blend-mode: difference`; proportions are derived from the golden ratio. A three-mode theme toggle (system / light / dark) is persisted to `localStorage` and applied before first paint.

Calendar visualisation (no text): the current **month** is a solid dot orbiting outside the tick ring, the **date of month** a small dot at the same orbit on its minute angle, and the **day of week** seven circles on an inner perimeter (hollow ring = inactive, filled = today).

UI lives in a collapsible control rail (bottom-right): stroke-weight slider, minute / hour / day / date / month visibility toggles, and a continuous/ticking mode toggle, alongside an always-visible controls and theme button.
