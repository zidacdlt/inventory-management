# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System - Full-stack demo with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

## Critical Tool Usage Rules

### Subagents
- **vue-expert**: **MANDATORY** for any `.vue` file creation or significant modification
- **code-reviewer**: Use after writing significant code
- **Explore**: Use for codebase structure questions and pattern searches
- **backend-api-test** skill: Use when writing/modifying tests in `tests/backend/`

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for all GitHub operations
  - Exception: local branches — use `git checkout -b`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Frontend: `http://localhost:3000`, API: `http://localhost:8001`

## Commands

```bash
# Backend (run from server/)
uv run python main.py          # dev server on port 8001

# Frontend (run from client/)
npm install && npm run dev     # dev server on port 3000
npm run build                  # production build → client/dist/

# Tests (run from tests/)
uv run pytest backend/ -v                          # all backend tests
uv run pytest backend/test_inventory.py -v         # single file
uv run pytest backend/ -k "test_name" -v           # single test by name
```

On macOS/Linux, `./scripts/start.sh` starts both servers. Windows requires two separate terminals.

## Architecture

**Stack**: Vue 3 + Vite (port 3000) → FastAPI (port 8001) → in-memory JSON data

**Data flow**: Vue filter composable → `client/src/api.js` (axios) → FastAPI query params → `apply_filters()` helper → Pydantic model → response

**Filter system**: 4 global filters (Time Period, Warehouse, Category, Order Status) live in the `useFilters` composable as a singleton. All views read from it; `FilterBar.vue` writes to it. Filters pass as query params — a value of `'all'` skips that filter in the backend.

**Reactivity pattern**: Raw API data stored in refs (`allOrders`, `inventoryItems`). All derived values (totals, filtered lists, chart data) are `computed` properties — never mutate raw data directly.

**State composables** (`client/src/composables/`):
- `useFilters.js` — filter refs + `getCurrentFilters()` / `resetFilters()`
- `useAuth.js` — mock auth, hardcoded user (John Doe / 田中太郎), language-aware tasks
- `useI18n.js` — locale switching (en/ja), persisted to localStorage, USD↔JPY at 150

**Backend data**: All JSON files in `server/data/` are loaded once at startup via `server/mock_data.py`. Mutations are in-memory only — restart to reset. To regenerate sample data run `server/generate_data.py`.

**Month filtering**: Supports `YYYY-MM` format and quarters (`Q1-2025`). Quarter-to-month mapping lives in `QUARTER_MAP` in `server/main.py`. Inventory endpoints do **not** support month filtering (no time dimension on inventory).

**Unimplemented API stubs**: `api.js` has methods for tasks, purchase orders, and backlog purchase orders that have no backend counterpart — they will 404.

## API Endpoints

| Endpoint | Filters |
|---|---|
| `GET /api/inventory` | warehouse, category |
| `GET /api/orders` | warehouse, category, status, month |
| `GET /api/dashboard/summary` | warehouse, category, status, month |
| `GET /api/demand` | none |
| `GET /api/backlog` | none |
| `GET /api/spending/{summary,monthly,categories,transactions}` | none |
| `GET /api/reports/{quarterly,monthly-trends}` | none |

## Design System

- Colors: Slate/gray (`#0f172a`, `#64748b`, `#e2e8f0`)
- Status colors: green (delivered), blue (shipped), yellow (processing), red (backordered)
- Charts: custom SVG; layouts: CSS Grid; no external UI framework; no emojis in UI
- Global styles in `client/src/App.vue`; component styles are scoped

## Key Constraints

- Revenue goals hard-coded: $800K/month (single warehouse), $9.6M YTD (all months)
- Always use unique `v-for` keys (`sku`, `month`, etc.) — never `index`
- Validate dates before calling `.getMonth()` — API dates can be null
- When changing JSON data shape, update the corresponding Pydantic model in `server/main.py`
- Warehouses: San Francisco, London, Tokyo — Categories: Circuit Boards, Sensors, Actuators, Controllers, Power Supplies
