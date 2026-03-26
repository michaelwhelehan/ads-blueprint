---
name: ads-generate
description: Day-to-day ad generation workflow - fetch brand assets from a website, generate the creative matrix, produce images via Gemini + text compositing, review in the gallery, and upload to Meta.
argument-hint: [step]
allowed-tools: Bash, Read, Write, Grep, Glob, Agent, WebFetch, WebSearch, AskUserQuestion
---

# Ads Generation Workflow

Run the day-to-day ad generation pipeline after the project has been bootstrapped. This skill covers fetching/refreshing brand assets, generating the creative matrix, producing images, reviewing them, and uploading to Meta.

## Specs Reference

Read the relevant spec before executing each step:

- `specs/05-creative-matrix.md` — Matrix generation algorithm
- `specs/06-image-prompt-system.md` — Prompt templates for image and copy generation
- `specs/07-image-generation.md` — Gemini image generation with reference images
- `specs/08-text-compositing.md` — Sharp + opentype.js text rendering onto images
- `specs/09-review-gallery.md` — Express review server at localhost:3847
- `specs/10-meta-upload.md` — Meta Graph API upload pipeline
- `specs/14-asset-management.md` — Asset directory conventions and naming patterns

## How to Run

If the user provides a specific step as `$ARGUMENTS`, run only that step:
- `fetch-assets` — Step 1
- `matrix` — Step 2
- `generate` — Step 3
- `review` — Step 4
- `upload` — Step 5

If no step is provided, guide through the full workflow interactively, confirming before each step.

## Prerequisites Check

Before running any step, verify the project is bootstrapped:

1. Check that `project-config/brand.json` exists. If not, tell the user to run `/ads-bootstrap` first.
2. Check that `project-config/ad-inputs.json` exists (has themes and personas).
3. Check that `project-config/campaign-config.json` exists (needed for upload step).
4. Check that `package.json` exists and has the expected npm scripts (`ads:matrix`, `ads:generate`, `ads:review`, `ads:upload`).
5. Check that `.env` exists and has `GEMINI_API_KEY` set (needed for matrix and generate steps).

If any required file is missing, tell the user which file is missing and suggest running `/ads-bootstrap` or creating it manually.

---

## Step 1: Fetch Brand Assets (`fetch-assets`)

Scan a website URL and download brand images (logos, hero images, product photos) into the `assets/` directory structure. This step can be re-run to refresh assets at any time.

### Flow

1. Ask the user:
   > What website URL should I scan for brand assets? (Or type "skip" to skip asset fetching)

2. If a URL is provided:

   a. Use `WebFetch` to download the website HTML.

   b. Parse the HTML to extract image URLs and metadata:
      - `<meta property="og:image">` — Open Graph image (high priority, often a good hero image)
      - `<meta property="og:title">` and `<meta property="og:description">` — Brand context
      - `<link rel="icon">`, `<link rel="apple-touch-icon">` — Favicon/icon candidates
      - `<img>` tags where the `class`, `alt`, `id`, or `src` contains "logo" — Logo candidates
      - `<img>` tags where the `class`, `alt`, or `src` contains "hero", "banner", "feature" — Hero image candidates
      - `<img>` tags where the `class`, `alt`, or `src` contains "product", "item", "shop" — Product photo candidates
      - Large `<img>` tags in `<header>`, `<main>`, or early in the DOM — General brand imagery
      - CSS background-image URLs from inline styles
      - Resolve all relative URLs to absolute using the base URL

   c. If the site has navigation links to product pages, gallery pages, or about pages, consider fetching those with `WebFetch` too for additional images.

   d. Ensure the asset directories exist:
      ```bash
      mkdir -p assets/logos assets/fonts assets/brand-photos/downloaded assets/physical-items
      ```

   e. For each discovered image URL, use `Bash` with `curl` to download:
      ```bash
      curl -sL -o "assets/brand-photos/downloaded/{descriptive-name}.{ext}" "{image-url}"
      ```
      Name files descriptively based on their source:
      - `og-image.jpg` — from og:image meta tag
      - `logo-header.png` — logo found in header
      - `logo-favicon.png` — from apple-touch-icon or favicon
      - `hero-1.jpg`, `hero-2.jpg` — hero/banner images
      - `product-1.jpg`, `product-2.jpg` — product images
      - `brand-photo-1.jpg` — general brand imagery

   f. After downloading, verify each file is a valid image (not a redirect page or error):
      ```bash
      file assets/brand-photos/downloaded/*.{jpg,png,jpeg,webp} 2>/dev/null
      ```
      Delete any files that are not actual images (HTML pages, empty files, etc.).

   g. Report what was found:
      - List each downloaded image with its source context (og:image, logo, hero, etc.)
      - Show file sizes
      - Show any metadata extracted (title, description)

   h. Ask the user to review the downloaded images and guide them on next steps:
      > I downloaded {N} images to `assets/brand-photos/downloaded/`. Here is what I found:
      > {list of images with descriptions}
      >
      > To use these in ad generation:
      > - Move your best logo to `assets/logos/logo.png`
      > - Move product photos to `assets/physical-items/`
      > - Create theme directories in `assets/brand-photos/{theme-id}/` and add outfit/reference images
      >
      > Want me to help organize these? I can:
      > 1. Copy the best logo candidate to `assets/logos/logo.png`
      > 2. Move product-looking images to `assets/physical-items/`
      > 3. Create theme directories from your ad-inputs.json themes

   i. If the user agrees, help organize:
      - Read `project-config/ad-inputs.json` to get theme IDs
      - Create `assets/brand-photos/{theme-id}/` for each theme
      - Copy/move images based on user guidance
      - Rename files to match expected naming patterns (e.g. `logo.png`, `outfit-1.jpg`)

3. If user skips:
   - Verify asset directories exist, create if missing:
     ```bash
     mkdir -p assets/logos assets/fonts assets/brand-photos assets/physical-items
     ```
   - Show current asset inventory:
     ```bash
     echo "=== Logos ===" && ls -la assets/logos/ 2>/dev/null
     echo "=== Fonts ===" && ls -la assets/fonts/ 2>/dev/null
     echo "=== Brand Photos ===" && ls -laR assets/brand-photos/ 2>/dev/null
     echo "=== Physical Items ===" && ls -la assets/physical-items/ 2>/dev/null
     ```
   - Report what assets are present and what might be missing

### Asset Validation

After fetching or when skipping, validate the minimum assets needed for generation:

- **Fonts** (REQUIRED): Check `assets/fonts/` for `.ttf` files. If no fonts found, warn:
  > No font files found in `assets/fonts/`. Text compositing requires at least a heading font (Bold) and body font (Bold) in TTF format. Place your font files there before running `generate`.
- **Logo** (recommended): Check `assets/logos/` for `logo.{png,jpg,jpeg,svg}`. If missing, note that logo will be skipped in compositing.
- **Physical items** (recommended): If `assets/physical-items/` is empty, note that product reference images won't be sent to Gemini.

---

## Step 2: Generate Creative Matrix (`matrix`)

Generate the creative matrix — a Cartesian product of active themes x concepts x personas x formats, with AI-generated ad copy for each entry.

### Flow

1. Verify prerequisites:
   - `project-config/brand.json` exists
   - `project-config/ad-inputs.json` exists with themes and personas
   - `.env` has `GEMINI_API_KEY` set

2. Show what will be generated by reading `project-config/ad-inputs.json`:
   - Count active themes (apply seasonal filtering: themes with `seasonal.activeFrom` and `seasonal.activeTo` are only active if today's date falls within the window, using DD-MM format comparison)
   - Count concepts per active theme
   - Count personas
   - Calculate total entries: `active_themes * total_concepts * personas * 1` (one format per combination from the 4:1 feed/story cycling pool)
   - Report:
     > Matrix will contain approximately {N} entries:
     > - {X} active themes with {Y} total concepts
     > - {Z} personas
     > - Format distribution: ~80% feed (1080x1080), ~20% story (1080x1920)

3. Ask user to confirm:
   > Ready to generate the creative matrix? This will call Gemini to generate ad copy for each entry (rate-limited at 1 req/sec). Estimated time: ~{N} seconds.

4. Run the matrix generation:
   ```bash
   npm run ads:matrix
   ```

5. After completion, read the generated matrix file to show a summary:
   - Check for round-aware path first: `output/ads/rounds/round-{N}/matrix.json`
   - Fall back to legacy path: `output/ads/matrix.json`
   - Report:
     > Matrix generated with {N} entries:
     > - {X} feed format, {Y} story format
     > - Themes: {list of theme names}
     > - Personas: {list of persona names}
     > - All entries have status "pending" — ready for image generation.

6. If the command fails:
   - Check if `GEMINI_API_KEY` is set: `grep GEMINI_API_KEY .env`
   - Check if the npm script exists: `grep "ads:matrix" package.json`
   - Check for TypeScript errors: `npx tsc --noEmit 2>&1 | head -20`
   - Report the error and suggest fixes

---

## Step 3: Generate Images (`generate`)

Generate ad images via Gemini and composite text/branding onto them using Sharp and opentype.js.

### Flow

1. Verify prerequisites:
   - Matrix file exists (run `matrix` step first if not)
   - `.env` has `GEMINI_API_KEY` set
   - Font files exist in `assets/fonts/` (required for text compositing)

2. Read the matrix file and count pending entries:
   - Check round-aware path: `output/ads/rounds/round-{N}/matrix.json`
   - Fall back to legacy: `output/ads/matrix.json`
   - Count entries with `status: "pending"` and `carriedForward` not true
   - Report:
     > Found {N} pending entries to generate. Already generated: {M}. Already approved: {A}.

3. Ask user about scope:
   > How many images would you like to generate?
   > - Enter a number for a test batch (recommended: 3-5 for first run)
   > - Enter "all" to generate all {N} pending images
   > - Estimated time per image: ~5-10 seconds (rate-limited at 1 req per 2 seconds)

4. Run generation:
   ```bash
   # For limited batch:
   npm run ads:generate -- --limit={N}

   # For all:
   npm run ads:generate
   ```

5. Monitor progress — the command will log each image as it generates. After completion:
   - Re-read the matrix file
   - Count entries by status
   - Check the images directory for generated files:
     ```bash
     ls -la output/ads/rounds/round-*/images/ 2>/dev/null || ls -la output/ads/images/ 2>/dev/null
     ```
   - Report:
     > Generation complete:
     > - Generated: {N} new images
     > - Failed: {F} (if any)
     > - Total pending remaining: {P}
     > - Images saved to: {path}

6. If generation fails:
   - Check `GEMINI_API_KEY` validity
   - Check for font file issues: `ls assets/fonts/*.ttf`
   - Check for rate limiting errors in the output
   - If partial progress was made (progressive save), report how many succeeded before failure
   - Common errors:
     - "Font not found" — Check font file names match what the code expects. Read `src/ads/generation/generate-images.ts` for the expected font paths.
     - "API key invalid" — Verify `GEMINI_API_KEY` in `.env`
     - "Rate limit exceeded" — Wait a few minutes and re-run (already-generated images will be skipped)
     - "Image generation failed" — Gemini may refuse some prompts; the pipeline logs and continues

7. After a test batch, suggest:
   > Want to review these images before generating the rest? Run the review step to see them at http://localhost:3847.

---

## Step 4: Launch Review Gallery (`review`)

Start the local Express review server to approve or reject generated images.

### Flow

1. Check that generated images exist:
   ```bash
   ls output/ads/rounds/round-*/images/*.png 2>/dev/null || ls output/ads/images/*.png 2>/dev/null
   ```
   If no images found, tell user to run the `generate` step first.

2. Check if port 3847 is already in use:
   ```bash
   lsof -ti:3847
   ```
   If occupied, warn the user and offer to kill the existing process:
   > Port 3847 is already in use (PID {pid}). Want me to stop the existing process?

3. Launch the review gallery:
   ```bash
   npm run ads:review
   ```
   Run this in the background so the user can continue interacting.

4. Tell the user:
   > Review gallery is running at http://localhost:3847
   >
   > In the gallery you can:
   > - View all generated images as cards with theme/persona/concept tags
   > - Click any image for a full-size lightbox view
   > - Filter by theme, persona, concept, or status
   > - Approve or reject individual images
   > - Use "Approve All Visible" or "Reject All Visible" for bulk actions
   > - Delete and regenerate specific images (resets to pending)
   >
   > If you are in round mode, only the current round is editable. Previous rounds are read-only.
   >
   > When you are done reviewing, come back here and I will show you the approval stats.

5. When the user returns, read the matrix file and report status counts:
   > Review summary:
   > - Approved: {A}
   > - Rejected: {R}
   > - Generated (not yet reviewed): {G}
   > - Pending (not yet generated): {P}

6. Based on the results, suggest next steps:
   - If there are pending entries: "You have {P} entries still pending. Run `generate` to create images for them."
   - If all generated are reviewed: "All images reviewed. {A} approved and ready for upload."
   - If many rejected: "You rejected {R} images. You can delete them in the gallery and re-run `generate` to regenerate, or adjust your prompt template in `project-config/prompt-template.json` for different results."

---

## Step 5: Upload to Meta (`upload`)

Upload approved ad creatives to Meta Ads via the Graph API. All ads are created PAUSED.

### Flow

1. Verify prerequisites:
   - `project-config/campaign-config.json` exists
   - `.env` has the required Meta environment variables:
     ```bash
     grep -c "META_ACCESS_TOKEN" .env && grep -c "META_AD_ACCOUNT_ID" .env && grep -c "META_BUSINESS_ID" .env
     ```
   - If any Meta variables are missing, tell the user:
     > Missing Meta API credentials in `.env`. You need:
     > - `META_ACCESS_TOKEN` — System user token with `ads_management` permission
     > - `META_AD_ACCOUNT_ID` — Your ad account ID (auto-prefixed with `act_`)
     > - `META_BUSINESS_ID` — Your Business Manager ID

2. Read the matrix and count approved entries:
   - Report:
     > Found {A} approved ads ready for upload.
     > Campaign: {campaign name from campaign-config.json}
     > Budget: ${daily_budget}/day ({CBO or ABO})

3. Warn the user:
   > **Important**: All ads will be created as PAUSED in Meta Ads Manager. You will need to manually activate them after reviewing in Ads Manager.
   >
   > The upload will:
   > 1. Create or reuse a campaign
   > 2. Create ad sets per persona (with targeting from ad-inputs.json)
   > 3. Upload image creatives
   > 4. Create ads linking creatives to ad sets
   >
   > Budget is sent in cents (e.g. $50/day = 5000 cents).
   >
   > Ready to upload?

4. Run the upload:
   ```bash
   npm run ads:upload
   ```

5. After completion, re-read the matrix and report:
   > Upload complete:
   > - Uploaded: {U} ads (all PAUSED)
   > - Failed: {F} (if any)
   >
   > Next steps:
   > 1. Go to Meta Ads Manager to review and activate your ads
   > 2. After ads have been running, use `npm run ads:report last_7d` to check performance
   > 3. For multi-round optimization, use `npm run ads:select-winners` after collecting data

6. If upload fails:
   - Check Meta API token validity
   - Check ad account ID format
   - Look for rate limiting (Meta API rate limit is handled at 1500ms between calls)
   - Common errors:
     - "Invalid OAuth access token" — Token expired, generate a new one
     - "Ad account not found" — Check `META_AD_ACCOUNT_ID` (should be numbers only, `act_` prefix is added automatically)
     - "Page not found" — Verify `META_PAGE_ID` in `.env` or `campaign-config.json`

---

## Full Workflow (No Step Specified)

When no specific step is given, guide the user through the complete workflow:

1. Run the prerequisites check. Report project status.
2. Ask:
   > I can guide you through the full ad generation workflow:
   > 1. **Fetch assets** — Download brand images from your website
   > 2. **Matrix** — Generate the creative matrix with AI ad copy
   > 3. **Generate** — Create ad images via Gemini + text compositing
   > 4. **Review** — Launch the gallery to approve/reject images
   > 5. **Upload** — Upload approved ads to Meta (all PAUSED)
   >
   > Where would you like to start? (Enter a step number, or "1" to start from the beginning)
3. Run each step in sequence, confirming before each one.
4. After each step, ask if the user wants to continue to the next step or stop.

## Behaviour Notes

- Before each step, check what already exists — do not overwrite generated images or matrix entries without asking.
- If a matrix already exists with generated/approved entries, warn before regenerating:
  > A matrix already exists with {N} generated and {A} approved entries. Regenerating will overwrite this. Continue?
- Support round-aware paths: always check for `output/ads/rounds/rounds-index.json` first.
- When showing file paths, use the actual paths found on disk (round-aware or legacy).
- If the user says "skip" for any question, proceed with defaults or move to the next step.
- Report timings when possible (how long generation took, estimated time remaining).
- If an npm script fails, check for TypeScript compilation errors first with `npx tsc --noEmit`.
