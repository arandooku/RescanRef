<!-- Generated: 2026-04-25 | Files scanned: 1 (index.html) | Token estimate: ~400 -->

# Backend

## Status
**No backend.** Pure client-side static SPA. No HTTP routes, no API, no server, no middleware.

## Pseudo-Backend (client-side compute)

The closest analogues to backend layers live entirely in the browser:

```
PARSER LAYER     (raw scan text → structured JSON)
   parseNmapPort        index.html:5944
   parseNmapScript      index.html:5979   sub-parsers: SSL cert, ciphers, methods, vulns, sec-headers
   parseCurlHeaders     index.html:6110
   parseRawHeaders      index.html:6158
   normalizeHeaderInput index.html:6174
   splitNmapByHost      index.html:5894   multi-host nmap output → array of single-host blobs
   autoDetectScanType   index.html:5916   pattern match → 'nmap-port' | 'nmap-script' | 'curl-headers'
   detectHeaderInputType index.html:6150  → 'curl' | 'raw' | 'nmap'

ANALYSIS LAYER   (structured data → findings)
   analyzeHeaders       index.html:6221   evaluates HEADER_RULES + HEADER_KNOWLEDGE.valueChecks

COMMAND BUILDER  (state → shell command)
   buildCraftedCommand  index.html:4852   single target
   buildBatchCommand    index.html:4798   .bat / .cmd loop over targets.txt
   colorCodeCommand     index.html:4896   highlighted HTML preview

PERSISTENCE      (blob → user download)
   exportRescanEntries  index.html:6927   → {version:1, entries:[…]} JSON
   exportAuditEntries   index.html:6981   → {version:1, auditEntries:[…]} JSON
   downloadTargetList   index.html:5079   → plaintext targets.txt
```

## "Routes" (sidebar navigation actions)
```
click sidebar nav-item        →  navigateTo({type, key})
click craft option pill       →  craftState mutation → re-render
click toggle pill             →  craftState.toggles[flag] = !x → re-render
click "Compare Selected"      →  rescanComparing=true → renderComparisonView
click "Matrix View"           →  rescanComparing='matrix' → renderMatrixView
click "Export JSON"           →  exportRescanEntries / exportAuditEntries
drop file on drop-zone        →  handleFilesDrop → parse → classifyModal
paste in paste-textarea       →  handlePasteImport → parse → classifyModal
```

## No Server-Side Concerns
| Concern | Status |
|---|---|
| Auth | n/a (offline) |
| Rate limiting | n/a |
| CSRF / XSS | local DOM only; `escapeHtml()` (4390) used in render paths |
| SQL | n/a (no DB) |
| Sessions | n/a |
| Logging | n/a |
| API versioning | JSON sidecar `version: 1` field on export |
