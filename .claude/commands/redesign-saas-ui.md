---
description: Redesign the Vue 3 app UI into a modern SaaS-style interface with a vertical sidebar navigation
---

Redesign this Vue 3 application's UI into a polished, modern SaaS-style interface. Replace the horizontal top nav bar with a fixed vertical sidebar on the left. Preserve all existing functionality — only change layout and visual styling.

## Step 1 — Read before touching anything

Read these files in full before writing a single line:
- `client/src/App.vue` — current nav structure, global styles, component imports
- `client/src/main.js` — all routes and their paths
- Any view files referenced in the router to understand page titles and icons needed

## Step 2 — Design spec to implement

### Layout structure

```
┌─────────────────────────────────────────────────────┐
│  SIDEBAR (260px fixed) │  MAIN CONTENT (flex: 1)    │
│  ─────────────────────  │  ─────────────────────     │
│  [Logo / Brand]         │  [FilterBar if present]    │
│                         │                            │
│  [Nav items]            │  <router-view />           │
│    - Overview           │                            │
│    - Inventory          │                            │
│    - Orders             │                            │
│    - Finance            │                            │
│    - Demand Forecast    │                            │
│    - Reports            │                            │
│    - Restocking         │                            │
│                         │                            │
│  [bottom spacer]        │                            │
│  [Language switcher]    │                            │
│  [Profile menu]         │                            │
└─────────────────────────────────────────────────────┘
```

### Sidebar CSS spec
- Width: 260px, fixed position, full viewport height
- Background: `#0f172a` (dark navy) — gives strong visual contrast
- Border-right: none (shadow instead: `box-shadow: 2px 0 8px rgba(0,0,0,0.12)`)
- z-index: 100

### Brand / logo area (top of sidebar)
- Padding: 24px 20px 20px
- Company name: white, font-size 1.125rem, font-weight 700
- Subtitle: `#94a3b8`, font-size 0.75rem, margin-top 2px
- Bottom border: `1px solid rgba(255,255,255,0.08)`

### Nav items
- Each `<router-link>` gets padding: 10px 16px, border-radius 8px, margin: 2px 12px
- Default color: `#94a3b8`, font-size 0.875rem, font-weight 500
- Hover: background `rgba(255,255,255,0.06)`, color `#e2e8f0`
- Active (`.active`): background `rgba(59,130,246,0.2)`, color `#60a5fa`, font-weight 600
- Each nav item has a small inline SVG icon (16×16) to the left of the label
- Nav section label (optional): uppercase `#475569`, font-size 0.65rem, letter-spacing 1px, padding 16px 20px 6px — group items under "MENU" or "MANAGEMENT"

### SVG icons to use (simple, clean, 16×16 viewBox)
Use these inline SVG paths for each route:
- Overview/Dashboard: `<path d="M3 3h5v5H3zm0 9h5v5H3zm9-9h5v5h-5zm0 9h5v5h-5z"/>` (grid)
- Inventory: `<path d="M21 16V8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16z"/>` (box)
- Orders: `<path d="M9 5H7a2 2 0 0 0-2 2v12a2 2 0 0 0 2 2h10a2 2 0 0 0 2-2V7a2 2 0 0 0-2-2h-2M9 5a2 2 0 0 0 2 2h2a2 2 0 0 0 2-2M9 5a2 2 0 0 0 2-2h2a2 2 0 0 0 2 2"/>` (clipboard)
- Finance/Spending: `<path d="M12 2v20M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"/>` (dollar)
- Demand: `<polyline points="22 12 18 12 15 21 9 3 6 12 2 12"/>` (trend line)
- Reports: `<path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/>` (file)
- Restocking: `<path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/><circle cx="12" cy="12" r="3"/>` (refresh/restock — or use: `<polyline points="23 4 23 10 17 10"/><polyline points="1 20 1 14 7 14"/><path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"/>`)

### Bottom of sidebar
- Flex-column with margin-top: auto to push to bottom
- `LanguageSwitcher` styled for dark sidebar (invert colors if needed, or wrap in a div with padding 12px 16px)
- `ProfileMenu` trigger button: show user avatar initials circle + name, styled for dark bg
- Padding-bottom: 16px

### Main content area
- margin-left: 260px (matches sidebar width)
- Padding: 0 (let each view handle its own padding)
- `FilterBar` sits at the top of the content area inside a white sticky bar if it exists, or just flows normally
- `<router-view>` below

### Global style updates (update the `<style>` block in App.vue)
- Remove all `.top-nav`, `.nav-container`, `.nav-tabs` styles
- Add `.sidebar`, `.sidebar-brand`, `.sidebar-nav`, `.nav-item`, `.nav-section-label`, `.sidebar-bottom`, `.content-area` styles
- Keep all card, table, badge, stat-card, loading, error, page-header styles — those are used by views
- Add CSS variables at `:root` for sidebar width: `--sidebar-width: 260px`
- Smooth page transitions: add `transition: margin 0.2s ease` on `.content-area`

## Step 3 — Use vue-expert for all Vue file edits

You MUST use the vue-expert subagent to make all changes to `.vue` files. Provide it with:
1. The full content of the current `App.vue`
2. The exact design spec above
3. Instructions to rewrite the App.vue template and styles while keeping the script section identical (same imports, same setup logic, same components registered)

The vue-expert should produce a complete rewritten `App.vue` — not a patch.

## Step 4 — Verify

After the agent completes:
1. Check that `client/src/App.vue` was actually written
2. Grep for `.top-nav` in App.vue — it should be gone from the template
3. Grep for `.sidebar` in App.vue — it should exist
4. Confirm `<router-view>` is still present in the template
5. Confirm `FilterBar`, `ProfileMenu`, `TasksModal`, `ProfileDetailsModal`, `LanguageSwitcher` components are still in the template

## Step 5 — Report

Tell the user:
- What changed (layout structure, color scheme, removed elements, added elements)
- That they can preview at http://localhost:3000
- Any caveats (e.g. FilterBar positioning, mobile responsiveness not addressed)
