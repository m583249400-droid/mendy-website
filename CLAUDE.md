# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project shape

Single-file static web app: everything (HTML, CSS, JS) lives in `index.html` (~1700 lines). There is no build system, no package manager, no test runner, no framework, and no dependencies installed locally. The only files in the repo are `index.html` and `README.md`.

UI language is Hebrew, layout is RTL (`<html lang="he" dir="rtl">`), and the app is designed mobile-first (fixed bottom navigation, `env(safe-area-inset-bottom)`, sticky page top). It is a personal productivity / Jewish-life dashboard for one user ("Mendy"), located in Beit Shemesh.

## Running and "testing"

Just open `index.html` in a browser — no server required. For local iteration with auto-reload, any static server works (e.g. `python3 -m http.server` from the repo root, then visit `http://localhost:8000/`). There are no automated tests; verify changes by opening the page and exercising the affected tab.

The page calls one external API at runtime: `https://www.hebcal.com/zmanim?...` (Jewish prayer times). Network failures there fall back to a friendly error in the zmanim grid — keep that fallback intact when editing `fetchZmanim()`.

## Architecture

### Tabs and routing
Top-level UI is a tab system. The seven tabs are declared in one place:

```js
const TABS=['home','zmanim','sw','tasks','shop','journal','settings'];
```

Each tab corresponds to a `<div id="tab-{name}" class="tab-page">` in markup and a button in `.bnav` (bottom nav) at the bottom of `<body>`. `showTab(name)` (around line 1424) toggles `display`, sets the active nav button by index (so nav button order MUST match `TABS` order), and lazy-renders that tab's content. When adding a tab you must update all four: the `TABS` array, the markup section, the bottom-nav button, and the lazy-render dispatch inside `showTab`.

The `sw` ("stam stopwatch") tab has its own internal sub-tab system driven by `stamTab(...)` and `.stam-tab` / `.stam-page` classes — independent from the top-level tabs.

### State and persistence
All persistence is `localStorage`, accessed only through three helpers defined near the top of `<script>`:

- `sg(key)` — get string
- `ss(key, value)` — set string
- `sp(key, default)` — get + JSON.parse with fallback

These are wrapped around an `S` object that silently falls back to an in-memory map if `localStorage` throws (private mode, etc.). Always go through these helpers — never touch `localStorage` directly, or the fallback breaks.

Known storage keys (non-exhaustive): `tasks`, `dm_YYYY-MM-DD` (per-day done map), `streak`, `journal`, `hm` (heatmap counts), `cq` (custom quotes), `qi`, `artIdx`, `energy`, `todos`, `shop`. Per-day keys use `TODAY()` which returns `new Date().toISOString().slice(0,10)`.

`initState()` (called once in the IIFE at the bottom of the script) hydrates the in-memory globals `tasks`, `doneMap`, `streak`, `journalData` from storage. The IIFE also starts two intervals: a 1s `tick()` for the clock and a 5-minute `fetchZmanim()` refresh.

### Rendering pattern
There is no virtual DOM. Each section has a `renderX()` function that rewrites the relevant container's `innerHTML` from current in-memory state, then re-binds via inline `onclick=` attributes in the generated markup. After mutating state you must (a) persist via `ss(...)`, then (b) call the matching `renderX()`. Common pairs: `tasks` → `renderBlocks()` + `renderTimeline()` + `updateRing()`; `todos` → `renderTodos()`; `shopItems` → `renderShop()`; `journalData` → `loadJEntry()`.

### Task model
A task is `{id, name, dur, time, icon, cat}`. `dur` is a string like `"30m"`, `"1h"`, `"1.5h"` — parsed by `pDur()` and labelled by `dLbl()`. Categories are CSS classes prefixed `c-` (`c-prayer`, `c-torah`, `c-writing`, `c-music`, `c-custom`) and map to icons via `CAT_ICONS` and to timeline colors via the `cm` map inside `renderTimeline()`. Defaults live in `DEF_TASKS`.

### Drag-and-drop
The home-tab "blocks" grid supports reordering via both HTML5 drag events (desktop) and a custom touch implementation (`handleTouchStart/Move/End`, `touchDragEl`/`touchClone` globals). Both paths must end with `ss('tasks', ...)` and `renderBlocks()`. If you change the block markup, keep the `data-id`, `data-idx`, and `.block` selectors used by both paths.

## Code style conventions

The script is intentionally written in a dense, minified-looking style: single-letter locals (`d`, `t`, `el`), arrow expressions on one line, multiple statements per line separated by `;`, abbreviated function names (`sg`, `ss`, `pDur`, `dLbl`, `nextZK`). Match this style when editing existing code — do not "clean it up" or expand it into multi-line idiomatic JS, since the file is meant to stay readable as a single artifact and diffs should stay small. New top-level sections are separated by `// ══════` banner comments.

User-visible strings are Hebrew. Keep them in Hebrew when editing UI text; do not translate to English.

## Git workflow

Active development branch for this task: `claude/add-claude-documentation-GVLNe`. Commit and push there; do not push to `main` without explicit instruction.
