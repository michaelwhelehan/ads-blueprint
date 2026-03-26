# Ads Blueprint

A specification-driven Meta Ads automation pipeline that generates ad creatives with AI, composites branded text overlays, uploads to Meta Ads, and iteratively optimizes through multi-round performance analysis.

## How It Works

```
brand.json + ad-inputs.json + campaign-config.json
  → ads:matrix        Generate creative matrix (themes x concepts x personas x formats)
  → ads:generate      Generate images via Gemini + composite branded text with Sharp
  → ads:review        Launch local gallery to approve/reject creatives
  → ads:upload        Upload approved ads to Meta (all created PAUSED)
  → ads:report        Pull performance metrics from Meta Insights API
  → ads:select-winners + ads:new-round → loop back for next optimization round
```

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js (ESM) + TypeScript (strict) |
| AI | Google Gemini API — image generation + ad copy |
| Ads Platform | Meta Graph API v21.0 |
| Image Processing | Sharp (compositing, gradients) + opentype.js (font rendering) |
| Validation | Zod |
| Review UI | Express.js (local server on port 3847) |
| Dev Runner | tsx |

## Project Status

This is a **specification repository**. The `specs/` directory contains 14 detailed implementation specs that define the full pipeline. No application code exists yet.

To scaffold the project from the specs, use the `/ads-bootstrap` skill in Claude Code.

## Specs

| # | Spec | Description |
|---|---|---|
| 01 | [Project Setup](specs/01-project-setup.md) | Dependencies, tsconfig, directory structure |
| 02 | [Brand Config](specs/02-brand-config.md) | Zod schema for brand identity (colors, typography, voice) |
| 03 | [Ad Inputs](specs/03-ad-inputs.md) | Themes, concepts, personas, format distribution |
| 04 | [Campaign Config](specs/04-campaign-config.md) | Meta campaign settings, budget strategies, tracking |
| 05 | [Creative Matrix](specs/05-creative-matrix.md) | Cartesian product algorithm, ad copy generation |
| 06 | [Image Prompt System](specs/06-image-prompt-system.md) | Gemini prompt construction for images and copy |
| 07 | [Image Generation](specs/07-image-generation.md) | Multi-part Gemini input with reference images |
| 08 | [Text Compositing](specs/08-text-compositing.md) | Sharp + opentype.js rendering pipeline |
| 09 | [Review Gallery](specs/09-review-gallery.md) | Express routes, inline HTML UI, round navigation |
| 10 | [Meta Upload](specs/10-meta-upload.md) | Graph API endpoints, retry logic, CTA mapping |
| 11 | [Performance Reporting](specs/11-performance-reporting.md) | Insights API, demographic breakdowns, HTML reports |
| 12 | [Multi-Round Optimization](specs/12-multi-round-optimization.md) | Winner selection, new round creation, concept tracking |
| 13 | [Shared Utilities](specs/13-shared-utilities.md) | Env, paths, rate limiting, rounds helpers |
| 14 | [Asset Management](specs/14-asset-management.md) | Asset directory conventions, naming, size recommendations |

## Pipeline Commands

```bash
# Full pipeline (run in order)
npm run ads:matrix                    # Generate creative matrix
npm run ads:generate                  # Generate images + composite text
npm run ads:generate -- --limit=3     # Generate only 3 (for testing)
npm run ads:review                    # Launch review gallery at localhost:3847
npm run ads:upload                    # Upload approved ads to Meta
npm run ads:report last_7d            # Pull performance metrics

# Multi-round optimization
npm run ads:select-winners            # Select top performers by CTR
npm run ads:new-round                 # Create next round from winners + fresh concepts
```

## Configuration

Three JSON files in `project-config/` drive the pipeline:

- **`brand.json`** — Brand identity: colors, typography, voice, selling points
- **`ad-inputs.json`** — Themes (visual worlds with concepts) + personas (audience segments with Meta targeting)
- **`campaign-config.json`** — Meta campaign settings: objective, budget, page IDs, pixel/tracking

## Environment Variables

| Variable | Required | Notes |
|---|---|---|
| `GEMINI_API_KEY` | Yes | Image generation + ad copy |
| `META_ACCESS_TOKEN` | Yes | System user token with `ads_management` scope |
| `META_AD_ACCOUNT_ID` | Yes | Auto-prefixed with `act_` |
| `META_BUSINESS_ID` | Yes | Business Manager ID |
| `META_PAGE_ID` | No | Can be set in `campaign-config.json` instead |
| `META_INSTAGRAM_ACCOUNT_ID` | No | Enables Instagram placements |

## Architecture

```
src/
├── shared/              Shared config, types, utilities
│   ├── brand-config.ts    Brand loading + Zod validation
│   ├── types.ts           Core type definitions
│   ├── env.ts             Environment variable loading
│   ├── paths.ts           Output path resolution
│   ├── rate-limit.ts      Per-API rate limiter
│   └── rounds.ts          Round detection + path helpers
├── ads/
│   ├── matrix/            Creative matrix generation + prompt builder
│   ├── generation/        Gemini image gen + Sharp/opentype compositing
│   ├── review/            Express review gallery (single inline HTML page)
│   ├── upload/            Meta Graph API wrapper + upload orchestration
│   └── reporting/         Meta Insights API + HTML report generation
└── scripts/
    ├── select-winners.ts  Winner selection by CTR
    └── new-round.ts       Next round creation from winners + untried concepts
```

## Key Design Decisions

- **Creative matrix** is a Cartesian product of themes x concepts x personas x formats (4:1 feed-to-story ratio)
- **All ads upload as PAUSED** — nothing goes live without manual activation
- **Progressive save** — matrix and upload state saved after each operation to prevent data loss
- **Text compositing** uses 3x supersample rendering for crisp text at all sizes
- **Round-aware paths** — modules auto-detect `rounds-index.json` and switch between round-based and flat output directories

## License

Private — all rights reserved.
