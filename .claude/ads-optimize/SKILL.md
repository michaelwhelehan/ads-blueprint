---
name: ads-optimize
description: Run the iterative ad optimization workflow - select winners from performance data, create a new round with fresh concept variations, generate images, review, and upload to Meta.
argument-hint: [step]
allowed-tools: Bash, Read, Grep, Glob, Agent, TaskCreate, TaskUpdate
---

# Ad Optimization Workflow

Run the iterative ad optimization loop. Each round keeps proven winners, tests new concept variations for winning themes, and drops underperformers -- progressively converging on the best-performing creative x audience combinations.

The full workflow is:

1. **report** - Pull performance data from Meta for the current round
2. **select-winners** - Auto-select top 10 ads by CTR, write winners.json
3. **new-round** - Create next round (copy winners + generate new concept variations)
4. **generate** - Generate images for new pending entries
5. **review** - Open the review gallery to approve/reject
6. **upload** - Upload approved ads to Meta as a new campaign

## Specs Reference

Read these specs for implementation details:

- `specs/11-performance-reporting.md` -- Insights API, demographic breakdowns, HTML report
- `specs/12-multi-round-optimization.md` -- Winner selection, new round creation, concept tracking

## How to Run

If the user provides a specific step as an argument (`$ARGUMENTS`), run only that step. Otherwise, guide them through the full workflow interactively.

Before starting, always check the current round state:

```bash
cat output/ads/rounds/rounds-index.json 2>/dev/null || echo "No rounds index found"
```

If no rounds index exists, inform the user they need to run the initial pipeline first (`npm run ads:matrix` with rounds enabled) before optimization can begin.

### Step Details

**Step 1: Report** (`report`)

Ask the user for two things:

1. **Date range** -- Suggest `last_7d` as default. Available presets: `today`, `yesterday`, `last_3d`, `last_7d`, `last_14d`, `last_28d`, `last_30d`, `last_90d`.
2. **Campaign ID** -- Optional. If they provide one, pass it with `--campaign`. Otherwise the script reads from the current round's upload log automatically.

With campaign ID:

```bash
npm run ads:report last_7d -- --campaign <campaign_id>
```

Without campaign ID (reads from current round's upload log):

```bash
npm run ads:report last_7d
```

For a specific round:

```bash
npm run ads:report last_7d -- --round=<round_number>
```

After running, review the output and highlight:
- **Summary metrics**: Total spend, impressions, clicks, overall CTR, CPC, conversions
- **Top 5 ads by clicks** with their CTR and spend
- **Demographic insights**: Which age/gender segments, countries, and placements perform best
- **Interest/persona performance**: Which audience segments are delivering results

The script also generates an HTML report at `output/ads/rounds/round-{N}/report.html` and auto-opens it in the browser.

**Step 2: Select Winners** (`select-winners`)

```bash
npm run ads:select-winners -- last_7d
```

The user can specify a different date range if needed (e.g. `last_30d` for more data).

**How winner selection works:**
- Fetches ad-level insights from Meta for the current round's campaign
- Filters out ads with fewer than **100 impressions** (not enough data to judge)
- Extracts the entry ID from the ad name (strips "Ad - " prefix)
- Sorts all qualifying ads by **CTR descending**
- Takes the **top 10** winners
- Writes `winners.json` to the current round directory

After running, read the winners file and display a summary table showing each winner's entry ID, CTR, impressions, clicks, and spend:

```bash
cat output/ads/rounds/round-$(cat output/ads/rounds/rounds-index.json | python3 -c "import sys,json; print(json.load(sys.stdin)['currentRound'])")/winners.json
```

If no ads meet the minimum impressions threshold, suggest:
- Waiting longer for data to accumulate
- Using a wider date range (e.g. `last_14d` or `last_30d`)
- Checking that the campaign is active in Meta Ads Manager

**Step 3: New Round** (`new-round`)

```bash
npm run ads:new-round
```

**What this does:**
1. Reads the current round's `winners.json`
2. Creates a new round directory (`round-{N+1}`)
3. **Copies winners forward**: Each winning ad's image is copied to the new round's images directory, and its matrix entry is marked as `carriedForward: true` with `status: "approved"` (skips review and image generation)
4. **Finds untried concepts**: For each winning theme, scans all previous rounds to find concepts that haven't been used yet. Pairs untried concepts with winning personas to create new matrix entries with `status: "pending"`
5. **Generates ad copy**: Calls Gemini to generate headlines and bullet points for each new entry (rate-limited at 1000ms between calls)
6. Saves the combined matrix (winners + new entries) and updates `rounds-index.json`

After running, display:
- How many winners were carried forward
- How many new concept variations were created
- Which themes have remaining untried concepts
- Which themes have exhausted all concepts (if any)

If no untried concepts are found for any winning theme, inform the user that all concept variations have been tested. They may want to add new concepts to `project-config/ad-inputs.json` before creating another round.

**Step 4: Generate Images** (`generate`)

```bash
npm run ads:generate
```

This generates images only for **pending entries** -- carried-forward winners already have their images copied and are skipped. The process:
- Calls Gemini image API for each pending entry
- Composites text overlays (headline, bullets, CTA) using Sharp + opentype.js
- Saves images to `output/ads/rounds/round-{N}/images/`
- Progressive save: matrix is updated after each image generation

This step calls the Gemini image API and may take a while depending on the number of new entries. The Gemini image API is rate-limited at 2000ms between calls.

**Step 5: Review** (`review`)

```bash
npm run ads:review
```

Starts the local review gallery server at http://localhost:3847. Tell the user:
- Open http://localhost:3847 in their browser
- The gallery shows the current round with carried-forward winners badged separately
- Review each new ad: approve or reject
- Carried-forward winners are already approved and shown for reference
- When done reviewing, come back to the terminal

Wait for the user to confirm they've finished reviewing before proceeding to upload.

**Step 6: Upload** (`upload`)

```bash
npm run ads:upload
```

Uploads all approved ads (both carried-forward winners and newly approved entries) to Meta as a new campaign. The upload process:
- Creates a new campaign (PAUSED)
- Creates ad sets per persona (with CBO or ABO budget per campaign config)
- Uploads creative images and creates ad creatives
- Creates ads linking creatives to ad sets
- Saves upload log to `output/ads/rounds/round-{N}/upload-log.json`
- Updates `rounds-index.json` with the new campaign ID

After upload, show:
- The new campaign ID
- Total number of ads uploaded
- Remind the user that **all ads are created PAUSED** and need to be activated in Meta Ads Manager
- Suggest waiting at least 3-7 days for performance data to accumulate before running the next optimization round

## Key Files

- **Rounds index**: `output/ads/rounds/rounds-index.json` -- tracks current round number and metadata for all rounds
- **Current round matrix**: `output/ads/rounds/round-{N}/matrix.json` -- all entries for the round (winners + new)
- **Winners**: `output/ads/rounds/round-{N}/winners.json` -- selected winners with CTR, impressions, clicks, spend
- **Upload log**: `output/ads/rounds/round-{N}/upload-log.json` -- Meta campaign/ad set/ad IDs from upload
- **Performance report**: `output/ads/rounds/round-{N}/report.html` -- HTML report with images and demographics
- **Ad inputs snapshot**: `output/ads/rounds/round-{N}/ad-inputs.json` -- snapshot of themes/personas used
- **Round utilities**: `src/shared/utils/rounds.ts` -- helper functions for round detection and paths

## Behaviour

- Before each step, check the current round state by reading `output/ads/rounds/rounds-index.json`
- If running the full workflow, confirm with the user before proceeding to each step
- If `select-winners` finds no ads meeting the minimum impressions threshold (100), suggest waiting or adjusting the date range
- If `new-round` finds no untried concepts for any winning theme, inform the user that all concepts have been exhausted and suggest adding new concepts to `project-config/ad-inputs.json`
- After `upload`, remind the user that ads are PAUSED and need activation in Meta Ads Manager
- Track concept usage across rounds: each theme x concept combination is only ever used once across all rounds to avoid repeating creatives
- If any step fails, read the error output carefully and suggest fixes before retrying
- Do not skip steps in the full workflow without user confirmation -- each step depends on the previous one completing successfully
