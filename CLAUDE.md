# ScanRef

Single-file offline HTML application (`index.html`, ~6000 lines) for security scanning
reference, command crafting (nmap/curl), and HTTP header auditing. Opens via `file://` protocol.

## Architecture

- **Everything is in `index.html`** — CSS, JS, and data are all inline. No external files, no build step.
- Data-driven: commands, craft tool definitions, header rules, and docs are JS objects/arrays at the top of the file
- Rendering logic follows the data sections
- Sidebar navigation with three groups: Reference (cheatsheets), Craft (interactive builders), Docs (flag references)
- Header Audit: standalone section for analyzing HTTP security headers (curl/raw/nmap input)
- Rescan History: saves/compares scan results over time via JSON file alongside HTML

## Constraints

- No external dependencies — fully offline, no CDNs, no fetch calls
- No localStorage — data edits go in the HTML source; history persists via JSON sidecar files
- Monokai color palette with dark/light themes via CSS custom properties
- Target: Windows laptops with OneDrive sync, some with restricted internet

## Working with this codebase

- There is no build, lint, or test command — the app is a single HTML file
- To test changes, open `index.html` in a browser (file:// protocol)
- Design specs live in `docs/superpowers/specs/`; implementation plans in `docs/superpowers/plans/`
- When editing, use section comment markers (e.g. `/* ===== Section Name ===== */`) to navigate the large file
- Keep all changes within `index.html` unless creating docs
