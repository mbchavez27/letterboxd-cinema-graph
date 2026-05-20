# Project Specification
## Letterboxd Actor-Movie Social Graph Explorer

**Type:** Full-stack web application with ML layer  
**Goal:** Interactive graph explorer that maps relationships between actors and films using Letterboxd data — browse co-star networks, find paths between actors, and get graph-powered recommendations  
**Target:** Portfolio-quality project, foundation for future research paper

---

## Core Idea

Films are not independent items. They are assemblages of collaborators — actors, directors, cinematographers — who carry relationships across their careers. This app makes that structure visible and explorable.

A user searches for an actor. Their co-star network expands outward. Click a co-star, their network expands. Ask for the shortest path between two actors. Find the most connected actor in a genre. Get film recommendations not from genre tags, but from graph proximity.

---

## What It Is NOT (scope boundaries)

- Not a Letterboxd clone or replacement
- Not a social network for users (no accounts in v1)
- Not a full knowledge graph (actors and films only, no crew in v1)
- Not real-time (static dataset, refreshed periodically)

---

## Tech Stack

### Data & ML
| Tool | Purpose |
|---|---|
| Python | All data processing and ML |
| Hugging Face `datasets` | Load the 847k-film Letterboxd dataset |
| Pandas | Data cleaning and edge list construction |
| NetworkX | Graph preprocessing and validation before DB import |
| Neo4j (Community Edition) | Primary graph database — stores nodes, edges, runs traversals |
| Neo4j Graph Data Science (GDS) | Built-in PageRank, Louvain clustering, shortest path — no ML code needed for v1 |
| Node2Vec (Python library) | Learn actor/film embeddings from graph structure (v2) |
| pgvector or Pinecone | Vector similarity search for recommendations (v2) |
| TMDB API | Enrich actor nodes with photos, bio, nationality |

### Backend
| Tool | Purpose |
|---|---|
| Python | Primary language |
| FastAPI | REST API framework — serves graph query results to frontend |
| Uvicorn | ASGI server to run FastAPI |
| neo4j Python driver | Connect FastAPI to Neo4j via Bolt protocol |
| Pydantic | Request/response data validation |
| python-dotenv | Environment variable management |

### Frontend
| Tool | Purpose |
|---|---|
| React | Component framework |
| TypeScript | Type safety across the frontend |
| Sigma.js | Graph canvas — renders force-directed actor/film networks interactively |
| TanStack Query | Data fetching, caching, loading states |
| Zustand | Lightweight global state (selected actor, graph depth, filters) |
| Tailwind CSS | Utility styling |
| Vite | Build tool and dev server |

### Infrastructure (local dev)
| Tool | Purpose |
|---|---|
| Docker + Docker Compose | Run Neo4j locally without manual install |
| Git + GitHub | Version control |

---

## Data Model

### Nodes

**Actor**
- `id` — TMDB actor ID
- `name` — full name
- `known_for_genres` — array of genres they appear in most
- `career_start` — year of first film
- `total_films` — count of films in dataset
- `avg_film_rating` — mean Letterboxd rating across their films
- `photo_url` — from TMDB

**Film**
- `id` — Letterboxd/TMDB film ID
- `title`
- `year`
- `genres` — array
- `letterboxd_rating` — float, out of 5
- `synopsis`
- `poster_url`
- `director`

### Edges

**APPEARED_IN** (Actor → Film)
- `role` — lead or supporting (if available)

**CO_STARRED_WITH** (Actor ↔ Actor) — derived, projected from bipartite
- `film_count` — number of shared films
- `avg_shared_film_rating` — quality signal for the collaboration
- `last_collaboration_year`

---

## System Architecture

```
┌─────────────────────────────────────┐
│           React Frontend            │
│   Sigma.js graph canvas             │
│   Search, filters, detail panel     │
└────────────────┬────────────────────┘
                 │ HTTP (JSON)
┌────────────────▼────────────────────┐
│           FastAPI Backend           │
│   Graph query endpoints             │
│   ML inference endpoints (v2)       │
└──────────┬─────────────┬────────────┘
           │             │
┌──────────▼──┐    ┌─────▼──────────┐
│   Neo4j     │    │  Vector Store  │
│   Graph DB  │    │  (v2 only)     │
│   Cypher    │    │  pgvector /    │
│   + GDS     │    │  Pinecone      │
└─────────────┘    └────────────────┘
```

---

## API Endpoints

### v1 — Graph queries

| Method | Endpoint | What it returns |
|---|---|---|
| `GET` | `/actor/search?q={name}` | Matching actors with basic info |
| `GET` | `/actor/{id}/neighborhood?depth={n}` | Ego network subgraph up to N hops |
| `GET` | `/actor/{id}/stats` | Centrality scores, top co-stars, genre breakdown |
| `GET` | `/path?from={id}&to={id}` | Shortest path between two actors |
| `GET` | `/genre/{genre}/top-actors` | Most central actors in a genre subgraph |
| `GET` | `/film/{id}` | Film detail with cast network |

### v2 — ML recommendations

| Method | Endpoint | What it returns |
|---|---|---|
| `GET` | `/film/{id}/similar` | Films similar by graph embedding |
| `GET` | `/actor/{id}/similar` | Actors with similar collaboration patterns |

All responses return a consistent graph payload: `{ nodes: [...], edges: [...], meta: {...} }` so the frontend renders without transformation.

---

## Build Phases

### Phase 1 — Data Pipeline
**Goal:** Populated Neo4j database you can query from the command line  
**Deliverable:** Working Cypher queries returning sensible results

Steps:
1. Load the Letterboxd Hugging Face dataset with the `datasets` Python library
2. Inspect the schema — understand what fields are available, how cast is structured, what's missing or null
3. Clean the data — handle missing cast arrays, normalize names, filter films with no cast data
4. Build the edge list — for every film, emit `(actor) -[APPEARED_IN]-> (film)` pairs as a flat CSV
5. Set up Neo4j locally via Docker — one `docker-compose.yml` file, Neo4j running on localhost
6. Import nodes and edges into Neo4j using the `neo4j-admin import` bulk loader (faster than row-by-row)
7. Enrich actor nodes — call TMDB API for photo URLs, bios, nationality; respect rate limits (40 req/10s)
8. Project the co-star graph — use Neo4j GDS to project `CO_STARRED_WITH` edges from the bipartite graph
9. Validate — spot-check known actor pairs (e.g. DiCaprio + Scorsese films) to confirm edges are correct

---

### Phase 2 — Backend API
**Goal:** REST API you can hit from Postman and get graph JSON back  
**Deliverable:** All v1 endpoints returning correct responses

Steps:
1. Set up FastAPI project structure — routers, schemas, db connection module
2. Write Neo4j connection utility — singleton driver, session management, error handling
3. Implement actor search endpoint — full-text index on actor name in Neo4j, return top matches
4. Implement neighborhood endpoint — parameterized Cypher query for ego network at depth N; cap depth at 3 to avoid exploding subgraphs
5. Implement shortest path endpoint — Cypher `shortestPath()` between two actor node IDs
6. Implement centrality endpoint — query precomputed PageRank scores from Neo4j GDS
7. Implement genre top-actors endpoint — filter subgraph by genre, return top N by centrality
8. Standardize response schema — all graph endpoints return `{ nodes, edges, meta }` consistently
9. Add basic error handling — 404 for unknown actors, timeout handling for deep traversals
10. Test all endpoints with Postman against real data

---

### Phase 3 — Frontend Graph Explorer
**Goal:** React app where you can search an actor, see their network, and navigate by clicking  
**Deliverable:** Working interactive graph in the browser

Steps:
1. Set up React + TypeScript project with Vite
2. Set up TanStack Query for all API calls and Zustand for graph state
3. Build the search bar — autocomplete against `/actor/search`, select an actor to load their neighborhood
4. Integrate Sigma.js — render the graph payload from the API as a force-directed canvas
5. Style nodes — actor nodes as circles sized by centrality score, film nodes as smaller squares
6. Handle node clicks — clicking an actor node triggers a new neighborhood fetch and expands the graph
7. Build the detail panel — clicking a node shows its card (photo, stats, top connections) in a sidebar
8. Add depth control — slider or buttons to set neighborhood depth (1, 2, or 3 hops)
9. Add genre filter — dropdown to filter visible nodes to a specific genre
10. Add shortest path UI — two search fields, a "connect" button, highlight the path in the graph
11. Handle performance — for large neighborhoods, limit to top 50 edges by weight before rendering; show "expand" option for more

---

### Phase 4 — ML Recommendation Layer
**Goal:** "Films like this one" powered by graph embeddings, surfaced in the UI  
**Deliverable:** Working similar films/actors endpoint and recommendation card in the frontend

Steps:
1. Run Node2Vec on the co-star graph — train on the `CO_STARRED_WITH` edges using the `node2vec` Python library; output 128-dim embeddings per actor
2. Derive film embeddings — represent each film as the mean embedding of its cast
3. Store embeddings — load actor and film vectors into pgvector (Postgres extension) or Pinecone
4. Implement `/film/{id}/similar` endpoint — query vector store for nearest neighbors by cosine similarity, return top 10
5. Implement `/actor/{id}/similar` endpoint — same approach for actor vectors
6. Surface in the frontend — add a "Similar films" card in the film detail panel, powered by the new endpoint
7. Evaluate informally — spot-check that similar films for known titles make intuitive sense (e.g. similar to *There Will Be Blood* should surface other PTA films and films with Daniel Day-Lewis)

---

## Folder Structure

```
letterboxd-graph/
│
├── data/                        # Raw and processed data files
│   ├── raw/                     # Downloaded dataset files
│   ├── processed/               # Cleaned CSVs ready for Neo4j import
│   └── embeddings/              # Saved Node2Vec vectors
│
├── pipeline/                    # Data processing scripts
│   ├── load_dataset.py          # Fetch from Hugging Face
│   ├── clean.py                 # Normalize, filter nulls
│   ├── build_edges.py           # Construct edge list CSVs
│   ├── enrich_tmdb.py           # Call TMDB API for actor metadata
│   └── import_neo4j.py          # Bulk import into Neo4j
│
├── ml/                          # ML training scripts
│   ├── train_node2vec.py        # Train embeddings on co-star graph
│   ├── build_film_embeddings.py # Derive film vectors from cast embeddings
│   └── load_vectors.py          # Push vectors to pgvector / Pinecone
│
├── backend/                     # FastAPI application
│   ├── main.py                  # App entry point
│   ├── routers/
│   │   ├── actors.py
│   │   ├── films.py
│   │   └── recommendations.py
│   ├── schemas.py               # Pydantic request/response models
│   ├── db.py                    # Neo4j driver singleton
│   └── queries/                 # Cypher query strings, one file per domain
│       ├── actor_queries.py
│       └── film_queries.py
│
├── frontend/                    # React application
│   ├── src/
│   │   ├── components/
│   │   │   ├── GraphCanvas.tsx  # Sigma.js wrapper
│   │   │   ├── SearchBar.tsx
│   │   │   ├── DetailPanel.tsx
│   │   │   └── PathFinder.tsx
│   │   ├── store/               # Zustand state
│   │   ├── hooks/               # TanStack Query hooks
│   │   └── types/               # Shared TypeScript types
│   └── vite.config.ts
│
├── docker-compose.yml           # Neo4j + pgvector local setup
├── .env.example                 # Environment variables template
└── README.md
```

---

## Environment Variables

```
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password
TMDB_API_KEY=your_tmdb_key
PINECONE_API_KEY=your_pinecone_key        # v2 only
PGVECTOR_CONNECTION_STRING=...            # v2 only
```

---

## Key Constraints and Watch-outs

**Graph size at scale**
The full dataset will produce millions of co-star edges. Always serve ego-networks (neighborhood around one node) from the API — never dump the full graph to the frontend. Cap neighborhood depth at 3 hops maximum.

**TMDB rate limits**
TMDB allows 40 requests per 10 seconds. With 847k films and potentially millions of actors, enrichment needs to be batched and resumable. Write the enrichment script so it can be stopped and restarted without re-fetching already-processed actors.

**Neo4j memory**
The Community Edition has no cluster support but handles this dataset size fine on a local machine. Give Neo4j at least 4GB heap in the Docker config.

**Node2Vec training time**
Training on the full co-star graph can take hours. Start with a genre-filtered subgraph (e.g. drama films only) to validate the pipeline before running on the full dataset.

**Letterboxd data terms**
The HuggingFace dataset is MIT licensed for educational and research use. Keep this in mind if you deploy publicly — check Letterboxd's terms before serving their data from a public endpoint.

---

## Success Criteria per Phase

| Phase | Done when... |
|---|---|
| Phase 1 | Cypher query `MATCH (a:Actor {name: "Cate Blanchett"})-[:CO_STARRED_WITH]-(b) RETURN b` returns correct co-stars |
| Phase 2 | All v1 Postman requests return correct JSON within 500ms |
| Phase 3 | Can search "Tilda Swinton", see her network, click a co-star, navigate three hops, and find the path back |
| Phase 4 | Similar films for *Mulholland Drive* returns other Lynch films and films with Naomi Watts in top 10 |
