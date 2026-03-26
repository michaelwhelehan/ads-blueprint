# 12 — Multi-Round Optimization

The rounds system enables iterative ad optimization: upload ads, measure performance, select winners, create a new round with winners + fresh concepts, and repeat.

## npm Scripts

```bash
npm run ads:select-winners          # Select top 10 by CTR (default: last_7d)
npm run ads:select-winners -- last_30d  # Custom date range
npm run ads:new-round               # Create next round from winners
```

## Rounds Directory Structure

```
output/ads/rounds/
├── rounds-index.json               # Index of all rounds
├── round-1/
│   ├── matrix.json                 # Creative matrix (all entries)
│   ├── ad-inputs.json              # Snapshot of ad-inputs used
│   ├── images/                     # Generated ad images
│   │   └── {entry-id}.png
│   ├── upload-log.json             # Meta upload log
│   ├── winners.json                # Selected winners (after performance data)
│   └── report.html                 # Performance report
├── round-2/
│   ├── matrix.json                 # Winners from round 1 + new concepts
│   ├── ...
```

## Types

```typescript
export interface RoundMeta {
  round: number;
  createdAt: string;       // ISO 8601
  campaignId?: string;     // Set after upload
  description?: string;
}

export interface RoundsIndex {
  currentRound: number;
  rounds: RoundMeta[];
}

export interface WinnerEntry {
  entryId: string;
  ctr: number;
  impressions: number;
  clicks: number;
  spend: number;
}

export interface WinnersFile {
  round: number;
  selectedAt: string;      // ISO 8601
  topN: number;
  winners: WinnerEntry[];
}
```

## Step 1: Winner Selection

**Script**: `scripts/select-winners.ts`

**Constants:**
- `TOP_N = 10` — Number of winners to select
- `MIN_IMPRESSIONS = 100` — Minimum impressions threshold

### Algorithm

```
selectWinners():
  1. Get current round number
  2. Load upload log for current round → campaign ID + ad mapping
  3. Fetch ad-level insights from Meta (default: last_7d)
  4. Build candidates:
     for each insight row:
       skip if impressions < MIN_IMPRESSIONS
       extract entryId from ad name ("Ad - {entryId}" → "{entryId}")
       verify entryId exists in upload log
       add { entryId, ctr, impressions, clicks, spend }
  5. Sort candidates by CTR descending
  6. Take top TOP_N
  7. Write winners.json:
     {
       round: currentRound,
       selectedAt: ISO timestamp,
       topN: TOP_N,
       winners: [{ entryId, ctr, impressions, clicks, spend }]
     }
  8. Save to output/ads/rounds/round-{N}/winners.json
```

## Step 2: New Round Creation

**Script**: `scripts/new-round.ts`

### Algorithm

```
createNewRound():
  1. Load rounds index → prevRound = currentRound, newRound = prevRound + 1
  2. Load winners.json for prevRound
  3. Load prevRound matrix for full entry data
  4. Create new round directory + images directory

  --- Step A: Copy Winners Forward ---
  5. For each winner:
     a. Find entry in prevRound matrix by entryId
     b. Copy image file from round-{prev}/images/ to round-{new}/images/
     c. Create entry in new matrix:
        { ...prevEntry,
          status: "approved",
          sourceRound: prevRound,
          carriedForward: true }
     d. Track winnerThemeIds and winnerPersonaIds

  --- Step B: Find Untried Concepts ---
  6. Load ad-inputs.json
  7. Get all used concepts across ALL rounds:
     getAllUsedConcepts() → Map<themeId, Set<conceptId>>
     (scans every round's matrix)
  8. For each winning theme:
     a. Find theme in ad-inputs
     b. Filter concepts NOT in usedConcepts for this theme
     c. For each unused concept × each winning persona:
        format = formatPool[index % 5]
        create entry { id, theme, concept, persona, format, status: "pending" }
  9. If no unused concepts found → create round with winners only

  --- Step C: Generate Copy for New Entries ---
  10. For each pending entry:
      a. Build image prompt (deterministic)
      b. Generate ad copy via Gemini (rate-limited at 1000ms)

  --- Step D: Save Outputs ---
  11. Combine winners + new entries into matrix
  12. Save matrix to round-{new}/matrix.json
  13. Copy ad-inputs.json snapshot to round-{new}/ad-inputs.json
  14. Update rounds index:
      index.currentRound = newRound
      index.rounds.push({ round: newRound, createdAt: ISO, description: "..." })
      saveRoundsIndex(index)
```

## Concept Tracking

The system tracks which concepts have been used across all rounds to avoid repeating the same creative:

```typescript
function getAllUsedConcepts(): Map<string, Set<string>> {
  const used = new Map<string, Set<string>>();
  for each round in rounds index:
    matrix = loadRoundMatrix(round)
    for each entry in matrix:
      used.get(themeId) → add(conceptId)
  return used
}
```

This means each theme × concept combination is only used once across all rounds. When all concepts for a winning theme are exhausted, the system reports this and moves on.

## Carried-Forward Winners

Winners are marked with metadata to track their lineage:

```typescript
{
  status: "approved",        // Already approved — skip review
  sourceRound: prevRound,    // Where this winner originated
  carriedForward: true,      // Flag for UI display and image generation skip
}
```

The image generation step skips entries where `carriedForward: true` (images already exist). The review gallery shows a badge for carried-forward entries.

## Full Optimization Loop

```
Round 1:
  ads:matrix → ads:generate → ads:review → ads:upload
  (wait for performance data to accumulate)
  ads:report → ads:select-winners → ads:new-round

Round 2:
  ads:generate (new concepts only) → ads:review → ads:upload
  (wait for performance data)
  ads:report → ads:select-winners → ads:new-round

Round 3: ...
```

Each round:
- Keeps winning ads (proven performers)
- Tests new concept variations for winning themes
- Drops underperformers automatically
- Progressively converges on the best-performing creative × audience combinations

## Initialising the First Round

The rounds system starts automatically when `ads:matrix` detects a `rounds-index.json` file. To initialise:

1. Create `output/ads/rounds/rounds-index.json`:
```json
{
  "currentRound": 1,
  "rounds": [{ "round": 1, "createdAt": "2025-01-01T00:00:00.000Z" }]
}
```

2. Run `npm run ads:matrix` — matrix saves to `round-1/matrix.json`
3. Continue with normal pipeline flow
