# Multi-Target, Batch Loop, Matrix Comparer & Upload Guardrails

**Date:** 2026-04-15
**Branch:** feat/header-audit
**Approach:** Integrated Target List (Approach A) — all features flow within existing UI sections

## 1. Multi-Target Input

### Current State
Each craft tool (nmap/curl) has a single text input for target (IP/hostname/CIDR).

### New Behavior
Target input becomes a dual-mode field:

- **Single mode** (default): Current text input, unchanged behavior
- **List mode**: Textarea accepting one host per line + file import button

### UI Elements
- Mode toggle: pill/link next to target label — "Single | List"
- List mode shows:
  - Textarea with placeholder `one host per line`
  - "Import file" button accepting `.txt` files (one host per line)
  - Host count badge showing `N hosts`
- Hosts get trimmed, deduped, empty lines stripped on any input change

### Command Generation
- Single mode: current behavior, target inline in command
- List mode + batch OFF: generates `-iL targets.txt` command + instruction note "save hosts to targets.txt first"
- List mode + batch ON: generates CMD `for` loop (see Section 2)

## 2. CMD Batch Toggle

### Visibility
Appears in craft toggles area when: list mode active AND 2+ hosts entered.

### UI
- Toggle pill in existing toggles row — "Batch (CMD loop)"
- When enabled, command preview shows full batch script with syntax highlighting

### Generated Output

**Under 10 hosts — inline loop:**
```cmd
@echo off
for %%H in (host1 host2 host3) do (
    nmap -sS -T4 %%H
    echo ---
)
```

**10+ hosts — file-based loop:**
```cmd
@echo off
for /F "tokens=*" %%H in (targets.txt) do (
    nmap -sS -T4 %%H
    echo ---
)
```

Both patterns verified working on Windows 11 CMD.

### Behavior
- Under 10 hosts: `for %%H in (...)` — no external file needed
- 10+ hosts: `for /F ... in (targets.txt)` — shows "save list as targets.txt" instruction
- `echo ---` separator between results for later parsing
- Command preview shows full batch script
- Copy button copies entire script
### Edge Cases
- Batch toggle hidden when single target mode active
- Batch toggle auto-disables if user switches back to single mode
- Toggling batch off returns to normal single-command preview with `-iL` note

## 3. Matrix Comparer

### Location
Rescan History section — new view mode alongside existing side-by-side comparison.

### Entry Point
- Scan timeline gets multi-select checkboxes
- Select 2+ same-type scans → "Compare" (existing) and "Matrix" buttons appear
- Mixed scan types → matrix button disabled with tooltip "select same scan type"

### nmap-port Matrix
```
             | 22/tcp | 80/tcp | 443/tcp | 3389/tcp |
-------------|--------|--------|---------|----------|
192.168.1.1  | open   | open   | closed  | filtered |
192.168.1.2  | open   | closed | open    | open     |
10.0.0.5     | closed | open   | open    | closed   |
```

- Columns: union of all ports found across selected scans
- Cells color-coded: green=open, red=closed, yellow=filtered
- Sortable by clicking column headers

### curl-headers / Header Audit Matrix
```
             | X-Frame-Options | HSTS   | CSP    | Server     |
-------------|-----------------|--------|--------|------------|
app.prod.com | DENY            | max=31 | strict | (hidden)   |
app.uat.com  | SAMEORIGIN      | —      | —      | Apache/2.4 |
api.prod.com | DENY            | max=31 | strict | (hidden)   |
```

- Columns: union of all headers found across selected scans
- Missing headers show `—` in muted style
- Fingerprint/insecure values highlighted per existing HEADER_RULES

### Features
- Filter toggle: show only columns with differing values across hosts
- Export matrix as CSV
- Sticky first column (host names) on horizontal scroll
- Works within existing Prod/UAT environment tabs

## 4. JSON Upload Guardrails

### Rationale
`FileReader` + `JSON.parse` handles 50-100MB comfortably. Scan history files stay well under 5MB in practice. Guardrails are defensive, not solving a current problem.

### Implementation
Applied to both Rescan History and Header Audit file import handlers:

| File Size | Behavior |
|-----------|----------|
| Under 10MB | Load silently (current behavior) |
| 10-50MB | Warning toast: "Large file (Xmb). Loading may be slow." |
| Over 50MB | Block with error: "File too large. Max 50MB." |

### Additional Checks
- `try/catch` around `JSON.parse` — currently missing, would silently fail on malformed JSON
- On parse error: toast with "Invalid JSON format" message
- Entry count sanity check: if parsed entries > 5000, warn before merging

## Data Flow Summary

```
Target Input (Single/List mode)
    │
    ├─ Single → current command generation
    │
    └─ List → host list parsed
              │
              ├─ Batch OFF → command with -iL targets.txt + save note
              │
              └─ Batch ON → CMD for loop wrapping crafted command
                            │
                            ├─ <10 hosts → inline for %%H in (...)
                            └─ 10+ hosts → for /F in (targets.txt)

Rescan History
    │
    ├─ Select 2 same-type → Compare (existing side-by-side)
    │
    └─ Select 2+ same-type → Matrix View (new)
                              │
                              ├─ nmap-port → port×host matrix
                              └─ curl/audit → header×host matrix

File Upload (any handler)
    │
    ├─ Size check → warn/block
    ├─ JSON.parse try/catch → error toast
    └─ Entry count check → warn if >5000
```

## Terminal Aesthetic
All new UI follows existing design language:
- Dark-only, OCR A Extended monospace font
- Green accent (#00ff41), no border-radius/shadows/animations
- Pill toggles match existing craft toggle style
- Matrix table uses terminal grid styling consistent with existing comparison views
- Toast messages use existing notification pattern
