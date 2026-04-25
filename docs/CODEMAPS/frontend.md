<!-- Generated: 2026-04-25 | Files scanned: 1 (index.html) | Token estimate: ~950 -->

# Frontend

## Section Tree
```
ScanRef
 ├─ Reference (commands[].section)         dynamic — currently empty array
 ├─ Craft
 │   ├─ nmap     →  6 scan types, ports, timing, host disco, output, scripts, target list
 │   └─ curl     →  method, headers, body, follow, output, auth
 ├─ Docs
 │   ├─ nmap     →  ~75 entries grouped by category (Scan Type, Ports, Timing, …)
 │   └─ curl     →  flag reference with platform notes + warnings + tips
 ├─ Rescan       →  history timeline, drop/paste import, compare/matrix, export
 └─ Audit        →  header analyzer, save modal, timeline, comparison
```

## DOM Roots
```
#sidebar          aside, fixed-width nav
#sidebarNav       buildSidebar() output (5 tab groups)
#mainContent      router target — replaced wholesale per navigateTo
#classifyModal    rescan import → classify scan type/title/env
#auditSaveModal   audit input → save with title/target
#hamburger        mobile menu toggle
```

## Routing
```
activeSection = { type, key }
   │
   ▼
navigateTo(section, scrollToId)         index.html:4647
   │
   ▼
renderMainContent(scrollToId)           index.html:4657
   │
   ├─ type=reference  →  renderReferenceSection(container, sectionName)
   ├─ type=craft      →  renderCraftSection(container, toolKey)
   ├─ type=docs       →  renderDocsSection(container, toolKey, scrollToId)
   ├─ type=rescan     →  renderRescanSection(container)
   └─ type=audit      →  renderAuditSection(container)
```

## State (module-level vars)
```
activeSection      { type, key }
previousSection    { type, key }
searchQuery        string
craftState         { [toolKey]: {target, targetMode, targetList, targetFilename,
                                 batchMode, groups, toggles, customInputs} }
rescanEnv          'Prod' | 'UAT'
rescanEntries      [{ id, title, environment, target, timestamp, scanType,
                      rawInput, parsed }]
rescanSelected     [id, …]   max 2 for diff, ≥2 for matrix
rescanComparing    false | true | 'matrix'
pendingImportRaw / pendingImportParsed / pendingImportMulti
auditEntries       [{ id, title, target, timestamp, rawInput, inputType,
                      findings, counts }]
auditSelected / auditComparing
currentAuditResult { rawInput, inputType, meta, findings, counts }
```

## Render Pipeline
```
render*()                      → html string
container HTML swap            → DOM replaced (whole subtree)
attach*Handlers(container)     → bind click/input/change
```
No diffing. Every state mutation re-renders the active section.

## Key Render Functions
| Function | Lines | Output |
|---|---|---|
| `buildSidebar` | 4481 | nav tree with reference / craft / docs / rescan / audit groups |
| `renderReferenceSection` | 4704 | command cards for one section name |
| `renderCommandCard` | 4731 | single card with copy-to-clipboard |
| `renderCraftSection` | 4941 | option groups + toggle pills + target input + preview |
| `renderDocsSection` | 5359 | category-grouped doc cards, expandable on info click |
| `renderRescanSection` | 5504 | env tabs + drop zone + paste area + timeline + compare buttons |
| `renderScanTimeline` | 5565 | scans grouped by target, sorted by timestamp |
| `renderComparisonView` | 6320 | 2-scan diff dispatcher |
| `renderMatrixView` | 6369 | hosts × ports/headers grid |
| `renderPortMatrix` / `renderHeaderMatrix` | 6389 / 6444 | matrix tables |
| `renderPortComparison` etc. | 6503 onward | per-scan-type diff renderers |
| `renderAuditSection` | 7213 | analyzer textarea + report + timeline |
| `renderAuditReport` | 7254 | findings cards (missing / deprecated / fingerprint / value issues) |
| `renderHeaderAnalyzer` | 7339 | per-header row with knowledge popover |
| `renderAuditComparisonView` | 7664 | 2-audit diff |
| `renderSearchResults` | 7087 | flat result list across all section types |
| `renderEmptyState` | 4693 | empty placeholder |

## Craft Builder Internals
```
getCraftState(toolKey)       4780  init from defaults if absent
buildCraftedCommand          4852  → "nmap -sS -T4 192.168.1.1"
buildBatchCommand            4798  → .bat or .cmd loop over targets.txt
colorCodeCommand             4896  → HTML span tree for preview (.cmd-tool/.cmd-flag/.cmd-arg/.cmd-timing)
parseTargetList              5062  split on whitespace/commas/newlines
generateTargetFilename       5069  timestamped default
downloadTargetList           5079  Blob → .txt download
attachCraftHandlers          5093  bind option click, toggle pill, target input, copy, download
```

## Import / Parse Pipeline
```
drop / paste / file dialog
   │
   ▼
handleFilesDrop / handlePasteImport       5806 / 5832
   │
   ├─ JSON?   → load entries directly
   ├─ multi-host nmap? → splitNmapByHost  5894  → multi-modal classify
   └─ single host
       │
       ▼
   autoDetectScanType  5916   (nmap-port | nmap-script | curl-headers)
       │
       ▼
   parseScanOutput     5933   → dispatch
       │
       ├─ parseNmapPort     5944
       ├─ parseNmapScript   5979   (SSL cert, ciphers, methods, vulns, sec-headers)
       └─ parseCurlHeaders  6110
       │
       ▼
   classifyModal opens → user fills title/env/target/timestamp/type
       │
       ▼
   rescanEntries.push(entry)
```

## Audit Pipeline
```
paste curl/raw/nmap http output
   │
   ▼
detectHeaderInputType   6150   (curl | raw | nmap)
parseRawHeaders         6158
normalizeHeaderInput    6174   → headers map
analyzeHeaders          6221   → findings { missing, deprecated, fingerprint, valueIssues }
   │
   ▼
renderAuditReport / renderHeaderAnalyzer
   │
   ▼
saveModal → auditEntries.push
```

## Search
`performSearch` (7000) iterates commands, craftTools, docs, rescanEntries, auditEntries → counts per `type:key` shown as sidebar badges, results rendered via `renderSearchResults`. `/` keybinding focuses input.

## Mobile
Hamburger toggle (`#hamburger`) opens sidebar via `.sidebar-overlay`. Three breakpoints: 900px (tablet), 767px (mobile), 400px (small phone). Styles at index.html:1910–2143.
