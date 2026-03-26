# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **specification repository** for a Meta Ads automation pipeline. The `specs/` directory contains 14 detailed implementation specs. No application code exists yet — the specs define what to build.

The pipeline generates ad creatives via Google Gemini, composites text/branding with Sharp, and uploads to Meta Ads via Graph API v21.0. It supports iterative optimization through a multi-round system.

## Bootstrapping

Use the `/ads-bootstrap` skill to scaffold the project interactively. It walks through brand configuration, persona setup, theme creation, campaign config, and full code scaffolding from the specs.

## Tech Stack

- **Runtime**: Node.js (ESM, `"type": "module"`)
- **Language**: TypeScript (strict, ES2022 target, bundler module resolution)
- **AI**: Google Gemini API (`@google/genai`) — `gemini-3.1-flash-image-preview` for images, `gemini-3.1-flash-lite-preview` for ad copy
- **Ads Platform**: Meta Graph API v21.0
- **Image Processing**: Sharp (compositing, gradients) + opentype.js (font rendering to SVG paths)
- **Validation**: Zod
- **Review UI**: Express.js local server on port 3847
- **Runner**: tsx (dev execution without compilation)

## Build & Run Commands

```bash
# Type check
npx tsc --noEmit

# Pipeline steps (in order)
npm run ads:matrix                    # Generate creative matrix (themes x concepts x personas x formats)
npm run ads:generate                  # Generate images via Gemini + composite text
npm run ads:generate -- --limit=3     # Generate only 3 images (for testing)
npm run ads:review                    # Launch review gallery at http://localhost:3847
npm run ads:upload                    # Upload approved ads to Meta (all PAUSED)
npm run ads:report last_7d            # Pull performance metrics
npm run ads:report -- --round=2       # Report for specific round

# Multi-round optimization
npm run ads:select-winners            # Select top 10 by CTR
npm run ads:new-round                 # Create next round from winners + untried concepts
```

## Architecture

### Pipeline Flow

```
brand.json + ad-inputs.json + campaign-config.json
    → ads:matrix (Cartesian product + Gemini copy generation)
    → ads:generate (Gemini image gen + Sharp text compositing)
    → ads:review (Express gallery for approve/reject)
    → ads:upload (Meta Graph API: campaign → ad sets → creatives → ads)
    → ads:report (Meta Insights API → console tables + HTML report)
    → ads:select-winners → ads:new-round → loops back to ads:generate
```

### Key Source Layout

```
src/shared/          — Brand config, types, utilities (env, paths, rate-limit, rounds)
src/ads/matrix/      — Creative matrix generation + prompt builder
src/ads/generation/  — Gemini image generation + Sharp/opentype text compositing
src/ads/review/      — Express review gallery (single inline HTML page, no frontend build)
src/ads/upload/      — Meta Graph API wrapper + upload orchestration
src/ads/reporting/   — Meta Insights API + HTML report generation
scripts/             — select-winners.ts, new-round.ts (round management)
```

### Configuration Files (project-config/)

- `brand.json` — Brand identity (colors, typography, voice, selling points). Validated by Zod `BrandConfigSchema`.
- `ad-inputs.json` — Themes (visual worlds with concepts) + personas (audience segments with Meta targeting).
- `campaign-config.json` — Meta campaign settings (objective, budget, page IDs, tracking/pixel).
- `prompt-template.json` — Optional photography style and requirements overrides for image generation prompts.

### Path Aliases

TSConfig defines `@shared/*` → `src/shared/*` and `@ads/*` → `src/ads/*`.

## Critical Design Patterns

### Creative Matrix

The matrix is a Cartesian product: **active themes x concepts x personas x format** (4:1 feed-to-story ratio from a cycling pool of 5). Entry IDs follow `{theme.id}-{concept.id}-{persona.id}-{format.name}` — this naming convention is parsed by the review gallery and round system.

### Round-Aware vs Legacy

All modules detect `rounds-index.json` to switch between round paths (`output/ads/rounds/round-{N}/`) and legacy flat paths (`output/ads/`). Use the pattern:
```typescript
if (roundsIndexExists()) return roundMatrix(currentRoundNum());
return paths.output.ads.matrix;
```

### Rate Limiting

Different delays per API: Gemini copy 1000ms, Gemini image 2000ms, Meta API 1500ms. All use `createRateLimiter()`.

### Progressive Save

Matrix is saved after each image generation and each ad upload to prevent data loss on crash.

### Text Compositing

Text is rendered at 3x supersample via opentype.js SVG paths, then downscaled with Sharp. White text with black outline + drop shadow on gradient overlays for universal readability. All dimensions are proportional to image width/height.

### Meta Upload

All ads created PAUSED. Budget sent in cents (dollars x 100). Ad account ID auto-prefixed with `act_`. CTA strings mapped from friendly names to Meta API constants.

## Environment Variables

| Variable | Required | Notes |
|---|---|---|
| `GEMINI_API_KEY` | Yes | Image generation + ad copy |
| `META_ACCESS_TOKEN` | Yes | System user token with `ads_management` |
| `META_AD_ACCOUNT_ID` | Yes | Auto-prefixed with `act_` |
| `META_BUSINESS_ID` | Yes | Business Manager ID |
| `META_PAGE_ID` | No | Can be set in campaign-config.json instead |
| `META_INSTAGRAM_ACCOUNT_ID` | No | Enables Instagram placements |

## Specs Reference

When implementing or modifying a module, read the corresponding spec in `specs/` first:

- `01-project-setup.md` — Dependencies, tsconfig, directory structure, full pipeline diagram
- `02-brand-config.md` — Zod schema, loading pattern, field usage across pipeline
- `03-ad-inputs.md` — Themes/concepts/personas types, seasonal filtering algorithm, format distribution
- `04-campaign-config.md` — Campaign types, CBO vs ABO budget strategies, tracking config
- `05-creative-matrix.md` — Matrix algorithm, ad copy Gemini prompt, status lifecycle
- `06-image-prompt-system.md` — buildImagePrompt() and buildCopyGenerationPrompt() algorithms
- `07-image-generation.md` — Multi-part Gemini input with reference images, asset loading
- `08-text-compositing.md` — Sharp + opentype.js rendering pipeline, layout for feed/story formats
- `09-review-gallery.md` — Express routes, round navigation, inline HTML UI
- `10-meta-upload.md` — Graph API endpoints, upload pipeline, retry logic, CTA mapping
- `11-performance-reporting.md` — Insights API, demographic breakdowns, HTML report
- `12-multi-round-optimization.md` — Winner selection, new round creation, concept tracking
- `13-shared-utilities.md` — env, paths, rate-limit, rounds helpers
- `14-asset-management.md` — Asset directory conventions, naming patterns, size recommendations
