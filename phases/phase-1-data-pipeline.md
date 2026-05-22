# Phase 1 — Data Pipeline

## Goal

Populated Neo4j database you can query from the command line.

**Deliverable:** Working Cypher queries returning sensible results.

---

## Steps

| # | Step | Tool | File |
|---|------|------|------|
| 1 | Load dataset from Hugging Face | Notebook | `scripts/pipeline/notebooks/01_explore_dataset.ipynb` |
| 2 | Inspect schema — understand `cast`, `directors`, `genres`, `reviews` structure | Notebook | `scripts/pipeline/notebooks/01_explore_dataset.ipynb` |
| 3 | Clean data — drop empty casts, normalize actor names, parse ratings | Notebook → Script | `scripts/pipeline/notebooks/02_clean_data.ipynb` → `scripts/pipeline/clean.py` |
| 4 | Build actor nodes CSV | Script | `scripts/pipeline/build_edges.py` |
| 5 | Build film nodes CSV | Script | `scripts/pipeline/build_edges.py` |
| 6 | Build edge list CSV — `(actor)-[APPEARED_IN]->(film)` | Script | `scripts/pipeline/build_edges.py` |
| 7 | Set up Neo4j via Docker | Config | `docker-compose.yml` (root) |
| 8 | Bulk import into Neo4j | Script | `scripts/pipeline/import_neo4j.py` |
| 9 | Project co-star graph via GDS | Script | `scripts/pipeline/import_neo4j.py` |
| 10 | Validate — spot-check queries | Notebook | `scripts/pipeline/notebooks/03_validate_results.ipynb` |

---

## Notebook vs Script Split

### Notebooks (exploratory, iterative, visual)

- **`01_explore_dataset.ipynb`** — Load the dataset, inspect schema, check distributions, understand data quirks
- **`02_clean_data.ipynb`** — Prototype cleaning logic with before/after samples, validate transformations visually
- **`03_validate_results.ipynb`** — Run spot-check Cypher queries, display sample results, verify correctness

### Python Scripts (reproducible, pipeline-ready)

- **`load_dataset.py`** — Fetch dataset from Hugging Face (reusable by notebook)
- **`clean.py`** — Production cleaning pipeline (extracted from notebook findings)
- **`build_edges.py`** — Generate `actors.csv`, `films.csv`, `appeared_in.csv` deterministically
- **`import_neo4j.py`** — Bulk import CSVs into Neo4j + run GDS projection

### Workflow

1. Explore and prototype in notebooks
2. Once logic is solid, extract to `.py` scripts for reproducible pipeline
3. Run scripts end-to-end: `load_dataset.py` → `clean.py` → `build_edges.py` → `import_neo4j.py`
4. Validate results in `03_validate_results.ipynb`

---

## Output Files

All outputs go to `data/processed/`:

| File | Description |
|------|-------------|
| `actors.csv` | One row per unique actor with derived attributes (career start, total films, avg rating) |
| `films.csv` | One row per film with all dataset fields |
| `appeared_in.csv` | Edge list: `actor_name, film_url` |

---

## Dependencies

```bash
uv sync  # installs all Phase 1 deps
```

Core: `datasets`, `pandas`, `networkx`, `neo4j`, `python-dotenv`, `jupyter`, `matplotlib`, `seaborn`

---

## Success Criteria

`MATCH (a:Actor {name: "Cate Blanchett"})-[:CO_STARRED_WITH]-(b) RETURN b` returns correct co-stars.
