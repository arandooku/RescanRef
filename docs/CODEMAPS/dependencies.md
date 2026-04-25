<!-- Generated: 2026-04-25 | Files scanned: 1 (index.html) | Token estimate: ~350 -->

# Dependencies

## Runtime Dependencies
**Zero.** No npm, no CDN, no fetch, no network calls. App opens via `file://`.

```
$ grep -E "https?://|<script src|<link" index.html | head
```
- No `<script src>` tags except inline.
- No `<link rel="stylesheet">` to external CSS.
- `https://` URLs appear only as **reference** strings inside `docs[*].docUrl` (links to nmap.org, curl.se, MDN) — rendered as `<a target="_blank">` in doc cards. Not loaded by the app.

## Browser APIs Used
| API | Where | Purpose |
|---|---|---|
| `document.*` | throughout | DOM manipulation |
| `Blob` + `URL.createObjectURL` | exportRescanEntries (6931), exportAuditEntries (6984), downloadTargetList (5082) | trigger file download |
| `navigator.clipboard.writeText` | copyToClipboard (4419) | copy command preview |
| `document.execCommand('copy')` | copyToClipboard fallback (4421) | clipboard fallback for `file://` |
| FileReader / drag events | handleFilesDrop (5806), handleAuditFileDrop (7629) | JSON/text import |
| `JSON.parse` / `JSON.stringify` | sidecar import/export | schema v1 format |
| `Date` / `Date.toISOString` | formatDate (4462), generateTargetFilename (5069) | timestamps |

## Build / Tooling
**None.** No bundler, no transpiler, no preprocessor, no test runner, no linter.
- ES5-compatible JS (uses `var`, `function`, no modules, mixed `let`/`const` for state).
- Targets evergreen browsers; runs offline on Windows laptops.

## Constraints (CLAUDE.md)
- Must remain a single-file HTML.
- No external deps allowed — fully offline.
- Some target machines have restricted internet — anything fetched at runtime would break the app.

## Repo-Internal Docs
```
docs/CODEMAPS/                  this directory (architecture/backend/frontend/data/dependencies)
docs/superpowers/specs/         design docs (dated)
   2026-04-10-scanref-design.md
   2026-04-10-header-audit-design.md
   2026-04-14-htrace-redesign-design.md
   2026-04-15-multi-target-batch-matrix-design.md
docs/superpowers/plans/         implementation plans (dated)
   2026-04-10-scanref-implementation.md
   2026-04-10-header-audit-implementation.md
   2026-04-14-htrace-redesign-implementation.md
   2026-04-15-multi-target-batch-matrix.md
CLAUDE.md                       project rules
.reports/                       (auto-generated, this run creates codemap-diff.txt)
```

## External Reference URLs (in doc data, not loaded)
- nmap.org (man pages)
- curl.se (curl manual)
- developer.mozilla.org (MDN HTTP header docs — referenced from HEADER_KNOWLEDGE comments)
- owasp.org (security header guidance)
