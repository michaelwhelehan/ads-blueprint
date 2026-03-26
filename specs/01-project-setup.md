# 01 — Project Setup

Everything needed to initialise the Meta Ads pipeline project from scratch.

## Tech Stack

- **Runtime**: Node.js (ESM)
- **Language**: TypeScript
- **AI**: Google Gemini API (`@google/genai`) — image generation + ad copy
- **Ads Platform**: Meta Graph API v21.0 — campaign management
- **Image Processing**: Sharp (compositing) + opentype.js (text rendering)
- **Validation**: Zod
- **Review UI**: Express.js (local dev server)

## Dependencies

### Runtime

```json
{
  "@google/genai": "^1.44.0",
  "dotenv": "^17.3.1",
  "express": "^5.2.1",
  "opentype.js": "^1.3.4",
  "sharp": "^0.34.5",
  "zod": "^4.3.6"
}
```

### Dev

```json
{
  "@types/express": "^5.0.6",
  "@types/node": "^25.4.0",
  "tsx": "^4.21.0",
  "typescript": "^5.9.3"
}
```

## TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "strict": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@shared/*": ["src/shared/*"],
      "@ads/*": ["src/ads/*"]
    }
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "output"]
}
```

## package.json Essentials

```json
{
  "type": "module",
  "scripts": {
    "ads:matrix": "tsx src/ads/matrix/generate-matrix.ts",
    "ads:generate": "tsx src/ads/generation/generate-images.ts",
    "ads:review": "tsx src/ads/review/gallery-server.ts",
    "ads:upload": "tsx src/ads/upload/meta-uploader.ts",
    "ads:report": "tsx src/ads/reporting/performance.ts",
    "ads:select-winners": "tsx scripts/select-winners.ts",
    "ads:new-round": "tsx scripts/new-round.ts"
  }
}
```

## Directory Structure

```
{project-root}/
├── src/
│   ├── shared/
│   │   ├── brand/           # Brand config schema + loader
│   │   │   └── brand-config.ts
│   │   ├── types/           # TypeScript interfaces
│   │   │   └── ads.ts
│   │   └── utils/           # Shared helpers
│   │       ├── env.ts       # Environment variable accessors
│   │       ├── paths.ts     # Centralised path resolution
│   │       ├── rate-limit.ts # Throttle for API calls
│   │       └── rounds.ts   # Multi-round utilities
│   │
│   └── ads/
│       ├── matrix/          # Creative matrix generation
│       │   ├── generate-matrix.ts
│       │   └── prompt-builder.ts
│       ├── generation/      # Image generation + compositing
│       │   └── generate-images.ts
│       ├── review/          # Local gallery server
│       │   └── gallery-server.ts
│       ├── upload/          # Meta API integration
│       │   ├── meta-api.ts
│       │   └── meta-uploader.ts
│       └── reporting/       # Performance analytics
│           └── performance.ts
│
├── scripts/                 # Round management scripts
│   ├── select-winners.ts
│   └── new-round.ts
│
├── assets/                  # Brand assets (see spec 14)
│   ├── logos/
│   ├── fonts/
│   ├── brand-photos/
│   └── physical-items/
│
├── project-config/          # Runtime configuration
│   ├── brand.json
│   ├── ad-inputs.json
│   ├── campaign-config.json
│   └── prompt-template.json (optional)
│
├── output/                  # Generated content (gitignored)
│   └── ads/
│       ├── rounds/
│       │   ├── rounds-index.json
│       │   └── round-{N}/
│       │       ├── matrix.json
│       │       ├── images/
│       │       ├── upload-log.json
│       │       ├── winners.json
│       │       ├── ad-inputs.json (snapshot)
│       │       └── report.html
│       ├── matrix.json      (legacy, pre-rounds)
│       ├── images/          (legacy)
│       └── campaigns/
│           └── upload-log.json (legacy)
│
├── .env
├── .gitignore
├── package.json
└── tsconfig.json
```

## Environment Variables

Create a `.env` file at the project root:

| Variable | Required | Description |
|---|---|---|
| `GEMINI_API_KEY` | Yes | Google Gemini API key for image generation and ad copy |
| `META_ACCESS_TOKEN` | Yes | Meta system user token with `ads_management` permission |
| `META_AD_ACCOUNT_ID` | Yes | Ad account ID (with or without `act_` prefix — auto-added) |
| `META_BUSINESS_ID` | Yes | Meta Business Manager ID |
| `META_PAGE_ID` | No | Facebook Page ID (can also be set in campaign-config.json) |
| `META_INSTAGRAM_ACCOUNT_ID` | No | Instagram Business Account ID |

## .gitignore

```
node_modules/
dist/
output/
.env
*.png
*.jpg
.DS_Store
```

## Pipeline Flow

```
brand.json + ad-inputs.json + campaign-config.json
                    │
                    ▼
            ┌──────────────┐
            │  ads:matrix   │  Generate creative matrix (themes × concepts × personas × formats)
            └──────┬───────┘
                   ▼
            ┌──────────────┐
            │ ads:generate  │  Generate images via Gemini + composite text/branding
            └──────┬───────┘
                   ▼
            ┌──────────────┐
            │  ads:review   │  Local gallery — approve/reject images
            └──────┬───────┘
                   ▼
            ┌──────────────┐
            │  ads:upload   │  Upload approved ads to Meta (PAUSED)
            └──────┬───────┘
                   ▼
            ┌──────────────┐
            │  ads:report   │  Pull performance metrics from Meta
            └──────┬───────┘
                   ▼
          ┌────────────────────┐
          │ ads:select-winners │  Pick top N by CTR
          └────────┬───────────┘
                   ▼
          ┌────────────────────┐
          │   ads:new-round    │  Copy winners + add untried concepts → next round
          └────────────────────┘
                   │
                   └──── loops back to ads:generate ────►
```
