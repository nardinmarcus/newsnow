# NewsNow — AI Coding Guide

## Project Overview

NewsNow is a real-time news aggregation dashboard. Frontend in React (Vite), backend in Nitro (h3), deployable to Cloudflare Pages, Vercel, Bun, or Node.js. Supports MCP server for AI agent integration.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 19, Vite 7, TanStack Router, Jotai, UnoCSS, Framer Motion |
| Backend | Nitro (h3), built via `vite-plugin-with-nitro` |
| Database | Cloudflare D1 (prod), better-sqlite3 (local dev), bun-sqlite (Bun) |
| Auth | GitHub OAuth + JWT (jose) |
| Package Manager | pnpm 10 |
| Language | TypeScript 5.9 strict |

## Directory Map

```
shared/          → Shared code (types, sources config, utils) — used by BOTH frontend & backend
  types.ts       → Core types: SourceID, Source, NewsItem, SourceResponse, etc.
  sources.ts     → Generated source registry (from sources.json)
  pre-sources.ts → Source definitions with column/type/sub metadata
  consts.ts      → TTL (30min), Interval (10min), Site info
  site.ts        → Site metadata (name, URL, repo, author, copyright)
  type.util.ts   → TypeScript utility types (UnionToIntersection, etc.)
server/          → Nitro backend (import alias: `#/`)
  api/           → API route handlers
    s/index.ts   → GET /api/s?id=xxx — fetch single source items
    s/entire.post.ts → POST /api/s/entire — fetch multiple sources at once
    oauth/github.ts → GitHub OAuth callback
    latest.ts    → GET /api/latest
    login.ts, me.ts, enable-login.ts → Auth endpoints
    mcp.post.ts  → MCP server HTTP endpoint
  database/      → cache.ts (news cache with TTL), user.ts (GitHub user data)
  mcp/server.ts  → MCP server definition (tool: get_hotest_latest_news)
  sources/       → Individual source fetchers (one file per source/platform)
  middleware/    → Auth middleware (JWT verification)
  utils/         → Server-side utilities
src/             → React frontend (import alias: `~/`)
  components/    → UI components (column/, header/, footer/, common/)
  routes/        → TanStack Router file-based routes
  hooks/         → Custom hooks (auto-imported)
  atoms/         → Jotai atoms (auto-imported)
  styles/        → CSS/UnoCSS
scripts/         → Build scripts (favicon.ts, source.ts — regenerate sources.json)
public/          → Static assets (favicons, PWA icons, OG image)
```

## Key Architecture

### Build Pipeline
1. `pnpm presource` runs first: generates `shared/sources.json` from `shared/pre-sources.ts` + favicons
2. Vite builds the React frontend
3. Nitro builds the server side (bundled into `dist/output/`)
4. `dist/output/public/` → static frontend; `dist/output/server/` → Node server entry

### Source System
- Sources are defined in `shared/pre-sources.ts` with columns, sub-sources, types (`hottest`/`realtime`)
- Each source has a fetcher in `server/sources/<name>.ts` using `defineSource()`
- The `NewsItem` interface (in `shared/types.ts`) is the universal return shape
- To add a source: add config in `pre-sources.ts` + implement fetcher in `server/sources/` + run `pnpm presource`

### Database
- Uses db0 (unjs) for connector abstraction
- Local: better-sqlite3 → file `db.sqlite` in project root
- Cloudflare: D1 with binding `NEWSNOW_DB`
- Vercel: no built-in DB (user must configure manually)
- Bun: bun-sqlite
- Tables: cache (news items with TTL), user (GitHub user + JWT token)

### Environment Variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `G_CLIENT_ID` | For OAuth | GitHub OAuth App Client ID |
| `G_CLIENT_SECRET` | For OAuth | GitHub OAuth App Client Secret |
| `JWT_SECRET` | For OAuth | JWT signing secret (⚠️  ensure consistent across deployments, or every restart will invalidate existing login sessions) |
| `INIT_TABLE` | First run | Set to `true` to create DB tables |
| `ENABLE_CACHE` | Optional | Enable news caching (default: true in CF) |

### Deployment Presets

| Platform | Env var | Database | Notes |
|----------|---------|----------|-------|
| Cloudflare Pages | `CF_PAGES=1` | D1 | wrangler.toml configured |
| Vercel | `VERCEL=1` | None by default | Must configure external DB |
| Bun | `BUN=1` | bun-sqlite | |
| Docker/Node | (none) | better-sqlite3 | `docker compose up` |

## Rules & Constraints

- **Do NOT edit `shared/sources.json` directly** — it's generated. Edit `shared/pre-sources.ts` and run `pnpm presource`
- **`shared/` code runs in both browser and Node** — never use Node-only APIs here
- **All source fetchers must return `NewsItem[]`** — see `shared/types.ts` for the interface
- **API routes are auto-discovered by Nitro** from `server/api/` directory structure — just create the file
- **Auto-imports are configured in vite.config.ts** — `src/hooks/`, `shared/`, `src/utils/`, `src/atoms/` are auto-imported in frontend. Server auto-imports `server/utils/` and `shared/` via nitro.config.ts
- **Package manager is pnpm** — `packageManager` field is set, use `pnpm` not npm/yarn
- **wrangler.toml vars override `.env.server`** when deploying to Cloudflare Pages — keep them in sync
- **JWT_SECRET must be consistent** — changing it invalidates all existing sessions. Set it once and never rotate unless you're okay with logging everyone out
- **Cache TTL is 30 min** (`shared/consts.ts`), refresh interval 10 min — logged-in users can force-refresh

## Common Tasks

### Add a new data source
1. Add source config to `shared/pre-sources.ts`
2. Create/update fetcher in `server/sources/<name>.ts` using `defineSource()`
3. Run `pnpm presource` to regenerate `sources.json`
4. Run `pnpm dev` to test

### Run locally
```sh
pnpm dev
```
Starts both Vite (frontend HMR) and Nitro (backend). Make sure `.env.server` exists with required vars.

### Build for production
```sh
pnpm build
```

### MCP server
The MCP server is embedded — run the project normally, then configure MCP clients:
```json
{
  "mcpServers": {
    "newsnow": {
      "command": "npx",
      "args": ["-y", "newsnow-mcp-server"],
      "env": { "BASE_URL": "https://newsnow.namooca.com" }
    }
  }
}
```
Or access via HTTP POST to `/api/mcp`.

### Deployment
```sh
pnpm deploy   # Cloudflare Pages
# or
docker compose up   # Docker
```

## Project Conventions

- **ESLint**: Uses `@ourongxing/eslint-config` with React plugins
- **Formatting**: Pre-commit lint-staged runs `eslint --fix` on all files
- **Git hooks**: simple-git-hooks runs lint-staged on pre-commit
- **Versioning**: `pnpm release` uses bumpp for version bumps
- **Testing**: Vitest, config in `vitest.config.ts`
