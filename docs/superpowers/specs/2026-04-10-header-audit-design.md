# Header Audit Feature — Design Spec

**Date:** 2026-04-10
**Status:** Approved
**Based on:** [humble](https://github.com/rfc-st/humble) HTTP security header analyzer

## Overview

A standalone "Header Audit" section in RescanRef that analyzes HTTP response headers for security issues. Users paste `curl -I` output, raw headers, or nmap script output and get a structured report showing missing, deprecated, insecure, and fingerprint-leaking headers with OWASP-recommended fixes. Audits are saveable and comparable over time.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Location | Separate sidebar section ("Header Audit") | Keeps it focused; doesn't clutter Rescan History |
| Analysis depth | Core checks (no deep CSP parsing) | Covers most actionable findings; CSP can be added later |
| Input methods | Paste + file import | Offline-first; covers common workflows |
| Report content | Findings + OWASP recommended values | Directly actionable without external links |
| Persistence | Saveable + comparable over time | Core app value is tracking security posture changes |
| Grading | None | Findings and counts are sufficient |
| Architecture | Single file with section markers | Consistent with existing monolithic offline design |

## Data Model

### Header Rules Knowledge Base

A single `HEADER_RULES` constant containing all analysis data:

#### `missing` — 15 critical security headers

Each entry: `{ header, description, recommended }`.

Headers (from humble's `missing.txt`): Cache-Control, Clear-Site-Data, Content-Type, Content-Security-Policy, Cross-Origin-Embedder-Policy, Cross-Origin-Opener-Policy, Cross-Origin-Resource-Policy, Integrity-Policy, NEL, Permissions-Policy, Referrer-Policy, Strict-Transport-Security, X-Content-Type-Options, X-Frame-Options, X-Permitted-Cross-Domain-Policies.

The `recommended` field contains the OWASP-recommended value string (from humble's `owasp_best_practices.txt`).

#### `deprecated` — ~15 headers that should not be present

Each entry: `{ header, reason }`.

Headers: Accept-CH-Lifetime, Content-DPR, Digest, Expect-CT, Feature-Policy, Large-Allocation, P3P, Pragma, Public-Key-Pins, Report-To, Tk, Warning, X-Content-Security-Policy, X-Download-Options, X-Pad, X-SourceMap, X-UA-Compatible, X-Webkit-CSP, X-XSS-Protection.

#### `fingerprint` — ~1,280 header-to-technology mappings

Each entry: `{ header, pattern, technology }`.

Sourced from humble's `fingerprint.txt`. The `pattern` field is `'*'` for headers whose mere presence is a fingerprint, or a specific value substring for headers like `Server` that need value matching.

#### `security` — 62 known security-relevant headers

A flat array of header name strings used to identify which present headers count as "enabled security headers."

#### `valueChecks` — per-header validation rules

An object keyed by header name, containing validation rules:

- **Strict-Transport-Security**: `max-age` must be >= 31536000; should contain `includeSubDomains`.
- **Referrer-Policy**: insecure values are `unsafe-url`, `no-referrer-when-downgrade`, empty string. Recommended: `strict-origin-when-cross-origin`.
- **X-Frame-Options**: valid values are `DENY`, `SAMEORIGIN`. `ALLOW-FROM` is deprecated.
- **X-Content-Type-Options**: must be `nosniff`.
- **Permissions-Policy**: flag deprecated features (`speaker`, `vibrate`, `vr`) and overly broad values (`*`).

### Audit Result Schema

```
{
  id:         string (UUID),
  title:      string (user-provided label),
  target:     string (auto-detected or user-entered URL/host),
  timestamp:  string (ISO date),
  rawInput:   string (original pasted text),
  inputType:  'curl-headers' | 'raw-headers' | 'nmap-script' | 'file',
  findings: {
    enabled:     [{ header, value }],
    missing:     [{ header, description, recommended }],
    deprecated:  [{ header, value, reason }],
    fingerprint: [{ header, value, technology }],
    insecure:    [{ header, value, issue, recommended }],
    empty:       [{ header }]
  },
  counts: {
    enabled: number,
    missing: number,
    deprecated: number,
    fingerprint: number,
    insecure: number,
    empty: number
  }
}
```

## Input Parsing

### Accepted Formats

1. **curl -I output** — starts with `HTTP/` status line followed by `Header: value` lines. Call existing `parseCurlHeaders()` and extract `.headers` from its return value (no modification to existing function needed).
2. **Raw headers** — `Header: value` lines without a status line (e.g., from browser DevTools). Detected by absence of `HTTP/` prefix on first non-empty line. New parsing logic in `parseRawHeaders()`.
3. **nmap http-security-headers script** — NSE output containing `| http-security-headers:` marker. Call existing `parseNmapScript()` and extract header data from the relevant script entry (no modification to existing function needed).
4. **File import** — `.txt` file containing any of the above formats. Read via `FileReader` API.

### Normalization

All input formats normalize to: `{ headers: { 'header-name': 'value', ... }, meta: { statusCode, protocol, source } }`. Header names are stored as-cased but matched case-insensitively during analysis.

### Auto-detection

`detectHeaderInputType(raw)` uses these heuristics (first match wins):
1. Contains `| http-security-headers:` -> `nmap-script`
2. First non-empty line starts with `HTTP/` -> `curl-headers`
3. Lines match `SomeHeader: value` pattern -> `raw-headers`

## Analysis Engine

`analyzeHeaders(normalizedInput)` runs six checks in order, building the findings object:

1. **Enabled** — match present headers against the 62 known security headers list.
2. **Missing** — check which of the 15 critical headers are absent. Special case: skip `X-Frame-Options` missing finding if CSP contains `frame-ancestors`.
3. **Fingerprint** — match present headers against fingerprint patterns.
4. **Deprecated** — check if any deprecated headers are present.
5. **Insecure values** — run `valueChecks` rules against present headers.
6. **Empty** — flag headers with empty/whitespace-only values.

Returns the full audit result object.

## UI Layout

### Sidebar

New entry under its own category below Rescan History:

```
Header Audit
```

### Main Area

#### Input Zone (top)

- Textarea with placeholder showing expected formats
- Drag-and-drop overlay for `.txt` files (same pattern as Rescan History drop zone)
- File picker button
- "Analyze" button (primary action)
- "Clear" button (appears after analysis, resets input and report)

#### Report (middle, appears after analysis)

Vertical stack of collapsible sections, each with a count badge and color indicator:

| Section | Color | Content per item |
|---------|-------|-----------------|
| Missing Security Headers (N) | Red | Header name, why it matters, recommended value (copyable) |
| Deprecated Headers (N) | Yellow | Header name, current value, reason deprecated |
| Insecure Values (N) | Orange | Header, current value, what's wrong, recommended value |
| Fingerprint Headers (N) | Yellow | Header, value, what technology it leaks |
| Empty Headers (N) | Gray | Header name |
| Enabled Security Headers (N) | Green | Header, value |

Sections with findings start expanded. Sections with zero findings start collapsed and show green indicator.

#### Actions (below report)

- "Save Audit" button — opens a small modal for the user to title the audit, then saves to in-memory `auditEntries[]` array.
- "Compare" button (enabled when 2+ saved audits exist for same target) — opens comparison view.

#### Saved Audits Timeline (below actions)

Card-based timeline of saved audits, showing: title, target, timestamp, summary counts (e.g., "7 missing, 2 deprecated, 3 insecure"). Same visual style as Rescan History scan cards.

### Comparison View

Side-by-side table comparing older (left) vs newer (right) audit:

- **Green** — was missing/insecure, now fixed
- **Red** — was fine, now missing/insecure (regression)
- **Yellow** — value changed
- **Gray** — unchanged

Summary badge at top: "N fixed, N regressed, N changed, N same."

## Persistence

Same JSON export/import pattern as Rescan History:

- **Export**: downloads `YYYY-MM-DD_audit_<target>_<title>.json` containing `{ version: 1, entries: [...auditResults] }`
- **Import**: drag-drop JSON file to restore saved audits into `auditEntries[]`
- **In-memory**: `auditEntries[]` array, separate from `rescanEntries[]`

## Code Organization

All code lives in `index.html` with clear section markers:

```
// ═══════════════════════════════════════════
// HEADER AUDIT: DATA
// ═══════════════════════════════════════════
// HEADER_RULES constant (~1,400 lines for fingerprint data)

// ═══════════════════════════════════════════
// HEADER AUDIT: PARSER
// ═══════════════════════════════════════════
// parseHeaderInput(), detectHeaderInputType(), normalizeHeaders()

// ═══════════════════════════════════════════
// HEADER AUDIT: ANALYSIS ENGINE
// ═══════════════════════════════════════════
// analyzeHeaders(), check functions for each category

// ═══════════════════════════════════════════
// HEADER AUDIT: UI RENDERER
// ═══════════════════════════════════════════
// renderHeaderAudit(), renderAuditReport(), renderAuditComparison()
// renderAuditTimeline(), audit save/export/import handlers

// ═══════════════════════════════════════════
// HEADER AUDIT: CSS
// ═══════════════════════════════════════════
// Styles in the <style> block, under a Header Audit section marker
```

## Reuse from Existing Code

- `parseCurlHeaders()` — reuse for curl input parsing
- `parseNmapScript()` — reuse for nmap input parsing
- Copy-to-clipboard — reuse for recommended value code blocks
- Drop zone pattern — reuse drag-and-drop from Rescan History
- Comparison color scheme — reuse green/red/yellow/gray from existing comparison engine
- Card layout — reuse scan entry card style from Rescan History timeline
- Modal pattern — reuse classification modal pattern for "Save Audit" modal
- Theme support — inherits existing dark/light Monokai theme automatically

## Out of Scope

- Deep CSP directive-level analysis (future enhancement)
- Set-Cookie flag analysis (future enhancement)
- Browser compatibility / caniuse links
- URL fetching (breaks offline-first design)
- External MDN reference links (don't work on `file://`)
- Grading system
