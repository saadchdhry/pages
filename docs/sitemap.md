# Sitemap

```
github-pages/
├── index.html          — Analog clock (active)
├── old.html            — Deprecated original analog clock
├── favicon.png         — Site favicon (32×32 PNG)
└── docs/
    ├── sitemap.md      — This file
    └── architecture-analog.md
```

## Pages

### `/` · `index.html`

Optimised Bauhaus-inspired analog clock. SVG-rendered with two clock modes:

- **Continuous** (30 FPS): smooth sweeping hands
- **Ticking** (1 FPS): second hand snaps to second markers; startup sweep animation on page load

Three geometric hands (hour, minute, second + lollipop counterweight) with `mix-blend-mode: difference` blending. Tick marks sized by golden ratio. UI controls: stroke-weight slider, minute-tick toggle, continuous/ticking mode toggle.

### `/old.html`

Deprecated. The original analog clock, superseded by `index.html`. Kept for reference.
