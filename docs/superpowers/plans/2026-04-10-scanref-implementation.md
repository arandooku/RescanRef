# ScanRef Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single offline HTML file that serves as a command reference, interactive command crafter (nmap/curl), verbose docs, and rescan history tracker with side-by-side comparison.

**Architecture:** Single `index.html` with all CSS, JS, and data inline. Data-driven rendering — commands, craft tool definitions, and docs are JS objects at the top of the file. The app reads/writes JSON files for scan history via browser File API (drag & drop, file picker, download). No external dependencies, no localStorage, no network requests.

**Tech Stack:** Vanilla HTML/CSS/JS, system fonts, Monokai color palette, browser File API for JSON import/export.

---

## File Structure

- Create: `index.html` — the entire app (single file)
- Create: `.gitignore` — ignore `.superpowers/`, `history/` folder
- Create: `CLAUDE.md` — project context for Claude Code

All implementation tasks modify the same `index.html`. Each task adds a logical section to the file.

---

### Task 0: Project Setup + GitHub Repo

- [ ] **Step 1: Create .gitignore**

```gitignore
.superpowers/
history/
*.json
```

- [ ] **Step 2: Create the GitHub repo**

Run:
```bash
cd f:/Claude/RescanRef
gh repo create ScanRef --public --source=. --remote=origin
```

- [ ] **Step 3: Commit and push**

```bash
git add .gitignore
git commit -m "chore: add .gitignore"
git push -u origin master
```

---

### Task 1: HTML Skeleton + CSS Theme + Sidebar Layout

**Files:**
- Create: `index.html`

This is the foundation task. Creates the full HTML document structure with:
- Monokai CSS custom properties (dark + light themes)
- Complete CSS for all sections (sidebar, cards, craft UI, docs, history, comparison, responsive)
- Sidebar HTML (header, search, nav, theme toggle)
- Main content container
- Import modal HTML for Rescan History

- [ ] **Step 1: Create index.html with full CSS and HTML skeleton**

Write the complete `<!DOCTYPE html>` document with:
- `<style>` block containing all CSS custom properties for dark/light Monokai themes and all component styles
- Sidebar HTML with header ("ScanRef"), search input, nav container, theme toggle
- Main content container
- Import modal for Rescan History
- Hamburger button and overlay for mobile responsive

The CSS must cover: sidebar, command cards, craft UI (target input, pill buttons, toggle switches, custom inputs, command preview), docs entries (with platform sections), rescan history (dropzone, env tabs, scan timeline, comparison tables, raw diff panels), responsive breakpoints, and utility classes.

See the design spec at `docs/superpowers/specs/2026-04-10-scanref-design.md` for all color values, sizes, and layout details.

- [ ] **Step 2: Verify structure opens in browser**

Open `index.html` in browser. Should see an empty sidebar and main area with Monokai dark theme.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: HTML skeleton with Monokai CSS theme and sidebar layout"
```

---

### Task 2: Data Layer — Reference Commands

**Files:**
- Modify: `index.html` (add `<script>` section)

- [ ] **Step 1: Add the commands data array**

Inside a `<script>` tag at the bottom of `<body>`, add the `commands` array with starter entries for all 5 reference sections:

- **Git** (6 commands): undo last commit, interactive rebase, show file changes, stash with message, cherry-pick, list all branches
- **Docker** (5 commands): list containers, stop all, prune stopped, view logs, exec into container
- **PowerShell** (4 commands): find files, check listening ports, get service status, test network connectivity
- **SSH / Network** (4 commands): SSH with key, SSH tunnel, netstat, nslookup
- **npm / Node** (3 commands): list global packages, check outdated, clear cache

Each entry: `{ section, title, cmd, notes, tags }`.

See the design spec Data Structure section for the exact schema.

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add reference commands data (Git, Docker, PowerShell, SSH, npm)"
```

---

### Task 3: Data Layer — Craft Tool Definitions

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add the craftTools object**

Add the `craftTools` object with full definitions for both `nmap` and `curl`:

**nmap:** 3 groups (Scan Type with 6 options, Ports with 4 options, Timing with 6 options) + 12 toggles (OS Detection, Service Version, Aggressive, Verbose, Very Verbose, Skip host discovery, No DNS, Vuln scripts, Default scripts, Output to file, XML output, Show reason). Each option/toggle has a `docId` linking to its docs entry. Some toggles have `customInput: true` with a placeholder.

**curl:** 2 groups (Method with 6 options, Content Type with 3 options) + 12 toggles (Headers only, Verbose, Silent, Follow redirects, Insecure, Accept JSON, Bearer token, Show status code, Output to file, Connect timeout, Use proxy, Request body). Same docId and customInput pattern.

See spec sections "nmap Craft Options" and "curl Craft Options" for the complete lists.

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add craft tool definitions (nmap + curl)"
```

---

### Task 4: Data Layer — Docs Entries

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add the docs object with all nmap entries**

Add `docs.nmap` array with verbose entries for every nmap flag/option. Each entry:
```js
{ id, flag, category, title, description, platforms: { linux, windowsCmd, windowsPs }, example, warnings: [], tips: [] }
```

Must include entries for:
- 6 scan types: -sS, -sT, -sU, -sA, -sV, -sn
- 4 port options: default, -p-, --top-ports, -p custom
- 6 timing templates: T0-T5
- 12 toggles: -O, -sV, -A, -v, -vv, -Pn, -n, --script vuln, --script=default, -oN, -oX, --reason

Each description must be verbose (3-5 sentences minimum). Each `platforms` object must document Linux behavior, Windows CMD behavior (admin elevation, Npcap requirements, path differences), and Windows PowerShell behavior (alias conflicts, escaping quirks).

- [ ] **Step 2: Add the docs object with all curl entries**

Add `docs.curl` array with verbose entries for every curl flag/option. Same structure. Must include entries for:
- 6 methods: GET, POST, PUT, DELETE, PATCH, HEAD
- 3 content types: none, JSON, form-urlencoded
- 12 toggles: -I, -v, -s, -L, -k, Accept JSON, Bearer token, -w status, -o, --connect-timeout, -x proxy, -d data

Platform notes must document the PowerShell `curl` alias issue (`Invoke-WebRequest`), CMD JSON escaping, and Linux defaults.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add verbose docs entries for nmap and curl with platform notes"
```

---

### Task 5: App State + Utility Functions

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add app state variables**

```js
let activeSection = { group: 'reference', key: 'Git' };
let previousSection = null;
let craftState = {};
let scanEntries = [];
let searchQuery = '';
let isDark = true;
let historyEnv = 'Prod';
let selectedScans = [];
let compareMode = false;
```

- [ ] **Step 2: Add utility functions**

- `uuid()` — generate UUID v4
- `copyToClipboard(text, btnEl)` — copy to clipboard with `navigator.clipboard.writeText()` and fallback to `document.execCommand('copy')`, update button text to "Copied!" for 1.5s
- `esc(str)` — HTML-escape a string using a temporary div
- `highlightText(text, query)` — wrap search matches in `<span class="highlight">`
- `toggleTheme()` — toggle `light` class on `<html>`, update icon/label

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add app state and utility functions"
```

---

### Task 6: Sidebar Rendering + Navigation

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add sidebar rendering**

- `getReferenceSections()` — extract unique section names from `commands` array
- `renderSidebar()` — build sidebar HTML with group labels (Reference, Craft, Docs, Rescan History), items, active state highlighting, and match counts during search
- `countMatches(group, key)` — count search matches for a section

- [ ] **Step 2: Add navigation functions**

- `navigate(group, key, scrollToId)` — set active section, re-render sidebar + main, optionally scroll to element, close mobile sidebar
- `navigateToDoc(tool, docId)` — save current section as `previousSection`, navigate to docs with scroll
- `navigateBackToCraft()` — restore previous craft section
- `renderMain()` — dispatch to the correct render function based on `activeSection.group`

- [ ] **Step 3: Verify navigation works**

Open in browser. Click sidebar items — main area should update. Sections should highlight when active.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: sidebar rendering and section navigation"
```

---

### Task 7: Reference Section Renderer

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add renderReferenceSection(section)**

Filter `commands` by section name (and search query if active). Render each as a command card with:
- Title with search highlighting
- Tag badge
- Code block with copy button (calls `copyToClipboard`)
- Optional notes with search highlighting
- "No commands match" message when filter is empty

- [ ] **Step 2: Verify in browser**

Click "Git" in sidebar. Should see 6 command cards. Click Copy on one — should copy to clipboard and show "Copied!".

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: reference section rendering with copy buttons"
```

---

### Task 8: Craft Section — State + Rendering

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add craft state management**

- `initCraftState(tool)` — initialize state for a craft tool with default selections from `craftTools` definition
- `selectPill(tool, groupName, flag, btnEl)` — update group selection, re-render
- `toggleFlag(tool, flag, btnEl)` — toggle a boolean flag, show/hide custom input if applicable

- [ ] **Step 2: Add renderCraftSection(tool)**

Render the full craft UI:
1. Header (tool name + description)
2. Target input (bound to `craftState[tool].target`, pink border)
3. Each group as a card with pill buttons (active = pink, with "Compare all →" doc link)
4. Custom input that appears when a pill with `customInput: true` is selected
5. Toggles in a 2-column grid with switch buttons, ⓘ info links, and conditional custom inputs
6. Crafted command output card at bottom with green border and copy button

- [ ] **Step 3: Add buildCraftedCommand(tool)**

Assemble the command string from craft state:
- Start with tool name
- Add selected group flags (with custom input values where applicable)
- Add enabled toggle flags (with custom input values)
- Handle special cases (e.g., Bearer token format)
- End with target (or `<target>` placeholder)

- [ ] **Step 4: Add updateCraftPreview(tool)**

Update the preview element with the built command. Apply color classes:
- `.flag` (pink) for group option flags
- `.opt` (yellow) for toggle flags
- `.target` (green) for the target

- [ ] **Step 5: Verify in browser**

Click "nmap" in sidebar. Verify:
- Target input updates preview
- Pill buttons switch (one active per group)
- Toggles turn on/off
- Custom inputs appear/hide
- Preview updates in real-time
- Copy button works

Repeat for "curl".

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: craft section with interactive nmap and curl builders"
```

---

### Task 9: Docs Section Renderer + Craft↔Docs Navigation

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add renderDocsSection(tool)**

Render all doc entries for a tool:
- "← Back to Crafter" link (shown when navigated from craft)
- Section header
- Each entry as a doc card with:
  - Flag (large monospace) + category badge
  - Title
  - Description paragraph
  - Three platform sections (Linux, Windows CMD, Windows PowerShell) with distinct colored labels
  - Example in code block
  - Warning items (yellow)
  - Tip items

- [ ] **Step 2: Verify Craft → Docs → Craft flow**

1. Click "nmap" under Craft
2. Click ⓘ next to "-O OS Detection"
3. Should navigate to Docs, scrolled to the -O entry
4. "← Back to nmap Crafter" link should appear
5. Click it — should return to Craft with all selections preserved

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: docs section renderer with craft↔docs navigation"
```

---

### Task 10: Parsers — nmap Port, nmap Script, curl Headers

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add auto-detection functions**

- `detectScanType(raw)` — identify nmap-port, nmap-script, or curl-headers from raw text patterns
- `detectTarget(raw)` — extract target IP/host from raw output

- [ ] **Step 2: Add parseNmapPort(raw)**

Parse nmap normal output (-oN format):
- Extract target from "Nmap scan report for <target>"
- Extract host status and latency
- Parse the PORT/STATE/SERVICE/VERSION table using regex
- Extract OS details
- Return structured object matching spec's parsed structure

- [ ] **Step 3: Add parseNmapScript(raw)**

Parse nmap script output. First call `parseNmapPort` for base data, then:
- Parse `ssl-cert` blocks → subject, issuer, validFrom, validTo, SANs, expired, daysRemaining
- Parse `ssl-enum-ciphers` blocks → TLS versions with cipher lists and strength ratings
- Parse `http-methods` blocks → methods list and risky methods
- Parse `http-security-headers` blocks → present headers with values, missing headers
- Parse `vulners` blocks → service CPE, CVE list with scores
- Fallback: store unrecognized script blocks as raw text

- [ ] **Step 4: Add parseCurlHeaders(raw)**

Parse curl -I or curl -v output:
- Handle both formats (curl -I: plain headers; curl -v: lines starting with `< `)
- Extract status line (protocol + status code)
- Parse each header key:value pair
- Run security headers audit against the known checklist (11 headers)
- Return structured object with headers array and securityHeaders (present/missing)

- [ ] **Step 5: Add parseRawOutput(raw, scanType) dispatcher**

Route to the correct parser based on scan type.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: parsers for nmap port, nmap script, and curl headers"
```

---

### Task 11: Rescan History — UI, Import, File I/O

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add renderHistorySection()**

Render the history UI:
- Load/Save/Import buttons
- Drop zone for JSON files
- Prod/UAT environment tabs
- Scan entries grouped by target, sorted by timestamp descending
- Selectable entries (click to select, max 2, highlight selected)
- Compare button (appears when 2 selected)

- [ ] **Step 2: Add file I/O functions**

- `loadHistoryFiles()` — open file picker for .json, read and merge entries
- `readHistoryFile(file)` — parse JSON, deduplicate by id, merge into `scanEntries`
- `saveAllHistory()` — download all entries as single JSON file
- `saveSingleEntry(entry)` — download one entry with auto-generated filename
- `downloadJSON(data, filename)` — create blob, trigger download

- [ ] **Step 3: Add import modal functions**

- `openImportModal(rawText)` — show modal, optionally pre-fill raw output
- `closeImportModal()` — hide modal
- `autoDetectImportFields(raw)` — auto-detect scan type and target from pasted/dropped content
- `confirmImport()` — validate, parse, create entry, save, close modal

- [ ] **Step 4: Add history dropzone**

- `initHistoryDropzone()` — set up drag/drop listeners. JSON files → load as history. Other files → open import modal with file contents.

- [ ] **Step 5: Verify import flow**

1. Click "Scan History" → click "Import Scan Result"
2. Paste sample nmap output into textarea
3. Verify auto-detection fills in scan type and target
4. Enter title, click Import
5. Entry should appear in timeline
6. JSON file should auto-download

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: rescan history UI with import, drag & drop, and JSON file I/O"
```

---

### Task 12: Comparison Engine + Comparison View

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add comparison functions**

- `comparePorts(oldPorts, newPorts)` — compare port arrays, return diff items with status (fixed/new-issue/changed/unchanged)
- `compareSecurityHeaders(oldHdrs, newHdrs)` — compare security header present/missing lists

- [ ] **Step 2: Add renderComparisonView()**

When two scans are selected and compared:
- Back button to return to timeline
- Header bar with both scan titles + timestamps
- Summary badges (N fixed, N new, N changed, N unchanged)
- **Port comparison table**: Port | Service | Old State | New State | Version — color-coded rows
- **Security headers table**: Header | Old Value | New Value — color-coded rows
- **SSL cert comparison** (if both have ssl-cert scripts): field-by-field table
- **CVE comparison** (if both have vulners scripts): CVE | Score | Status (fixed/new/unchanged)
- **HTTP methods comparison** (if both have http-methods scripts)
- **Raw output** panel: side-by-side raw text

- [ ] **Step 3: Verify comparison**

1. Import two scan results for the same target (with different data)
2. Select both in the timeline
3. Click Compare
4. Verify side-by-side view shows correct color-coded diffs

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: comparison engine with side-by-side diff view"
```

---

### Task 13: Search + Keyboard Shortcuts + Responsive

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add search**

- Debounced input listener (200ms) on search input
- Updates `searchQuery`, re-renders sidebar (with match counts) and main content
- Reference section filters by title, cmd, notes, tags
- `/` keyboard shortcut focuses search
- `Esc` clears search and unfocuses

- [ ] **Step 2: Add responsive sidebar**

- `toggleSidebar()` — toggle `.open` class on sidebar and `.visible` on overlay
- Hamburger button triggers toggle

- [ ] **Step 3: Verify**

- Type "ssh" in search → only matching commands shown
- Press `/` → search focuses
- Press Esc → search clears
- Resize to <768px → hamburger appears, sidebar slides in/out

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: search, keyboard shortcuts, and responsive sidebar"
```

---

### Task 14: Init on Load

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add init calls at script bottom**

```js
renderSidebar();
renderMain();
```

- [ ] **Step 2: Final end-to-end verification**

Open `index.html` in browser and test the full flow:
1. Reference sections render with copy buttons
2. Craft nmap: set target, toggle options, copy crafted command
3. Craft curl: set target, toggle options, copy crafted command
4. Click ⓘ → navigates to Docs → Back to Craft works
5. Docs show all entries with platform sections
6. Rescan History: import a scan, save JSON, load JSON
7. Compare two scans side-by-side
8. Search works across sections
9. Theme toggle works
10. Responsive at narrow width

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: init rendering and final integration"
```

---

### Task 15: GitHub Push + CLAUDE.md

- [ ] **Step 1: Create CLAUDE.md**

```markdown
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ScanRef is a single offline HTML file (`index.html`) that serves as a command reference, interactive command crafter, and rescan history tracker. It lives in a OneDrive folder and syncs across Office 365 devices. Zero external dependencies.

## Architecture

Everything is in one file: `index.html`. The file is structured as:

1. **`<style>`** — All CSS with Monokai theme custom properties (dark/light)
2. **`<body>`** — Sidebar + main content container + import modal
3. **`<script>`** — All JS, organized in this order:
   - **Data arrays** (`commands`, `craftTools`, `docs`) — edit these to add/modify content
   - **App state** — reactive variables that drive rendering
   - **Utility functions** — clipboard, escaping, highlighting
   - **Sidebar + navigation** — renders sidebar, handles section switching
   - **Section renderers** — one function per section type (reference, craft, docs, history)
   - **Craft engine** — state management, command building, preview rendering
   - **Parsers** — nmap port, nmap script, curl headers
   - **Comparison engine** — diff logic for ports, headers, certs, CVEs
   - **File I/O** — JSON import/export via File API
   - **Search + keyboard shortcuts**
   - **Init** — initial render calls

## Key Constraints

- Single HTML file — no build step, no bundling
- Fully offline — no CDNs, no external fonts, no fetch calls
- No localStorage — theme resets on reload, scan history lives in JSON files
- Must work via file:// protocol in Chrome/Edge on Windows
- All data (commands, docs) is hardcoded in JS arrays at the top of the script

## Adding Content

- **New reference command**: Add object to `commands` array. New sections auto-appear in sidebar.
- **New craft tool**: Add key to `craftTools` with groups/toggles. Add matching `docs` entries.
- **New craft option**: Add to relevant tool's groups or toggles array + add docs entry with matching docId.

## Scan History

- Stored as JSON files in a `history/` folder alongside the HTML
- One file per scan session (default) or consolidated
- Import via drag & drop, file picker, or copy-paste
- Parsers handle: nmap port scans, nmap script scans (ssl-cert, ssl-enum-ciphers, http-methods, http-security-headers, vulners), curl header responses
```

- [ ] **Step 2: Push everything**

```bash
git add CLAUDE.md index.html
git commit -m "docs: add CLAUDE.md and finalize index.html"
git push
```
