# 03 — Ad Inputs (Themes, Concepts, Personas)

Defines the creative building blocks: themes (visual worlds), concepts (specific scenes), and personas (audience segments with Meta targeting).

## File Location

`project-config/ad-inputs.json`

## TypeScript Types

```typescript
// src/shared/types/ads.ts

export interface SeasonalWindow {
  activeFrom: string; // "DD-MM" format, e.g. "01-12" for Dec 1
  activeTo: string;   // "DD-MM" format, e.g. "05-01" for Jan 5
}

export interface AdConcept {
  id: string;                // kebab-case, e.g. "speakeasy-table"
  sceneDescription: string;  // Detailed visual brief for image generation
  hook: string;              // Ad headline, max ~40 chars
  bullets: string[];         // 3 key features/benefits for ad copy
}

export interface AdTheme {
  id: string;                // kebab-case, e.g. "jazz-age-jubilee"
  name: string;              // Display name
  description: string;       // Long-form theme description
  seasonal?: SeasonalWindow; // Optional — if set, theme only active in window
  concepts: AdConcept[];     // 2-3 concept variations per theme
}

export interface MetaTargeting {
  age_min: number;
  age_max: number;
  genders?: number[];  // 1=male, 2=female (omit for all)
  interests: { id: string; name: string }[];
  geo_locations?: { countries: string[] };
}

export interface AdPersona {
  id: string;                // kebab-case, e.g. "corporate-planners"
  name: string;              // Display name
  description: string;       // Who this persona is
  painPoints: string[];      // What problems they have
  targeting: MetaTargeting;  // Meta Ads targeting parameters
}

export interface AdInputs {
  themes: AdTheme[];
  personas: AdPersona[];
}
```

## Ad Formats

Defined as constants — not part of ad-inputs.json:

```typescript
export interface AdFormat {
  name: string;
  width: number;
  height: number;
  placement: string;
}

export const AD_FORMATS: Record<string, AdFormat> = {
  "feed-square":    { name: "feed-square",    width: 1080, height: 1080, placement: "feed" },
  "story-portrait": { name: "story-portrait", width: 1080, height: 1920, placement: "story" },
};
```

## Format Distribution

Formats are assigned from a cycling pool with a **4:1 feed-to-story ratio**:

```typescript
const formatPool: AdFormat[] = [
  FORMATS["feed-square"],    // index 0
  FORMATS["feed-square"],    // index 1
  FORMATS["feed-square"],    // index 2
  FORMATS["feed-square"],    // index 3
  FORMATS["story-portrait"], // index 4
];
// Each combination gets: formatPool[index % 5]
```

## Seasonal Filtering

Themes with a `seasonal` window are only included when the current date falls within the range:

```
Algorithm: isThemeActive(theme, now)
  if no seasonal window → always active
  convert activeFrom and activeTo to MMDD integers
  convert today to MMDD integer
  if from <= to (same year): active when today >= from AND today <= to
  if from > to (year boundary, e.g. Dec→Jan): active when today >= from OR today <= to
```

## Naming Conventions

IDs use **kebab-case** and combine into filenames:

```
{theme.id}-{concept.id}-{persona.id}-{format.name}.png
```

Example: `jazz-age-jubilee-speakeasy-table-dinner-hosts-feed-square.png`

This convention is critical — the review gallery and round system parse filenames to extract metadata.

## Matrix Combination Algorithm

The creative matrix is the **Cartesian product** of active themes × concepts × personas × format (from pool):

```
for each active theme:
  for each concept in theme.concepts:
    for each persona:
      format = formatPool[index % 5]
      index++
      create entry with id: "{theme}-{concept}-{persona}-{format}"
```

With 4 themes × 3 concepts × 3 personas = 36 entries (roughly 29 feed + 7 story).

## Finding Meta Interest IDs

Meta interest IDs are required for targeting. Use the Targeting Search API:

```
GET https://graph.facebook.com/v21.0/search
  ?type=adinterest
  &q={keyword}
  &access_token={token}
```

Returns `{ data: [{ id: "6003...", name: "Board games", ... }] }`.

## Example ad-inputs.json

```json
{
  "themes": [
    {
      "id": "{theme-id}",
      "name": "{Theme Display Name}",
      "description": "{Detailed theme description for image generation context}",
      "concepts": [
        {
          "id": "{concept-id}",
          "sceneDescription": "{Detailed visual scene description — what the AI should generate}",
          "hook": "{Short punchy headline, max 40 chars}",
          "bullets": [
            "{Benefit 1}",
            "{Benefit 2}",
            "{Benefit 3}"
          ]
        },
        {
          "id": "{concept-id-2}",
          "sceneDescription": "{Another scene for the same theme}",
          "hook": "{Another headline}",
          "bullets": ["{...}", "{...}", "{...}"]
        }
      ]
    },
    {
      "id": "{seasonal-theme-id}",
      "name": "{Seasonal Theme}",
      "description": "{...}",
      "seasonal": {
        "activeFrom": "01-12",
        "activeTo": "05-01"
      },
      "concepts": [{"...": "..."}]
    }
  ],
  "personas": [
    {
      "id": "{persona-id}",
      "name": "{Persona Display Name}",
      "description": "{Who this audience segment is}",
      "painPoints": [
        "{What problem do they have 1}",
        "{What problem do they have 2}"
      ],
      "targeting": {
        "age_min": 25,
        "age_max": 45,
        "genders": [2],
        "interests": [
          { "id": "6003384...", "name": "{Interest keyword}" },
          { "id": "6003107...", "name": "{Interest keyword 2}" }
        ],
        "geo_locations": { "countries": ["US", "GB", "AU"] }
      }
    }
  ]
}
```
