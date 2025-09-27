# 🚀 FloatChat-ARGO

> **AI-powered conversational interface for ARGO ocean float data discovery & visualization**
>
> **Team:** Cypher • **Category:** Software / PoC • **Organization:** MoES / INCOIS (inspiration & datasets)

---

![FloatChat Banner](https://img.shields.io/badge/FloatChat-ARGO-blue?style=for-the-badge\&logo=netlify)

**FloatChat-ARGO** makes oceanographic exploration human-friendly: ask natural-language questions about ARGO float data and get visualizations, tables, and actionable exports. This repository is a polished Proof-of-Concept (PoC) demonstrating an end-to-end pipeline that ingests ARGO NetCDF files, stores structured data in PostgreSQL and embeddings in a vector database, and uses Retrieval-Augmented Generation (RAG) + LLMs to translate human queries into data queries and visual responses.

---

## 🧭 Table of Contents

1. [Why FloatChat-ARGO?](#-why-floatchat-argo)
2. [Highlights / Features](#-highlights--features)
3. [Tech Stack & Architecture](#-tech-stack--architecture)
4. [Quickstart (Dev) — Get it running locally](#-quickstart-dev---get-it-running-locally)
5. [Ingest pipeline (NetCDF → Parquet → Postgres)](#-ingest-pipeline-netcdf--parquet--postgres)
6. [Vector DB & RAG pipeline](#-vector-db--rag-pipeline)
7. [Frontend: Dashboard & Chat UX](#-frontend-dashboard--chat-ux)
8. [Example queries & sample outputs](#-example-queries--sample-outputs)
9. [Project layout & important files](#-project-layout--important-files)
10. [Roadmap & milestones](#-roadmap--milestones)
11. [Contributing & Good first issues](#-contributing--good-first-issues)
12. [Security & Data Handling](#-security--data-handling)
13. [License & Acknowledgements](#-license--acknowledgements)
14. [Contacts & Team Cypher](#-contacts--team-cypher)

---

## 🌊 Why FloatChat-ARGO?

Ocean data can be intimidating: NetCDF, multidimensional arrays, geospatial indexing, and domain-specific variables make it cumbersome for non-experts. FloatChat-ARGO lowers the barrier — users can ask questions like:

* *“Show me salinity profiles near the equator in March 2023.”*
* *“Compare chlorophyll-a and oxygen profiles in the Arabian Sea over the last 6 months.”*
* *“Which ARGO floats are closest to 18.5°N, 72.8°E?”*

and receive an immediate, interactive reply: plots, maps, CSV/NetCDF snippets, and plain-English explanations.

---

## ✨ Highlights / Features

* **End-to-end ingestion:** NetCDF → Parquet → PostgreSQL schema for ARGO profile metadata and measurements.
* **Vectorized retrieval:** Build embeddings for float metadata, notes, and profile summaries; store in FAISS/Chroma for semantic retrieval.
* **RAG-powered LLM:** Retrieve relevant context with vector DB + LLM to map NL queries to SQL and generate user-facing answers (Model Context Protocol).
* **Interactive dashboard:** Streamlit-based UI (configurable to Dash) with map view, profile viewer, comparisons, and a chat panel.
* **Exportable outputs:** Export filtered subsets to CSV, ASCII, or NetCDF for downstream analysis.
* **Modular & extensible:** Designed to extend beyond ARGO (BGC floats, gliders, satellite tiles).

---

## 🧰 Tech Stack & Architecture

**Languages & frameworks:** Python 3.10+, FastAPI, Streamlit, SQLAlchemy, Pandas, xarray

**Datastores & infra:** PostgreSQL (relational records), FAISS or Chroma (vector index), Parquet for columnar snapshots

**LLM & RAG:** OpenAI / local LLMs (LLaMA/Mistral/QWEN) with a retrieval layer + lightweight prompt orchestration (Model Context Protocol)

**Visualizations:** Plotly for interactive plots, Leaflet (via `folium`) for map tiles & float tracks

**Containerization:** Docker + Docker Compose for local dev

**CI/CD:** GitHub Actions (lint, unit tests, Docker build) — optional deployment to Render or any container host

### 🏗 Architecture (High-level)

```
[NetCDF Files] --> [Ingest Scripts] --> Parquet / Postgres
                                   |
                                   v
                            [Embedding Generator]
                                   |
                                   v
                           [Vector DB (FAISS/Chroma)]
                                   |
[User Chat Query] --> [Backend RAG Service + LLM] --> SQL / Data Retrieval --> Response + Visuals
                                   |
                                   v
                               [Frontend (Streamlit)]
```

---

## ⚡ Quickstart (Dev) — Get it running locally

> This section gets you a working PoC on your machine with a *small sample* dataset (do NOT commit full NetCDF files to the repo).

### Prerequisites

* Python 3.10+ installed
* Docker & Docker Compose
* `git` installed

### Clone & start

```bash
git clone https://github.com/<your-org>/FloatChat-ARGO.git
cd FloatChat-ARGO
```

### 1) Start infra (Postgres sample + vector service)

A `docker-compose.yml` is provided for quick local setup. It starts PostgreSQL and a vector service (if using Chromadb or a small FAISS containerized helper).

```bash
# start infra
docker-compose up -d
```

### 2) Create virtualenv & install deps

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 3) Configure environment

Create a `.env` with the following example variables (don't commit this file):

```
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/floatchat
VECTOR_DB_URL=http://localhost:8000
OPENAI_API_KEY=sk-...
LLM_MODEL=gpt-4o-mini
```

### 4) Ingest a small sample NetCDF (example)

We include a tiny `data/sample_argo.nc` file in the repository for demo purposes. To ingest it:

```bash
python src/ingest/ingest_argo_sample.py --input data/sample_argo.nc
```

This will: parse the NetCDF, write a Parquet snapshot to `data/` and insert profile metadata & measurements into Postgres.

### 5) Run backend

```bash
# launch FastAPI backend
uvicorn src.backend.app:app --reload --host 0.0.0.0 --port 8000
```

### 6) Run frontend

```bash
streamlit run src.frontend.app.py
```

Then open `http://localhost:8501` and start chatting with the dataset!

---

## 🧾 Ingest pipeline (NetCDF → Parquet → Postgres)

**Goals:** Extract ARGO profiles, transform variables, index metadata and time/space fields.

**Key steps implemented in `src/ingest/`:**

1. Read NetCDF using `xarray` and `netCDF4`. Parse profiles and convert to tidy `pandas` DataFrames.
2. Normalize variable names (e.g., `TEMP` → `temperature`) and handle missing flags (NaN / fill values).
3. Persist a Parquet snapshot for reproducibility and offline processing.
4. Upsert per-profile metadata into PostgreSQL (float_id, profile_id, lon, lat, date, depth range, variable list).
5. Compute a short text summary per profile (e.g., "Float 590123 — 2023-03-15 — Temp max 29.1°C") and store as a candidate for embeddings.

**Schema snippet (simplified)**

```sql
CREATE TABLE profiles (
  id SERIAL PRIMARY KEY,
  float_wmo INT,
  profile_index INT,
  timestamp TIMESTAMP,
  lat DOUBLE PRECISION,
  lon DOUBLE PRECISION,
  min_depth DOUBLE PRECISION,
  max_depth DOUBLE PRECISION,
  variables JSONB,
  summary TEXT
);

CREATE TABLE measurements (
  id SERIAL PRIMARY KEY,
  profile_id INT REFERENCES profiles(id),
  depth DOUBLE PRECISION,
  temperature DOUBLE PRECISION,
  salinity DOUBLE PRECISION,
  oxygen DOUBLE PRECISION,
  -- other params as available
  recorded_at TIMESTAMP
);
```

---

## 🧠 Vector DB & RAG pipeline

**Purpose:** Combine semantic retrieval (vector DB) with LLM reasoning for robust query interpretation and SQL generation.

**Flow:**

1. For each profile record, compute a small embedding using an embedding model (OpenAI or local embedding model).
2. Store embeddings + metadata in FAISS or Chroma.
3. On user query: retrieve top-k semantically-similar profiles (by embedding), include their summaries and metadata as RAG context.
4. Pass the combined prompt to the LLM with instructions to:

   * infer user intent, and
   * (if requested) generate SQL to run against Postgres, or produce a visualization plan.

**Example Prompt Template (conceptual):**

```
You are a data assistant. Relevant profile summaries: [..]. User query: "Show me salinity profiles near the equator in March 2023".
Task: return SQL to select matching profiles or provide a plan to visualize results.
```

**Safety tip:** LLM-generated SQL queries must be validated — use parameterized queries and a restricted SQL execution layer to avoid malicious or unintended queries.

---

## 🖥 Frontend: Dashboard & Chat UX

We ship a Streamlit dashboard with:

* **Map (Leaflet)** showing float trajectories and float markers. Clicking a float opens a profile viewer.
* **Profile Viewer**: depth vs variable plots (temperature, salinity, oxygen, BGC params).
* **Compare Mode**: overlay multiple profiles by float or time-window.
* **Chat Panel**: plain-language input; responses may include plots, tables, and suggested follow-ups.

**UX tips implemented:**

* Inline suggested prompts for novice users (e.g., "Compare salinity in Arabian Sea — last 6 months")
* Result-caching to speed repeat queries
* Export buttons for CSV & NetCDF snippets

---

## 🔎 Example queries & sample outputs

Here are typical user interactions: **(illustrative examples)**

1. **User:** `Show me salinity profiles near the equator in March 2023.`

   * Backend: RAG finds profiles within lat ± 5° around 0°, time in March 2023 → returns 12 profiles.
   * Frontend: Plots individual salinity vs depth curves with a mean profile and hoverable points.
   * Exports: `results/mar2023_equator_salinity.csv` or NetCDF subset.

2. **User:** `Compare chlorophyll and oxygen in the Arabian Sea for the last 6 months.`

   * Backend: filter by region bounding box (Arabian Sea), time = now - 6 months, retrieve BGC variables, and compute statistics (mean, std) per profile.
   * Frontend: dual-axis plot (depth vs value) for both variables, with shading for variability.

3. **User:** `Nearest ARGO floats to 18.5N, 72.8E.`

   * Backend: spatial index query to find nearby float positions, returns top-5 with distances; map centers on the location and highlights floats.

---

## 📁 Project layout & important files

```
FloatChat-ARGO/
├─ .github/                 # CI, PR templates, issue templates
├─ data/                    # sample datasets (small) — DO NOT include full data
├─ docs/                    # architecture, flowcharts, ROADMAP.md
├─ src/
│  ├─ ingest/               # NetCDF parsing & ingestion
│  ├─ backend/              # FastAPI + RAG orchestration
│  ├─ vectordb/             # embedding & FAISS/Chroma helpers
│  └─ frontend/             # Streamlit app & visualization code
├─ infra/                   # Docker, docker-compose, k8s manifests
├─ tests/
├─ README.md
├─ requirements.txt
└─ docker-compose.yml
```

---

## 🛣 Roadmap & Milestones

**Sprint 0 — PoC setup (this repo)**

* Minimal ingest for small NetCDF sample
* Postgres schema & simple API
* Streamlit basic map & profile plot

**Sprint 1 — RAG + Vector DB**

* Embedding pipeline and FAISS/Chroma index
* LLM orchestrator for SQL generation & explanation

**Sprint 2 — UX polish & export**

* Compare mode, better map interactions, caching
* Export to CSV/NetCDF/ASCII

**Sprint 3 — Additional datasets & deployment**

* Add BGC floats, glider support, satellite imagery overlays
* Deploy to Render / GCP / Kubernetes

---

## 🤝 Contributing & Good First Issues

We welcome contributors — students, oceanographers, data engineers, and designers. Suggested first issues:

* `good-first-issue`: Add a small sample NetCDF + ingestion notebook
* `backend`: Implement Postgres schema migrations
* `frontend`: Add hover info to Leaflet markers for floats

Please read `CONTRIBUTING.md` and `CODE_OF_CONDUCT.md` before contributing.

---

## 🔒 Security & Data Handling

* **Never** commit full-size NetCDF files to GitHub. Use small samples and downloader scripts.
* Use GitHub Secrets for API keys (OpenAI, model endpoints).
* Validate/parameterize any SQL generated by LLMs before execution.
* Use role-based DB credentials for the app (read-only for queries where possible).

---

## 🧾 License & Acknowledgements

This project is released under the **MIT License** — see `LICENSE`.

Acknowledgements:

* ARGO Global Data Repository (sample data references)
* INCOIS & MoES (inspiration & Indian ARGO program)
* Open-source libraries: xarray, pandas, SQLAlchemy, FastAPI, Streamlit, Plotly, FAISS/Chroma

---

## 📬 Contacts & Team Cypher

* **Team:** Cypher
* **Repo:** `FloatChat-ARGO`
* **Lead / Maintainer:** `zamanrashid114@gmail.com` 

---




*Made with 🌊 & ⚙️ by Team Cypher — FloatChat-ARGO*
