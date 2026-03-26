# 09 — Review Gallery

A local Express server that provides a web UI for reviewing, approving, and rejecting generated ad images.

## npm Script

```bash
npm run ads:review
# Opens at http://localhost:3847
```

## Source File

`src/ads/review/gallery-server.ts`

## Server Setup

```typescript
import express from "express";
const PORT = 3847;
const app = express();
app.use(express.json());
app.listen(PORT, () => console.log(`Gallery: http://localhost:${PORT}`));
```

## API Routes

### Read Operations

| Route | Method | Description |
|---|---|---|
| `GET /` | GET | Gallery HTML page (single-page app) |
| `GET /api/rounds` | GET | Returns rounds index JSON (null if legacy mode) |
| `GET /api/round/:num/matrix` | GET | Returns matrix for specific round |
| `GET /api/matrix` | GET | Returns legacy matrix (pre-rounds) |
| `GET /api/stats` | GET | Returns status counts `{ pending, generated, approved, rejected, uploaded }` |
| `GET /rounds/:num/image/:filename` | GET | Serves an image from a round's images directory |
| `GET /image/:filename` | GET | Serves an image from legacy images directory |

### Write Operations (current round only)

| Route | Method | Body | Description |
|---|---|---|---|
| `POST /api/status` | POST | `{ id: string, status: string }` | Update entry status (approve/reject) |
| `POST /api/delete` | POST | `{ id: string }` | Delete entry image and reset to pending |

## Status Update Logic

```
POST /api/status { id, status }:
  1. Load matrix (current round or legacy)
  2. Find entry by id
  3. Set entry.status = status ("approved" or "rejected")
  4. Save matrix immediately
  5. Return { ok: true }
```

## Delete Logic

```
POST /api/delete { id }:
  1. Load matrix
  2. Find entry by id
  3. If entry.imagePath exists:
     - Delete the image file from disk
     - Clear entry.imagePath
  4. Set entry.status = "pending"
  5. Save matrix
  6. Return { ok: true }
```

## UI Features

The gallery is a single HTML page served inline from the server (no separate frontend build).

### Round Navigation
- Dropdown selector showing all rounds from rounds-index.json
- Defaults to current (latest) round
- Older rounds are **read-only** — approve/reject/delete buttons hidden
- Read-only banner displayed for non-current rounds

### Card Display
Each ad entry shows:
- Thumbnail image (clickable for lightbox full-size view)
- Entry ID
- Theme, persona, concept as tags
- Hook text and bullet points
- Status badge (colour-coded border)
- Carried-forward winner badge (if `carriedForward: true`)

### Filtering
Toolbar with dropdown filters for:
- Theme
- Persona
- Concept
- Status (all, pending, generated, approved, rejected)

Filters are populated dynamically from the current matrix.

### Bulk Operations
- **Approve All Visible** — approves all visible (filtered) entries
- **Reject All Visible** — rejects all visible entries

### Visual Indicators
- `approved` → green border
- `rejected` → red border, 50% opacity
- `generated` → default border
- `pending` → default border (no image)

### Lightbox
Click any image to view full-size in a modal overlay. Press Escape or click outside to close.

## Round Awareness

```
isRoundMode():
  return rounds-index.json exists

getCurrentRound():
  return loadRoundsIndex().currentRound

// Matrix loading:
if request for round N:
  load from output/ads/rounds/round-{N}/matrix.json
else if legacy:
  load from output/ads/matrix.json

// Actions only allowed for:
if round === getCurrentRound()
```

## Data Persistence

All changes are written directly back to the matrix JSON file immediately. There is no separate database — the matrix file is the source of truth.

## Styling

Dark theme (background `#0a0a0a`) designed for reviewing visual content. Responsive grid layout that adapts to screen width. Cards use `grid-template-columns: repeat(auto-fill, minmax(340px, 1fr))`.
