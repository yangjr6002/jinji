# AGENTS.md

## Project overview

Private Space is a Chinese-language, browser-only personal dossier demo. It is a self-contained single-page application: HTML, CSS, and vanilla JavaScript all live in one file, with no package manager, build step, or automated test suite.

The app manages highly sensitive *demo* data such as accounts, assets, identity photos, contacts, timelines, and family relationships. The UI clearly warns users not to enter real secrets.

## Repository layout

- `index.html` — canonical application and primary edit target. It contains markup, styles, state, rendering, and event handlers.
- `.gitignore` — ignores local Claude configuration.

## Run and verify

There is no install, build, lint, or test command. For a representative local preview, serve the repository root:

```sh
python3 -m http.server 4173 --bind 127.0.0.1
```

Open [http://127.0.0.1:4173/](http://127.0.0.1:4173/) in a modern browser. Stop the server with `Ctrl+C` when finished. Opening `index.html` directly also works for most UI checks, but the local server is preferred when verifying CDN-loaded dependencies and browser storage behavior.

After changes, at minimum:

1. Confirm the document parses by opening it in a browser and checking the developer console for JavaScript errors.
2. Exercise the relevant workflow, including save/reload when it changes persisted data.
3. Check `git diff --check` for whitespace errors.

For interaction-heavy changes, also check narrow and wide layouts, modal dismissal, keyboard behavior, and at least one create/edit/delete path. For family-tree work, include Dagre auto-layout, pan/zoom, node editing, adding or linking a parent/child/spouse, and parent/child/spouse connector rendering.

## Architecture and state

- The data model starts at `DEFAULT_DATA`; `DATA` is the live mutable state.
- `boot()` loads persisted data, normalizes/migrates it through `normalizeData()`, then calls `render()`.
- Prefer updating `DATA`, then calling `save()` and `render()` (and `toast()` when appropriate). `save()` drives persistence and undo/redo snapshots.
- Persistence uses `window.storage` when supplied by Claude Artifacts and falls back to `localStorage` for normal browser use.
- Browser persistence uses the single `pw_dossier_data_v6` key.
- Preserve backward compatibility when changing persisted shapes: extend `normalizeData()` or add a focused migration. Do not assume every browser has newly seeded `DEFAULT_DATA`.
- Backup import/export serializes the whole `DATA` object. Schema changes should keep imported older backups usable.
- User-entered strings rendered into HTML must go through `esc()`. Keep this rule when adding fields or templates.

### Family-tree data and layout

- The family graph uses Cytoscape.js with the Dagre layout extension, loaded from CDNs. Do not replace it with hand-positioned DOM/SVG code.
- A family member uses `id`, `name`, `relation`, `parentIds`, and optional `spouseIds`; spouse-label text is stored separately in `familySpouseLabels`, keyed by the sorted pair of member IDs.
- `familyGraphElements()` maps persistent data to Cytoscape nodes and edges. For a child with two or more recorded parents, it creates a render-only `union:` node between parents and child; never persist this virtual node in `DATA`.
- Dagre calculates every member position at render time. Old `x`, `y`, `w`, and `h` values from the previous custom canvas are removed during `normalizeData()` because they are not compatible with the new renderer.
- Keep parent/child relationships as directed graph edges and spouse relationships as separately styled dashed edges. This is a relationship graph, not a strict one-parent tree.
- Use Cytoscape's built-in pan, zoom, selection, and node event handling. A node click opens the existing-app editing flow; data changes must still call `save()` and `render()`.

## UI conventions

- Rendering is imperative vanilla JavaScript and uses inline `onclick`/event handlers; follow the surrounding pattern rather than introducing a framework or build tooling for a small change.
- Most views are routed through `VIEW`, `go()`, and `render()`. Item detail tabs are tracked independently in `openTabs` / `activeTabKey`.
- Reusable UI helpers near the end of the file include `openModal`, `confirmDelete`, `field`, `moduleShell`, and `emptyState`.
- Text is primarily Simplified Chinese. Match the existing wording, low-saturation visual palette, and accessible button titles.
- Preserve undo/redo behavior for every meaningful state mutation. Avoid direct `localStorage` writes outside the persistence helpers.
- Destructive actions should use `confirmDelete`.

## External dependencies and privacy

- Google Fonts and Leaflet are loaded from CDNs.
- World-map geocoding uses OpenStreetMap/Nominatim and therefore needs network access. Keep a graceful fallback for unavailable remote resources.
- Images are stored as data URLs in browser persistence; this can exhaust local storage. Do not add large seed images or silently upload user data.
- Never add production-looking real credentials, account numbers, ID images, or personal details to defaults, fixtures, screenshots, or commits.

## Change workflow

1. Inspect `git status` first; preserve unrelated user changes.
2. Make the focused update in `index.html`.
3. Run the verification checks above and report any browser-only checks that were not possible.

Use `apply_patch` for targeted edits. Do not introduce generated/minified output, dependencies, or a build system unless the task explicitly requires it.
