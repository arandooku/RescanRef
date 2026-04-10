# ScanRef Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single offline HTML file for shell/terminal command reference with interactive command crafting, verbose docs, and scan history comparison.

**Architecture:** Everything lives in one `index.html` file — HTML structure, CSS (via `<style>`), JS data objects, and rendering logic (via `<script>`). Data-driven: commands, craft tools, and docs are JS objects at the top of the script block; rendering functions consume them. Rescan history uses JSON files loaded via drag-and-drop / file picker. No external dependencies, no network requests, no localStorage (except theme toggle).

**Tech Stack:** Vanilla HTML/CSS/JS, CSS custom properties for theming, `navigator.clipboard` API, `FileReader` API for JSON import, Blob download for JSON export.

---

## File Structure

This is a single-file application. All code lives in `index.html`:

```
RescanRef/
  index.html          # The entire application (HTML + CSS + JS)
```

Logical sections within `index.html` (top to bottom):

1. `<style>` — CSS custom properties (Monokai palette), layout grid, component styles, responsive breakpoints, dark/light themes
2. `<body>` — HTML skeleton: sidebar, main area, mobile hamburger
3. `<script>` — Data section: `commands[]`, `craftTools{}`, `docs{}`
4. `<script>` — App logic: rendering, navigation, search, craft builders, parsers, comparison engine

Tasks build the file incrementally. Each task adds to the single file and produces a working result verifiable by opening `index.html` in a browser.

---

## Task 1: Create index.html — Full Application

**Files:**
- Create: `index.html`

This task creates the complete application in one shot: HTML skeleton, all CSS, all JS logic (sidebar, rendering, craft builders, parsers, comparison engine, search, theme toggle, keyboard shortcuts, responsive hamburger), and empty data placeholders.

- [ ] **Step 1: Create the complete `index.html`**

The file includes everything from the spec:
- Full Monokai dark/light CSS theme with custom properties
- Sidebar layout (fixed 220px, hamburger on mobile)
- Search bar with debounced filtering
- Reference section card renderer
- Craft section builder (pill groups, toggles, custom inputs, live command preview)
- Docs section renderer with platform info
- Rescan History UI (env tabs, drop zone, paste import, classify modal, scan timeline, comparison)
- All parsers (nmap port, nmap script, curl headers)
- Full comparison engine (port diff, script diff, header diff, raw diff)
- Clipboard copy with fallback
- Theme toggle with localStorage persistence
- Keyboard shortcuts (/, Esc, 1-9)

Data arrays are empty — populated in subsequent tasks.

- [ ] **Step 2: Verify the skeleton works**

Open `index.html` in a browser. Verify:
- Dark Monokai background, pink "ScanRef" title in sidebar
- Search bar present
- Theme toggle switches between dark and light
- On narrow viewport (<768px), sidebar collapses and hamburger appears
- `/` focuses search bar, `Esc` clears it
- "Rescan History" appears in sidebar and shows Prod/UAT tabs, drop zone

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add HTML skeleton with Monokai theme, sidebar layout, and app engine"
```

---

## Task 2: Reference Data — Starter Commands

**Files:**
- Modify: `index.html` (replace `const commands = [];`)

- [ ] **Step 1: Populate the `commands` array**

Replace the empty `commands` array with ~35 entries across 5 sections: Git (8), Docker (8), PowerShell (6), SSH/Network (6), npm/Node (7). Each entry: `{ section, title, cmd, notes, tags }`.

- [ ] **Step 2: Verify**

Sidebar shows Reference sections. Click each — cards render with tag badges, code blocks, copy buttons. Search filters across sections.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add starter reference commands for all 5 sections"
```

---

## Task 3: nmap Craft Tool Definition

**Files:**
- Modify: `index.html` (add `nmap` key to `craftTools`)

- [ ] **Step 1: Add nmap craft definition**

3 groups (Scan Type with 6 options, Ports with 4 options, Timing with 6 options) and 12 toggles. Each option/toggle has a `docId` linking to docs.

- [ ] **Step 2: Verify**

nmap crafter renders with target input, pill groups, toggles. Crafted command updates live. Custom inputs appear for Custom range ports, -oN, -oX. Copy works.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add nmap craft tool with scan types, ports, timing, and toggles"
```

---

## Task 4: curl Craft Tool Definition

**Files:**
- Modify: `index.html` (add `curl` key to `craftTools`)

- [ ] **Step 1: Add curl craft definition**

2 groups (Method with 6 options, Data/Body with 4 options) and 14 toggles covering headers, auth, options.

- [ ] **Step 2: Verify**

curl crafter shows method pills, data/body options, header toggles, option toggles. Custom inputs for bearer token, custom header, output file, timeout, proxy. Command updates live.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add curl craft tool with methods, headers, body, and options"
```

---

## Task 5: nmap Docs Entries

**Files:**
- Modify: `index.html` (add `nmap` key to `docs`)

- [ ] **Step 1: Add nmap docs**

28 entries covering all scan types (6), port options (4), timing levels (6), and toggles (12). Each entry: `{ id, flag, category, title, description, example, platforms: { linux, windows, powershell }, warnings[], tips[] }`. IDs match `docId` values from Task 3.

- [ ] **Step 2: Verify**

Docs section renders with flag badges, category pills, descriptions, platform info blocks. Info icons in nmap Craft navigate to correct doc entries. "Back to nmap Craft" link works.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add nmap docs with full flag reference and platform info"
```

---

## Task 6: curl Docs Entries

**Files:**
- Modify: `index.html` (add `curl` key to `docs`)

- [ ] **Step 1: Add curl docs**

25 entries covering methods (6), data/body (4), headers (5), and options (10). Same structure as nmap docs. IDs match `docId` values from Task 4.

- [ ] **Step 2: Verify**

curl Reference section renders correctly. Info icons in curl Craft navigate to correct entries. "Back to curl Craft" works. Search works.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add curl docs with full flag reference and platform info"
```

---

## Task 7: End-to-End Verification & Polish

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Test all user flows**

1. Reference: all 5 sections render, copy works
2. Craft nmap: target + pills + toggles + crafted command + copy
3. Craft curl: same
4. Docs nmap: entries render, platform info, Back to Craft
5. Docs curl: same
6. Craft -> Docs navigation: info icons jump to correct entries
7. Rescan History: Prod/UAT tabs, paste import, auto-detect, classify, save (JSON download)
8. Comparison: load JSON files, select 2 scans, view diff with color coding
9. Search: filters all sections, match counts in sidebar, yellow highlighting
10. Theme toggle: dark/light switch, readability
11. Responsive: hamburger menu below 768px
12. Keyboard: `/` search, `Esc` clear, `1-9` switch sections

- [ ] **Step 2: Fix issues found**

- [ ] **Step 3: Final commit**

```bash
git add index.html
git commit -m "fix: polish interactions and fix issues found in testing"
```
