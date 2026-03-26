# 14 — Asset Management

How brand assets (logos, fonts, product photos, theme reference images) are organised and flow into the image generation pipeline.

## Directory Structure

```
assets/
├── logos/
│   └── logo.{png|jpg|jpeg|svg}       # Primary brand logo
│
├── fonts/
│   ├── {HeadingFont}-Bold.ttf         # For hook text (top of image)
│   ├── {BodyFont}-Bold.ttf            # For bullet points
│   └── {BodyFont}-Regular.ttf         # Optional lighter weight
│
├── brand-photos/
│   ├── {theme-id}/                    # One directory per theme (kebab-case = theme.id)
│   │   ├── outfit-1.{png|jpg}         # Style/costume reference images
│   │   ├── outfit-2.{png|jpg}         # Multiple outfits cycled through
│   │   ├── outfit-3.{png|jpg}
│   │   └── pdf-cover.{png|jpg}        # Optional product cover image
│   ├── {another-theme-id}/
│   │   └── ...
│   └── downloaded/                    # Images downloaded from website scanner
│       └── ...
│
└── physical-items/
    ├── {item-1}.{png|jpg}             # Product photos
    ├── {item-2}.{png|jpg}             # (your actual product)
    └── {item-3}.{png|jpg}
```

## How Assets Are Loaded

The `loadBrandAssets()` function in `generate-images.ts` scans these directories:

```
loadBrandAssets() → BrandAssets:

  // 1. Theme assets
  scan assets/brand-photos/ for subdirectories:
    for each directory:
      outfits = files matching /^outfit.*\.(png|jpg|jpeg)$/i
      pdfCover = file matching /^pdf-cover\.(png|jpg|jpeg)$/i or null
      store in Map<themeId, { outfits, pdfCover }>

  // 2. Logo
  scan assets/logos/ for file matching /^logo\.(png|jpg|jpeg|svg)$/i
  (first match wins)

  // 3. Physical items
  scan assets/physical-items/ for *.png, *.jpg, *.jpeg
  exclude any files you want to hide (e.g. "covert-*" prefix pattern)
```

## How Assets Flow Into Image Generation

### Physical Items → Reference Images

Physical item photos are sent to Gemini as reference images with a text label explaining what they are and how they should appear in the scene:

```
[PHYSICAL ITEMS - {description of product items and instructions}]
{image 1}
{image 2}
```

All physical items are sent with every image generation call.

### Theme Outfits → Style Reference

For each entry, the theme's outfit images provide style/costume reference. Outfits are cycled through:

```
outfit = themeAssets.outfits[outfitIndex % outfits.length]
```

Sent with a label:
```
[OUTFIT REFERENCE - {description of what this reference shows and how to use it}]
{outfit image}
```

### Logo → Text Compositing

The logo is not sent to Gemini — it's composited onto the generated image after generation:

- Resized to `imgWidth × 0.07` (7% of image width)
- Placed at bottom-left with padding
- Brand URL text rendered next to it

### Fonts → Text Rendering

Font TTF files are loaded by `opentype.js` for text path generation:

- **Heading font (Bold)**: Used for hook text at the top of the image
- **Body font (Bold)**: Used for bullet points above the logo

## Asset Naming Conventions

### Logos
- `logo.png` or `logo.jpg` or `logo.jpeg` or `logo.svg`
- Only one logo file needed — first match by the regex is used
- Transparent PNG recommended for clean compositing

### Fonts
- Must be `.ttf` format (TrueType) — required by opentype.js
- File names should match what you reference in code (e.g. `ZillaSlab-Bold.ttf`, `Inter-Bold.ttf`)

### Brand Photos / Theme Directories
- Directory name **must match** `theme.id` from `ad-inputs.json` (kebab-case)
- Example: theme with `id: "jazz-age-jubilee"` → directory `assets/brand-photos/jazz-age-jubilee/`
- Outfit files must start with `outfit` (case-insensitive)
- Having 2-4 outfit variations per theme adds diversity to generated images

### Physical Items
- Any descriptive filename works
- These are your actual product photos — whatever you sell
- Exclude items from generation by prefixing with a pattern you filter out

## Size Recommendations

| Asset Type | Minimum Size | Maximum File Size | Notes |
|---|---|---|---|
| Logo | 200×200px | 2MB | Transparent PNG preferred |
| Outfit reference | 512×512px | 5MB | Photorealistic, clear costume/style |
| Physical items | 512×512px | 5MB | Clean product photos |
| Fonts | N/A | 1MB | TTF format only |

Larger reference images give Gemini better detail to work from, but very large files (>10MB) slow down API calls since they're base64-encoded.

## Downloaded Images (from Website Scanner)

The bootstrap skill downloads brand images to `assets/brand-photos/downloaded/`. After scanning:

1. Review downloaded images
2. Move relevant ones to appropriate theme directories or `physical-items/`
3. Rename to match expected patterns (`logo.png`, `outfit-1.png`, etc.)
4. Delete irrelevant downloads

## Missing Assets

The pipeline handles missing assets gracefully:

- **No logo**: Logo and brand URL are skipped in compositing
- **No theme outfits**: No outfit reference sent to Gemini (images still generate)
- **No physical items**: No product reference sent (may affect product accuracy)
- **No fonts**: Text rendering will fail — fonts are required for compositing

## Adding Assets for a New Theme

When you add a new theme to `ad-inputs.json`:

1. Create `assets/brand-photos/{theme-id}/`
2. Add 2-4 outfit reference images
3. Optionally add a `pdf-cover.png` or product image
4. Run `npm run ads:generate` — the theme will automatically use the new assets
