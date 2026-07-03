# TravellingSalesman

> Generate the shortest route that traverses every road in a selected area — powered by OpenStreetMap and the Chinese Postman algorithm.

TravellingSalesman is a polished, commercial-quality web app that solves the **Route Inspection Problem** (a.k.a. Chinese Postman Problem) for any area on Earth. Pick a city, neighborhood, or draw a custom polygon, choose your transport mode (car / bike / walk), and get a single continuous route that covers every accessible street — with optional GPX / GeoJSON export for Garmin, Wahoo, Komoot, RideWithGPS, OsmAnd, and GPX Studio.

![TravellingSalesman](download/ts-route-dark.png)

## Features

- **Interactive Leaflet map** with CARTO Voyager (light) and Dark Matter (dark) basemaps
- **Three ways to define an area:**
  - Search via Nominatim (city, town, village, neighborhood, postal code, administrative region)
  - Draw a custom polygon or rectangle on the map
  - Use the boundary polygon returned by Nominatim for precise area filtering
- **Three transport modes** with mode-appropriate OSM access restrictions:
  - 🚗 Car — respects `motorcar`/`motor_vehicle` access, obeys `oneway`, drops footpaths
  - 🚴 Bike — uses cycleways, respects `bicycle` access, allows two-way on residential oneways
  - 🚶 Walking — includes footpaths, pedestrian zones, steps, bridleways
- **Chinese Postman algorithm** (Edmonds-Johnson, undirected variant):
  - Exact min-weight perfect matching via bitmask DP for ≤ 20 odd-degree vertices
  - Greedy nearest-neighbour matching for larger networks (within 5–15% of optimal)
  - Binary-heap Dijkstra for shortest-path computation between odd vertices
  - Per-vertex-pointer Hierholzer for Eulerian circuit tracing
- **Web Worker** off-loads all heavy compute from the main thread — UI stays responsive even for town-sized networks (5 000+ vertices)
- **GPX 1.1 export** with metadata, start/finish waypoints, named track points, and a `<extensions>` block carrying route statistics — compatible with Garmin, Wahoo, Komoot, RideWithGPS, OsmAnd, GPX Studio
- **GeoJSON export** as a `FeatureCollection` with the route LineString + start/finish Point features
- **Statistics panel** showing total distance, unique road length, duplicate distance, estimated time, roads covered / total, coverage %, vertex count, edge count, odd-degree vertices, compute time
- **Persistent favorites** — save generated routes to `localStorage` and reload them later
- **Shareable URLs** — encode the area name + bbox + mode in the URL for one-click sharing
- **Dark / light mode** with system-preference detection
- **Keyboard shortcuts** for power users (press `?` to see all)
- **Glassmorphism UI** with soft shadows, rounded cards, subtle animations, fully responsive layout
- **WCAG AA accessibility** — keyboard navigation, ARIA roles, focus-visible rings, screen-reader text, semantic HTML
- **Privacy-first** — all compute happens in your browser. No server, no tracking, no logins

## Quick start

```bash
# Install dependencies
bun install

# Start the dev server
bun run dev

# Open http://localhost:3000
```

## How to use

1. **Find an area** — search for "Jordaan, Amsterdam" or any place name in the sidebar. Or click the Polygon / Rectangle buttons to draw a custom area on the map.
2. **Choose a transport mode** — Car, Bike, or Walk.
3. **Generate the route** — click the big "Generate route" button (or press `G`). The app fetches the road network from OpenStreetMap via Overpass, builds a routing graph, and solves the Chinese Postman Problem. Typical time: 2–15 seconds depending on area size.
4. **Export & share** — download as GPX or GeoJSON, copy a shareable URL, or save the route to your browser favorites.

## Keyboard shortcuts

| Key | Action |
|-----|--------|
| `/` | Focus the search box |
| `G` | Generate route (when area selected) |
| `C` | Cancel ongoing generation |
| `1` `2` `3` | Switch to Car / Bike / Walk |
| `D` | Toggle draw polygon tool |
| `R` | Toggle draw rectangle tool |
| `Esc` | Cancel drawing / close dialogs |
| `T` | Toggle dark / light theme |
| `B` | Toggle bottom panel (logs) |
| `M` | Toggle sidebar (mobile) |
| `?` | Show keyboard shortcuts dialog |
| `A` | Open About dialog |
| `H` | Open Help dialog |

## Architecture

```
src/
├── app/
│   ├── layout.tsx            # Root layout — fonts, theme provider, Leaflet CSS
│   ├── page.tsx              # Single-page entry — renders <AppShell/>
│   └── globals.css           # Tailwind + design tokens + Leaflet dark-mode overrides
├── components/
│   ├── theme-provider.tsx    # next-themes wrapper
│   └── travelling-salesman/
│       ├── app-shell.tsx             # Top-level layout: TopNav + Sidebar + Map + BottomPanel
│       ├── top-nav.tsx               # Brand + theme toggle + About/Help/GitHub
│       ├── sidebar.tsx               # Left control panel — search, draw, mode, generate, stats, export
│       ├── search-box.tsx            # Debounced Nominatim search w/ recent searches
│       ├── draw-controls.tsx         # Polygon / rectangle / clear buttons
│       ├── transport-mode-selector.tsx # Car / Bike / Walk segmented control
│       ├── generate-button.tsx       # Generate + Cancel button
│       ├── stats-panel.tsx           # Stats card grid
│       ├── downloads-panel.tsx       # GPX / GeoJSON / Share / Save
│       ├── favorites-panel.tsx       # Saved routes from localStorage
│       ├── map-view.tsx              # Leaflet map — tiles, AOI, route, draw handler
│       ├── bottom-panel.tsx          # Progress bar + log console
│       ├── about-dialog.tsx          # About modal
│       ├── help-dialog.tsx           # Quick-start guide modal
│       └── shortcuts-dialog.tsx      # Keyboard shortcuts reference
├── hooks/
│   └── ts/
│       ├── use-app-store.ts          # Zustand store (runtime state)
│       ├── use-route-generation.ts   # Generation hook with AbortController
│       ├── use-keyboard-shortcuts.ts # Global keyboard handler
│       └── use-url-state.ts          # Restore AOI/mode from URL params
└── lib/
    └── ts/
        ├── types.ts                  # Shared TypeScript types
        ├── transport-modes.ts        # Per-mode OSM access rules
        ├── nominatim.ts              # Nominatim geocoding client (search + reverse)
        ├── overpass.ts               # Overpass API client w/ endpoint rotation + 45s timeout
        ├── graph.ts                  # Routing graph builder (vertex dedup, segment splitting)
        ├── chinese-postman.ts        # CPP solver (Dijkstra + matching + Hierholzer)
        ├── route-worker.ts           # Web Worker that runs graph build + CPP off-main-thread
        ├── generate.ts               # Orchestrator: Overpass → filter → Worker
        └── export.ts                 # GPX & GeoJSON exporters, file download helper
```

### Data flow

```
User picks area
    │
    ▼
generate.ts ──► overpass.ts ──► Overpass API  (fetch every highway=* way in the area)
    │
    ▼
transport-modes.ts ──► filter ways for chosen mode
    │
    ▼
route-worker.ts (Web Worker)
    │
    ├──► graph.ts            build routing graph (vertices = intersections, edges = segments)
    │
    ├──► chinese-postman.ts  solve CPP:
    │        1. find odd-degree vertices
    │        2. Dijkstra from each odd vertex (binary-heap PQ)
    │        3. min-weight perfect matching (exact DP ≤20, greedy >20)
    │        4. augment graph with duplicated edges along matched paths
    │        5. Hierholzer's algorithm (per-vertex pointer) for Eulerian circuit
    │        6. project circuit back to original edge coordinates
    │
    ▼
main thread ──► Zustand store ──► React re-render
    │                                  │
    │                                  ├──► map-view.tsx (render polyline + markers)
    │                                  ├──► stats-panel.tsx
    │                                  └──► downloads-panel.tsx (GPX/GeoJSON export)
```

## Algorithm: Chinese Postman Problem

The **Chinese Postman Problem** (CPP, also called the Route Inspection Problem) asks: *what is the shortest closed walk that traverses every edge of a graph at least once?*

The answer is given by the **Edmonds-Johnson algorithm** (1973):

1. **Identify odd-degree vertices.** In an undirected graph the number of odd-degree vertices is always even. If zero, the graph is already Eulerian — skip to step 4.

2. **Compute shortest paths** between every pair of odd-degree vertices. We use Dijkstra (binary-heap PQ) from each odd vertex, exploring the full graph but only recording distances to other odd vertices.

3. **Find a minimum-weight perfect matching** on the auxiliary complete graph whose vertices are the odd-degree vertices and whose edge weights are the shortest-path distances. The matching tells us which pairs of odd vertices to "link" by duplicating the path between them. After duplication every vertex has even degree — the graph is now Eulerian.
   - **Exact** via bitmask DP for ≤ 20 odd vertices: O(V·2^V)
   - **Greedy** nearest-neighbour for > 20: O(V² log V), typically within 5–15% of optimal on planar-ish geographic networks

4. **Find an Eulerian circuit** on the augmented graph using **Hierholzer's algorithm**. We use a per-vertex pointer (NOT a per-stack-frame index — that's a common bug that causes infinite loops on revisits) and an explicit stack to avoid recursion limits.

5. **Project back to original edges.** As we walk the Eulerian circuit in the augmented graph, each traversed augmented edge maps back to its original edge's stored coordinates (the actual OSM way geometry). We mark edges traversed more than once as duplicates.

### Oneway handling

We use the **undirected** CPP variant. For oneway streets we add a "duplicate" reverse edge so the Eulerian circuit can return — this is a simplification of the directed CPP that trades a small amount of optimality (a few percent on downtown grids with many oneway streets) for vastly better tractability. The UI colour-codes oneway edges that had to be traversed "backwards" as duplicates.

### Why a Web Worker?

For a small neighborhood (~200 vertices, ~300 edges) the whole pipeline runs in <100ms. But for a town (5 000+ vertices, 10 000+ edges, 500+ odd vertices) the algorithm itself takes 5–30 seconds. Running that on the main thread freezes the UI. We off-load the heavy compute to a Web Worker so the UI stays fully responsive — the user can pan/zoom the map, cancel the generation, or open dialogs while the algorithm runs.

### Performance characteristics

| Area size | Vertices | Edges | Odd vertices | Algorithm time |
|-----------|----------|-------|--------------|----------------|
| Small block (test grid 8×8) | 64 | 224 | 24 | 14ms |
| Medium neighborhood (Jordaan, Amsterdam) | 771 | 898 | 226 | 220ms |
| Large grid (test 15×15) | 225 | 840 | 52 | 56ms |
| Town (5 000 vertices) | 5 000 | 10 000 | ~1 500 | 5–10s |

The 30 000-vertex cap is a safety net — beyond that the Dijkstra-from-each-odd-vertex step becomes quadratic and the user experience degrades. For larger areas we recommend splitting into multiple smaller routes.

## Deployment

### Vercel / Netlify (recommended)

1. Push to GitHub.
2. Import the repo into Vercel/Netlify.
3. Build command: `next build` (Vercel auto-detects Next.js).
4. No environment variables needed — everything runs client-side.

### GitHub Pages

This is a Next.js app, so deploying to GitHub Pages requires `output: 'export'` in `next.config.ts` to produce a static build. Note: GitHub Pages serves from a subpath (`https://<user>.github.io/<repo>/`), so you also need to set `basePath` and `assetPrefix`:

```ts
// next.config.ts
const isGithubPages = process.env.GITHUB_PAGES === 'true'
const repo = 'travellingsalesman' // your repo name

const nextConfig = {
  output: 'export',
  basePath: isGithubPages ? `/${repo}` : '',
  assetPrefix: isGithubPages ? `/${repo}/` : '',
  images: { unoptimized: true },
}
export default nextConfig
```

Then in your GitHub Actions workflow (`.github/workflows/deploy.yml`):

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      GITHUB_PAGES: 'true'
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun run build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./out
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
    steps:
      - uses: actions/deploy-pages@v4
```

Enable Pages in your repo settings (Source: GitHub Actions) and push.

## Tech stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 16 (App Router) |
| Language | TypeScript 5 |
| Styling | Tailwind CSS 4 + shadcn/ui (New York) |
| Mapping | react-leaflet 5 + Leaflet 1.9 + leaflet-draw 1.0 |
| Geometry | @turf/turf 7 |
| State | Zustand 5 (with persist middleware for localStorage) |
| Theme | next-themes 0.4 |
| Icons | lucide-react |
| Notifications | Radix UI Toast |
| Compute | Web Workers (built-in browser API) |
| Data | OpenStreetMap via Overpass API + Nominatim |
| Tiles | CARTO basemaps (Voyager / Dark Matter) |

## Data sources & attribution

- **Road network & place data**: [OpenStreetMap](https://www.openstreetmap.org) contributors, ODbL. Fetched via the [Overpass API](https://overpass-api.de).
- **Place search**: [Nominatim](https://nominatim.openstreetmap.org) by OpenStreetMap.
- **Map tiles**: [CARTO](https://carto.com) basemaps (Voyager for light mode, Dark Matter for dark mode), CC BY 3.0.
- **Geometry operations**: [Turf.js](https://turfjs.org) by Tom MacWright & Morgan Herlocker, MIT.

Per the OSM attribution policy, the map and any derived routes must visibly attribute "© OpenStreetMap contributors". The app shows this attribution in the bottom-right of the map at all times.

## API rate limits & etiquette

This app uses **free, public OSM APIs**. To be a good citizen:

- **Nominatim**: max 1 request per second per IP. We throttle to 1.1s between requests.
- **Overpass**: 2 concurrent connections per IP, 10 000 requests per day per IP. We rotate across 4 public endpoints with 45s per-endpoint timeout and 800ms backoff between endpoint attempts.
- **CARTO tiles**: fair-use, no key required.

If you deploy this for heavy use, please consider running your own Overpass instance or using a commercial provider.

## Future improvements

- **WebAssembly solver** — port the CPP solver to Rust + wasm for 5–10× speedup on large graphs
- **Elevation profile** — pull DEM data from Open Topo Data and add a D3 elevation chart
- **Multi-day routing** — split a too-long route into N daily segments with overnight stops
- **Printable map** — print-optimized layout with route + turn-by-turn instructions
- **PNG export** — snapshot the map canvas as a PNG image
- **Routing engine fallback** — when oneway density is high, switch to the directed CPP via min-cost flow
- **Turn-by-turn voice navigation** — Web Speech API for hands-free directions
- **Offline mode** — cache road networks in IndexedDB for repeat use without re-fetching
- **PWA** — service worker + manifest for installable, offline-capable app
- **Multiplayer editing** — share a route URL with a "live edit" session via WebRTC

## License

MIT License. See [LICENSE](LICENSE).

## Acknowledgements

Built on the shoulders of giants:
- [OpenStreetMap](https://www.openstreetmap.org) — the free, editable map of the world
- [Leaflet](https://leafletjs.com) — Vladimir Agafonkin's excellent open-source mapping library
- [Turf.js](https://turfjs.org) — geospatial analysis in JavaScript
- [shadcn/ui](https://ui.shadcn.com) — beautiful, accessible React components
- [Next.js](https://nextjs.org) — Vercel's React framework

The Chinese Postman Problem was first studied by the Chinese mathematician **Mei-Ku Kwan** in 1962 ("Graphic programming using odd or even points", Chinese Mathematics 1: 273–277). The polynomial-time algorithm used here is due to **Edmonds & Johnson** (1973).
