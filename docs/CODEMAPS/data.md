<!-- Generated: 2026-04-25 | Files scanned: 1 (index.html) | Token estimate: ~700 -->

# Data

## Storage Model
**No database. No localStorage.** Two persistence channels:
1. **JS literals in `index.html`** — static reference data (commands, craftTools, docs, HEADER_RULES, HEADER_KNOWLEDGE).
2. **Downloaded JSON sidecar files** — user-saved scan/audit history (manual import/export, `version:1`).

## Static Data (in-source)

### `commands[]`  (index.html:2250)
Reference cards. Currently empty array — placeholder for future reference content.
```
{ section, title, command, description, tags?, … }
```

### `craftTools{}`  (index.html:2254)
Interactive command builders.
```
craftTools.nmap   index.html:2255
   .name, .description, .targetPlaceholder
   .groups[]   single-select option groups
       Scan Type:  -sS -sT -sU -sA -sV -sn
       Ports:      default | -p- | --top-ports 100 | -p custom
       Timing, Host Discovery, Output, Scripts, …
   .toggles[]  on/off flags

craftTools.curl   index.html:2318
   .groups, .toggles  (method, headers, body, follow, output, auth)
```

### `docs{}`  (index.html:2365)
Flag-level documentation.
```
docs.nmap[]   index.html:2366    ~75 entries
docs.curl[]   index.html:3234    flag reference

entry shape:
   { id, flag, category, title, docUrl, description,
     example, platforms:{linux,windows,powershell}, warnings[], tips[] }
```

### `HEADER_RULES{}`  (index.html:3785)
HTTP header audit ruleset.
```
.missing[]      ~15 required security headers + recommended values
.deprecated[]   ~18 deprecated headers + reason
.fingerprint[]  ~80 server/CDN/CMS disclosure patterns (Server, X-Powered-By, …)
.security[]     ~50 known security-relevant header names (lowercase)
.valueChecks{}  per-header value validators
   'strict-transport-security' .checks[]   max-age threshold, includeSubDomains, preload
   referrer-policy / x-frame-options / permissions-policy / etc.
```

### `HEADER_KNOWLEDGE{}`  (index.html:4036)
Per-header explainer functions: take header value → human-readable bullets. Covers HSTS, CSP, Referrer-Policy, Permissions-Policy, Cache-Control, ETag, Set-Cookie, Keep-Alive, CF-RAY, CF-Cache-Status, Retry-After, Server-Timing, Content-Type, Content-Length, etc.

## Runtime State (module vars)
```
activeSection         { type:'reference'|'craft'|'docs'|'rescan'|'audit', key }
previousSection
searchQuery
craftState[toolKey]   { target, targetMode, targetList[], targetFilename,
                        batchMode:'bat'|'cmd', groups{}, toggles{}, customInputs{} }
rescanEnv             'Prod'|'UAT'
rescanEntries[]       see schema below
rescanSelected[]
rescanComparing       false | true | 'matrix'
pendingImportRaw / pendingImportParsed / pendingImportMulti[]
auditEntries[]        see schema below
auditSelected[] / auditComparing
currentAuditResult    transient analyzer state
```

## Sidecar JSON Schemas

### Rescan export — `<date>_rescan-<env>-export.json`
```
{
  "version": 1,
  "entries": [
    {
      "id":          "uuid-like",
      "title":       "Prod Web Server - Initial Scan",
      "environment": "Prod" | "UAT",
      "target":      "192.168.1.1 | hostname",
      "timestamp":   "ISO 8601",
      "scanType":    "nmap-port" | "nmap-script" | "curl-headers",
      "rawInput":    "<original output>",
      "parsed":      { … type-specific structure … }
    }
  ]
}
```

### Audit export — `<date>_audit-export.json`
```
{
  "version": 1,
  "auditEntries": [
    {
      "id":        "uuid-like",
      "title":     "example.com - Q1 Audit",
      "target":    "example.com",
      "timestamp": "ISO 8601",
      "rawInput":  "<original headers>",
      "inputType": "curl" | "raw" | "nmap",
      "findings":  { missing[], deprecated[], fingerprint[], valueIssues[] },
      "counts":    { missing:N, deprecated:N, fingerprint:N, valueIssues:N }
    }
  ]
}
```

### Parsed payload shapes (rescanEntries[].parsed)
```
nmap-port:   { ports:[{port,proto,state,service,version}], hostInfo }
nmap-script: { ports[], scripts:{ ssl-cert, ssl-enum-ciphers, http-methods,
                                  vulns, http-security-headers, … } }
curl-headers:{ status, statusText, headers:{name:value} }
```

## Migration History
None — schema is `version: 1` since inception. No migration framework. Forward-compatibility relies on the parser tolerating unknown fields.

## Cardinality
| Asset | Count |
|---|---|
| nmap docs entries | ~75 |
| curl docs entries | ~50 |
| HEADER_RULES.missing | 15 |
| HEADER_RULES.deprecated | 18 |
| HEADER_RULES.fingerprint | 80+ |
| HEADER_RULES.security | 50+ |
| craftTools | 2 (nmap, curl) |
| commands[] | 0 (placeholder) |
