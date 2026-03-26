# 04 — Campaign Configuration

Defines how ads are structured and uploaded to Meta: campaign objective, budget strategy, tracking, and page IDs.

## File Location

`project-config/campaign-config.json`

## TypeScript Type

```typescript
// src/shared/types/ads.ts

export interface CampaignConfig {
  campaignName: string;
  objective:
    | "OUTCOME_SALES"
    | "OUTCOME_LEADS"
    | "OUTCOME_TRAFFIC"
    | "OUTCOME_AWARENESS"
    | "OUTCOME_ENGAGEMENT"
    | "OUTCOME_APP_PROMOTION";
  budgetOptimization: "CBO" | "ABO";
  dailyBudget?: number;         // In dollars (converted to cents for API)
  pageId: string;                // Facebook Page ID
  instagramActorId?: string;     // Instagram Business Account ID
  websiteUrl?: string;           // Landing page base URL
  tracking: {
    method: "conversions-api" | "pixel" | "both";
    pixelId?: string;            // Meta Pixel ID
    datasetId?: string;          // Conversions API dataset ID
    conversionEvent: string;     // e.g. "Purchase", "AddToCart", "Lead"
  };
}
```

## Field Reference

### Campaign-Level

| Field | Description |
|---|---|
| `campaignName` | Display name in Meta Ads Manager |
| `objective` | What the campaign optimises for. `OUTCOME_SALES` optimises for conversions; `OUTCOME_TRAFFIC` for link clicks |
| `budgetOptimization` | `CBO` = budget set at campaign level, Meta distributes across ad sets. `ABO` = budget set per ad set |
| `dailyBudget` | Daily budget in **dollars**. Converted to cents (×100) when sent to Meta API |

### Page & Platform

| Field | Description |
|---|---|
| `pageId` | Facebook Page ID — required for all ads. Find in Page Settings → About |
| `instagramActorId` | Instagram Business Account ID — enables Instagram placements. Find via `GET /{pageId}?fields=instagram_business_account` |
| `websiteUrl` | Base URL for ad links. Theme ID is appended: `{websiteUrl}/theme/{theme.id}` |

### Tracking

| Field | Description |
|---|---|
| `tracking.method` | `"conversions-api"` for server-side tracking, `"pixel"` for client-side, `"both"` for dual |
| `tracking.pixelId` | Meta Pixel ID — used with `"pixel"` or `"both"` method |
| `tracking.datasetId` | Conversions API dataset ID — used as `pixel_id` in promoted_object |
| `tracking.conversionEvent` | Event name to optimise for, e.g. `"Purchase"`. Uppercased for the API |

## Budget Strategies

### CBO (Campaign Budget Optimization) — Recommended

- Budget set at **campaign level** (`dailyBudget` passed to `createCampaign`)
- Meta automatically distributes spend across ad sets based on performance
- Best for: letting Meta find the best-performing audience segments

### ABO (Ad Set Budget Optimization)

- Budget set at **ad set level** (`dailyBudget` passed to each `createAdSet`)
- You control exactly how much each persona segment receives
- Best for: controlled A/B testing where each persona gets equal spend

## How Config Flows Into Upload

```
campaignConfig.objective      → createCampaign(objective)
campaignConfig.dailyBudget    → createCampaign(daily_budget) if CBO
                              → createAdSet(daily_budget) if ABO
campaignConfig.pageId         → createAdCreative(object_story_spec.page_id)
campaignConfig.instagramActorId → createAdSet(instagramActorId) for platform targeting
                                → createAdCreative(object_story_spec.instagram_actor_id)
campaignConfig.websiteUrl     → createAdCreative(link_data.link) as "{url}/theme/{theme.id}"
campaignConfig.tracking       → createAdSet(promoted_object: { pixel_id, custom_event_type })
```

## Example campaign-config.json

```json
{
  "campaignName": "{Your Brand} - {Campaign Description}",
  "objective": "OUTCOME_SALES",
  "budgetOptimization": "CBO",
  "dailyBudget": 20,
  "pageId": "{your-facebook-page-id}",
  "instagramActorId": "{your-instagram-actor-id}",
  "websiteUrl": "https://{your-domain.com}",
  "tracking": {
    "method": "conversions-api",
    "datasetId": "{your-dataset-id}",
    "conversionEvent": "Purchase"
  }
}
```
