# FloatChat-ARGO

**AI-powered conversational interface for ARGO ocean data discovery & visualization**  
Team: Cypher

---

## What is this?
FloatChat-ARGO is a proof-of-concept system that ingests ARGO NetCDF data, stores structured records in PostgreSQL, embeddings in a vector DB (FAISS/Chroma), and exposes a RAG-powered chatbot + interactive dashboard for querying and visualizing ARGO floats.

## Features (PoC)
- NetCDF ingestion → Parquet / Postgres
- Embedding + vector index for spatial/metadata retrieval
- Natural language → SQL translation via RAG (Model Context Protocol)
- Interactive dashboard: trajectories, depth-time plots, profile comparisons
- Export: CSV / ASCII / NetCDF subsets

## Quickstart (dev)
### Prereqs
- Python 3.10+, Docker, Docker Compose, PostgreSQL

### Local dev
```bash
# 1. Clone
git clone https://github.com/<your-org>/FloatChat-ARGO.git
cd FloatChat-ARGO

# 2. Build and start DB + vector (example)
docker-compose up -d

# 3. Install Python deps
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 4. Run ingest (example)
python src/ingest/ingest_argo_sample.py --input data/sample.nc

# 5. Run backend
uvicorn src.backend.app:app --reload

# 6. Run frontend (Streamlit)
streamlit run src.frontend.app.py
