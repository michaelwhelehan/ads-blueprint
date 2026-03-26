# 11 — Performance Reporting

Pulls ad performance data from Meta's Insights API, displays console tables, and generates an HTML report with images and demographic breakdowns.

## npm Script

```bash
npm run ads:report              # Default: last 7 days
npm run ads:report last_30d     # Custom date range
npm run ads:report -- --campaign {campaign_id}  # Specific campaign
npm run ads:report -- --round=2 # Specific round
```

## Source File

`src/ads/reporting/performance.ts`

## Date Presets

| Preset | Days Back |
|---|---|
| `today` | 0 |
| `yesterday` | 1 |
| `last_3d` | 2 |
| `last_7d` | 6 |
| `last_14d` | 13 |
| `last_28d` | 27 |
| `last_30d` | 29 |
| `last_90d` | 89 |

Presets are converted to `time_range` JSON for the API:

```typescript
function datePresetToTimeRange(preset: string): Record<string, string> {
  // Calculate since/until dates from preset
  return { time_range: JSON.stringify({ since: "YYYY-MM-DD", until: "YYYY-MM-DD" }) };
}
```

## Insights API

### Ad-Level Performance

```
GET /{campaign_id}/insights
  ?fields=ad_name,adset_name,impressions,clicks,ctr,cpc,spend,actions
  &level=ad
  &time_range={"since":"2025-01-01","until":"2025-01-07"}
```

**Response type:**

```typescript
interface MetaInsightRow {
  ad_name: string;
  adset_name?: string;
  impressions: string;    // Note: strings, not numbers
  clicks: string;
  ctr: string;
  cpc: string;
  spend: string;
  actions?: MetaInsightAction[];
}

interface MetaInsightAction {
  action_type: string;    // e.g. "offsite_conversion", "link_click"
  value: string;
}
```

### Demographic Breakdowns

```
GET /{campaign_id}/insights
  ?fields=impressions,clicks,spend,ctr,cpc,actions
  &level=campaign
  &breakdowns={breakdown_type}
  &time_range=...
```

**Breakdown types:**
- `age,gender` — Age and gender segments
- `country` — Geographic breakdown
- `publisher_platform,platform_position` — Facebook vs Instagram, feed vs stories

**Response type:**

```typescript
interface MetaDemographicRow {
  age?: string;
  gender?: string;
  country?: string;
  publisher_platform?: string;
  platform_position?: string;
  impressions: string;
  clicks: string;
  spend: string;
  ctr: string;
  cpc: string;
  actions?: MetaInsightAction[];
}
```

### Pagination

Insights responses may be paginated. Follow `paging.next` URLs:

```typescript
async function fetchAllPages<T>(initialUrl: string): Promise<T[]> {
  const all: T[] = [];
  let url = initialUrl;
  while (url) {
    const data = await metaFetch(url);
    all.push(...data.data);
    url = data.paging?.next;
  }
  return all;
}
```

## Algorithm

```
main():
  1. Parse CLI args (datePreset, campaignId, roundNum)
  2. Load upload log (from round or campaign arg) → get campaign ID
  3. Fetch ad-level insights
  4. Convert to PerformanceRow[], sort by clicks descending
  5. Print console table
  6. Calculate and print summary (total spend, impressions, clicks, CTR, CPC, conversions)
  7. Print top 5 by clicks
  8. Fetch demographic breakdowns in parallel:
     - Age & Gender
     - Country
     - Platform & Placement
  9. Print demographic tables
  10. Fetch ad-set-level insights → map to personas → print interest performance
  11. Build HTML report with images
  12. Save report.html
  13. Open in browser (macOS: execFile("open", [htmlPath]))
```

## Conversion Extraction

```typescript
const conversions = insight.actions?.find(
  (a) => a.action_type === "offsite_conversion"
)?.value || "0";
```

## Interest Performance

Maps ad set performance back to persona targeting:

```
printInterestPerformance():
  1. Load matrix → build adSetId → persona mapping from upload log
  2. Fetch adset-level insights
  3. Match insights to personas by ad set name pattern: "{campaignName} - {persona.name}"
  4. Display persona name, interest keywords, and performance metrics
```

## HTML Report

Generated as a self-contained HTML file with inline CSS and JavaScript.

### Components

1. **Summary cards** — Total spend, impressions, clicks, CTR, CPC, conversions
2. **Ad performance grid** — Card per ad with:
   - Thumbnail image (from local files)
   - Metrics (impressions, clicks, CTR, CPC, spend, conversions)
   - Click-to-enlarge lightbox
3. **Demographic tables** — Age/Gender, Country, Platform/Placement
4. **Dark theme** — Consistent with review gallery styling

### Image Path Resolution

Ad images are linked using relative paths from the HTML file location:

```typescript
function buildAdImageMap(log, matrix, insights, roundNum):
  for each insight:
    strip "Ad - " prefix from ad_name
    find matching entry in matrix (via upload log or direct ID match)
    resolve image path relative to report directory
  return Map<adName, relativePath>
```

## Output

- **Console**: Formatted tables and summary
- **HTML**: `output/ads/rounds/round-{N}/report.html` (or `output/ads/report.html` legacy)
- Auto-opens in default browser on macOS
