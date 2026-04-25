<!-- Generated: 2026-04-25 | Files scanned: 1 (index.html, 7799 lines) | Token estimate: ~750 -->

# Architecture

## Project Type
Single-file static SPA. No build, no deps, offline-only (`file://`). One artifact: `index.html` (~7799 lines, was ~6730 in CLAUDE.md — drift).

## Layout
```
index.html
 ├─ <style>            lines 7–2143    (28 sections)
 ├─ <body> markup      lines 2144–2243 (sidebar + main + 2 modals)
 └─ <script>           lines 2245–7797
     ├─ DATA           2250–3958  (commands, craftTools, docs, HEADER_RULES, HEADER_KNOWLEDGE)
     ├─ STATE          3760–3783  (activeSection, craftState, rescan*, audit*)
     ├─ UTILS          4383–4475  (id/escape/toast/copy/highlight/slugify/formatDate)
     ├─ SIDEBAR        4481–4646  (buildSidebar, updateSidebarActive, matchCounts)
     ├─ ROUTER         4647–4702  (navigateTo, renderMainContent, renderEmptyState)
     ├─ REFERENCE      4704–4778  (renderReferenceSection, renderCommandCard, copy handlers)
     ├─ CRAFT          4780–5358  (state, build/colorCode commands, target list, batch script)
     ├─ DOCS           5359–5503  (renderDocsSection)
     ├─ RESCAN         5504–5805  (timeline, handlers, parsers, comparison)
     ├─ IMPORT/PARSE   5806–6219  (drop, paste, splitNmapByHost, parseNmap*, parseCurlHeaders)
     ├─ AUDIT ANALYZE  6221–6318  (analyzeHeaders + HEADER_RULES eval)
     ├─ COMPARE        6320–6925  (renderComparisonView, MatrixView, port/header/script/SSL/cipher/methods/secHeaders/vuln/header/raw)
     ├─ EXPORT         6927–6994  (exportRescanEntries, exportAuditEntries → JSON sidecars)
     ├─ SEARCH         7000–7196  (performSearch across all 5 sections, renderSearchResults)
     ├─ AUDIT VIEW     7213–7773  (renderAuditSection/Report/Timeline/Comparison)
     └─ INIT           7775–7796  (initTheme, buildSidebar, navigateTo first ref/craft)
```

## Five Sections (sidebar tabs)
```
type=reference  →  static command cards filtered by section name
type=craft      →  interactive nmap/curl builder + batch script
type=docs       →  nmap/curl flag reference with categories
type=rescan     →  scan history timeline + JSON import/export + diff
type=audit      →  HTTP header analyzer + saved audits + comparison
```

## Data Flow
```
user action (click/paste/drop)
     │
     ▼
event handler  ──▶  mutate state vars (craftState / rescanEntries / auditEntries)
     │
     ▼
renderMainContent()  ──▶  swap container HTML  ──▶  attach*Handlers()
     │
     ▼
JSON sidecar (export only — no localStorage, no fetch)
```

## Constraints (from CLAUDE.md)
- No external deps. No CDNs. No fetch. No localStorage.
- Dark terminal aesthetic (OCR A Extended, green accent, no radius/shadow/animation).
- Edits go in HTML source; persistence via downloaded JSON files.
- Target: Windows laptops + OneDrive sync; restricted internet.

## Render Pattern
String concatenation → assigned into container → `attach*Handlers(container)` rebinds events. No virtual DOM, no reactivity. Re-render on state change.

## Naming Pattern
- `render*()` build HTML strings
- `attach*Handlers()` bind events post-render
- `parse*()` extract structured data from raw scan/header text
- `build*Command()` synth nmap/curl from craftState
- `colorCodeCommand()` syntax-highlighted preview
