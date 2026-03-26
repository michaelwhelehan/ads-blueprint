# 10 — Meta Upload

Uploads approved ad creatives to Meta via the Graph API v21.0. Creates a campaign → ad sets (grouped by persona) → ad creatives → ads. Everything is created PAUSED.

## npm Script

```bash
npm run ads:upload
```

## Source Files

- `src/ads/upload/meta-api.ts` — Low-level API wrapper
- `src/ads/upload/meta-uploader.ts` — Upload orchestration pipeline

## Meta Graph API v21.0

### Base URL

```
https://graph.facebook.com/v21.0
```

### Authentication

All requests include:
```
Authorization: Bearer {META_ACCESS_TOKEN}
```

### Fetch Wrapper

```typescript
async function metaFetch<T>(endpoint: string, options?: RequestInit): Promise<T> {
  const url = endpoint.startsWith("http") ? endpoint : `${API_BASE}/${endpoint}`;
  const response = await fetch(url, {
    ...options,
    signal: AbortSignal.timeout(30_000),  // 30s timeout
    headers: {
      Authorization: `Bearer ${env.metaAccessToken()}`,
      ...options?.headers,
    },
  });
  const data = await response.json();
  if (!response.ok) throw new Error(`Meta API error (${response.status}): ${data.error?.message}`);
  return data;
}
```

## API Endpoints

### 1. Image Upload

```
POST /{ad_account_id}/adimages
Content-Type: multipart/form-data

Body (FormData):
  filename: string (basename of file)
  source: Blob (image bytes)

Response: { images: { [filename]: { hash: string } } }
Returns: image hash (string)
```

### 2. Campaign Creation

```
POST /{ad_account_id}/campaigns
Content-Type: application/x-www-form-urlencoded

Body (URLSearchParams):
  name: string
  objective: string                    // e.g. "OUTCOME_SALES"
  special_ad_categories: "[]"          // Required, empty array as string
  status: "PAUSED"
  buying_type: "AUCTION"
  bid_strategy: "LOWEST_COST_WITHOUT_CAP"
  daily_budget?: string               // In CENTS (multiply dollars by 100)

Response: { id: string }
Returns: campaign ID
```

### 3. Ad Set Creation

```
POST /{ad_account_id}/adsets
Content-Type: application/x-www-form-urlencoded

Body (URLSearchParams):
  name: string                         // "{campaignName} - {persona.name}"
  campaign_id: string
  optimization_goal: "OFFSITE_CONVERSIONS"   // Default
  billing_event: "IMPRESSIONS"                // Default
  status: "PAUSED"
  targeting: string                    // JSON.stringify of targeting object (see below)
  daily_budget?: string               // In CENTS (only for ABO mode)
  promoted_object?: string            // JSON.stringify (see below)

Response: { id: string }
Returns: ad set ID
```

**Targeting Object Structure:**

```json
{
  "age_min": 25,
  "age_max": 45,
  "genders": [2],                      // Optional: 1=male, 2=female
  "geo_locations": { "countries": ["US", "GB"] },
  "flexible_spec": [
    { "interests": [{ "id": "6003...", "name": "..." }] }
  ],
  "targeting_automation": { "advantage_audience": 0 },
  "publisher_platforms": ["facebook", "instagram"]  // If Instagram enabled
}
```

**Promoted Object (for conversion tracking):**

```json
{
  "pixel_id": "{dataset_id}",
  "custom_event_type": "PURCHASE"       // Uppercased conversion event
}
```

### 4. Ad Creative Creation

```
POST /{ad_account_id}/adcreatives
Content-Type: application/x-www-form-urlencoded

Body (URLSearchParams):
  name: string                          // "Creative - {entry.id}"
  object_story_spec: string             // JSON.stringify (see below)

Response: { id: string }
Returns: creative ID
```

**Object Story Spec Structure:**

```json
{
  "page_id": "{facebook_page_id}",
  "instagram_actor_id": "{ig_actor_id}",  // Optional
  "link_data": {
    "image_hash": "{hash_from_upload}",
    "message": "{adCopy.primaryText}",
    "name": "{adCopy.headline}",
    "link": "{websiteUrl}/theme/{theme.id}",
    "call_to_action": {
      "type": "LEARN_MORE",              // See CTA mapping below
      "value": { "link": "{websiteUrl}/theme/{theme.id}" }
    }
  }
}
```

### 5. Ad Creation

```
POST /{ad_account_id}/ads
Content-Type: application/x-www-form-urlencoded

Body (URLSearchParams):
  name: string                          // "Ad - {entry.id}"
  adset_id: string
  creative: string                      // JSON.stringify({ creative_id: "{id}" })
  status: "PAUSED"

Response: { id: string }
Returns: ad ID
```

### 6. Status Update

```
POST /{object_id}
Content-Type: application/x-www-form-urlencoded

Body: status=ACTIVE|PAUSED
```

### 7. Insights (see spec 11)

```
GET /{object_id}/insights?fields=...&level=...&time_range=...
```

## CTA Type Mapping

Ad copy CTA strings are converted to Meta API constants:

```typescript
const mapping = {
  "Learn More": "LEARN_MORE",
  "Sign Up":    "SIGN_UP",
  "Shop Now":   "SHOP_NOW",
  "Download":   "DOWNLOAD",
  "Get Offer":  "GET_OFFER",
};

// Overrides for invalid CTAs that Gemini might return:
const overrides = {
  "GET_STARTED": "LEARN_MORE",
  "TRY_FREE":    "LEARN_MORE",
  "BUY":         "SHOP_NOW",
};
```

## Upload Pipeline Algorithm

```
main():
  1. Load matrix, filter entries where status === "approved" AND imagePath exists
  2. If none → exit with message
  3. Load campaign config
  4. Create campaign (PAUSED):
     campaignId = createCampaign({
       name, objective,
       dailyBudget: CBO ? config.dailyBudget : undefined,
       status: "PAUSED"
     })
  5. Group approved entries by persona.id
  6. For each persona group:
     a. Create ad set (PAUSED):
        adSetId = createAdSet({
          name: "{campaignName} - {persona.name}",
          campaignId,
          dailyBudget: ABO ? config.dailyBudget : undefined,
          targeting: persona.targeting,
          promotedObject: if tracking.datasetId exists,
          instagramActorId: config.instagramActorId
        })
     b. For each entry in group (with retry):
        i.   Upload image → imageHash
        ii.  Create creative → creativeId
        iii. Create ad → adId
        iv.  Set entry.status = "uploaded"
        v.   Record { adSetId, adId, entryId } in upload log
     c. Save matrix after each ad (progressive)
  7. Save upload log
  8. Update rounds index with campaignId (if round mode)
```

## Retry Logic

```typescript
async function withRetry<T>(fn, maxAttempts = 3, baseDelayMs = 2000): Promise<T> {
  for (attempt = 1 to maxAttempts):
    try: return await fn()
    catch (err):
      // Don't retry on 4xx auth/client errors (except 429 rate limit)
      if error is 4xx (not 429): throw immediately
      if not last attempt:
        delay = baseDelayMs × 2^(attempt-1)  // 2s, 4s, 8s
        wait(delay)
  throw last error
}
```

## Rate Limiting

1500ms between Meta API calls via `createRateLimiter(1500)`. This is conservative to avoid rate limits.

## Upload Log

Saved to `output/ads/rounds/round-{N}/upload-log.json` (or `output/ads/campaigns/upload-log.json` legacy):

```json
{
  "campaignId": "{meta_campaign_id}",
  "uploadedAt": "2025-01-15T10:30:00.000Z",
  "ads": [
    { "adSetId": "{id}", "adId": "{id}", "entryId": "{matrix_entry_id}" },
    ...
  ]
}
```

## Error Response Types

```typescript
interface MetaApiError {
  error?: {
    message: string;
    type: string;         // e.g. "OAuthException"
    code: number;         // e.g. 190 (invalid token), 100 (invalid parameter)
    error_subcode?: number;
    error_user_msg?: string;
    fbtrace_id?: string;
  };
}
```

## Important Notes

- **All ads are created PAUSED** — manual activation required in Meta Ads Manager
- **Budget in cents** — `dailyBudget` in config is dollars, multiplied by 100 for API
- **Ad account ID prefix** — automatically adds `act_` prefix if missing
- **Website URL** — theme ID is appended as path: `{websiteUrl}/theme/{theme.id}`
