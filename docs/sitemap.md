# Sitemap

```
github-pages/
├── index.html          — Digital clock  (default landing page)
├── analog.html         — Bauhaus analog clock
├── favicon.png         — Site favicon (32×32 PNG)
└── docs/
    ├── sitemap.md      — This file
    └── architecture-analog.md
```

## Pages

### `/` · `index.html`
A minimal 24-hour digital clock.
Displays `HH:MM` in large weight-100 type with seconds in a smaller weight alongside, and the current date below. Ticks every second via `setInterval`. Adapts to system dark/light mode.

### `/analog.html`
A Bauhaus-inspired monotone analog clock.
SVG-rendered with smooth `requestAnimationFrame` animation. Three geometric hands (hour, minute, second) with `mix-blend-mode: difference` blending — hands invert each other's colour where they cross. No numerals; geometric tick marks only.

## Navigation

The two pages are linked by a persistent tab in the bottom-right corner of each page.

| Current page | Tab icon | Destination |
|---|---|---|
| `index.html` | ○ thin circle | `analog.html` |
| `analog.html` | • filled dot | `index.html` |

The tab sits at 18% opacity and rises to 55% on hover.
