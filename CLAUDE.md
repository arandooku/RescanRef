# ScanRef

Single-file offline HTML application (`index.html`, ~8100 lines) for security scanning
reference, command crafting (nmap/curl), and HTTP header auditing. Opens via `file://` protocol.

## Architecture

- **Everything is in `index.html`** — CSS, JS, and data are all inline. No external files, no build step.
- Data-driven: commands, craft tool definitions, header rules, and docs are JS objects/arrays at the top of the file
- Rendering logic follows the data sections
- Sidebar navigation with two tool groups: Craft (nmap/curl interactive builders), Docs (flag references with official doc URLs)
- Header Audit: standalone section for analyzing HTTP security headers (curl/raw/nmap input)
- Rescan History: saves/compares scan results over time via JSON file alongside HTML

## Constraints

- No external dependencies — fully offline, no CDNs, no fetch calls
- No localStorage — data edits go in the HTML source; history persists via JSON sidecar files
- Dark-only terminal aesthetic — OCR A Extended monospace font, green accent, no border-radius/shadows/animations
- Target: Windows laptops with OneDrive sync, some with restricted internet

## Working with this codebase

- There is no build, lint, or test command — the app is a single HTML file
- To test changes, open `index.html` in a browser (file:// protocol)
- Design specs in `docs/superpowers/specs/`, implementation plans in `docs/superpowers/plans/` (prefixed by date)
- When editing, use section comment markers (e.g. `/* ===== Section Name ===== */`) to navigate the large file
- Keep all changes within `index.html` unless creating docs

## Current state

- `feat/header-audit` branch: terminal-aesthetic redesign (dark-only, OCR A Extended font, green accent, terminal-prompt headers)
- Removed PS/SSH reference sections; added official doc URLs to all nmap/curl references
- Craft toggles redesigned as compact pills; doc entries auto-expand on info click
- Header Audit and Rescan History: comparison views with separate save/export flows
- Mobile responsive adjustments applied
- URL hash routing for deep-linkable sections (`#craft/nmap`, `#docs/curl`, `#audit`, `#rescan`)
- Keyboard shortcuts: `/` search, `?` help overlay, `c/d/r/h` jump to section, `Esc` close modal, `1-9` Nth nav item
- Print stylesheet, `:focus-visible` rings, `prefers-reduced-motion` respect, auto-resizing paste textareas
- Response Viewer: standalone `#viewer` section — paste raw curl output, auto-detects and beautifies JSON / HTTP headers / HTML / XML (native `JSON.parse` + `DOMParser`, no deps)
- Architecture codemaps in `docs/CODEMAPS/` (architecture, frontend, backend, data, dependencies)
