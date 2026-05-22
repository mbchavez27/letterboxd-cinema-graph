# Phase 4 — ML Recommendation Layer

## Goal

"Films like this one" powered by graph embeddings, surfaced in the UI.

**Deliverable:** Working similar films/actors endpoints and recommendation cards in the frontend.

---

## Steps

| # | Step | File |
|---|------|------|
| 1 | Run Node2Vec on the co-star graph — train on `CO_STARRED_WITH` edges; output 128-dim embeddings per actor | `scripts/ml/train_node2vec.py` |
| 2 | Derive film embeddings — represent each film as the mean embedding of its cast actors | `scripts/ml/build_film_embeddings.py` |
| 3 | Store embeddings in pgvector — set up Postgres + pgvector via Docker, load all vectors | `scripts/ml/load_vectors.py` |
| 4 | Implement `/film/{url}/similar` endpoint — query pgvector for nearest neighbors by cosine similarity | `backend/routers/recommendations.py` |
| 5 | Implement `/actor/{name}/similar` endpoint — same approach for actor vectors | `backend/routers/recommendations.py` |
| 6 | Surface in the frontend — add "Similar films" card in film detail panel, "Similar actors" in actor panel | `frontend/src/components/DetailPanel.tsx` |
| 7 | Evaluate informally — spot-check results for known films (e.g. _There Will Be Blood_ should surface other PTA films) | — |

---

## ML Training Scripts

```
scripts/ml/
├── train_node2vec.py        # Train Node2Vec on CO_STARRED_WITH graph
├── build_film_embeddings.py # Derive film vectors from mean cast embeddings
└── load_vectors.py          # Push actor + film vectors into pgvector
```

### Training Workflow

1. Validate pipeline on a genre-filtered subgraph first (e.g. Drama only)
2. Run full training overnight — can take several hours on a laptop
3. Save embeddings to `data/embeddings/`

---

## API Endpoints

| Method | Endpoint | What it returns |
|---|---|---|
| `GET` | `/film/{url}/similar` | Films similar by graph embedding (top 10) |
| `GET` | `/actor/{name}/similar` | Actors with similar collaboration patterns (top 10) |

---

## Dependencies

```bash
uv sync --group ml
```

Core: `node2vec`, `numpy`, `scikit-learn`
Infra: Postgres + pgvector (via `docker-compose.yml`)

---

## Docker Setup

Add pgvector service to `docker-compose.yml`:

```yaml
services:
  neo4j:
    # ... existing Neo4j config

  pgvector:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: ${PG_USER:-postgres}
      POSTGRES_PASSWORD: ${PG_PASSWORD:-postgres}
      POSTGRES_DB: cinegraph
    ports:
      - "5432:5432"
    volumes:
      - pgvector_data:/var/lib/postgresql/data

volumes:
  pgvector_data:
```

---

## Output Files

All embeddings saved to `data/embeddings/`:

| File | Description |
|------|-------------|
| `actor_embeddings.npy` | 128-dim Node2Vec vectors per actor |
| `film_embeddings.npy` | Mean-cast-derived vectors per film |

---

## Success Criteria

Similar films for _Mulholland Drive_ returns other Lynch films and films with Naomi Watts in top 10.
