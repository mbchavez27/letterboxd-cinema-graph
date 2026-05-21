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

## Dataset

**Source:** [pkchwy/letterboxd-all-movie-data](https://huggingface.co/datasets/pkchwy/letterboxd-all-movie-data) on Hugging Face  
**Size:** 847,209 films  
**License:** MIT — free for educational and research use

Each record contains:

```json
{
  "url": "https://letterboxd.com/film/come-and-see/",
  "title": "Come and See",
  "year": "1985",
  "directors": ["Elem Klimov"],
  "genres": ["War", "Drama"],
  "cast": ["Aleksei Kravchenko", "Olga Mironova"],
  "synopsis": "The invasion of a village in Byelorussia...",
  "rating": "4.62 out of 5",
  "poster_url": "https://a.ltrbxd.com/resized/film-poster/...",
  "reviews": [
    {
      "username": "cameron fetter",
      "review_text": "as soon as this film ended...",
      "likes": "11662"
    }
  ]
}
```

No external API needed — `cast`, `poster_url`, `rating`, `genres`, and `reviews` with usernames are all included.

---

## Tech Stack

### Data & ML

| Tool                           | Purpose                                                                         |
| ------------------------------ | ------------------------------------------------------------------------------- |
| Python                         | All data processing and ML                                                      |
| Hugging Face `datasets`        | Load the 847k-film Letterboxd dataset                                           |
| Pandas                         | Data cleaning and edge list construction                                        |
| NetworkX                       | Graph preprocessing and validation before DB import                             |
| Neo4j (Community Edition)      | Primary graph database — stores nodes, edges, runs traversals                   |
| Neo4j Graph Data Science (GDS) | Built-in PageRank, Louvain clustering, shortest path — no ML code needed for v1 |
| Node2Vec (Python library)      | Learn actor/film embeddings from graph structure (v2)                           |
| pgvector                       | Vector similarity search for recommendations (v2)                               |

### Backend

| Tool                | Purpose                                                     |
| ------------------- | ----------------------------------------------------------- |
| Python              | Primary language                                            |
| FastAPI             | REST API framework — serves graph query results to frontend |
| Uvicorn             | ASGI server to run FastAPI                                  |
| neo4j Python driver | Connect FastAPI to Neo4j via Bolt protocol                  |
| Pydantic            | Request/response data validation                            |
| python-dotenv       | Environment variable management                             |

### Frontend

| Tool           | Purpose                                                                 |
| -------------- | ----------------------------------------------------------------------- |
| React          | Component framework                                                     |
| TypeScript     | Type safety across the frontend                                         |
| Sigma.js       | Graph canvas — renders force-directed actor/film networks interactively |
| TanStack Query | Data fetching, caching, loading states                                  |
| Zustand        | Lightweight global state (selected actor, graph depth, filters)         |
| Tailwind CSS   | Utility styling                                                         |
| Vite           | Build tool and dev server                                               |

### Infrastructure (local dev)

| Tool                    | Purpose                                             |
| ----------------------- | --------------------------------------------------- |
| Docker + Docker Compose | Run Neo4j + pgvector locally without manual install |
| Git + GitHub            | Version control                                     |

---

## Data Model

### Nodes

**Actor**

- `name` — full name (primary key, normalized)
- `known_for_genres` — array of genres they appear in most (derived from dataset)
- `career_start` — year of earliest film in dataset
- `total_films` — count of films in dataset
- `avg_film_rating` — mean Letterboxd rating across their films

**Film**

- `url` — Letterboxd film URL (primary key)
- `title`
- `year`
- `genres` — array
- `letterboxd_rating` — float, out of 5
- `synopsis`
- `poster_url` — from dataset directly
- `directors` — array

### Edges

**APPEARED_IN** (Actor → Film)

- No additional properties in v1 — role (lead/supporting) not available in dataset

**CO_STARRED_WITH** (Actor ↔ Actor) — derived, projected from bipartite

- `film_count` — number of shared films
- `avg_shared_film_rating` — mean Letterboxd rating of shared films
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
│   Neo4j     │    │  pgvector      │
│   Graph DB  │    │  (v2 only)     │
│   Cypher    │    │  Node2Vec      │
│   + GDS     │    │  embeddings    │
└─────────────┘    └────────────────┘
```

---

## API Endpoints

### v1 — Graph queries

| Method | Endpoint                               | What it returns                                  |
| ------ | -------------------------------------- | ------------------------------------------------ |
| `GET`  | `/actor/search?q={name}`               | Matching actors with basic info                  |
| `GET`  | `/actor/{name}/neighborhood?depth={n}` | Ego network subgraph up to N hops                |
| `GET`  | `/actor/{name}/stats`                  | Centrality scores, top co-stars, genre breakdown |
| `GET`  | `/path?from={name}&to={name}`          | Shortest path between two actors                 |
| `GET`  | `/genre/{genre}/top-actors`            | Most central actors in a genre subgraph          |
| `GET`  | `/film/{url}/detail`                   | Film detail with cast network                    |

### v2 — ML recommendations

| Method | Endpoint                | What it returns                            |
| ------ | ----------------------- | ------------------------------------------ |
| `GET`  | `/film/{url}/similar`   | Films similar by graph embedding           |
| `GET`  | `/actor/{name}/similar` | Actors with similar collaboration patterns |

All responses return a consistent graph payload: `{ nodes: [...], edges: [...], meta: {...} }` so the frontend renders without transformation.

---

## Build Phases

### Phase 1 — Data Pipeline

**Goal:** Populated Neo4j database you can query from the command line  
**Deliverable:** Working Cypher queries returning sensible results

1. Load the dataset with `load_dataset("pkchwy/letterboxd-all-movie-data")` from Hugging Face
2. Inspect the schema — understand how `cast`, `directors`, `genres`, and `reviews` are structured
3. Clean the data — drop films with empty `cast` arrays, normalize actor names (strip whitespace, consistent casing), parse `rating` string to float
4. Build actor nodes CSV — one row per unique actor name with derived attributes (career start, total films, avg rating)
5. Build film nodes CSV — one row per film with all fields from the dataset
6. Build edge list CSV — for every film, emit one `(actor)-[APPEARED_IN]->(film)` row per cast member
7. Set up Neo4j locally via Docker — `docker-compose.yml` with Neo4j Community Edition, 4GB heap allocated
8. Bulk import nodes and edges into Neo4j using `neo4j-admin import`
9. Project the co-star graph — use Neo4j GDS to derive `CO_STARRED_WITH` edges from the bipartite graph, computing `film_count` and `avg_shared_film_rating` as edge properties
10. Validate — run spot-check queries (e.g. DiCaprio + Scorsese co-stars, Tilda Swinton genre breakdown) to confirm data is correct

---

### Phase 2 — Backend API

**Goal:** REST API you can hit from Postman and get graph JSON back  
**Deliverable:** All v1 endpoints returning correct responses

1. Set up FastAPI project structure — routers, schemas, db connection module
2. Write Neo4j connection utility — singleton driver, session management, error handling
3. Implement actor search endpoint — full-text index on actor `name` in Neo4j, return top matches
4. Implement neighborhood endpoint — parameterized Cypher query for ego network at depth N; cap depth at 3 to avoid exploding subgraphs
5. Implement shortest path endpoint — Cypher `shortestPath()` between two actor nodes
6. Implement centrality endpoint — query precomputed PageRank scores stored by Neo4j GDS
7. Implement genre top-actors endpoint — filter subgraph by genre, return top N actors by centrality
8. Implement film detail endpoint — return film node with its full cast as a small subgraph
9. Standardize response schema — all graph endpoints return `{ nodes, edges, meta }` consistently
10. Add basic error handling — 404 for unknown actors, timeout handling for deep traversals
11. Test all endpoints with Postman against real data

---

### Phase 3 — Frontend Graph Explorer

**Goal:** React app where you can search an actor, see their network, and navigate by clicking  
**Deliverable:** Working interactive graph in the browser

1. Set up React + TypeScript project with Vite
2. Set up TanStack Query for all API calls and Zustand for graph state (selected node, depth, active filters)
3. Build the search bar — autocomplete against `/actor/search`, select an actor to load their neighborhood
4. Integrate Sigma.js — render the graph payload from the API as a force-directed canvas
5. Style nodes — actor nodes as circles sized by centrality score, film nodes as smaller squares; color by genre
6. Handle node clicks — clicking an actor node triggers a new neighborhood fetch and expands the graph
7. Build the detail panel — clicking a node shows its card (poster for films, stats, top connections) in a sidebar
8. Add depth control — slider or toggle buttons for neighborhood depth (1, 2, or 3 hops)
9. Add genre filter — dropdown to filter visible nodes to a specific genre
10. Add shortest path UI — two actor search fields, a "connect" button, highlight the returned path in the graph
11. Handle performance — limit rendered edges to top 50 by `film_count` weight for large neighborhoods; show "load more" for expansion

---

### Phase 4 — ML Recommendation Layer

**Goal:** "Films like this one" powered by graph embeddings, surfaced in the UI  
**Deliverable:** Working similar films/actors endpoints and recommendation cards in the frontend

1. Run Node2Vec on the co-star graph — train on `CO_STARRED_WITH` edges using the `node2vec` Python library; output 128-dim embeddings per actor; start with a genre-filtered subgraph to validate before full run
2. Derive film embeddings — represent each film as the mean embedding of its cast actors
3. Store embeddings in pgvector — set up Postgres + pgvector extension via Docker, load all actor and film vectors
4. Implement `/film/{url}/similar` endpoint — query pgvector for nearest neighbors by cosine similarity, return top 10
5. Implement `/actor/{name}/similar` endpoint — same approach for actor vectors
6. Surface in the frontend — add a "Similar films" card in the film detail panel and "Similar actors" in the actor panel
7. Evaluate informally — spot-check results for known films (e.g. _There Will Be Blood_ should surface other PTA films and Daniel Day-Lewis films in top 10)

---

## Folder Structure

```
letterboxd-cinema-graph/
│
├── .github/
│   └── workflows/               # CI workflows (placeholder for now)
│
├── data/                        # Never committed to git — fully gitignored
│   ├── raw/                     # Downloaded Hugging Face dataset files
│   ├── processed/               # Cleaned CSVs ready for Neo4j bulk import
│   │   ├── actors.csv           # One row per unique actor (derived attributes)
│   │   ├── films.csv            # One row per film (all dataset fields)
│   │   └── appeared_in.csv      # Edge list: actor_name, film_url
│   └── embeddings/              # Saved Node2Vec vectors (.npy) — Phase 4
│
├── scripts/                     # Offline scripts — run manually, not a live service
│   │
│   ├── pipeline/                # Phase 1 — data pipeline
│   │   ├── load_dataset.py      # Fetch dataset from Hugging Face
│   │   ├── clean.py             # Normalize names, filter nulls, parse ratings
│   │   ├── build_edges.py       # Build actors.csv, films.csv, appeared_in.csv
│   │   └── import_neo4j.py      # Bulk import CSVs into Neo4j + run GDS projection
│   │
│   └── ml/                      # Phase 4 — ML recommendation layer
│       ├── train_node2vec.py        # Train Node2Vec on CO_STARRED_WITH graph
│       ├── build_film_embeddings.py # Derive film vectors from mean cast embeddings
│       └── load_vectors.py          # Push actor + film vectors into pgvector
│
├── backend/                     # Phase 2 — FastAPI application (live service)
│   ├── main.py                  # App entry point, router registration
│   ├── config.py                # Settings loaded from .env via pydantic-settings
│   ├── schemas.py               # Pydantic request/response models
│   ├── db.py                    # Neo4j driver singleton
│   ├── routers/
│   │   ├── actors.py            # /actor/* endpoints
│   │   ├── films.py             # /film/* endpoints
│   │   └── recommendations.py  # /film/{url}/similar, /actor/{name}/similar — Phase 4
│   └── queries/
│       ├── actor_queries.py     # Cypher strings for actor endpoints
│       └── film_queries.py      # Cypher strings for film endpoints
│
├── frontend/                    # Phase 3 — React application (live service)
│   ├── public/                  # Static assets
│   ├── src/
│   │   ├── api/
│   │   │   └── client.ts        # Axios/fetch client + typed API call functions
│   │   ├── components/
│   │   │   ├── GraphCanvas.tsx  # Sigma.js wrapper — renders force-directed graph
│   │   │   ├── SearchBar.tsx    # Actor autocomplete search
│   │   │   ├── DetailPanel.tsx  # Sidebar card for selected actor or film node
│   │   │   ├── PathFinder.tsx   # Two-actor shortest path UI
│   │   │   └── ui/              # Reusable primitives
│   │   │       ├── Button.tsx
│   │   │       ├── Card.tsx
│   │   │       └── Badge.tsx
│   │   ├── hooks/               # TanStack Query data hooks, one per API call
│   │   │   ├── useActorSearch.ts        # Calls /actor/search
│   │   │   ├── useActorNeighborhood.ts  # Calls /actor/{name}/neighborhood
│   │   │   └── useShortestPath.ts       # Calls /path
│   │   ├── store/
│   │   │   └── graphStore.ts    # Zustand — selected node, depth, active filters
│   │   └── types/
│   │       └── graph.ts         # Shared TypeScript interfaces (Node, Edge, GraphPayload)
│   └── vite.config.ts
│
├── docker-compose.yml           # Neo4j + pgvector local setup
├── .env.example                 # Environment variable template (no secrets)
├── .gitignore                   # Ignores data/, .env, node_modules, __pycache__
└── README.md
```

---

## Environment Variables

```
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password
PGVECTOR_CONNECTION_STRING=postgresql://user:pass@localhost:5432/cinegraph   # v2 only
```

---

## Key Constraints and Watch-outs

**Graph size at scale**
The full dataset will produce millions of co-star edges. Always serve ego-networks from the API — never dump the full graph to the frontend. Cap neighborhood depth at 3 hops maximum.

**Actor identity**
The dataset has no actor IDs — actors are identified by name string only. Names must be normalized consistently (whitespace, casing) during cleaning or the same actor will appear as multiple nodes.

**Neo4j memory**
Give Neo4j at least 4GB heap in the Docker config. Set this in `docker-compose.yml` via `NEO4J_server_memory_heap_max__size=4G`.

**Node2Vec training time**
Training on the full co-star graph can take several hours on a laptop. Always validate the pipeline on a genre-filtered subgraph first (e.g. Drama only), then run full training overnight.

**Letterboxd data terms**
The dataset is MIT licensed for educational and research use. Check Letterboxd's terms of service before deploying publicly — their underlying data is sourced from TMDB and carries its own attribution requirements.

---

## Success Criteria per Phase

| Phase   | Done when...                                                                                              |
| ------- | --------------------------------------------------------------------------------------------------------- |
| Phase 1 | `MATCH (a:Actor {name: "Cate Blanchett"})-[:CO_STARRED_WITH]-(b) RETURN b` returns correct co-stars       |
| Phase 2 | All v1 Postman requests return correct `{ nodes, edges, meta }` JSON within 500ms                         |
| Phase 3 | Can search "Tilda Swinton", see her network, click a co-star, navigate three hops, and find the path back |
| Phase 4 | Similar films for _Mulholland Drive_ returns other Lynch films and films with Naomi Watts in top 10       |
