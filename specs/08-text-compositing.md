# 08 — Text Compositing

After image generation, text (hook, bullets, logo, brand URL) is composited onto each image using Sharp and opentype.js. This creates the final ad-ready image.

## Dependencies

- **sharp** — Image resizing, compositing, gradient overlays
- **opentype.js** — Font loading, text measurement, SVG path generation

## Source

`src/ads/generation/generate-images.ts` — functions `compositeTextAndBranding()` and `renderText()`

## Font Setup

### Font Files

Place font files in `assets/fonts/`:

```
assets/fonts/
  {HeadingFont}-Bold.ttf      # For hook text (top of image)
  {BodyFont}-Bold.ttf          # For bullet points
  {BodyFont}-Regular.ttf       # Optional, for lighter body text
```

### Font Loading

```typescript
const FONTS = {
  headingBold: join(paths.root, "assets", "fonts", "{HeadingFont}-Bold.ttf"),
  bodyBold:    join(paths.root, "assets", "fonts", "{BodyFont}-Bold.ttf"),
  bodyRegular: join(paths.root, "assets", "fonts", "{BodyFont}-Regular.ttf"),
};

// Cache loaded fonts to avoid re-parsing on every call
const fontCache = new Map<string, opentype.Font>();

function loadFont(fontPath: string): opentype.Font {
  let font = fontCache.get(fontPath);
  if (!font) {
    font = opentype.loadSync(fontPath);
    fontCache.set(fontPath, font);
  }
  return font;
}
```

## Text Rendering Pipeline

### renderText()

Renders text to a PNG buffer with drop shadow and outline for readability.

```
renderText(text, width, fontSize, options = { font: "body", align: "center" }):
  1. Select font file based on options.font ("heading" or "body")
  2. Load font with opentype.js
  3. Scale = 3 (3x supersample for crisp text)
  4. Wrap text into lines using word wrapping:
     wrapText(otFont, text, scaledFontSize, scaledWidth):
       for each word:
         measure test line width with otFont.getAdvanceWidth()
         if exceeds maxWidth → push current line, start new
       return lines[]
  5. Calculate SVG dimensions:
     svgHeight = lines.length × (scaledFontSize × 1.2) + scaledFontSize × 0.3
  6. Generate SVG paths for each line:
     for each line:
       measure lineWidth for alignment
       x = (scaledWidth - lineWidth) / 2  (if center-aligned)
       y = scaledFontSize + i × lineHeight
       path = otFont.getPath(line, x, y, scaledFontSize)
       append path.toSVG()
  7. Build SVG with three layers:
     strokeWidth = scaledFontSize × 0.06
     shadowOffset = scaledFontSize × 0.03

     <svg>
       <!-- Shadow layer: black, 60% opacity, offset down-right -->
       <g fill="black" fill-opacity="0.6" transform="translate({offset},{offset})">{paths}</g>
       <!-- Outline layer: black fill + stroke for thickness -->
       <g fill="black" stroke="black" stroke-width="{strokeWidth}" stroke-linejoin="round">{paths}</g>
       <!-- White fill on top -->
       <g fill="white">{paths}</g>
     </svg>
  8. Rasterise with sharp: resize from 3x to target width
     sharp(Buffer.from(svg)).resize({ width }).png().toBuffer()
```

### Why 3x Supersample?

Rendering at 3× the target size and downscaling produces anti-aliased, crisp text without relying on the rasteriser's built-in anti-aliasing. This is critical for small text on ad images.

## Compositing Pipeline

### compositeTextAndBranding()

```
compositeTextAndBranding(imageBuffer, entry, logoPath):
  targetWidth = entry.format.width   // 1080
  targetHeight = entry.format.height // 1080 or 1920
  isSquare = targetWidth === targetHeight

  1. Resize image to target dimensions (cover mode)

  2. Top gradient overlay (square format only):
     SVG linearGradient from top to 25%:
       top: black at 80% opacity
       25% down: transparent
     Composite over image

  3. Bottom gradient overlay (both formats):
     SVG linearGradient from bottom to 50%:
       bottom: black at 80% opacity
       50% up: transparent
     Composite over image

  4. Calculate text sizes:
     hookSize    = imgWidth × (isSquare ? 0.055 : 0.08)
     messageSize = imgWidth × (isSquare ? 0.03  : 0.04)
     textPadding = imgWidth × 0.1   // 10% margin on each side

  5. Render hook text (top):
     hookImage = renderText(concept.hook, imgWidth - textPadding×2, hookSize, { font: "heading", align: "center" })
     hookTop = imgHeight × (isSquare ? 0.06 : 0.18)
     Place at (textPadding, hookTop)

  6. Render logo + brand URL (bottom):
     logoSize = imgWidth × 0.07
     brandingTop = imgHeight - logoSize - 40 - (isSquare ? 0 : imgHeight × 0.05)
     Resize logo to logoSize × logoSize (contain mode, transparent background)
     Place logo at (40, brandingTop)
     Render brand URL text at (40 + logoSize + 12, brandingTop centered vertically)

  7. Render bullet points (above logo, bottom-up):
     bulletGap = imgHeight × 0.008
     bulletPaddingAboveLogo = imgHeight × 0.03
     startY = brandingTop - bulletPaddingAboveLogo
     For each bullet (reversed order, placed bottom-up):
       bulletImage = renderText("• {bullet}", imgWidth - textPadding×2, messageSize, { font: "body", align: "center" })
       startY -= bulletImage.height
       Place at (textPadding, startY)
       startY -= bulletGap

  8. Composite all layers onto image
     Return final PNG buffer
```

## Visual Layout

### Feed Square (1080×1080)

```
┌──────────────────────────┐
│   ▓▓▓ top gradient ▓▓▓  │
│      HOOK TEXT HERE      │  ← 6% from top, heading font
│                          │
│                          │
│     (generated image     │
│      fills entire        │
│      background)         │
│                          │
│   • Bullet point one     │  ← above logo, body font
│   • Bullet point two     │
│   • Bullet point three   │
│   🔲 brand-url.com       │  ← logo + URL at bottom
│   ▓▓▓ bottom gradient ▓▓ │
└──────────────────────────┘
```

### Story Portrait (1080×1920)

Same layout but proportionally adjusted:
- Hook at 18% from top (more space)
- Larger font sizes (8% and 4% of width)
- More breathing room at bottom for logo

## Key Design Decisions

1. **White text with black outline + shadow** — readable on any background
2. **Gradient overlays** — ensure text contrast without masking the image
3. **SVG path rendering** — font-independent, pixel-perfect at any size
4. **Progressive compositing** — each layer added as a Sharp composite operation
5. **Format-aware sizing** — all dimensions are proportional to image width/height
