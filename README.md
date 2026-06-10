# FixFirst Edge

[![CI](https://github.com/Ridwannurudeen/fixfirst-edge/actions/workflows/ci.yml/badge.svg?branch=master)](https://github.com/Ridwannurudeen/fixfirst-edge/actions/workflows/ci.yml)

**Demo:** [2-minute walkthrough on YouTube](https://youtu.be/5-cMMQXlYo0?si=uo1Q_DZjigILzCZI) · **Live landing page:** [edge.gudman.xyz](https://edge.gudman.xyz)

## What This Is

**Offline maintenance assistant for factory technicians.** Runs entirely on a laptop—no internet needed. You search by photo, error code, voice note, or manual text. System returns three things: the relevant manual page, a past similar incident (with fix + downtime), and the replacement part number. All traceable. No AI hallucination.

**Built for Actian VectorAI DB Challenge.** Uses all three required features: named vectors (text + image + voice in one collection), filtered search (error codes, machine models, part numbers), and hybrid RRF fusion (dense embeddings + metadata-filtered retrieval combined). Runs 100% offline on a 16 GB laptop, CPU-only. No outbound calls in diagnosis path. Response time: **~850 ms median, ~1100 ms at 95th percentile** (see [Benchmark](#benchmark)).

---

## Why This Matters

**Problem:** Industrial sites have bad connectivity. Technicians near broken machines often have no signal. Cloud "AI helpers" are also unauditable—you can't trace an LLM's answer back to the actual manual or the actual past incident.

**Solution:** Everything local. Everything traceable. Search → returns linked evidence from three sources (manual + past incident + parts DB). Each line points to a real row in the database.

Example output:
> **Manual:** conveyor_manual.pdf p.42 — "Error E04 indicates thermal overload on drive motor…"
>
> **Similar past incident:** CX-200 machine, same error. Fixed by replacing thermal overload relay and re-torquing motor mounts. Downtime: 45 min.
>
> **Part to order:** OL-E04-R (Thermal Overload Relay)

---

## How It Uses Actian VectorAI DB

Three features. Each solves a real problem:

### 1. Named Vectors — One collection, three embedding spaces

Single `incidents` collection holds three vector types side-by-side:

| Vector | Model | Dims | What it searches |
|---|---|---|---|
| `text_vec` | BAAI/bge-small-en-v1.5 | 384 | Manual text, incident descriptions, error codes |
| `image_vec` | sentence-transformers/clip-ViT-B-32 | 512 | Photos of damaged parts, schematics |
| `audio_text_vec` | bge-small-en (over whisper transcripts) | 384 | Voice notes from technicians |

No separate collections. No joining across stores. One document can carry all three vectors. Search by text, image, or voice—all hit the same collection. See [`backend/app/db.py`](backend/app/db.py) — `init_collection()` builds the schema, `upsert()` accepts any mix of vectors.

### 2. Filtered Search — Keyword metadata narrows results

Every query filtered by: machine type, model number, error code, part number, fault severity, doc type. Filters are indexed. Means a part recommendation for a "CX-200 with error E04" only returns parts that fit that exact machine and code. See [`backend/app/db.py:_build_filter`](backend/app/db.py) and [`backend/app/services/search_service.py`](backend/app/services/search_service.py).

### 3. Hybrid Fusion (RRF) — Dense embeddings + exact metadata match in one query

Text search runs **reciprocal rank fusion** across two lanes:
- Dense ANN on `text_vec` (top-50 dense hits)
- Actian-native lane: re-runs ANN with strict metadata filters (`fault_code`, `model_no`, `part_no`) extracted from query

At ingest, manuals/incidents/parts backfill those identifiers into indexed fields. RRF merges both lanes (k=60). This matters: error code `E04` and part ID `CX-200` get boosted by exact filters, while symptom phrases like "motor tripped" still match via dense embeddings. See [`backend/app/db.py:search_hybrid`](backend/app/db.py).

---

## How Diagnosis Works Across Modalities

`/api/diagnose` returns three things—manual section, past incident, candidate part—regardless of input type. Here's the flow per modality:

### Text query
Hybrid RRF directly. Dense ANN + identifier-filtered ANN, both lanes in one collection. Example: "E04 motor overload on CX-200" → hits manual chunk + incident + part in one fused pass.

### Voice query
Transcribed locally (faster-whisper, tiny.en model, CPU) to text. Same hybrid path as text. Also fuses `audio_text_vec` ANN so prior voice notes surface even if wording differs.

### Image query
`image_vec` ANN (CLIP space) finds nearest schematic or tagged incident photo. Hit's metadata (`model_no`, `fault_code`, `machine_type`) becomes seed for text hybrid path. Text hybrid then fetches manual + incident + part.

**Why image→text bridge?** Pure image diagnosis is weak. Two schematics look similar but belong to different faults. Parts are ID'd by number, not pixels. Real signal is the **metadata filed with the image** (`model_no`, `fault_code`, technician notes). Actian's filters let one photo match unlock the full case graph (manual/incident/part). See [`backend/app/services/search_service.py:diagnose`](backend/app/services/search_service.py).

This is why filtered search is core, not decoration. It's the glue turning three separate vector spaces into one cross-modal case graph.

---

## Why Offline-Only

**Connectivity:** Plants have poor signal. Techs near machines often disconnected.

**Auditability:** Cloud LLMs generate untrackable prose. Technician can't verify against actual manual. FixFirst returns structured evidence—every line traceable to a manual page or a past incident row.

**Data egress:** Many industrial sites forbid sending photos/error logs to cloud.

**Reliability:** Works even if cloud API is down or slow.

---

## Architecture

```
┌─────────────────────────┐        ┌─────────────────────────────────┐
│  Next.js 14 (frontend)  │  HTTP  │  FastAPI (backend)              │
│  UploadZone, SearchBar, │ ─────► │  /api/ingest/*                  │
│  FilterPanel,           │        │  /api/search/*                  │
│  DiagnosePanel,         │        │  /api/diagnose                  │
│  OfflineBanner          │        │  /api/incident/save             │
└─────────────────────────┘        └──────────────┬──────────────────┘
                                                  │
                      ┌───────────────────────────┼───────────────────────────┐
                      ▼                           ▼                           ▼
          ┌─────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────┐
          │  pipelines/         │   │  services/              │   │  db.py              │
          │  text_embedder      │   │  ingest_service         │   │  init_collection    │
          │  image_embedder     │   │  search_service         │   │  upsert             │
          │  audio_transcriber  │   │  diagnose_service       │   │  search_text/image/ │
          │  pdf_chunker        │   │                         │   │   audio/hybrid      │
          │  csv_loader         │   │                         │   │  health             │
          └─────────┬───────────┘   └───────────┬─────────────┘   └──────────┬──────────┘
                    │                           │                            │
                    ▼                           ▼                            ▼
          bge-small, CLIP-ViT-B-32,    builds metadata,           gRPC to Actian VectorAI DB
          whisper tiny.en              calls embedders + db       :50051 (Docker)
          (cached locally, CPU)                                   collection: `incidents`
                                                                  named vectors: 3
                                                                  filters: 6 keyword fields
                                                                  hybrid: RRF(ANN, identifier-filtered ANN)
```

### Stack

- **DB**: Actian VectorAI DB (Docker)
- **Backend**: Python 3.11/3.12, FastAPI, pdfplumber, sentence-transformers, faster-whisper
- **Frontend**: Next.js 14 (App Router), TypeScript, Tailwind CSS
- **Models**: All local, CPU-only, ~1.3 GB cached on first run

---

## Quickstart

### Prerequisites

- Docker
- Python 3.11 or 3.12
- Node 20+
- ~2 GB disk for model cache (first run)

### 1. Start Actian

```bash
docker compose up -d actian
```

Or standalone:

```bash
docker run -d --name vectoraidb -p 50051:50051 \
  --restart unless-stopped williamimoh/actian-vectorai-db:latest
```

Full stack one command (all services + auto-build):

```bash
docker compose --profile full up --build
```

Landing page (static marketing, hosted at [edge.gudman.xyz](https://edge.gudman.xyz)):
- `.github/workflows/deploy-landing-pages.yml` publishes mirror to GitHub Pages as fallback
- Cover asset: `landing/cover.svg`

### 2. Install backend

```bash
cd backend
python3.11 -m venv .venv && source .venv/bin/activate
pip install /path/to/actian_vectorai-0.1.0b2-py3-none-any.whl  # Actian beta wheel first
pip install -r requirements.txt
```

**Note:** Requires Python 3.11 or 3.12. SDK wheel must install before `requirements.txt` (requirements.txt pins `actian-vectorai==0.1.0b2`).

### 3. Seed data

Place files in `data/raw/`:

```
manuals/*.pdf               → equipment docs
images/*.{jpg,png}          → damage photos
voice/*.wav                 → voice notes (16 kHz mono)
incidents.csv               → columns: id, machine_type, model_no, fault_code, severity, symptom, fix_applied, downtime_min, parts_used
parts.csv                   → columns: part_no, name, machine_type, model_no
error_codes.csv             → columns: fault_code, machine_type, description
```

Sample fixtures in [`data/fixtures/`](data/fixtures/). Quick start:

```bash
cp data/fixtures/*.csv data/raw/
```

Generate full demo corpus (3 PDFs, 6 images, 5 voice notes) offline:

```bash
pip install reportlab pillow pyttsx3
python scripts/gen_demo_assets.py
```

Outputs → `data/raw/manuals/`, `data/raw/images/`, `data/raw/voice/` (no models, no network).

### 4. Run backend + ingest

```bash
cd backend
uvicorn app.main:app --host 127.0.0.1 --port 8000 &
PYTHONPATH=. python scripts/bulk_ingest.py
```

### 5. Run frontend

```bash
cd frontend
npm install
npm run dev
```

Open http://localhost:3000

Frontend defaults to `http://localhost:8000` backend. For split deployments, set `NEXT_PUBLIC_API_URL` env var. Backend allows CORS from `localhost:3000` and `127.0.0.1:3000` by default. Override with `CORS_ORIGINS`.

### 6. Verify offline

```bash
cd backend
python scripts/verify_offline.py
```

Tests `/api/health` + full `/api/diagnose` flow (text, image, voice). Skips image/voice checks if fixtures missing. Disconnect WiFi after ingest—app keeps working. Status banner shows: **OFFLINE — Ready locally** (Actian up, collection ready), **OFFLINE — Local DB not initialized** (backend up, collection missing), or **OFFLINE — Local API unavailable** (frontend can't reach backend).

---

## API Reference

Main entry point: `POST /api/diagnose` (fuses text + image + voice, returns templated evidence). Lower-level endpoints expose each modality + filters separately for programmatic use.

| Method | Path | Does |
|---|---|---|
| GET | `/api/health` | `{status, online, db, collection_ready}` |
| POST | `/api/ingest/manual` | multipart: PDF + machine_type + model_no |
| POST | `/api/ingest/incident` | JSON row → auto-detects incident vs error_code |
| POST | `/api/ingest/image` | multipart: image + machine_type + optional model_no/fault_code/severity/part_no |
| POST | `/api/ingest/voice` | multipart: WAV + machine_type |
| POST | `/api/ingest/part` | JSON row |
| POST | `/api/search/text` | JSON: `{query, filters?}` → RRF hybrid (dense ANN + identifier-filtered ANN) |
| POST | `/api/search/image` | multipart: image + filters → `image_vec` ANN |
| POST | `/api/search/voice` | multipart: WAV + filters → `audio_text_vec` ANN fused with transcript hybrid |
| POST | `/api/search/multimodal` | multipart: text + image + audio + filters → RRF across modalities |
| POST | `/api/diagnose` | multipart: query + image + voice + filters → **primary UI entry point**, templated evidence answer |
| POST | `/api/incident/save` | JSON verified incident row → live-indexed |

---

## Testing

```bash
cd backend
pytest tests -q
# 21 passed
```

All pipelines unit-tested (mock embedders, no network, no models needed for CI).

## Benchmark

End-to-end `/api/diagnose` latency over 20 mixed queries (error codes, part numbers, symptoms, machine IDs) against full demo corpus (3 PDFs, 30 incidents, 25 parts, 13 error codes, 6 images, 5 voice notes). 16 GB laptop, CPU-only, WSL2 + Docker, all models local:

| Metric | ms |
|---|---|
| **p50** | 843 |
| **p95** | 1093 |
| mean | 900 |
| min | 781 |
| max | 1094 |

Stable across 3 runs. Each request: embeds query → hybrid RRF (dense + identifier-filtered ANN) → 3 per-slot retrievals (manual/incident/part) = 9 vector ops per request. 20/20 queries returned full evidence set. Reproduce: `python backend/scripts/bench_diagnose.py`.

---

## Submission Readiness

Before DoraHacks final submission:

```bash
python scripts/check_submission_readiness.py
```

Fails if repo contains placeholder demo/team fields or landing page still using fallback block instead of real embed.

---

## Project Structure

```
fixfirst-edge/
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI app
│   │   ├── config.py            # env settings
│   │   ├── db.py                # Actian wrapper (named vectors, filters, hybrid RRF)
│   │   ├── schemas.py           # pydantic models
│   │   ├── routers/             # ingest, search, diagnose, health
│   │   ├── pipelines/           # text/image/audio/pdf/csv processors
│   │   └── services/            # ingest, search, diagnose logic
│   ├── scripts/
│   │   ├── bulk_ingest.py       # walks data/raw/, calls ingest endpoints
│   │   ├── verify_offline.py    # diagnose verifier
│   │   └── pull_actian.sh       # docker pull retry
│   └── tests/
├── frontend/
│   ├── app/page.tsx             # single-page UI
│   ├── components/              # UploadZone, SearchBar, FilterPanel, DiagnosePanel, …
│   └── lib/api.ts               # typed fetch wrapper
├── data/
│   ├── raw/                     # owner-supplied + fixture CSVs
│   └── processed/               # model cache (gitignored)
├── docker-compose.yml
└── CODEX_BRIEF.md               # implementation plan
```

---

## License

MIT — see [LICENSE](LICENSE).
