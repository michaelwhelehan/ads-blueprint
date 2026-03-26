# 13 — Shared Utilities

Utility modules shared across the entire ads pipeline: environment variables, path management, rate limiting, and round helpers.

## Environment Helpers

**File**: `src/shared/utils/env.ts`

```typescript
import { config } from "dotenv";
import { resolve } from "path";

config({ path: resolve(process.cwd(), ".env") });

function requireEnv(key: string): string {
  const value = process.env[key];
  if (!value) throw new Error(`Missing required environment variable: ${key}`);
  return value;
}

function optionalEnv(key: string, fallback: string = ""): string {
  return process.env[key] || fallback;
}

export const env = {
  geminiApiKey:    () => requireEnv("GEMINI_API_KEY"),
  metaAccessToken: () => requireEnv("META_ACCESS_TOKEN"),
  metaAdAccountId: () => {
    const id = requireEnv("META_AD_ACCOUNT_ID");
    return id.startsWith("act_") ? id : `act_${id}`;  // Auto-prefix
  },
  metaBusinessId:  () => requireEnv("META_BUSINESS_ID"),
  metaPageId:      () => optionalEnv("META_PAGE_ID"),
};
```

**Key design**: Lazy getters — env vars are only read when first accessed, so missing vars only error when that specific feature is used.

## Path Management

**File**: `src/shared/utils/paths.ts`

```typescript
import { resolve, join } from "path";
import { mkdirSync } from "fs";

const ROOT = resolve(process.cwd());

export const paths = {
  root: ROOT,
  output: {
    ads: {
      root:        join(ROOT, "output", "ads"),
      images:      join(ROOT, "output", "ads", "images"),
      copy:        join(ROOT, "output", "ads", "copy"),
      campaigns:   join(ROOT, "output", "ads", "campaigns"),
      matrix:      join(ROOT, "output", "ads", "matrix.json"),
      rounds:      join(ROOT, "output", "ads", "rounds"),
      roundsIndex: join(ROOT, "output", "ads", "rounds", "rounds-index.json"),
    },
  },
  assets: {
    logos: join(ROOT, "assets", "logos"),
    fonts: join(ROOT, "assets", "fonts"),
  },
  brandConfig:     join(ROOT, "project-config", "brand.json"),
  adInputs:        join(ROOT, "project-config", "ad-inputs.json"),
  campaignConfig:  join(ROOT, "project-config", "campaign-config.json"),
  promptTemplate:  join(ROOT, "project-config", "prompt-template.json"),
};

export function ensureDir(dir: string): void {
  mkdirSync(dir, { recursive: true });
}
```

**Usage pattern**: All modules import `paths` and use it for every file path. Never hardcode paths.

## Rate Limiting

**File**: `src/shared/utils/rate-limit.ts`

```typescript
export function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

export function createRateLimiter(delayMs: number) {
  let lastCall = 0;

  return async function throttle<T>(fn: () => Promise<T>): Promise<T> {
    const now = Date.now();
    const elapsed = now - lastCall;
    if (elapsed < delayMs) {
      await delay(delayMs - elapsed);
    }
    lastCall = Date.now();
    return fn();
  };
}
```

**Usage across pipeline:**

| Module | Delay | Rate |
|---|---|---|
| Matrix generation (Gemini copy) | 1000ms | 1 req/sec |
| Image generation (Gemini image) | 2000ms | 0.5 req/sec |
| New round creation (Gemini copy) | 1000ms | 1 req/sec |
| Meta upload (all API calls) | 1500ms | ~0.67 req/sec |

## Round Utilities

**File**: `src/shared/utils/rounds.ts`

### Index Management

```typescript
export function roundsIndexExists(): boolean
  // Returns true if rounds-index.json exists

export function loadRoundsIndex(): RoundsIndex
  // Parse and return rounds-index.json

export function saveRoundsIndex(index: RoundsIndex): void
  // Write rounds-index.json (ensures directory exists)

export function currentRoundNum(): number
  // Shortcut: loadRoundsIndex().currentRound
```

### Path Helpers

```typescript
export function roundDir(n: number): string
  // "output/ads/rounds/round-{n}"

export function roundMatrix(n: number): string
  // "output/ads/rounds/round-{n}/matrix.json"

export function roundImages(n: number): string
  // "output/ads/rounds/round-{n}/images"

export function roundUploadLog(n: number): string
  // "output/ads/rounds/round-{n}/upload-log.json"

export function roundWinners(n: number): string
  // "output/ads/rounds/round-{n}/winners.json"

export function roundAdInputs(n: number): string
  // "output/ads/rounds/round-{n}/ad-inputs.json"
```

### Data Loading

```typescript
export function loadRoundMatrix(n: number): CreativeMatrixEntry[]
  // Load and parse matrix for round n (returns [] if not found)

export function saveRoundMatrix(n: number, matrix: CreativeMatrixEntry[]): void
  // Save matrix to round directory (ensures dir exists)
```

### Filename Parsing

Ad metadata is encoded in image filenames. Parsing extracts theme, concept, persona, and format from right to left:

```typescript
export function parseAdFilename(filename: string, knownConcepts?: Set<string>): ParsedAdFilename
  // 1. Strip .png extension
  // 2. Match format from right (known list: "feed-square", "story-portrait")
  // 3. Match persona from right (known list from ad-inputs)
  // 4. Match concept from remaining string (using known concept IDs)
  // 5. Remaining = theme

export function loadKnownConcepts(): Set<string>
  // Load all concept IDs from ad-inputs.json

export function parseImageDirectory(dir: string, knownConcepts?): ParsedAdFilename[]
  // Parse all .png files in directory
```

**Note**: The known personas list is hardcoded in the module. When adapting for your project, update this list to match your persona IDs.

### Concept Tracking

```typescript
export function getAllUsedConcepts(): Map<string, Set<string>>
  // Scan ALL round matrices
  // Return map: themeId → Set of conceptIds used
  // Used by new-round to find untried concepts
```

### Display Names

```typescript
export function toDisplayName(slug: string): string
  // "jazz-age-jubilee" → "Jazz Age Jubilee"
  // Replace hyphens with spaces, capitalise first letter of each word
```

## Round-Aware Code Pattern

All core modules follow this pattern to support both round-based and legacy (pre-rounds) operation:

```typescript
function getMatrixPath(): string {
  if (roundsIndexExists()) return roundMatrix(currentRoundNum());
  return paths.output.ads.matrix;
}

function getImagesDir(): string {
  if (roundsIndexExists()) return roundImages(currentRoundNum());
  return paths.output.ads.images;
}
```

This allows the codebase to work before the rounds system is initialised (using flat `output/ads/` paths) and after (using `output/ads/rounds/round-{N}/` paths).
