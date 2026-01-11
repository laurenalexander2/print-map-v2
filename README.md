# Print Map (AIC Prints) — v2 

Check it out -> https://print-map.vercel.app/

Code: https://drive.google.com/drive/folders/1cQR2dP4bL_aVMTDX9bBd4297sHz9a5r7?usp=drive_link

## From artwork to position on the map (semantic → spatial pipeline)

Every print rendered on the map passes through **four representational stages**. Each stage deliberately changes *what information is preserved* and *what is discarded* in order to make the next stage possible.

```
Artwork metadata
   ↓
Natural-language string
   ↓
512-dimensional semantic vector
   ↓
2-dimensional manifold (UMAP)
   ↓
Discrete grid position (renderable layout)
```

Understanding this pipeline is essential to understanding what the map does — and what it does *not* do.

---

### 1. Artwork → natural language (semantic surface)

Each artwork is first reduced to a single, ordered text string:

```
"{title}. {artist}. {medium}. {date}. {origin}."
```

This string is the **only semantic input** to the system.

Design intent:

* It encodes meaning using human language rather than hand-engineered features.
* Field order is fixed so the model sees consistent structure.
* Missing fields are omitted to avoid injecting noise.
* The string is intentionally concise; verbosity does not improve semantic geometry.

At this stage, the artwork exists purely as **language**, with no notion of distance, similarity, or layout.

---

### 2. Natural language → 512-dimensional vector (semantic geometry)

The text string is passed through the **CLIP text encoder**, producing a vector in ℝ⁵¹².

This step is often the most opaque, but conceptually it does one thing:

> It embeds the *meaning* of the sentence into a high-dimensional space where semantic similarity corresponds to geometric proximity.

Key properties of this space:

* Individual dimensions have **no human-interpretable meaning**.
* Meaning is distributed across all 512 dimensions.
* Two vectors being close (high cosine similarity) means the texts are conceptually similar, even if they share no words.
* Medium, era, artist, subject matter, and tone all influence position simultaneously.

This is not classification and not clustering.
It is **continuous semantic geometry**.

After this step:

* Every artwork is a point in a dense semantic field.
* Local neighborhoods are meaningful.
* The space is impossible to visualize or render directly.

This vector space is the **semantic ground truth** of the system.

---

### 3. 512-D → 2-D (manifold projection with UMAP)

UMAP projects the 512-dimensional semantic space into two dimensions.

What UMAP preserves:

* **Local relationships**: points that were near each other in 512-D tend to remain near in 2-D.
* Neighborhood structure and cluster coherence.

What UMAP discards:

* Absolute distances
* Global geometry
* Metric interpretability

Important consequence:

> The 2-D output should be read as a **topological surface**, not a precise map.

UMAP answers only one question:

* “Which works are semantically close to each other?”

It does not solve layout problems such as overlap, spacing, or visual readability.

The result is a **continuous semantic manifold**, not a renderable layout.

---

### 4. 2-D → grid position (discrete, renderable layout)

UMAP coordinates overlap heavily when thousands of points are rendered.

To make the map usable, the system converts the 2-D manifold into a **dense, overlap-free grid**:

1. UMAP coordinates are normalized to a unit square.
2. Each point is assigned a **Morton (Z-order) index**, which linearizes 2-D space while preserving locality.
3. Points are sorted by this index.
4. They are assigned sequentially to grid cells.

This step:

* eliminates overlaps,
* guarantees deterministic placement,
* preserves local semantic continuity,
* sacrifices global geometric meaning.

The result is not a faithful geometric projection of UMAP.
It is a **space-filling ordering of semantic similarity**.

---

### What the map actually represents

When a user pans the map, they are not navigating raw UMAP space.

They are navigating a **discrete tiling** whose adjacency relationships were induced by semantic similarity in language space.

Two prints are neighbors on the screen because:

* their textual descriptions were close in 512-D semantic space,
* that proximity survived UMAP,
* and their relative order was preserved by the space-filling curve.

This chain of transformations is what allows the frontend to remain:

* static,
* deterministic,
* backend-free,
* and scalable to tens of thousands of works.

---


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

### 3. Text-first layout (v2)
v2 uses **text embeddings only** (title, artist, medium, date, origin).  
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

## Expected behavior (v2)

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
