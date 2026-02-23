# Algorithm Improvement Research (v2.1+)

Based on NotebookLM analysis and typographic theory.

## 1. X-height correction (HIGH PRIORITY)

**Golden Ratio Typography (GRT) formula:**
```
c_f = (x - x_0) × f
```
Where `x` = actual xHeightRatio, `x_0 ≈ 0.618` (golden ideal), `f` = scaling factor.

Applied to line-height:
```
lineHeight = fontSize × (baseRatio + xHeightFactor) × contextMultiplier × ...
xHeightFactor = (xHeightRatio - 0.45) × scaleFactor
```

Fonts with large x-height (Roboto 0.528, Verdana 0.545) visually "consume" inter-line space → need more line-height.

**Implementation**: add `xHeightRatio: number` to FontProfile. Estimated values:

| Font | xHeightRatio | Source |
|------|-------------|--------|
| Inter | 0.536 | OS/2 table |
| Roboto | 0.528 | OS/2 table |
| Open Sans | 0.536 | OS/2 table |
| Montserrat | 0.486 | OS/2 table |
| Lato | 0.516 | OS/2 table |
| Poppins | 0.520 | OS/2 table |
| Noto Sans | 0.536 | OS/2 table |
| Raleway | 0.486 | OS/2 table |
| SF Pro | 0.510 | Apple spec |
| Manrope | 0.520 | OS/2 table |
| Playfair Display | 0.460 | OS/2 table |
| Merriweather | 0.480 | OS/2 table |
| DM Sans | 0.520 | OS/2 table |
| Druk | 0.530 | estimated |
| Garamond | 0.430 | classic ref |
| Futura | 0.486 | classic ref |
| Verdana | 0.545 | classic ref |
| Helvetica | 0.523 | classic ref |
| Times New Roman | 0.448 | classic ref |

Category fallback xHeightRatio:
- sans-serif: 0.52
- serif: 0.46
- mono: 0.50
- display: 0.50

## 2. Cyrillic script multiplier (HIGH PRIORITY)

Cyrillic has fewer ascenders/descenders → monotonous "fence" effect → needs more space.

**Detection**: regex on `node.characters`:
```typescript
const cyrillicRatio = (text.match(/[\u0400-\u04FF]/g) || []).length / text.length;
const isCyrillic = cyrillicRatio > 0.3; // dominant script
```

**Coefficients**:
- Line-height: `scriptMultiplier = isCyrillic ? 1.07 : 1.0` (+7%)
- Tracking: `scriptBoost = isCyrillic ? 0.04 : 0.0` (rectangular forms: п, и, н, ш, ц)

Edge case: mixed text "Hello Привет" → use dominant script (>30% threshold).

Some fonts have well-designed Cyrillic (e.g., Golos, PT Sans) — could add `cyrillicQuality: 'native' | 'adapted'` to reduce multiplier for native Cyrillic fonts.

## 3. Enhanced dark mode (MEDIUM PRIORITY)

Current: flat +1.5% LH, +1.5% tracking.
Research: tracking should be +3–8%, especially for small text (irradiation effect).

**Size-dependent bgAdjust for tracking**:
```
fontSize ≤ 12px: bgTrackAdj = 0.05
fontSize 13–20px: bgTrackAdj = 0.035
fontSize 21–32px: bgTrackAdj = 0.02
fontSize > 32px: bgTrackAdj = 0.015
```

Line-height bgAdjust stays at 1.015 (minimal effect on leading).

## 4. Width class: Condensed / Expanded (MEDIUM PRIORITY)

**Detection**: parse `fontName.style` for keywords:
```typescript
const style = fontName.style.toLowerCase();
const isCondensed = style.includes('condensed') || style.includes('narrow') || style.includes('compressed');
const isExpanded = style.includes('expanded') || style.includes('wide') || style.includes('extended');
```

**Coefficients**:
- Condensed → line-height ×1.05 (+5%), tracking +0.01 (loosen)
- Expanded → line-height ×0.97 (-3%), tracking -0.005 (tighten)
- Normal → no adjustment

At small sizes (<14px), condensed fonts need even more tracking: +0.02 instead of +0.01.

## 5. Line width / measure (MEDIUM PRIORITY)

Classic rule: wider columns → more line-height for return saccade.

**Available data**: `node.width` in Figma (px).
**Approximate characters per line**: `charsPerLine ≈ node.width / (fontSize × 0.5)` (rough estimate).

**Multiplier**:
```
charsPerLine < 30: measureMul = 0.95 (short lines → tighter LH)
charsPerLine 30–45: measureMul = 1.0
charsPerLine 45–75: measureMul = 1.0 (optimal range)
charsPerLine > 75: measureMul = 1.05 (wide columns → more LH)
```

Only apply for body context, not display/caption.

## 6. Aperture & stroke contrast (LOW PRIORITY)

Add to FontProfile:
```typescript
aperture?: 'open' | 'neutral' | 'closed';
strokeContrast?: 'low' | 'medium' | 'high';
```

**Aperture** (affects tracking at small sizes):
- closed (Helvetica, Futura): small-size tracking += 0.008
- neutral (Roboto, Inter): no adjustment
- open (Frutiger, Open Sans): small-size tracking -= 0.003

**Stroke contrast** (affects tracking):
- high (Didot, Bodoni, Playfair): tracking += 0.005 (reduce "striping")
- medium (Times, Georgia): no adjustment
- low (Futura, Helvetica, DM Sans): no adjustment

## 7. Sub-12px nonlinear scaling (LOW PRIORITY)

Current: linear sizeScale = 0.008 for ≤12px.
Research suggests nonlinear growth:

```
12px: trackingBoost = 1.0×
10px: trackingBoost = 1.3×
8px: trackingBoost = 1.8×
6px: trackingBoost = 2.5×
```

Formula: `smallBoost = fontSize < 12 ? Math.pow(12 / fontSize, 0.7) : 1.0`

## 8. Apple SF Pro reference values (CALIBRATION)

Use as unit test reference (±15% tolerance):

| Size | Weight | Expected tracking (px) |
|------|--------|----------------------|
| 34pt | Bold | -1.05 |
| 28pt | Bold | -0.80 |
| 22pt | Bold | -0.70 |
| 17pt | Semi | -0.43 |
| 17pt | Regular | -0.43 |
| 13pt | Regular | +0.03 |
| 12pt | Regular | +0.12 |

Line-height reference:
- Text (≤19pt): 120–130%
- Display (≥20pt): 110–120%

## Implementation priority

### v2.1 (next release)
1. xHeightRatio in FontProfile + correction formula
2. Cyrillic script detection + multipliers
3. Enhanced dark mode (size-dependent)
4. Condensed/Expanded detection

### v2.2
5. Line width (measure) factor
6. Aperture + stroke contrast in profiles
7. Sub-12px nonlinear curve
8. SF Pro calibration tests
