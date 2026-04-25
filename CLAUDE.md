# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

No build step. Open `index.html` directly in a browser:

```powershell
Start-Process "index.html"
```

To test changes: refresh the browser tab. There is no dev server, bundler, or package manager.

## Architecture

Everything lives in a single file: **`index.html`** (inline HTML + CSS + JS). There are no external JS dependencies, no modules, and no build tooling.

### Data flow

1. User fills the search form → `startSearch()` builds `searchStringsArray` via `buildQueries()`
2. `apiPost()` starts an Apify actor run (`compass/crawler-google-places` by default)
3. `pollRun()` polls `GET /actor-runs/{id}` every 4 seconds until `SUCCEEDED`
4. `apiGet()` fetches dataset items from `GET /datasets/{id}/items`
5. `normalise()` maps raw Google Maps fields to the canonical `COLS` keys
6. `renderResults()` → `buildTable()` → `renderRows()` writes the DOM table

### Key data structures

**`COLS` array** (line ~512) — single source of truth for all columns. Each entry has `key`, `label`, and `aliases` (field names the Apify actor might return). Adding a new column means adding one entry here; `normalise()`, `buildTable()`, `exportCSV()`, and `exportJSON()` all derive from it automatically.

**`activeCols`** — filtered subset of `COLS` containing only columns where at least one lead has data. Computed in `renderResults()`, used everywhere the table is built.

**`allLeads` / `filtered`** — `allLeads` is the full normalised dataset; `filtered` is the current view after `filterTable()` or `sortTable()` is applied.

### Persistence

| Key | Storage | Contents |
|-----|---------|----------|
| `sgleadgen_settings` | localStorage | `{ apiKey, actorId, extra }` |
| `sgleadgen_cache` | localStorage | `{ leads: [...], ts }` — last result set, restored on load |
| `sgleadgen_theme` | localStorage | `"dark"` or `"light"` |

### Theme system

CSS custom properties on `:root` (dark) and `[data-theme="light"]` (light). `applyTheme(t)` sets `document.documentElement.dataset.theme` and updates the toggle button emoji.

### MCP servers (`.mcp.json`)

- **`apify`** — `@apify/actors-mcp-server`, token from `.env`
- **`playwright`** — `@playwright/mcp`, for browser automation

## Extending the tool

**Add a new output column:** Add an entry to the `COLS` array with the right `aliases` matching the actor's output field names. No other changes needed.

**Swap the Apify actor:** Change `DEF_ACTOR` constant or use the Settings modal. The actor input is built in `startSearch()` — adjust the `input` object shape there if the new actor expects different fields.

**Add a new Singapore sector preset:** Add an `<option>` inside `#sectorSelect` in the HTML.
