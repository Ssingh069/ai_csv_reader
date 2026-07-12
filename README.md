# GrowEasy — CSV Importer

Upload a messy CSV of leads, normalize and extract structured contact data,
preview the results in a fast virtualized table, and persist each import to
MongoDB with a browsable history.

The app handles the real-world mess of exported lead lists — inconsistent
column names, multiple emails/phones in one cell, missing fields — and turns
them into clean, structured records.

---

## Features

- **Drag-and-drop CSV upload** with client-side preview (PapaParse).
- **Smart extraction** — normalizes columns and pulls out structured fields
  (name, email, phone, company, etc.) using a pluggable provider (**Gemini**
  or **OpenAI**).
- **Virtualized results table** (`@tanstack/react-virtual`) that stays smooth
  on thousands of rows.
- **Import history** — every run is saved to MongoDB and reopenable from the
  history drawer.
- **Batched, concurrent extraction** with retries and timeouts, tunable via env.
- **Graceful degradation** — runs stateless if no MongoDB is configured, and
  ships a `mock` provider so it works with no API key.

---

## Tech Stack

| Layer | Tech |
|-------|------|
| Frontend | React 18, Vite, React Router, Framer Motion, TanStack Virtual, PapaParse, Lucide |
| Backend | Node.js, Express, Multer, Zod, `p-limit`, Helmet, Morgan |
| Extraction | Google Gemini / OpenAI (pluggable, plus a built-in `mock`) |
| Database | MongoDB (Atlas or local — optional) |
| Infra | Docker, docker-compose, Render (`render.yaml`) |

---

## Project Structure

```
.
├── backend/            # Express API (parse → extract → persist)
│   └── src/
│       ├── routes/     # /api/v1/{health,imports}
│       ├── controllers/
│       ├── services/   # extraction providers + mongo
│       └── config/     # env validation (zod)
├── frontend/           # Vite + React SPA
├── samples/            # example CSVs to try
├── docker-compose.yml
└── render.yaml         # Render deployment blueprint
```

---

## Getting Started (local)

### Prerequisites
- Node.js 20+
- A Gemini API key ([free key from Google](https://aistudio.google.com/apikey)) — or set `AI_PROVIDER=mock` to skip it
- MongoDB (optional — [Atlas](https://cloud.mongodb.com) free tier or local). Omit to run stateless.

### 1. Backend

```bash
cd backend
cp .env.example .env      # then fill in GEMINI_API_KEY and (optionally) MONGODB_URI
npm install
npm start                 # http://localhost:8080
```

### 2. Frontend

```bash
cd frontend
npm install
npm run dev               # http://localhost:5173
```

Open **http://localhost:5173** and drop one of the files from `samples/`.

---

## Configuration

Backend env vars (see [`backend/.env.example`](backend/.env.example)):

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8080` | API port |
| `FRONTEND_ORIGIN` | `http://localhost:5173` | CORS allow-origin |
| `AI_PROVIDER` | `gemini` | `gemini` \| `openai` \| `mock` |
| `GEMINI_API_KEY` | — | required when provider is `gemini` |
| `GEMINI_MODEL` | `gemini-3.1-flash-lite` | Gemini model |
| `OPENAI_API_KEY` / `OPENAI_MODEL` | — / `gpt-4o-mini` | for the OpenAI provider |
| `BATCH_SIZE` | `20` | rows per extraction batch |
| `AI_CONCURRENCY` | `3` | parallel batches |
| `MAX_UPLOAD_MB` | `10` | upload size limit |
| `MONGODB_URI` | — | leave blank to run stateless |
| `MONGODB_DB` | `groweasy_csv_importer` | database name |

> **MongoDB Atlas + SRV note:** on some networks the DNS resolver refuses
> `mongodb+srv://` (SRV) lookups (`querySrv ECONNREFUSED`). If that happens,
> use the **direct** (non-SRV) connection string from Atlas → Connect →
> "Node.js 2.2.12 or earlier", which lists the shard hosts explicitly.

---

## API

Base path: `/api/v1`

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET`  | `/health` | Service status, provider, persistence state |
| `POST` | `/imports/parse` | Upload a CSV (`multipart/form-data`, field `file`); returns parsed rows + `importId` |
| `POST` | `/imports/:importId/extract` | Run extraction on a parsed import |
| `GET`  | `/imports/history` | List past imports |
| `GET`  | `/imports/history/:importId` | Fetch a single import's details |

---

## Docker

```bash
docker compose up --build
```

- Frontend → http://localhost:5173
- Backend  → http://localhost:8080

Set `GEMINI_API_KEY` (and optionally `MONGODB_URI`) in your shell or a `.env`
file before running. See [`docker-compose.yml`](docker-compose.yml).

---

## Deployment (Render)

A [`render.yaml`](render.yaml) blueprint is included for one-click deploy to
[Render](https://render.com). Configure `GEMINI_API_KEY` and `MONGODB_URI` as
environment variables / secrets in the Render dashboard.

---

## Sample Data

The `samples/` folder contains deliberately messy exports to test with:

- `facebook-lead-export.csv`, `google-ads-export.csv` — platform exports
- `messy-realestate.csv` — inconsistent columns
- `multi-email-multi-phone.csv` — multiple contacts per cell
- `marketing-agency.csv`, `reference-groweasy.csv`
