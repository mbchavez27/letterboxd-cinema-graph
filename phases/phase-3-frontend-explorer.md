# Phase 3 — Frontend Graph Explorer

## Goal

React app where you can search an actor, see their network, and navigate by clicking.

**Deliverable:** Working interactive graph in the browser.

---

## Steps

| # | Step | File |
|---|------|------|
| 1 | Set up React + TypeScript project with Vite | `frontend/` scaffold |
| 2 | Set up TanStack Query for all API calls and Zustand for graph state | `frontend/src/hooks/`, `frontend/src/store/` |
| 3 | Build the search bar — autocomplete against `/actor/search` | `frontend/src/components/SearchBar.tsx` |
| 4 | Integrate Sigma.js — render the graph payload from the API as a force-directed canvas | `frontend/src/components/GraphCanvas.tsx` |
| 5 | Style nodes — actor nodes as circles sized by centrality, film nodes as smaller squares; color by genre | `frontend/src/components/GraphCanvas.tsx` |
| 6 | Handle node clicks — clicking an actor triggers a new neighborhood fetch and expands the graph | `frontend/src/components/GraphCanvas.tsx` |
| 7 | Build the detail panel — clicking a node shows its card in a sidebar | `frontend/src/components/DetailPanel.tsx` |
| 8 | Add depth control — slider or toggle buttons for neighborhood depth (1, 2, or 3 hops) | `frontend/src/components/` |
| 9 | Add genre filter — dropdown to filter visible nodes to a specific genre | `frontend/src/components/` |
| 10 | Add shortest path UI — two actor search fields, a "connect" button, highlight the returned path | `frontend/src/components/PathFinder.tsx` |
| 11 | Handle performance — limit rendered edges to top 50 by `film_count` weight; show "load more" | `frontend/src/` |

---

## Project Structure

```
frontend/
├── public/                  # Static assets
├── src/
│   ├── api/
│   │   └── client.ts        # Fetch client + typed API call functions
│   ├── components/
│   │   ├── GraphCanvas.tsx  # Sigma.js wrapper — renders force-directed graph
│   │   ├── SearchBar.tsx    # Actor autocomplete search
│   │   ├── DetailPanel.tsx  # Sidebar card for selected actor or film node
│   │   ├── PathFinder.tsx   # Two-actor shortest path UI
│   │   └── ui/              # Reusable primitives
│   │       ├── Button.tsx
│   │       ├── Card.tsx
│   │       └── Badge.tsx
│   ├── hooks/               # TanStack Query data hooks, one per API call
│   │   ├── useActorSearch.ts
│   │   ├── useActorNeighborhood.ts
│   │   └── useShortestPath.ts
│   ├── store/
│   │   └── graphStore.ts    # Zustand — selected node, depth, active filters
│   └── types/
│       └── graph.ts         # Shared TypeScript interfaces (Node, Edge, GraphPayload)
└── vite.config.ts
```

---

## Dependencies

- React 18+
- TypeScript
- Sigma.js (graph rendering)
- TanStack Query (data fetching, caching)
- Zustand (global state)
- Tailwind CSS (styling)
- Vite (build tool)

Install (Phase 3):

```bash
cd frontend
npm install
```

---

## Running

```bash
cd frontend
npm run dev
```

App available at `http://localhost:5173`

---

## Success Criteria

Can search "Tilda Swinton", see her network, click a co-star, navigate three hops, and find the path back.
