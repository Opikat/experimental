# FineTune — Typography Auto-Tuning for Figma

Figma plugin that calculates optimal line-height and letter-spacing based on font metrics, weight, size, background brightness, and baseline grid — then exports in code-ready units.

**TypeBalance picks pretty numbers. FineTune picks pretty numbers that work in code.**

## Features

- **30 hand-tuned font profiles** — Inter, Roboto, SF Pro, PP Neue Montreal, Druk, and 25 more. Any unknown font uses category-based fallbacks.
- **Regressive line-height scale** — ~110% for display type, ~140% for body, ~155% for captions. Follows Bringhurst / Material Design 3 paradigm.
- **4px baseline grid snap** — line-height always aligns to vertical rhythm.
- **Background-aware** — auto-detects solid, gradient, and image fills. Adjusts for dark/light.
- **Multi-layer batch** — select a frame with 50 text layers, see deduplicated result cards grouped by font+weight+size. Apply all at once.
- **Auto-apply** — toggle on and values apply instantly when you select text. No extra clicks.
- **Code export** — CSS (`em`), CSS Fluid (`clamp()`), iOS (`kern`/`lineHeightMultiple`), Android (`letterSpacing`/`lineSpacingMultiplier`).

## Quick start

```bash
cd plugin
npm install
npm run build
```

In Figma: **Plugins** > **Development** > **Import plugin from manifest** > select `plugin/manifest.json`.

## Usage

1. Select any text layer or frame containing text layers
2. See calculated values for each unique font configuration
3. Click **Apply to selected** or enable **Auto-apply** in settings
4. Switch export tab (CSS / Fluid / iOS / Android) and copy code

## Line-height scale

| Context | Font size | Target |
|---------|-----------|--------|
| Display (large) | 96-128px+ | ~110% |
| Display | 48-64px | ~120% |
| Display (small) | 32-48px | ~125-130% |
| Body | 14-31px | ~140% |
| Caption | ≤13px | ~155% |

Line-height is snapped to the nearest 4px grid step for vertical rhythm.

## Letter-spacing (tracking) formula

Letter-spacing is calculated as a sum of five factors, each in em:

```
tracking = baseRatio + sizeScale + displayAdj + weightAdj + caseAdj + bgAdj
px value = fontSize * tracking
```

| Factor | What it does | Range |
|--------|-------------|-------|
| **baseRatio** | Per-font base tracking (from font profile) | -0.01 .. 0.01 em |
| **sizeScale** | Small text gets looser, large text tighter | +0.008 em at ≤12px, 0 at 14-24px, down to -0.03 em at 96px+ |
| **displayAdj** | Extra tightening for display context (≥32px) | -0.015 .. -0.03 em (per font) |
| **weightAdj** | Heavier weights → tighter tracking | interpolated from font profile per-weight table |
| **caseAdj** | Uppercase boost | +0.04 .. 0.07 em (per font) |
| **bgAdj** | Dark background → slightly looser | +0.015 em on dark, 0 on light |

Example: Inter Medium 48px, lowercase, dark background:

```
base    =  0.000
size    = -0.010  (48px: full tightening in 24-48 range)
display = -0.022  (Inter's displayTightening)
weight  = -0.002  (Medium w500)
case    =  0.000  (lowercase)
bg      = +0.015  (dark)
─────────────────
total   = -0.019 em → -0.91px
```

## Settings

- **Auto-apply on selection change** — optimized values are immediately applied to every text layer you select. Disable to preview first.
- **Save to Figma Variables** — creates a "FineTune" variable collection with line-height and letter-spacing tokens. Includes Light and Dark mode variants.

## Docs

- [Product description](docs/product.md) — detailed overview, architecture, roadmap
- [PRD](docs/prd-typography-plugin.md) — requirements, formulas, competitive analysis

## Development

```bash
cd plugin
npm run watch    # rebuild on file changes
npm run build    # production build
```

Build output: `plugin/dist/main.js` + `plugin/dist/ui.html` (single file with inlined JS+CSS).

Target: `es2017` (Figma sandbox constraint — no `??`, `?.`).

## License

MIT
