# opikat-personal-experiments
personal projects

## TypeTune — Figma Plugin

Figma plugin for auto-tuning line-height and letter-spacing with code-ready export.

**TypeBalance picks pretty numbers. TypeTune picks pretty numbers that work in code.**

### What it does

- Calculates optimal `line-height` and `letter-spacing` based on font metrics, size, weight, uppercase, and background brightness
- Snaps line-height to 4px baseline grid for vertical rhythm
- Detects dark/light background automatically (solid, gradient, image fills)
- Exports in platform-native units: CSS (`em`), CSS Fluid (`clamp()`), iOS (`kern`/`lineHeightMultiple`), Android (`letterSpacing`/`lineSpacingMultiplier`)

### Line-height scale

| Context | Font size | Target line-height |
|---------|-----------|-------------------|
| Display (large) | 96-128px+ | ~110% |
| Display | 48-64px | ~120% |
| Display (small) | 32-48px | ~125-130% |
| Body | 14-31px | ~140% |
| Caption | ≤13px | ~155% |

### Supported fonts (Tier 1)

Inter, Roboto, Open Sans, Montserrat, Lato, Poppins, Noto Sans, Raleway, Ubuntu, Nunito, Playfair Display, PT Sans, Merriweather, Rubik, Work Sans, DM Sans, Manrope, Space Grotesk, IBM Plex Sans, Jost, SF Pro, PP Neue Montreal, Golos, Graphik, Gilroy, TT Norms Pro, Basis Grotesque, Suisse Int'l, Druk, Helios

Unsupported fonts use category-based fallback formulas (sans-serif / serif / mono / display).

### Build & run

```bash
cd plugin
npm install
npm run build
```

In Figma: **Plugins** → **Development** → **Import plugin from manifest** → select `plugin/manifest.json`.

### Settings

- **Auto-apply on selection change** — when enabled, optimized values are immediately applied to every text layer you select, no need to click "Apply". Disable if you want to preview values first.
- **Save to Figma Variables** — creates a "TypeTune" variable collection with line-height and letter-spacing values for each text style. Useful for design systems — developers can read these variables directly. Includes Light and Dark mode variants.

---

## Figma Console MCP

This project is configured with [figma-console-mcp](https://github.com/southleft/figma-console-mcp) for AI-assisted Figma design workflows.

### Setup

1. **Register a Figma app** at https://www.figma.com/developers/apps and note your `client_id`, `client_secret`, and `redirect_uri`.

2. **Complete the OAuth authorization flow** — direct the user to:
   ```
   https://www.figma.com/oauth?client_id=<CLIENT_ID>&redirect_uri=<REDIRECT_URI>&scope=files:read&state=<STATE>&response_type=code
   ```
   Figma will redirect back with a `code` parameter.

3. **Exchange the code for an access token:**
   ```bash
   export FIGMA_CLIENT_ID="your_client_id"
   export FIGMA_CLIENT_SECRET="your_client_secret"
   export FIGMA_REDIRECT_URI="your_redirect_uri"
   export FIGMA_AUTH_CODE="code_from_callback"

   ./scripts/get-figma-token.sh
   ```

4. **Set the token** for the MCP server:
   ```bash
   export FIGMA_ACCESS_TOKEN="<token from step 3>"
   ```

5. **Restart Claude Code** — the MCP server will start automatically via `.mcp.json`.
