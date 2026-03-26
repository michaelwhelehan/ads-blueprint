# 07 — Image Generation

Generates ad images via Gemini's image generation model, using multi-part prompts with reference images. After generation, images are composited with text and branding (see spec 08).

## npm Script

```bash
npm run ads:generate
# Limit to N images (for testing):
npm run ads:generate -- --limit=3
```

## Source File

`src/ads/generation/generate-images.ts`

## Gemini API Setup

```typescript
import { GoogleGenAI, type GenerateContentConfig, type Part } from "@google/genai";

const ai = new GoogleGenAI({ apiKey: env.geminiApiKey() });
```

### Model

`gemini-3.1-flash-image-preview` — supports image generation with reference images.

### Generation Config

```typescript
const config: GenerateContentConfig = {
  responseModalities: ["TEXT", "IMAGE"],
  imageConfig: {
    aspectRatio: entry.format.placement === "story" ? "9:16" : "1:1",
  },
};
```

## Algorithm

```
main():
  1. Load matrix from current round (or legacy path)
  2. Refresh all image prompts from current template (allows template changes between runs)
  3. Save refreshed matrix
  4. Filter entries: status === "pending" AND NOT carriedForward
  5. Apply --limit flag if provided
  6. Load brand assets (see below)
  7. For each pending entry (rate-limited at 2000ms):
     a. Generate image via Gemini (multi-part prompt)
     b. If image returned:
        - Composite text and branding onto image (spec 08)
        - Save as "{entry.id}.png" in images directory
        - Set entry.imagePath = filename
        - Set entry.status = "generated"
     c. If failed: log error, continue to next
     d. Save matrix after each image (progressive save)
```

## Multi-Part Input Construction

Gemini receives an array of `Part` objects combining text labels and reference images:

```
buildParts(entry, assets):
  parts = []

  // 1. Physical items (product photos)
  if assets.physicalItems.length > 0:
    parts.push({ text: "[PHYSICAL ITEMS - {description of what these are and how to use them}]" })
    for each item in assets.physicalItems:
      parts.push(imagePartFromFile(item))

  // 2. Theme-specific reference images (e.g. costume/style reference)
  themeAssets = assets.themeAssets.get(entry.theme.id)
  if themeAssets and themeAssets.outfits.length > 0:
    outfit = themeAssets.outfits[outfitIndex % outfits.length]
    parts.push({ text: "[OUTFIT REFERENCE - {description of what this reference shows}]" })
    parts.push(imagePartFromFile(outfit))

  // 3. The text prompt
  parts.push({ text: entry.imagePrompt })

  return parts
```

### Image Part Helper

```typescript
function imagePartFromFile(filePath: string): Part {
  const data = readFileSync(filePath).toString("base64");
  const mimeType = filePath.endsWith(".png") ? "image/png" : "image/jpeg";
  return { inlineData: { mimeType, data } };
}
```

## API Call

```typescript
const response = await ai.models.generateContent({
  model: "gemini-3.1-flash-image-preview",
  contents: [{ role: "user", parts }],
  config,
});
```

## Response Extraction

```
extractImageBuffer(response):
  candidates = response.candidates
  if no candidates → return null
  for each part in candidates[0].content.parts:
    if part.inlineData.mimeType starts with "image/":
      return Buffer.from(part.inlineData.data, "base64")
  return null
```

## Asset Loading

### Algorithm

```
loadBrandAssets():
  // Per-theme assets
  scan assets/brand-photos/ for directories:
    for each theme directory:
      outfits = files matching /^outfit.*\.(png|jpg|jpeg)$/i
      pdfCover = file matching /^pdf-cover\.(png|jpg|jpeg)$/i (or null)
      store as Map<themeId, { outfits, pdfCover }>

  // Logo
  scan assets/logos/ for file matching /^logo\.(png|jpg|jpeg|svg)$/i

  // Physical items (product photos)
  scan assets/physical-items/ for *.png, *.jpg, *.jpeg
  exclude files starting with "covert-" (or similar pattern for items you want to hide)

  return { themeAssets, logo, physicalItems }
```

### Asset Types

```typescript
interface ThemeAssets {
  outfits: string[];        // Full paths to outfit reference images
  pdfCover: string | null;  // Optional product cover image
}

interface BrandAssets {
  themeAssets: Map<string, ThemeAssets>;  // keyed by theme.id
  logo: string | null;
  physicalItems: string[];
}
```

## Rate Limiting

2000ms between API calls (0.5 req/sec) via `createRateLimiter(2000)`.

Image generation is slower and more resource-intensive than text generation, so the rate limit is higher.

## Progressive Save

The matrix is saved to disk after **every** image generation. This prevents data loss if the process crashes mid-batch, and allows you to review partial results while generation continues.

## Output

- **Images**: `output/ads/rounds/round-{N}/images/{entry.id}.png` (or `output/ads/images/` legacy)
- **Matrix**: Updated in place with `imagePath` and `status: "generated"` for each successful image
