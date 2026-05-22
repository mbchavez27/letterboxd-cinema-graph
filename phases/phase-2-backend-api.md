# Phase 2 — Backend API

## Goal

REST API you can hit from Postman and get graph JSON back.

**Deliverable:** All v1 endpoints returning correct responses.

---

## Steps

| # | Step | File |
|---|------|------|
| 1 | Set up FastAPI project structure — routers, schemas, db connection module | `backend/main.py`, `backend/config.py`, `backend/schemas.py`, `backend/db.py` |
| 2 | Write Neo4j connection utility — singleton driver, session management, error handling | `backend/db.py` |
| 3 | Implement actor search endpoint — full-text index on actor `name` in Neo4j, return top matches | `backend/routers/actors.py` |
| 4 | Implement neighborhood endpoint — parameterized Cypher query for ego network at depth N; cap depth at 3 | `backend/routers/actors.py` |
| 5 | Implement shortest path endpoint — Cypher `shortestPath()` between two actor nodes | `backend/routers/actors.py` |
| 6 | Implement centrality endpoint — query precomputed PageRank scores stored by Neo4j GDS | `backend/routers/actors.py` |
| 7 | Implement genre top-actors endpoint — filter subgraph by genre, return top N actors by centrality | `backend/routers/actors.py` |
| 8 | Implement film detail endpoint — return film node with its full cast as a small subgraph | `backend/routers/films.py` |
| 9 | Standardize response schema — all graph endpoints return `{ nodes, edges, meta }` consistently | `backend/schemas.py` |
| 10 | Add basic error handling — 404 for unknown actors, timeout handling for deep traversals | `backend/routers/*.py` |
| 11 | Test all endpoints with Postman against real data | — |

---

## API Endpoints

| Method | Endpoint | What it returns |
|---|---|---|
| `GET` | `/actor/search?q={name}` | Matching actors with basic info |
| `GET` | `/actor/{name}/neighborhood?depth={n}` | Ego network subgraph up to N hops |
| `GET` | `/actor/{name}/stats` | Centrality scores, top co-stars, genre breakdown |
| `GET` | `/path?from={name}&to={name}` | Shortest path between two actors |
| `GET` | `/genre/{genre}/top-actors` | Most central actors in a genre subgraph |
| `GET` | `/film/{url}/detail` | Film detail with cast network |

All responses: `{ nodes: [...], edges: [...], meta: {...} }`

---

## Project Structure

```
backend/
├── main.py                  # App entry point, router registration
├── config.py                # Settings loaded from .env via pydantic-settings
├── schemas.py               # Pydantic request/response models
├── db.py                    # Neo4j driver singleton
├── routers/
│   ├── actors.py            # /actor/* endpoints
│   └── films.py             # /film/* endpoints
└── queries/
    ├── actor_queries.py     # Cypher strings for actor endpoints
    └── film_queries.py      # Cypher strings for film endpoints
```

---

## Dependencies

```bash
uv sync --group backend
```

Core: `fastapi`, `uvicorn`, `pydantic`, `pydantic-settings`, `neo4j` (already in Phase 1)

---

## Running

```bash
cd backend
uv run uvicorn main:app --reload
```

API available at `http://localhost:8000`
Interactive docs at `http://localhost:8000/docs`

---

## Success Criteria

All v1 Postman requests return correct `{ nodes, edges, meta }` JSON within 500ms.
