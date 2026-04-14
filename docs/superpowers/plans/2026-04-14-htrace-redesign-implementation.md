# htrace.sh Terminal Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign ScanRef's entire visual appearance to match the htrace.sh terminal aesthetic — OCR A Extended font, green-dominant Monokai palette, sharp geometry, no light theme.

**Architecture:** CSS rewrite + targeted HTML changes in a single file (`index.html`, ~6880 lines). All JS logic stays untouched. Changes are purely cosmetic. The file has CSS (lines 1-1925), HTML (lines 1926-1865ish), and JS (lines 1926-6880).

**Tech Stack:** Pure HTML/CSS/JS, single file, no build step.

**Spec:** `docs/superpowers/specs/2026-04-14-htrace-redesign-design.md`

---

### Task 1: Strip Light Theme & Remove Theme Toggle

Remove the dual-theme system. Single dark theme only.

**Files:**
- Modify: `index.html:2` (html tag)
- Modify: `index.html:9-68` (CSS custom properties)
- Modify: `index.html:316-345` (theme toggle CSS)
- Modify: `index.html:1852-1857` (theme toggle HTML)
- Modify: `index.html:4155-4175` (theme JS functions)
- Modify: `index.html:6292` (theme event listener)

- [ ] **Step 1: Remove data-theme from html tag**

Change line 2 from:
```html
<html lang="en" data-theme="dark">
```
to:
```html
<html lang="en">
```

- [ ] **Step 2: Convert CSS custom properties to single theme**

Replace the entire `:root[data-theme="dark"] { ... }` block (lines 9-38) with a plain `:root { ... }` block using the new color values from the spec:

```css
:root {
  --sidebar-bg: #0e100c;
  --main-bg: #0a0c08;
  --accent-pink: #f92672;
  --accent-green: #a6e22e;
  --accent-cyan: #66d9ef;
  --accent-yellow: #e6db74;
  --accent-purple: #ae81ff;
  --accent-orange: #fd971f;
  --text-primary: #e0e0d8;
  --text-secondary: #8a8c84;
  --text-muted: #4a4c44;
  --code-bg: #0a0c08;
  --card-bg: #0e100c;
  --card-border: #1e201a;
  --input-bg: #0a0c08;
  --input-border: #1e201a;
  --toggle-off-bg: #2a2c24;
  --toggle-on-bg: var(--accent-green);
  --scrollbar-thumb: #2a2c24;
  --scrollbar-track: transparent;
  --mark-bg: rgba(230, 219, 116, 0.2);
  --overlay-bg: rgba(0,0,0,0.8);
  --shadow: none;
  --card-hover-border: #3e403a;
  --sidebar-separator: rgba(255,255,255,0.04);
}
```

- [ ] **Step 3: Delete the light theme block**

Delete the entire `:root[data-theme="light"] { ... }` block (lines 40-68).

- [ ] **Step 4: Delete glow variables**

Remove these three lines from the dark theme block (they were on lines 33-35):
```css
  --glow-pink: 0 0 20px rgba(249, 38, 114, 0.15);
  --glow-green: 0 0 20px rgba(166, 226, 46, 0.12);
  --glow-cyan: 0 0 20px rgba(102, 217, 239, 0.12);
```

- [ ] **Step 5: Delete theme toggle CSS**

Delete the theme toggle CSS block (lines 316-345):
```css
.theme-toggle { ... }
.theme-toggle:hover { ... }
.theme-toggle-track { ... }
.theme-toggle-track::after { ... }
:root[data-theme="light"] .theme-toggle-track { ... }
:root[data-theme="light"] .theme-toggle-track::after { ... }
```

- [ ] **Step 6: Remove theme toggle HTML**

Replace the sidebar footer (lines 1852-1857):
```html
    <div class="sidebar-footer">
      <div class="theme-toggle" id="themeToggle">
        <div class="theme-toggle-track"></div>
        <span id="themeLabel">Dark</span>
      </div>
    </div>
```
with:
```html
    <div class="sidebar-footer">
      <span style="font-size:0.6rem; color:var(--text-muted); letter-spacing:1px;">SCANREF &middot; OFFLINE</span>
    </div>
```

- [ ] **Step 7: Neutralize theme JS**

Replace the three theme functions (lines 4155-4175) with no-ops so nothing breaks:
```js
function initTheme() {}
function toggleTheme() {}
function updateThemeLabel() {}
```

Remove the event listener on line 6292:
```js
document.getElementById('themeToggle').addEventListener('click', toggleTheme);
```

- [ ] **Step 8: Verify in browser**

Open `index.html` via file:// protocol. Confirm:
- No light theme toggle visible
- Dark colors render correctly
- No JS console errors

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "refactor: strip light theme and remove theme toggle"
```

---

### Task 2: Typography & Font

Switch the entire app to OCR A Extended monospace. Remove all sans-serif font-family declarations.

**Files:**
- Modify: `index.html:76` (body font-family)
- Modify: `index.html:137` (sidebar header h1 font)
- Modify: `index.html:413` (section-header h2 font)
- Modify: `index.html:503` (code-block font)
- Modify: `index.html:1362` (audit-textarea font)
- Multiple other font-family declarations throughout CSS

- [ ] **Step 1: Change body font-family**

Line 76, change:
```css
  font-family: 'Segoe UI', 'SF Pro Display', -apple-system, sans-serif;
```
to:
```css
  font-family: 'OCR A Extended', 'OCR A Std', Consolas, monospace;
  font-size: 13px;
```

- [ ] **Step 2: Remove redundant monospace font-family declarations**

Since body is now monospace, all elements inherit. Find and remove every standalone `font-family: 'Cascadia Code', 'Fira Code', 'JetBrains Mono', Consolas, monospace;` declaration. These appear on:
- `.sidebar-header h1` (line 137)
- `.section-header h2` (line 413)
- `.code-block` (line 503)
- `.audit-textarea` (line 1362)
- Any other `font-family` declarations referencing Cascadia/Fira/JetBrains

Replace each with `font-family: inherit;` or simply delete the line (they inherit from body).

- [ ] **Step 3: Verify in browser**

Open `index.html`. Confirm OCR A Extended renders everywhere — sidebar, headers, code blocks, inputs, buttons. All monospace, no sans-serif anywhere.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "style: switch to OCR A Extended monospace font throughout"
```

---

### Task 3: Geometry — Kill Border Radius, Shadows, Animations, Gradients

Strip all rounded corners, box shadows, entrance animations, and ambient gradients.

**Files:**
- Modify: `index.html` — approximately 50+ CSS rules across lines 86-1800

- [ ] **Step 1: Remove all keyframe animations**

Delete the four keyframe blocks (lines 99-114):
```css
@keyframes fadeInUp { ... }
@keyframes fadeIn { ... }
@keyframes slideInLeft { ... }
@keyframes pulseGlow { ... }
```

- [ ] **Step 2: Remove all animation usages**

Find and delete every `animation:` property line. These appear on:
- `.card` (line 442): `animation: fadeInUp 0.15s ease-out both;`
- `.card:nth-child(N)` (lines 444-450): `animation-delay` lines
- `.section-header` (line 408): `animation: fadeInUp 0.35s ease-out;`
- `.craft-section` area (line 543): `animation: fadeInUp 0.35s ease-out;`
- `.craft-output-container` (line 762): `animation: pulseGlow 4s ease-in-out infinite;`
- `.docs-section` area (line 813): `animation: fadeInUp 0.2s ease-out;`
- `.doc-card` (line 869): `animation: fadeInUp 0.15s ease-out both;`
- `.rescan-section` area (line 991): `animation: fadeInUp 0.35s ease-out;`
- `.modal` (line 1101): `animation: fadeInUp 0.3s ease-out;`

- [ ] **Step 3: Set all border-radius to 0**

Find every `border-radius` declaration in the CSS and set it to 0. There are ~45 instances. Use a bulk find-replace approach: replace all `border-radius: Npx;` and `border-radius: N%;` with `border-radius: 0;`.

Key locations:
- Scrollbar thumb (line 86)
- Mark highlight (line 93)
- Sidebar search input (line 157)
- Match count badges (line 211)
- Card (line 439)
- Tag badges (line 480)
- Code blocks (line 501)
- Copy button (line 516)
- All craft section inputs/buttons (lines 571-799)
- Toggle switches (lines 725, 735, 756)
- Docs section cards (lines 854-970)
- Rescan history elements (lines 998-1096)
- Modal (line 1096, 1125, 1137, 1147)
- Audit elements (lines 1351-1800+)

- [ ] **Step 4: Remove ambient gradient**

Delete the `.main-content::before` rule (lines 396-404):
```css
.main-content::before {
  content: '';
  position: fixed;
  top: -30%; right: -20%;
  width: 60%; height: 60%;
  background: radial-gradient(ellipse, rgba(249, 38, 114, 0.03) 0%, transparent 70%);
  pointer-events: none;
  z-index: 0;
}
```

- [ ] **Step 5: Remove hover transforms**

Find and remove all `transform: translateY(-1px)` from hover states. These appear on:
- `.craft-btn-run:hover` (line 805)
- `.import-btn:hover` (line 1078)
- `.modal-btn-save:hover` (line 1153)
- `.audit-btn-analyze:hover` (line 1381)
- `.compare-btn:hover` (line 1219)
- Any other `:hover` rules with `translateY`

Also remove associated `box-shadow` values from those same hover rules.

- [ ] **Step 6: Remove all remaining box-shadow declarations**

Remove `box-shadow` from:
- `.sidebar-header h1` text-shadow (line 143): `text-shadow: 0 0 24px rgba(249, 38, 114, 0.3);`
- Focus states with `box-shadow: 0 0 0 3px ...` (lines 166, 577, 648, 671, 1012, etc.)
- `.craft-output-container` glow (line 761)
- Any remaining `var(--glow-*)` references

Exception: keep `box-shadow` on status dot indicators if they exist (for the colored glow effect on audit pass/warn/fail dots).

- [ ] **Step 7: Verify in browser**

Open `index.html`. Confirm:
- Everything has sharp corners
- No fade-in animations on page load or section switch
- No ambient pink gradient in background
- No floating/lifting hover effects on buttons
- Clean, flat terminal appearance

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "style: remove border-radius, shadows, animations, gradients"
```

---

### Task 4: Accent Color Swap — Pink to Green

Change the primary UI accent from pink to green. Pink stays only for errors/failures/critical states and the SCANREF brand text.

**Files:**
- Modify: `index.html` — ~72 references to accent-pink/f92672/rgba(249,38,114) across CSS

**Important:** Do NOT change `.sidebar-header h1` color (line ~140) — it stays `var(--accent-pink)` as the brand/logo color. This is the one intentional pink UI element.

- [ ] **Step 1: Sidebar navigation — pink to green**

Change these rules to use green instead of pink:

`.nav-item:hover` (line ~195):
```css
.nav-item:hover {
  background: rgba(166, 226, 46, 0.06);
  color: var(--text-primary);
  border-left-color: rgba(166, 226, 46, 0.3);
}
```

`.nav-item.active` (line ~200):
```css
.nav-item.active {
  background: linear-gradient(90deg, rgba(166, 226, 46, 0.15) 0%, transparent 100%);
  color: var(--accent-green);
  font-weight: 600;
  border-left-color: var(--accent-green);
}
```

`.match-count` badge (lines ~208-215):
```css
.nav-item .match-count {
  font-size: 0.65rem;
  font-weight: 700;
  background: rgba(166, 226, 46, 0.15);
  color: var(--accent-green);
  padding: 1px 7px;
  min-width: 20px;
  text-align: center;
}
.nav-item.active .match-count { background: rgba(166, 226, 46, 0.25); }
```

- [ ] **Step 2: Tree nav — pink to green**

Same changes for `.nav-tree-parent` hover/active states (lines ~232-301):
- Replace all `rgba(249, 38, 114, ...)` with `rgba(166, 226, 46, ...)`
- Replace `var(--accent-pink)` with `var(--accent-green)`

Apply to: `.nav-tree-parent:hover`, `.nav-tree-parent.active`, `.nav-tree-child:hover`, `.nav-tree-child.active`

- [ ] **Step 3: Sidebar search — pink to green focus**

`.sidebar-search input:focus` (lines ~165-167):
```css
.sidebar-search input:focus {
  border-color: var(--accent-green);
  box-shadow: none;
}
```

- [ ] **Step 4: Hamburger — pink to green**

`.hamburger:hover` (line ~364):
```css
.hamburger:hover { border-color: var(--accent-green); }
```

- [ ] **Step 5: Craft section — pink to green**

Change all pink references in the craft section (lines ~539-808):
- `.craft-target:focus` border: green instead of pink
- `.craft-flag-btn.active` border/background: green instead of pink
- `.craft-flag-btn.active` checked state: green
- `.craft-select:focus`: green border
- `.crafted-command .cmd-flag`: change from `var(--accent-pink)` to `var(--accent-cyan)` (flags are cyan in htrace style)
- `.craft-btn-run`: green background instead of pink
- All associated hover states and box-shadows

- [ ] **Step 6: Rescan History — pink to green for UI elements**

Change non-error pink references (lines ~987-1305):
- `.env-tab:hover`, `.env-tab.active`: green instead of pink
- `.paste-textarea:focus`: green border
- `.scan-entry.selected`: green border/background
- `.compare-btn`: keep as-is or change to green (it's a primary action)
- `.import-btn`: already green, keep

Keep pink for: `.diff-table tr.diff-new-issue`, `.diff-line-added` (these represent regressions/new issues — pink is correct as error color)

- [ ] **Step 7: Header Audit — pink to green for UI, keep pink for failures**

Change (lines ~1351-1807):
- `.audit-textarea:focus`: green border
- `.audit-btn-analyze`: green background instead of pink
- `.audit-drop-overlay.dragover`: green border instead of pink

Keep pink for (these are correct as error/failure indicators):
- `.audit-section-count.red`
- `.ha-badge-danger`
- `.audit-comp-badge.new-issue`
- `.audit-comp-badge.regressed`
- `.audit-comp-group-label.new`
- `.audit-comp-item.new`

- [ ] **Step 8: Modal — pink to green**

- `.modal-field input:focus`: green border (line ~1130)
- Any modal pink accent references

- [ ] **Step 9: Selection glow & focus visible — pink to green**

Lines ~1336-1345:
- `.selection-glow`: green instead of pink
- `:focus-visible`: green outline instead of pink

- [ ] **Step 10: Search results highlight — pink to green**

Line ~1329: change search match color from pink to green.

- [ ] **Step 11: JS inline style references**

Lines ~5982 and ~6040 set `borderColor = 'var(--accent-pink)'` in JS. Change to `'var(--accent-green)'`.

- [ ] **Step 12: Verify in browser**

Open `index.html`. Click through every section:
- Sidebar: green active states, green hover, green match counts
- Reference cards: expand some, verify no pink except in error contexts
- Craft: build a command, verify green buttons/inputs
- Header Audit: paste some headers, verify green analyze button, green focus
- Rescan History: verify green selection states
- Check that error/failure indicators remain pink

- [ ] **Step 13: Commit**

```bash
git add index.html
git commit -m "style: swap primary accent from pink to green, keep pink for errors"
```

---

### Task 5: Section Headers — Terminal Prompt Style

Add terminal-prompt style lines above section headers via JS render functions (since headers are JS-generated).

**Files:**
- Modify: `index.html:406-423` (section-header CSS)
- Modify: `index.html:4407-4413` (reference section render)
- Modify: `index.html:4577-4584` (craft section render)
- Modify: `index.html:4793-4819` (docs section render)
- Modify: `index.html:4930` (rescan history render)
- Modify: `index.html:6306` (header audit render)

- [ ] **Step 1: Add section-prompt CSS**

Add new CSS after the `.section-header` rules (around line 423):
```css
.section-prompt {
  font-size: 0.75rem;
  color: var(--text-muted);
  margin-bottom: 4px;
  letter-spacing: 0.5px;
}
.section-prompt .cmd { color: var(--accent-green); font-weight: 700; }
.section-prompt .flag { color: var(--accent-cyan); }
.section-prompt .arg { color: var(--accent-yellow); }
```

- [ ] **Step 2: Restyle section-header h2**

Change `.section-header h2` (line ~413) to a terminal box style:
```css
.section-header h2 {
  font-size: 1rem;
  font-weight: 700;
  color: var(--accent-green);
  letter-spacing: 0.5px;
  display: inline-block;
  border: 1px solid rgba(166, 226, 46, 0.3);
  padding: 2px 10px;
}
```

- [ ] **Step 3: Update reference section render (JS)**

Find the render function around line 4407/4413. The current code builds:
```js
'<div class="section-header"><h2>' + escapeHtml(sectionName) + '</h2><p>' + count + ' commands</p></div>'
```

Change to:
```js
'<div class="section-header"><div class="section-prompt"># <span class="cmd">scanref</span> <span class="flag">--section</span> <span class="arg">' + escapeHtml(sectionName.toLowerCase().replace(/\s+/g, '-')) + '</span> <span class="flag">--count</span> <span class="arg">' + count + '</span></div><h2>' + escapeHtml(sectionName) + '</h2></div>'
```

- [ ] **Step 4: Update craft section render (JS)**

Find the craft render around line 4577/4584. Change similarly:
```js
'<div class="section-header"><div class="section-prompt"># <span class="cmd">scanref</span> <span class="flag">--craft</span> <span class="arg">' + escapeHtml(toolKey) + '</span></div><h2>' + escapeHtml(tool.name || toolKey) + '</h2></div>'
```

- [ ] **Step 5: Update docs section render (JS)**

Find the docs render around line 4793/4819. Change:
```js
'<div class="section-header"><div class="section-prompt"># <span class="cmd">scanref</span> <span class="flag">--docs</span> <span class="arg">' + escapeHtml(toolKey) + '</span></div><h2>' + escapeHtml(toolKey) + ' Reference</h2></div>'
```

- [ ] **Step 6: Update rescan history render (JS)**

Find the rescan render around line 4930. Change:
```js
'<div class="section-header"><div class="section-prompt"># <span class="cmd">scanref</span> <span class="flag">--history</span></div><h2>Rescan History</h2></div>'
```

- [ ] **Step 7: Update header audit render (JS)**

Find the audit render around line 6306. Change:
```js
'<div class="section-header"><div class="section-prompt"># <span class="cmd">scanref</span> <span class="flag">--audit</span> <span class="flag">--headers</span></div><h2>Header Audit</h2></div>'
```

- [ ] **Step 8: Update search results render (JS)**

Lines ~6176/6180 — add prompt to search results too:
```js
'<div class="section-header"><div class="section-prompt"># <span class="cmd">scanref</span> <span class="flag">--search</span> <span class="arg">"' + escapeHtml(searchQuery) + '"</span></div><h2>Search Results</h2></div>'
```

- [ ] **Step 9: Verify in browser**

Click through each section. Verify each one shows a terminal-prompt line above the section title. The title should appear as a green bordered box.

- [ ] **Step 10: Commit**

```bash
git add index.html
git commit -m "feat: add terminal-prompt style section headers"
```

---

### Task 6: Button & Input Restyling

Convert all buttons to terminal-style bordered text. Restyle all inputs for the terminal aesthetic.

**Files:**
- Modify: `index.html` — button CSS across craft, audit, rescan, modal sections

- [ ] **Step 1: Restyle craft section buttons**

`.craft-btn-run` and similar (around lines 799-808):
```css
.craft-btn-run {
  padding: 6px 16px;
  background: transparent;
  border: 1px solid var(--accent-green);
  color: var(--accent-green);
  font-family: inherit;
  font-size: 0.75rem;
  cursor: pointer;
  text-transform: uppercase;
  letter-spacing: 1px;
  transition: background 0.15s;
}
.craft-btn-run:hover {
  background: rgba(166, 226, 46, 0.06);
}
```

- [ ] **Step 2: Restyle audit buttons**

`.audit-btn-analyze` (lines ~1371-1381):
```css
.audit-btn-analyze {
  padding: 7px 18px;
  background: transparent;
  color: var(--accent-green);
  font-weight: 700;
  border: 1px solid var(--accent-green);
  cursor: pointer;
  font-family: inherit;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 1px;
  transition: background 0.15s;
}
.audit-btn-analyze:hover { background: rgba(166, 226, 46, 0.06); }
```

`.audit-btn-clear` (lines ~1382-1391):
```css
.audit-btn-clear {
  padding: 7px 14px;
  background: transparent;
  border: 1px solid var(--card-border);
  color: var(--text-secondary);
  cursor: pointer;
  font-family: inherit;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 1px;
  transition: border-color 0.15s;
}
.audit-btn-clear:hover { border-color: var(--text-muted); }
```

`.audit-btn-save` (lines ~1392-1402):
```css
.audit-btn-save {
  padding: 7px 14px;
  background: transparent;
  border: 1px solid var(--accent-green);
  color: var(--accent-green);
  font-weight: 700;
  cursor: pointer;
  font-family: inherit;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 1px;
  transition: background 0.15s;
}
.audit-btn-save:hover { background: rgba(166, 226, 46, 0.06); }
```

`.audit-btn-compare` (lines ~1403-1415):
```css
.audit-btn-compare {
  display: none;
  padding: 7px 18px;
  background: transparent;
  color: var(--accent-cyan);
  font-weight: 700;
  border: 1px solid var(--accent-cyan);
  cursor: pointer;
  font-family: inherit;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 1px;
  transition: background 0.15s;
}
.audit-btn-compare:hover { background: rgba(102, 217, 239, 0.06); }
.audit-btn-compare.visible { display: inline-block; }
```

- [ ] **Step 3: Restyle rescan history buttons**

Apply the same terminal-bordered pattern to `.compare-btn`, `.import-btn`, `.export-btn`, and any other buttons in the rescan section.

- [ ] **Step 4: Restyle modal buttons**

`.modal-btn-save`, `.modal-btn-cancel` — same terminal border pattern. Save = green, cancel = muted.

- [ ] **Step 5: Restyle copy buttons**

`.copy-btn` (lines ~511-537):
```css
.copy-btn {
  position: absolute;
  top: 5px; right: 5px;
  background: transparent;
  border: 1px solid var(--card-border);
  color: var(--text-muted);
  font-size: 0.6rem;
  font-family: inherit;
  padding: 2px 8px;
  cursor: pointer;
  transition: all 0.15s;
  letter-spacing: 1px;
  text-transform: uppercase;
}
.copy-btn:hover { border-color: var(--accent-green); color: var(--accent-green); }
```

- [ ] **Step 6: Restyle all inputs and textareas**

Ensure all input/textarea styles use:
- `border-radius: 0` (already handled in Task 3)
- `background: var(--input-bg)`
- `border: 1px solid var(--input-border)`
- `font-family: inherit`
- Focus: `border-color: var(--accent-green)`
- Placeholder: `color: #3a3c34`

- [ ] **Step 7: Verify in browser**

Test all interactive elements:
- Craft tool: toggle flags, click run
- Audit: analyze, clear, save, compare buttons
- Rescan: import, compare, export buttons
- Modal: save, cancel
- Copy buttons on code blocks

All should be flat bordered text, uppercase, no rounded corners.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "style: restyle all buttons and inputs to terminal aesthetic"
```

---

### Task 7: Card & Code Block Restyling

Refine reference cards and code blocks for the terminal look.

**Files:**
- Modify: `index.html:425-537` (card and code block CSS)

- [ ] **Step 1: Restyle cards**

`.card` (line ~436):
```css
.card {
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  overflow: hidden;
  transition: border-color 0.15s;
}
.card:hover { border-color: var(--card-hover-border); }
```

`.card-header` — change chevron color to green:
```css
.card-chevron {
  font-size: 0.6rem;
  color: var(--text-muted);
  transition: transform 0.12s ease;
  flex-shrink: 0;
  margin-top: 3px;
}
```

- [ ] **Step 2: Restyle code blocks**

`.code-block` (line ~498):
```css
.code-block {
  background: var(--code-bg);
  border: 1px solid var(--card-border);
  padding: 8px 40px 8px 12px;
  font-family: inherit;
  font-size: 0.8rem;
  color: var(--accent-green);
  overflow-x: auto;
  white-space: pre-wrap;
  word-break: break-all;
  line-height: 1.5;
}
```

- [ ] **Step 3: Verify in browser**

Expand some reference cards. Verify code blocks are flat, green text on dark bg, sharp edges.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "style: restyle cards and code blocks for terminal look"
```

---

### Task 8: Scrollbar & Final Polish

Fine-tune scrollbar, mark highlight, and any remaining elements.

**Files:**
- Modify: `index.html:84-96` (scrollbar, mark)
- Modify: `index.html` — any remaining untouched elements

- [ ] **Step 1: Restyle scrollbar**

```css
::-webkit-scrollbar { width: 4px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: var(--scrollbar-thumb); }
::-webkit-scrollbar-thumb:hover { background: var(--text-muted); }
```

- [ ] **Step 2: Restyle mark highlight**

```css
mark {
  background: var(--mark-bg);
  color: inherit;
  padding: 1px 3px;
  box-decoration-break: clone;
  -webkit-box-decoration-break: clone;
}
```

- [ ] **Step 3: Remove z-index from section-header and cards-grid**

The z-index was needed for the ambient gradient overlay. Remove:
- `.section-header { position: relative; z-index: 1; }` — delete these two properties
- `.cards-grid { position: relative; z-index: 1; }` — delete these two properties

- [ ] **Step 4: Audit the entire CSS for any remaining inconsistencies**

Scan through the full CSS for:
- Any remaining `border-radius` values > 0
- Any remaining pink hover states that should be green
- Any remaining `box-shadow` not on status dots
- Any remaining `animation:` properties
- Any remaining `font-family` with sans-serif

- [ ] **Step 5: Full browser walkthrough**

Open `index.html` and test every section:
1. Sidebar navigation — click through all sections
2. Reference cards — expand/collapse, copy commands
3. Craft tools — build nmap and curl commands
4. Docs — expand flag references
5. Header Audit — paste sample headers, analyze, save
6. Rescan History — import, select, compare
7. Search — use the search box
8. Modal — open classify modal

Verify consistent terminal aesthetic throughout.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "style: scrollbar, mark highlight, and final terminal polish"
```

---

### Task 9: Mobile Responsive Check

Verify the terminal aesthetic works on narrow viewports.

**Files:**
- Modify: `index.html` — mobile media queries if needed

- [ ] **Step 1: Test at mobile width**

Open browser dev tools, resize to 400px width. Check:
- Hamburger menu works and is styled correctly
- Sidebar opens/closes
- Cards stack properly in single column
- Code blocks don't overflow
- Buttons remain usable
- Text remains readable at small sizes

- [ ] **Step 2: Fix any mobile issues**

If any elements break at narrow widths, fix the relevant CSS. Common issues:
- Long terminal-prompt lines may need `overflow-x: auto` or `white-space: nowrap` with scroll
- Button rows may need `flex-wrap: wrap`

- [ ] **Step 3: Commit if changes needed**

```bash
git add index.html
git commit -m "fix: mobile responsive adjustments for terminal redesign"
```
