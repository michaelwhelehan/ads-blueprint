# 02 — Brand Configuration

The brand config is the single source of truth for all generated content — image prompts, ad copy, text compositing, and upload metadata.

## File Location

`project-config/brand.json`

## Zod Schema

```typescript
// src/shared/brand/brand-config.ts
import { z } from "zod";

export const BrandColorsSchema = z.object({
  primary: z.string(),     // Hex colour, e.g. "#1A1A2E"
  secondary: z.string(),
  accent: z.string(),
  background: z.string(),
  text: z.string(),
});

export const BrandConfigSchema = z.object({
  name: z.string(),
  tagline: z.string(),
  colors: BrandColorsSchema,
  typography: z.object({
    headingFont: z.string(),   // Descriptive name, e.g. "Zilla Slab"
    bodyFont: z.string(),      // Descriptive name, e.g. "Inter"
  }),
  sellingPoints: z.array(z.string()),
  targetAudience: z.array(
    z.object({
      persona: z.string(),
      description: z.string(),
      painPoints: z.array(z.string()),
      platforms: z.array(z.string()),
    })
  ),
  voice: z.object({
    tone: z.array(z.string()),     // e.g. ["playful", "confident"]
    style: z.string(),             // Free-form voice guide
    avoid: z.array(z.string()),    // Things to never say/do
  }),
  logoPath: z.string().optional(),
});

export type BrandConfig = z.infer<typeof BrandConfigSchema>;
export type BrandColors = z.infer<typeof BrandColorsSchema>;
```

## Loading Pattern

```typescript
import { readFileSync } from "fs";
import { BrandConfigSchema } from "./brand-config.js";
import { paths } from "../utils/paths.js";

function loadBrand(): BrandConfig {
  const raw = readFileSync(paths.brandConfig, "utf-8");
  return BrandConfigSchema.parse(JSON.parse(raw));
}
```

The Zod `.parse()` call validates the JSON and throws descriptive errors if any field is missing or has the wrong type.

## Field Reference

| Field | Type | Used In |
|---|---|---|
| `name` | string | Image prompts, ad copy prompts |
| `tagline` | string | Ad copy generation (product description) |
| `colors.primary` | hex string | Image prompt colour palette |
| `colors.secondary` | hex string | Image prompt colour palette |
| `colors.accent` | hex string | Image prompt colour palette |
| `colors.background` | hex string | Image prompt colour palette |
| `colors.text` | hex string | Image prompt colour palette |
| `typography.headingFont` | string | Descriptive — actual font files in `assets/fonts/` |
| `typography.bodyFont` | string | Descriptive — actual font files in `assets/fonts/` |
| `sellingPoints` | string[] | Ad copy generation (value propositions) |
| `targetAudience[].persona` | string | Brand context (separate from ad personas) |
| `targetAudience[].painPoints` | string[] | Brand context |
| `voice.tone` | string[] | Ad copy generation (brand voice) |
| `voice.style` | string | Ad copy generation (writing style guide) |
| `voice.avoid` | string[] | Ad copy generation (negative constraints) |
| `logoPath` | string? | Optional override for logo file path |

## Example brand.json

```json
{
  "name": "{Your Brand Name}",
  "tagline": "{One-sentence value proposition}",
  "colors": {
    "primary": "#1A1A2E",
    "secondary": "#D4AF37",
    "accent": "#8B0000",
    "background": "#0D0D0D",
    "text": "#F5F5F5"
  },
  "typography": {
    "headingFont": "Zilla Slab",
    "bodyFont": "Inter"
  },
  "sellingPoints": [
    "{Key benefit 1}",
    "{Key benefit 2}",
    "{Key benefit 3}",
    "{Key benefit 4}"
  ],
  "targetAudience": [
    {
      "persona": "{Audience segment name}",
      "description": "{Who they are and what they want}",
      "painPoints": [
        "{Pain point 1}",
        "{Pain point 2}"
      ],
      "platforms": ["Instagram", "Facebook"]
    }
  ],
  "voice": {
    "tone": ["playful", "confident", "inviting"],
    "style": "{Description of how the brand speaks}",
    "avoid": [
      "{Thing to never say 1}",
      "{Thing to never say 2}",
      "Mentioning AI or that content is AI-generated"
    ]
  }
}
```

## How Brand Config Flows Into the Pipeline

1. **Image prompts** (spec 06): Brand name, colour palette injected into every Gemini prompt
2. **Ad copy generation** (spec 05): Tagline, selling points, voice tone/style/avoid feed the copy prompt
3. **Text compositing** (spec 08): Font files from `assets/fonts/` correspond to typography names
4. **Upload** (spec 10): Brand website URL from campaign config used in ad links
