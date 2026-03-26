---
name: ads-bootstrap
description: Set up a new Meta Ads pipeline from scratch - scan a website for brand assets, configure brand identity, define ad themes and personas, generate all config files, and scaffold the project using the ads-blueprint specs.
argument-hint: [step]
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, Agent, WebFetch, WebSearch, AskUserQuestion
---

# Ads Pipeline Bootstrap

Set up a complete Meta Ads pipeline from scratch. This skill walks the user through an interactive setup process, generates all configuration files, and scaffolds the project.

## Specs Reference

All implementation specs are in `output/ads-blueprint/specs/`. Read the relevant spec before implementing each module:

- `01-project-setup.md` — Dependencies, tsconfig, directory structure
- `02-brand-config.md` — Brand identity schema
- `03-ad-inputs.md` — Themes, concepts, personas
- `04-campaign-config.md` — Meta campaign configuration
- `05-creative-matrix.md` — Matrix generation algorithm
- `06-image-prompt-system.md` — Prompt templates
- `07-image-generation.md` — Gemini image generation
- `08-text-compositing.md` — Sharp + opentype.js text rendering
- `09-review-gallery.md` — Express review server
- `10-meta-upload.md` — Meta Graph API integration
- `11-performance-reporting.md` — Performance analytics
- `12-multi-round-optimization.md` — Rounds system
- `13-shared-utilities.md` — Env, paths, rate limiting, rounds
- `14-asset-management.md` — Asset directory structure

## How to Run

If the user provides a specific step as `$ARGUMENTS`, run only that step. Otherwise, guide them through the full workflow interactively, confirming before each step.

## Step 1: Brand Asset Collection (`scan`)

Ask the user:
> Do you have a website URL I can scan for brand images and information? Or would you prefer to manually place images in the assets folder?

### If URL provided:

1. Use `WebFetch` to download the website HTML
2. Parse the HTML to identify:
   - `<meta property="og:image">` — Open Graph image
   - `<meta property="og:title">` and `<meta property="og:description">` — Brand name and tagline candidates
   - `<meta name="description">` — Description fallback
   - `<title>` — Title fallback
   - `<link rel="icon">` and `<link rel="apple-touch-icon">` — Favicon/icons
   - `<img>` tags with "logo" in class, alt, or src — Logo candidates
   - Large `<img>` tags — Hero images and product photos
   - Links to social media profiles (Facebook, Instagram) — for page ID hints
   - CSS custom properties or inline styles with colour values
3. For each discovered image URL:
   - Use `Bash` with `curl` to download to `assets/brand-photos/downloaded/`
   - Name files descriptively (e.g. `og-image.jpg`, `logo-from-header.png`, `hero-1.jpg`)
4. Report what was found:
   - List downloaded images with their source (og:image, logo, hero, etc.)
   - Show extracted metadata (title, description, colours)
   - Pre-fill brand config values from metadata
5. Ask user to review and confirm which images to keep

### If manual:

1. Create the assets directory structure:
   ```
   assets/logos/
   assets/fonts/
   assets/brand-photos/
   assets/physical-items/
   ```
2. Tell the user:
   - Place your logo in `assets/logos/` (name it `logo.png` or `logo.jpg`)
   - Place heading and body font TTF files in `assets/fonts/`
   - Place product photos in `assets/physical-items/`
   - Brand photos can be organized by theme later
3. Wait for user confirmation that they've placed their files

## Step 2: Brand Configuration (`brand`)

Ask the user these questions (pre-fill from website scan if available):

1. **Brand name**: "What is your brand name?"
2. **Tagline**: "What is your tagline or one-sentence value proposition?"
3. **Colors**: "What are your brand colours? I need hex codes for: primary, secondary, accent, background, and text colours."
   - If colours were detected from the website, suggest them
   - If user isn't sure, offer to pick a palette based on their logo/website
4. **Typography**: "What fonts do you use? I need a heading font and a body font name."
   - Remind them to place .ttf files in `assets/fonts/`
   - Suggest popular free font pairings if they're unsure
5. **Selling points**: "What are your top 4-6 key selling points or value propositions?"
6. **Brand voice**:
   - "What tone words describe your brand? (e.g. playful, professional, bold, warm)"
   - "Describe your brand's writing style in a sentence"
   - "What should your brand NEVER say or do in ads?"
   - Always include "Never mention AI or that content is AI-generated" in the avoid list

Generate `project-config/brand.json` using the schema from spec 02. Validate against `BrandConfigSchema` mentally before saving.

Show the generated config and ask for confirmation.

## Step 3: Target Audience / Personas (`personas`)

Ask the user:

> Describe 2-3 target audience personas. For each persona, I need:
> - A name/ID (e.g. "young-professionals")
> - A description of who they are
> - 2-3 pain points they have
> - Age range (min/max)
> - Gender targeting (optional — all, male, female)
> - Interest keywords for Meta targeting
> - Geographic targeting (which countries)

For Meta interest targeting, help the user by:
- Suggesting relevant interest categories based on their product
- Noting that exact Meta interest IDs will need to be looked up later via the Targeting Search API
- Using placeholder IDs (e.g. `"id": "TODO_LOOKUP"`) if exact IDs aren't available yet

Start building `project-config/ad-inputs.json` with the personas array.

## Step 4: Ad Themes & Concepts (`themes`)

Ask the user:

> Now let's create 2-4 ad themes. Each theme is a visual world for your ads.
>
> For each theme, I need:
> - A name/ID (kebab-case, e.g. "summer-vibes")
> - A display name
> - A description (this guides the AI's image generation)
> - Whether it's seasonal (and if so, active dates in DD-MM format)
> - 2-3 concept variations, each with:
>   - A concept ID (kebab-case)
>   - A scene description (detailed visual brief for image generation)
>   - A hook/headline (max 40 chars, punchy)
>   - 3 bullet points (key benefits)

Guide the user:
- Themes should be visually distinct from each other
- Scene descriptions should be detailed enough for AI image generation (describe the setting, people, mood, lighting)
- Hooks should be attention-grabbing and reference the product
- Bullets should address persona pain points

Complete `project-config/ad-inputs.json` with the themes array alongside the personas.

Show the complete file and ask for confirmation.

## Step 5: Campaign Configuration (`campaign`)

Ask the user:

1. **Campaign name**: "What should this campaign be called in Meta Ads Manager?"
2. **Objective**: Present options:
   - Sales (OUTCOME_SALES) — optimise for purchases/conversions *(recommended for e-commerce)*
   - Leads (OUTCOME_LEADS) — optimise for lead form submissions
   - Traffic (OUTCOME_TRAFFIC) — optimise for link clicks
   - Awareness (OUTCOME_AWARENESS) — optimise for reach
3. **Daily budget**: "What is your daily ad spend budget in dollars?"
4. **Budget strategy**:
   - CBO (recommended) — Meta auto-distributes budget across audiences
   - ABO — you control budget per audience segment
5. **Facebook Page ID**: "What is your Facebook Page ID? (Find in Page Settings → About)"
6. **Instagram**: "Do you have an Instagram Business Account connected? If so, what's the account ID?"
7. **Website URL**: "What is your website URL? (used as the ad landing page)"
8. **Tracking**:
   - "Do you have a Meta Pixel or Conversions API set up?"
   - If yes: "What is the Pixel/Dataset ID?"
   - "What conversion event do you want to optimise for? (e.g. Purchase, AddToCart, Lead)"

Generate `project-config/campaign-config.json` using the schema from spec 04.

## Step 6: Project Scaffolding (`scaffold`)

Now create the actual project code by reading the specs and implementing each module:

1. Read specs 01, 13 (project setup + shared utilities)
2. Create the directory structure from spec 01
3. Implement shared utilities:
   - `src/shared/utils/env.ts` (spec 13)
   - `src/shared/utils/paths.ts` (spec 13)
   - `src/shared/utils/rate-limit.ts` (spec 13)
   - `src/shared/utils/rounds.ts` (spec 13)
4. Implement types: `src/shared/types/ads.ts` (spec 03)
5. Implement brand config: `src/shared/brand/brand-config.ts` (spec 02)
6. Implement matrix generation (specs 05, 06):
   - `src/ads/matrix/prompt-builder.ts`
   - `src/ads/matrix/generate-matrix.ts`
7. Implement image generation (specs 07, 08):
   - `src/ads/generation/generate-images.ts`
8. Implement review gallery (spec 09):
   - `src/ads/review/gallery-server.ts`
9. Implement Meta upload (spec 10):
   - `src/ads/upload/meta-api.ts`
   - `src/ads/upload/meta-uploader.ts`
10. Implement reporting (spec 11):
    - `src/ads/reporting/performance.ts`
11. Implement round scripts (spec 12):
    - `scripts/select-winners.ts`
    - `scripts/new-round.ts`
12. Create `.env.example`, `.gitignore`
13. Run `npm install`

After scaffolding, verify the build: `npx tsc --noEmit`

## Step 7: First Run (`run`)

Guide the user through their first pipeline run:

1. **Create .env**: "Copy `.env.example` to `.env` and fill in your API keys"
   - Verify GEMINI_API_KEY is set
   - Verify META_ACCESS_TOKEN is set (if they plan to upload)

2. **Generate matrix**:
   ```bash
   npm run ads:matrix
   ```
   Show the matrix summary (number of entries, themes, personas)

3. **Generate a few test images**:
   ```bash
   npm run ads:generate -- --limit=3
   ```
   Start with 3 to validate the prompt and compositing look good

4. **Review**:
   ```bash
   npm run ads:review
   ```
   Tell user to open http://localhost:3847, review the 3 test images, approve/reject

5. **If images look good, generate the rest**:
   ```bash
   npm run ads:generate
   ```

6. **Upload to Meta** (when ready):
   ```bash
   npm run ads:upload
   ```
   Remind: all ads are created PAUSED — activate in Meta Ads Manager

7. **After ads have run, check performance**:
   ```bash
   npm run ads:report last_7d
   ```

## Behaviour Notes

- Before each step, check what already exists (don't overwrite existing configs without asking)
- If user says "skip" for any question, use sensible defaults
- Validate all generated JSON configs against their schemas
- Save configs incrementally — don't wait until the end
- If the user provides a step argument, jump directly to that step
- Show previews of generated configs and ask for confirmation before saving
- When implementing code (step 6), read the relevant spec first, then implement following the types and algorithms exactly
