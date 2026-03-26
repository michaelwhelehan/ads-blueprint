# 06 — Image Prompt System

The prompt system constructs detailed text prompts for Gemini image generation and ad copy generation. It supports an optional template file for iterative refinement.

## Source File

`src/ads/matrix/prompt-builder.ts`

## Prompt Template

### Interface

```typescript
interface PromptTemplate {
  photographyStyle: string[];   // Array of style directives
  requirements: string[];       // Array of hard constraints
}
```

### File Location

`project-config/prompt-template.json` (optional — defaults used if absent)

### Loading

```
loadTemplate():
  if prompt-template.json exists → parse and return
  else → return null (use defaults)
```

### Default Photography Style

When no template file exists, these defaults are used:

```typescript
const DEFAULT_PHOTOGRAPHY_STYLE = [
  "Shot on an iPhone — casual, candid, user-generated content (UGC) aesthetic",
  "Slightly imperfect framing, natural available lighting (not studio-lit), mild lens distortion",
  "Real-life setting that looks lived-in and unposed, not a styled photoshoot",
  "Natural skin tones, slight grain/noise, soft shadows — no dramatic studio lighting or airbrushing",
  "The image should feel like something a real person snapped at home, not a professional advertisement",
];
```

### Default Requirements

```typescript
const DEFAULT_REQUIREMENTS = [
  "Do NOT include ANY text, logos, brand names, URLs, or watermarks in the image — all text will be composited separately",
  "Only the provided printed game sheets as product materials — no illustrated props, no magnifying glasses, no wax seals, no knives, no pocket watches",
  "The entire image MUST be photorealistic — no illustrated, animated, or cartoon elements",
  "Use the full frame for the scene — do NOT darken or leave empty space at any part of the image",
  "Text will be overlaid with drop shadows, so the image can be vibrant and detailed edge-to-edge",
  "The design should feel native to {placement} placement",
];
```

Note: `{placement}` is replaced with the actual format placement ("feed" or "story") at build time.

### Customising for Your Brand

Replace the defaults with directives specific to your product. The photography style controls the entire visual identity. For example:

- **SaaS product**: "Clean, minimal workspace with laptop and product UI visible on screen"
- **Food brand**: "Overhead flat-lay on a rustic wooden table, natural daylight from the left"
- **Fashion**: "Street-style photography, shallow depth of field, urban background"

The requirements should include hard constraints about what must/must not appear in the image.

## buildImagePrompt()

Constructs the full prompt sent to Gemini for image generation.

### Algorithm

```
buildImagePrompt(entry: CreativeMatrixEntry, brand: BrandConfig) → string:
  1. Load template (or use defaults)
  2. Replace {placement} in requirements with entry.format.placement
  3. Assemble prompt lines:
     - "Create a {width}x{height} ad image for {brand.name}."
     -
     - "REFERENCE IMAGES PROVIDED:"
     - "- PHYSICAL ITEMS: {description of what product items should appear}"
     - "- OUTFIT REFERENCE: {description of costume/style reference}"
     -
     - "THEME: {theme.name} — {theme.description}"
     -
     - "SCENE DESCRIPTION:"
     - "{concept.sceneDescription}"
     -
     - "Target Audience: {persona.name} - {persona.description}"
     -
     - "Brand Color Palette:"
     - "- Primary color: {colors.primary}"
     - "- Secondary color: {colors.secondary}"
     - "- Accent color: {colors.accent}"
     - "- Background color: {colors.background}"
     -
     - "Photography Style (CRITICAL — this determines the entire look and feel):"
     - {each photographyStyle line prefixed with "- "}
     -
     - "Requirements:"
     - {each requirement line prefixed with "- "}
  4. Join all lines with "\n"
```

### Output Example

```
Create a 1080x1080 ad image for {Brand Name}.

REFERENCE IMAGES PROVIDED:
- PHYSICAL ITEMS: ...
- OUTFIT REFERENCE: ...

THEME: {Theme Name} — {Theme Description}

SCENE DESCRIPTION:
{Detailed scene description from concept}

Target Audience: {Persona Name} - {Persona Description}

Brand Color Palette:
- Primary color: #1A1A2E
- Secondary color: #D4AF37
- Accent color: #8B0000
- Background color: #0D0D0D

Photography Style (CRITICAL — this determines the entire look and feel):
- {style directive 1}
- {style directive 2}
...

Requirements:
- {requirement 1}
- {requirement 2}
...
```

## buildCopyGenerationPrompt()

Constructs the prompt for Gemini ad copy generation.

### Algorithm

```
buildCopyGenerationPrompt(entry: CreativeMatrixEntry, brand: BrandConfig) → string:
  Assemble lines:
  - "Write Meta ad copy for {brand.name}."
  -
  - "Product/Service: {brand.tagline}"
  - "Key Selling Points: {brand.sellingPoints.join(', ')}"
  -
  - "Theme: {theme.name} — {theme.description}"
  - "Hook Angle: {concept.hook}"
  - "Key Features: {concept.bullets.join(' · ')}"
  -
  - "Target Persona: {persona.name}"
  - "Their Pain Points: {persona.painPoints.join(', ')}"
  -
  - "Brand Voice: {voice.tone.join(', ')} | Style: {voice.style}"
  - "Avoid: {voice.avoid.join(', ')}"
  - "IMPORTANT: Never mention AI, artificial intelligence, or that content is AI-generated/AI-created/AI-powered."
  -
  - "Return JSON with exactly these fields:"
  - "{"
  - '  "headline": "max 40 chars, punchy, must reference {your product}",'
  - '  "primaryText": "max 125 chars, leads with hook, speaks to pain points, ends with value prop",'
  - '  "callToAction": "one of: Learn More, Sign Up, Shop Now, Download, Get Offer"'
  - "}"
  Join with "\n"
```

## Iterative Prompt Optimisation (Optional)

The template file can include an iteration history for systematic prompt refinement:

```json
{
  "version": 1,
  "bestScore": 95,
  "photographyStyle": ["..."],
  "requirements": ["..."],
  "history": [
    {
      "iteration": 1,
      "score": 70,
      "accepted": false,
      "photographyStyle": ["..."],
      "requirements": ["..."],
      "failedCriteria": ["face quality", "prop accuracy"],
      "reasoning": "Faces had artifacts, wrong props appeared"
    },
    {
      "iteration": 2,
      "score": 85,
      "accepted": true,
      "photographyStyle": ["..."],
      "requirements": ["..."],
      "failedCriteria": [],
      "reasoning": "Much improved, keeping this version"
    }
  ]
}
```

This allows you to track which prompt changes improved or degraded output quality, and systematically converge on an optimal prompt for your specific brand.
