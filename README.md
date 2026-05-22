# Letterboxd Cinema Graph

Interactive graph explorer that maps relationships between actors and films using Letterboxd data. Browse co-star networks, find paths between actors, and get graph-powered recommendations.

**Not a Letterboxd clone.** This is a portfolio-quality project and foundation for future research — it makes the hidden collaboration structure of cinema visible and explorable.

---

## Tech Stack

| Layer | Tools |
|---|---|
| Data & ML | Python, Hugging Face `datasets`, Pandas, NetworkX, Neo4j + GDS, Node2Vec |
| Backend | FastAPI, Uvicorn, Neo4j Python driver, Pydantic |
| Frontend | React, TypeScript, Sigma.js, TanStack Query, Zustand, Tailwind, Vite |
| Infra | Docker + Docker Compose (Neo4j, pgvector) |

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Python | 3.11+ | Data pipeline, backend |
| uv | latest | Environment and dependency management |
| Docker + Docker Compose | latest | Neo4j, pgvector |
| Node.js | 18+ | Frontend (Phase 3+) |

Install `uv` if you don't have it:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

---

## Quick Start

### 1. Clone and install dependencies

```bash
git clone <repo-url>
cd letterboxd-cinema-graph
uv sync
```

This installs all Phase 1 dependencies (data pipeline + notebooks).

### 2. Set up environment variables

```bash
cp .env.example .env
```

Edit `.env` with your Neo4j credentials:

```
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password
```

### 3. Start Neo4j

```bash
docker compose up -d
```

Neo4j will be available at `http://localhost:7474` (browser) and `bolt://localhost:7687` (driver).

### 4. Run the data pipeline

```bash
# Explore the dataset in notebooks
uv run jupyter notebook scripts/pipeline/notebooks/

# Or run the pipeline scripts directly
uv run python scripts/pipeline/load_dataset.py
uv run python scripts/pipeline/clean.py
uv run python scripts/pipeline/build_edges.py
uv run python scripts/pipeline/import_neo4j.py
```

### 5. Install additional dependency groups (later phases)

```bash
# Phase 2 — Backend API
uv sync --group backend

# Phase 4 — ML recommendations
uv sync --group ml

# Everything
uv sync --all-groups
```

---

## Development Guide

### Running Notebooks

```bash
uv run jupyter notebook
```

Notebooks are in `scripts/pipeline/notebooks/`:

| Notebook | Purpose |
|---|---|
| `01_explore_dataset.ipynb` | Load dataset, inspect schema, understand data structure |
| `02_clean_data.ipynb` | Prototype cleaning logic with before/after samples |
| `03_validate_results.ipynb` | Spot-check Neo4j queries, verify correctness |

### Running Pipeline Scripts

Scripts are in `scripts/pipeline/` and should be run in order:

1. `load_dataset.py` — Fetch from Hugging Face
2. `clean.py` — Normalize names, filter nulls, parse ratings
3. `build_edges.py` — Generate `actors.csv`, `films.csv`, `appeared_in.csv`
4. `import_neo4j.py` — Bulk import into Neo4j + GDS projection

### Neo4j

- Browser: `http://localhost:7474`
- Default credentials: `neo4j` / `neo4j` (change on first login)
- Heap: 4GB allocated in `docker-compose.yml`
- Stop: `docker compose down`
- Wipe and restart: `docker compose down -v && docker compose up -d`

### Data Directory

All data lives in `data/` and is fully gitignored:

```
data/
├── raw/          # Downloaded dataset (never modified)
├── processed/    # Cleaned CSVs for Neo4j import
│   ├── actors.csv
│   ├── films.csv
│   └── appeared_in.csv
└── embeddings/   # Node2Vec vectors (Phase 4)
```

---

## Project Structure

```
letterboxd-cinema-graph/
├── scripts/
│   └── pipeline/           # Phase 1 — data pipeline
│       ├── notebooks/      # Exploration, cleaning, validation
│       ├── load_dataset.py
│       ├── clean.py
│       ├── build_edges.py
│       └── import_neo4j.py
├── backend/                # Phase 2 — FastAPI
├── frontend/               # Phase 3 — React + Sigma.js
├── data/                   # Never committed to git
├── docker-compose.yml      # Neo4j + pgvector
├── pyproject.toml          # Python dependencies (uv)
├── PROJECT_SPECS.md        # Full project specification
└── README.md
```

---

## Build Phases

| Phase | Goal | Status |
|---|---|---|
| 1 — Data Pipeline | Populated Neo4j database with working Cypher queries | In Progress |
| [`phases/phase-1-data-pipeline.md`](phases/phase-1-data-pipeline.md) |
| 2 — Backend API | REST API returning graph JSON for all v1 endpoints | Planned |
| [`phases/phase-2-backend-api.md`](phases/phase-2-backend-api.md) |
| 3 — Frontend Explorer | React app with interactive Sigma.js graph canvas | Planned |
| [`phases/phase-3-frontend-explorer.md`](phases/phase-3-frontend-explorer.md) |
| 4 — ML Recommendations | Node2Vec embeddings + similar film/actor suggestions | Planned |
| [`phases/phase-4-ml-recommendations.md`](phases/phase-4-ml-recommendations.md) |

Full specifications: [`phases/`](phases/)

---

## Cross-Platform Notes

### Linux / macOS

Everything works out of the box. All commands in this guide use Unix-style paths and shell syntax.

### Windows

Use **WSL2** (Windows Subsystem for Linux). Native Windows is not recommended because:

- Neo4j Docker containers have path-mounting issues on native Windows
- Unix-style paths in scripts will break
- Docker Desktop performance is significantly worse without WSL2 backend

**Setup:**

1. Install WSL2: `wsl --install` (requires Windows 10/11)
2. Open your WSL terminal (Ubuntu recommended)
3. Install `uv`, Docker, and Node.js inside WSL
4. Clone the repo inside your WSL filesystem (`~/projects/`, not `/mnt/c/`)
5. Follow the Linux instructions above

**Important:** Keep the project inside the WSL filesystem (`~`), not in the Windows-mounted `/mnt/c/` directory. File I/O performance is drastically worse on mounted drives.

---

## License

MIT — free for educational and research use.

The Letterboxd dataset is sourced from TMDB and carries its own attribution requirements. Check Letterboxd's terms of service before deploying publicly.
