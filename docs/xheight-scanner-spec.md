# ТЗ: X-Height Scanner — Figma Plugin

## 1. Purpose

A standalone utility Figma plugin that scans fonts used in the current document (or all available fonts), measures their x-height ratio using Figma's rendering engine, and exports the data as a JSON table compatible with the FineTune plugin.

**Why a separate plugin:**
- FineTune runs on every text selection — measurement overhead must be zero
- Scanner runs once per project/file to generate a reusable font metrics database
- Different permission model: scanner needs `documentAccess: "dynamic-page"`, creates temporary nodes
- Output is a static JSON file that FineTune imports or fetches from a remote URL

---

## 2. Measurement technique

### 2.1 X-height ratio

For each font (family + style), the plugin:

1. Creates a temporary `TextNode` off-screen (x: -9999, y: -9999)
2. Sets font to the target family/style, fontSize = 100 (large for precision)
3. Sets `node.characters = "x"` (lowercase Latin x)
4. Reads `node.absoluteRenderBounds.height` → this is the x-height in px
5. Computes `xHeightRatio = renderBounds.height / fontSize`
6. Deletes the temporary node

**Why fontSize = 100:** Larger font size reduces rounding error in `absoluteRenderBounds`. At 100px, a 0.1px measurement error = 0.001 ratio error (negligible).

**Why "x" character:** The lowercase "x" has no ascenders, descenders, or overshoot in virtually all fonts — it sits exactly on the baseline and reaches exactly the x-height line.

### 2.2 Cap-height ratio (bonus metric)

Same technique but with `node.characters = "H"`:
```
capHeightRatio = renderBounds.height / fontSize
```

Cap-height is useful for optical vertical centering of text in buttons/badges.

### 2.3 Additional metrics (optional, v2)

| Metric | Character | Formula |
|--------|-----------|---------|
| Ascender ratio | "d" | `renderBounds.height / fontSize` |
| Descender depth | "p" | `(renderBounds.height - capHeight) / fontSize` |
| Average width | "aenosrt" | `renderBounds.width / (7 × fontSize)` |

---

## 3. Font enumeration

### 3.1 Strategy A: Scan fonts used in document

```typescript
// Collect all unique font families + styles from text nodes on all pages
await figma.loadAllPagesAsync();
const allTextNodes = figma.root.findAll(n => n.type === 'TEXT') as TextNode[];

const uniqueFonts = new Map<string, Set<string>>(); // family → Set<style>

for (const node of allTextNodes) {
  const segments = node.getStyledTextSegments(['fontName']);
  for (const seg of segments) {
    const { family, style } = seg.fontName;
    if (!uniqueFonts.has(family)) uniqueFonts.set(family, new Set());
    uniqueFonts.get(family)!.add(style);
  }
}
```

**Pros:** Only measures fonts actually in use; smaller output.
**Cons:** Misses fonts the designer might add later.

### 3.2 Strategy B: Manual font list input

User pastes a list of font names into the plugin UI. Plugin scans only those.

**Pros:** Full control.
**Cons:** Manual effort.

### 3.3 Recommendation

**Start with Strategy A** (document scan). Add a text input in UI for additional fonts the user wants to include. Combine both lists, deduplicate.

---

## 4. Processing flow

```
┌─────────────────────────────────────────┐
│  1. User clicks "Scan"                  │
│  2. Enumerate fonts (Strategy A + input)│
│  3. Show count: "Found N fonts"         │
│  4. User confirms → "Start measurement" │
│  5. For each font:                      │
│     a. loadFontAsync({ family, style }) │
│     b. Create temp TextNode             │
│     c. Set "x" → measure xHeightRatio   │
│     d. Set "H" → measure capHeightRatio │
│     e. Delete temp node                 │
│     f. Update progress bar: i/N         │
│  6. Build JSON output                   │
│  7. Display results table in UI         │
│  8. User copies JSON or exports         │
└─────────────────────────────────────────┘
```

### 4.1 Error handling

| Error | Handling |
|-------|----------|
| `loadFontAsync` fails (font not installed) | Skip font, mark as `"error": "not available"` in output |
| `absoluteRenderBounds` returns null | Retry once; if still null, mark as `"error": "no render bounds"` |
| Font has no lowercase "x" (symbol fonts) | Detect via `renderBounds.height === 0` or null, skip |
| Plugin timeout (many fonts) | Process in batches of 10, yield between batches via `setTimeout` |

### 4.2 Performance

- `loadFontAsync`: ~20–50ms per font (network/system dependent)
- Create + measure + delete node: ~10–30ms
- **Total per font: ~50–100ms**
- 100 fonts ≈ 5–10 seconds
- 500 fonts ≈ 25–50 seconds

**Batch processing:** Process 10 fonts, then `await new Promise(r => setTimeout(r, 0))` to yield to Figma's event loop and prevent UI freeze.

---

## 5. Output format

### 5.1 JSON structure (FineTune-compatible)

```json
{
  "version": "1.0",
  "generatedAt": "2025-01-15T12:00:00Z",
  "figmaFileId": "abc123...",
  "fontCount": 42,
  "fonts": {
    "Inter": {
      "xHeightRatio": 0.536,
      "capHeightRatio": 0.727,
      "styles": {
        "Regular": { "xHeightRatio": 0.536, "capHeightRatio": 0.727 },
        "Bold": { "xHeightRatio": 0.536, "capHeightRatio": 0.727 },
        "Italic": { "xHeightRatio": 0.534, "capHeightRatio": 0.727 }
      },
      "category": "sans-serif"
    },
    "Roboto": {
      "xHeightRatio": 0.528,
      "capHeightRatio": 0.711,
      "styles": { ... },
      "category": "sans-serif"
    }
  }
}
```

**Notes:**
- Top-level `xHeightRatio` per family = value from Regular style (or first available)
- Per-style values stored in `styles` for precision (italic may differ slightly)
- `category` auto-detected or user-specified (see Section 6)

### 5.2 FineTune integration

FineTune will read this JSON and use the `xHeightRatio` value in its GRT formula:

```typescript
// In FineTune calculator.ts
const xHeightFactor = (xHeightRatio - 0.52) * 0.15; // scale factor TBD
lineHeight = fontSize * baseRatio * (1 + xHeightFactor) * contextMul * ...;
```

The scanner JSON can be:
- Pasted into FineTune's `font-database.ts` as static data
- Hosted on GitHub as a remote JSON database
- Stored in `figma.clientStorage` for local persistence

---

## 6. Category auto-detection

The plugin should attempt to guess the font category for fonts not in FineTune's database:

```typescript
function guessCategory(family: string, style: string): FontCategory {
  const name = (family + ' ' + style).toLowerCase();

  if (/mono|code|consolas|courier|fira\s*code|jetbrains|source\s*code/i.test(name))
    return 'mono';
  if (/display|poster|headline|decorative|script|handwrit/i.test(name))
    return 'display';
  if (/serif|garamond|times|georgia|palatino|baskervill|merriweather|playfair|lora|crimson|libre\s*baskervill/i.test(name)
      && !/sans/i.test(name))
    return 'serif';
  return 'sans-serif';
}
```

User can override in the results table before export.

---

## 7. UI specification

### 7.1 Layout

```
┌──────────────────────────────────────────┐
│  X-Height Scanner                   v1.0 │
├──────────────────────────────────────────┤
│                                          │
│  Scan mode:                              │
│  ○ Fonts used in this file (recommended) │
│  ○ Custom font list                      │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ Additional fonts (one per line):   │  │
│  │                                    │  │
│  │                                    │  │
│  └────────────────────────────────────┘  │
│                                          │
│  [ Scan fonts ]                          │
│                                          │
├──────────────────────────────────────────┤
│  Found 42 unique fonts                   │
│  ┌──────────────────────────────────┐    │
│  │ Font          │x-height│ cap-h  │cat │
│  │───────────────┼────────┼────────┼────│
│  │ Inter Regular │ 0.536  │ 0.727  │sans│
│  │ Inter Bold    │ 0.536  │ 0.727  │sans│
│  │ Roboto Regular│ 0.528  │ 0.711  │sans│
│  │ ...           │  ...   │  ...   │ ...│
│  └──────────────────────────────────┘    │
│                                          │
│  Progress: ████████████░░░░ 28/42        │
│                                          │
├──────────────────────────────────────────┤
│  [ Copy JSON ]  [ Copy as TypeScript ]   │
└──────────────────────────────────────────┘
```

### 7.2 Interactions

| Element | Behavior |
|---------|----------|
| "Scan fonts" button | Enumerates fonts, shows count, starts measurement |
| Progress bar | Updates per font, shows current font name |
| Results table | Scrollable, sortable by column click |
| Category column | Editable dropdown (sans-serif / serif / mono / display) |
| "Copy JSON" | Copies full JSON to clipboard |
| "Copy as TypeScript" | Copies as `Record<string, { xHeightRatio: number; capHeightRatio: number; category: FontCategory }>` |

### 7.3 Plugin window size

- Width: 400px
- Height: 500px (resizable)

---

## 8. Technical architecture

### 8.1 manifest.json

```json
{
  "name": "X-Height Scanner",
  "id": "xheight-scanner-unique-id",
  "api": "1.0.0",
  "main": "dist/main.js",
  "ui": "dist/ui.html",
  "documentAccess": "dynamic-page",
  "editorType": ["figma"],
  "networkAccess": { "allowedDomains": ["none"] }
}
```

`documentAccess: "dynamic-page"` — needed to scan all pages for text nodes.

### 8.2 File structure

```
xheight-scanner/
├── manifest.json
├── package.json
├── build.mjs              # esbuild bundler (same as FineTune)
├── src/
│   ├── main.ts            # Plugin sandbox code
│   │   ├── enumerateFonts()
│   │   ├── measureFont()
│   │   ├── processBatch()
│   │   └── message handlers
│   ├── ui/
│   │   ├── index.tsx       # Preact UI
│   │   └── styles.css
│   └── types.ts
└── dist/                   # Build output
```

### 8.3 Main sandbox code (main.ts) — key functions

```typescript
interface FontMeasurement {
  family: string;
  style: string;
  xHeightRatio: number;
  capHeightRatio: number;
  category: FontCategory;
  error?: string;
}

async function measureFont(
  family: string,
  style: string
): Promise<FontMeasurement> {
  const fontName = { family, style };

  try {
    await figma.loadFontAsync(fontName);
  } catch {
    return { family, style, xHeightRatio: 0, capHeightRatio: 0,
             category: 'sans-serif', error: 'Font not available' };
  }

  const node = figma.createText();
  node.x = -9999;
  node.y = -9999;
  node.fontName = fontName;
  node.fontSize = 100;

  // Measure x-height
  node.characters = 'x';
  await new Promise(r => setTimeout(r, 10)); // ensure render
  const xBounds = node.absoluteRenderBounds;
  const xHeight = xBounds ? xBounds.height / 100 : 0;

  // Measure cap-height
  node.characters = 'H';
  await new Promise(r => setTimeout(r, 10));
  const capBounds = node.absoluteRenderBounds;
  const capHeight = capBounds ? capBounds.height / 100 : 0;

  node.remove();

  return {
    family,
    style,
    xHeightRatio: Math.round(xHeight * 1000) / 1000,
    capHeightRatio: Math.round(capHeight * 1000) / 1000,
    category: guessCategory(family, style),
  };
}

async function processBatch(
  fonts: Array<{ family: string; style: string }>,
  batchSize = 10
): Promise<FontMeasurement[]> {
  const results: FontMeasurement[] = [];

  for (let i = 0; i < fonts.length; i += batchSize) {
    const batch = fonts.slice(i, i + batchSize);

    for (const font of batch) {
      const result = await measureFont(font.family, font.style);
      results.push(result);

      // Report progress to UI
      figma.ui.postMessage({
        type: 'progress',
        current: results.length,
        total: fonts.length,
        currentFont: `${font.family} ${font.style}`,
      });
    }

    // Yield to event loop between batches
    await new Promise(r => setTimeout(r, 0));
  }

  return results;
}
```

---

## 9. Export formats

### 9.1 JSON (primary)

See Section 5.1 above.

### 9.2 TypeScript (for FineTune font-database.ts)

```typescript
// Generated by X-Height Scanner
export const XHEIGHT_DATA: Record<string, {
  xHeightRatio: number;
  capHeightRatio: number;
  category: string;
}> = {
  "Inter": { xHeightRatio: 0.536, capHeightRatio: 0.727, category: "sans-serif" },
  "Roboto": { xHeightRatio: 0.528, capHeightRatio: 0.711, category: "sans-serif" },
  "Open Sans": { xHeightRatio: 0.536, capHeightRatio: 0.714, category: "sans-serif" },
  // ...
};
```

### 9.3 CSV (for spreadsheet analysis)

```
Font Family,Style,X-Height Ratio,Cap-Height Ratio,Category
Inter,Regular,0.536,0.727,sans-serif
Inter,Bold,0.536,0.727,sans-serif
Roboto,Regular,0.528,0.711,sans-serif
```

---

## 10. Edge cases

| Case | Handling |
|------|----------|
| Variable fonts (e.g., Inter Variable) | Measure at Regular (400) weight. Variable fonts may appear as a single style "Regular" or with axis names |
| Font not installed on user's machine | `loadFontAsync` will throw → mark as error, skip |
| Symbol/icon fonts (no lowercase "x") | `absoluteRenderBounds` returns null or height ≈ 0 → mark as `"icon-font"`, skip |
| CJK fonts (no Latin "x") | Detect via family name heuristics or zero xHeight → skip with note |
| Very large documents (1000+ text nodes) | Enumeration is fast (findAll is native); measurement is the bottleneck |
| Duplicate font family with different foundries | Key by exact `family` string — Figma treats them as separate |
| Already measured fonts (re-scan) | Check if data exists in `figma.clientStorage`, offer "use cached" option |

---

## 11. Acceptance criteria

1. Plugin scans all text nodes across all pages and extracts unique font families + styles
2. For each font, measures x-height ratio with ≤0.005 error vs. reference values (see docs/algorithm-research.md)
3. Progress bar updates in real-time during measurement
4. Results displayed in a sortable table
5. User can edit auto-detected category before export
6. "Copy JSON" produces valid JSON parseable by `JSON.parse()`
7. "Copy as TypeScript" produces valid TS code
8. Fonts that fail to load are listed with error status, not silently skipped
9. Plugin handles documents with 0 text nodes gracefully ("No text nodes found")
10. Measurement of 50 fonts completes within 10 seconds

---

## 12. Validation

Cross-reference scanner output against known values from OS/2 font tables:

| Font | Expected xHeightRatio | Tolerance |
|------|-----------------------|-----------|
| Inter | 0.536 | ±0.01 |
| Roboto | 0.528 | ±0.01 |
| Montserrat | 0.486 | ±0.01 |
| Playfair Display | 0.460 | ±0.01 |
| Times New Roman | 0.448 | ±0.01 |
| Verdana | 0.545 | ±0.01 |
| Helvetica | 0.523 | ±0.01 |

If measured values deviate by more than ±0.015 from OS/2 reference, investigate whether `absoluteRenderBounds` includes optical overshoot or ink traps that affect the measurement.

---

## 13. Future enhancements (v2)

- **Auto-upload to GitHub:** Push JSON to a configured GitHub repo (requires `networkAccess` update)
- **Batch comparison:** Compare measurements across Figma files to detect inconsistencies
- **Width class detection:** Measure average character width to classify condensed/normal/expanded
- **Stroke contrast estimation:** Compare thin vs. thick strokes via render bounds of "o" and "l"
- **Aperture openness:** Measure "e" counter size relative to overall height
- **Integration with FineTune:** Direct push of measurements to FineTune's `clientStorage`
