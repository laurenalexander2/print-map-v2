# Print Map (AIC Prints) — v2 https://print-map.vercel.app/

A local-run, NFT-style visual map of **Art Institute of Chicago prints**, rendered as a dense, non-overlapping wall of thumbnails arranged by **similarity** and streamed directly from AIC’s IIIF servers.

This project is intentionally split into **two decoupled parts**:

1. **A dataset builder** that produces a single `points.json`
2. **A frontend map app** that renders that file

Once `points.json` exists, the frontend has **no backend dependency**.

---

## What this project is

A zoomable, pannable wall of thousands of real museum print images, laid out so **similar works are near each other**, with no overlaps, no placeholders, and no locally stored images.

---

## Architecture overview

```
AIC API / AIC Dump
        │
        ▼
aic-points-builder  (Python)
  • fetch prints
  • text embeddings (CLIP)
  • UMAP → 2D
  • locality-preserving grid
  • write points.json
        │
        ▼
print-map frontend (React + Vite)
  • load points.json
  • render dense image wall
  • pan / zoom
  • side detail card
  • IIIF image streaming
```

---

## Key design decisions

### 1. Dataset is precomputed
All expensive work (API calls, embeddings, dimensionality reduction, layout) happens **once** in the builder.  
The frontend only reads static JSON.

### 2. No images are stored locally
All thumbnails and detail images are streamed from **AIC IIIF** endpoints.  
Only URLs are stored in `points.json`.

### 3. Text-first layout (v1)
v1 uses **text embeddings only** (title, artist, medium, date, origin).  
Image embeddings are deferred to v3.  
The UI automatically disables Visual mode when unavailable.

### 4. NFT-style dense wall (no overlaps)
Raw UMAP coordinates overlap.  
To fix this without destroying neighborhood structure, points are:
- sorted using a space-filling curve (Morton / Z-order) over UMAP space
- assigned to a dense grid

Result:
- All images touch
- Similar items remain adjacent
- No jitter, no collisions

### 5. Hard separation of concerns
The map app never talks to AIC.  
The builder never knows about React.  
`points.json` is the contract.

---

## The dataset: `points.json`

### Required location

```
print-map/
└── frontend/
    └── public/
        └── data/
            └── points.json
```

### Schema (simplified)

```json
{
  "meta": {
    "source": "Art Institute of Chicago API",
    "count": 2000,
    "generated_at_unix": 1700000000,
    "filters": {
      "prints_only": true,
      "public_domain": true,
      "has_image": true,
      "grid_size": [60, 34]
    }
  },
  "points": [
    {
      "id": 89503,
      "title": "Under the Wave off Kanagawa",
      "artist": "Katsushika Hokusai",
      "date": "1830/33",
      "medium": "Color woodblock print; oban",
      "origin": "Japan",
      "thumb_url": "...",
      "detail_url": "...",
      "museum_url": "...",
      "coords_text": [0.41, 0.72],
      "coords_visual": null,
      "grid": [12, 19]
    }
  ]
}
```

If `grid` exists, the frontend uses it and **overlap is impossible**.

---

## Dataset builder (separate repo)

### What it does
- Fetches **only prints** with images
- Handles AIC pagination limits safely
- Builds curated text embeddings (CLIP)
- Projects to 2D with UMAP
- Converts continuous space to a dense grid
- Writes a single `points.json`

### Data sources
- **AIC API** (fast iteration, page ≤ 10)
- **AIC full data dump** (recommended for full corpus)

### Example (API)

```bash
python build_points.py   --source api   --max_items 2000   --embeddings text   --grid_layout text   --out points.json
```

### Example (dump)

```bash
python build_points.py   --source dump   --dump_root ~/aic_dump/json   --max_items 50000   --embeddings text   --grid_layout text   --out points.json
```

---

## Frontend (print-map)

### Stack
- React
- TypeScript
- Vite

### Run instructions

```bash
cd print-map/frontend
npm install
npm run dev
```

Open the printed local URL (usually `http://localhost:5173`).

---

## Expected behavior (v1)

- Dense wall of touching thumbnails
- No overlaps
- Pan + zoom enabled
- Clicking opens a side detail card
- Text mode enabled
- Visual mode disabled (until v3)

---

## Known constraints

- No runtime API calls
- No image embeddings yet
- Grid slightly quantizes exact UMAP distances (by design)

---

## Future work (v3)

- Image embeddings + second grid
- Animated reflow between grids
- Progressive loading
- Optional WebGL renderer

---

## Core principle

> **Everything revolves around `points.json`.**

Build it once, drop it in, and the UI is deterministic.
