# Multi-Target, Batch Loop, Matrix Comparer & Upload Guardrails Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add multi-target host input, CMD batch loop generation, N-host matrix comparison, and JSON upload guardrails to the ScanRef single-file HTML app.

**Architecture:** All changes go in `index.html`. Multi-target input extends the existing craft tool target area. Batch toggle integrates into existing craft toggles. Matrix comparer is a new view mode in Rescan History alongside the existing 2-scan comparison. Upload guardrails wrap existing `handleFilesDrop` and `handleAuditFileDrop` functions.

**Tech Stack:** Vanilla HTML/CSS/JS, no dependencies. Single-file app opened via `file://` protocol.

---

## File Structure

All changes in one file:

- **Modify:** `index.html`
  - CSS section (~line 1232): New styles for target list mode, matrix table, upload toasts
  - `craftState` initialization (~line 4321): Add `targetMode`, `targetList`, `batchEnabled` fields
  - `buildCraftedCommand` (~line 4339): Handle list mode and batch generation
  - `colorCodeCommand` (~line 4377): Handle batch script rendering
  - `renderCraftSection` (~line 4415): Dual-mode target input, batch toggle
  - `attachCraftHandlers` (~line 4504): List mode handlers, file import, batch toggle
  - `renderRescanSection` (~line 4780): Matrix view button
  - `renderScanTimeline` (~line 4829): Multi-select checkboxes
  - New function `renderMatrixView`: Matrix comparison for N hosts
  - `handleFilesDrop` (~line 4974): Size/parse guardrails
  - `handleAuditFileDrop` (~line 6568): Size/parse guardrails

**Spec:** `docs/superpowers/specs/2026-04-15-multi-target-batch-matrix-design.md`

---

### Task 1: Add craftState Target List Fields

**Files:**
- Modify: `index.html:4321-4337` (getCraftState function)

- [ ] **Step 1: Extend `getCraftState` to include target list fields**

In `getCraftState` (line 4321), change the state initialization from:

```javascript
var state = { target: '', groups: {}, toggles: {}, customInputs: {} };
```

to:

```javascript
var state = { target: '', targetMode: 'single', targetList: [], batchEnabled: false, groups: {}, toggles: {}, customInputs: {} };
```

- [ ] **Step 2: Verify no breakage — open `index.html` in browser**

Open `index.html` via `file://`, navigate to Craft > nmap and Craft > curl. Confirm both still render and generate commands correctly. The new fields are unused at this point so behavior should be identical.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: extend craftState with targetMode, targetList, batchEnabled fields"
```

---

### Task 2: CSS for Multi-Target Input and Batch

**Files:**
- Modify: `index.html` CSS section (find the `.target-input-wrapper` styles, around line 1232+)

- [ ] **Step 1: Find existing target input CSS**

Search for `.target-input-wrapper` in the CSS section to locate where to add new styles.

- [ ] **Step 2: Add CSS for target mode toggle, list textarea, host badge, and batch output**

Add these styles immediately after the existing `.target-input-wrapper` and `.target-input` rules:

```css
/* ===== Multi-Target Input ===== */
.target-mode-toggle {
  display: inline-flex;
  gap: 0;
  margin-left: 10px;
  font-size: 0.7rem;
}
.target-mode-toggle button {
  background: transparent;
  border: 1px solid var(--border);
  color: var(--text-muted);
  padding: 1px 8px;
  cursor: pointer;
  font-family: inherit;
  font-size: 0.7rem;
}
.target-mode-toggle button:first-child {
  border-right: none;
}
.target-mode-toggle button.active {
  background: rgba(0, 255, 65, 0.1);
  color: var(--accent-green);
  border-color: var(--accent-green);
}
.target-list-area {
  display: none;
  flex-direction: column;
  gap: 6px;
  margin-top: 6px;
}
.target-list-area.visible {
  display: flex;
}
.target-list-textarea {
  width: 100%;
  min-height: 80px;
  max-height: 200px;
  background: var(--bg-input);
  color: var(--text);
  border: 1px solid var(--border);
  padding: 6px 8px;
  font-family: inherit;
  font-size: 0.8rem;
  resize: vertical;
}
.target-list-toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.7rem;
}
.target-list-toolbar .host-count {
  color: var(--accent-cyan);
  font-size: 0.7rem;
}
.target-list-toolbar .import-file-btn {
  background: transparent;
  border: 1px solid var(--border);
  color: var(--text-muted);
  padding: 2px 8px;
  cursor: pointer;
  font-family: inherit;
  font-size: 0.7rem;
}
.target-list-toolbar .import-file-btn:hover {
  color: var(--accent-green);
  border-color: var(--accent-green);
}
.target-list-file-input {
  display: none;
}

/* ===== Batch Script Output ===== */
.batch-script-note {
  color: var(--text-muted);
  font-size: 0.7rem;
  margin-top: 4px;
  font-style: italic;
}
.crafted-command.batch-mode {
  white-space: pre;
  font-size: 0.75rem;
  line-height: 1.5;
}
```

- [ ] **Step 3: Verify CSS loads — open `index.html`, confirm no visual regressions in craft sections**

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "style: add CSS for multi-target input, list mode, and batch output"
```

---

### Task 3: Render Dual-Mode Target Input

**Files:**
- Modify: `index.html:4415-4502` (renderCraftSection function)

- [ ] **Step 1: Replace the target input block in `renderCraftSection`**

In `renderCraftSection` (line 4415), find the target input block (lines 4427-4432):

```javascript
  // Target input
  html += '<div class="target-input-wrapper">' +
    '<label>Target</label>' +
    '<input type="text" class="target-input" id="craftTarget" value="' + escapeHtml(state.target) +
    '" placeholder="' + escapeHtml(tool.targetPlaceholder || 'Enter target') + '">' +
    '</div>';
```

Replace with:

```javascript
  // Target input — dual mode
  html += '<div class="target-input-wrapper">' +
    '<label>Target' +
    '<span class="target-mode-toggle">' +
    '<button class="target-mode-btn' + (state.targetMode === 'single' ? ' active' : '') + '" data-mode="single">Single</button>' +
    '<button class="target-mode-btn' + (state.targetMode === 'list' ? ' active' : '') + '" data-mode="list">List</button>' +
    '</span></label>' +
    '<input type="text" class="target-input' + (state.targetMode === 'list' ? '" style="display:none' : '') + '" id="craftTarget" value="' + escapeHtml(state.target) +
    '" placeholder="' + escapeHtml(tool.targetPlaceholder || 'Enter target') + '">' +
    '<div class="target-list-area' + (state.targetMode === 'list' ? ' visible' : '') + '">' +
    '<textarea class="target-list-textarea" id="craftTargetList" placeholder="one host per line">' + escapeHtml(state.targetList.join('\n')) + '</textarea>' +
    '<div class="target-list-toolbar">' +
    '<span class="host-count" id="hostCount">' + state.targetList.length + ' hosts</span>' +
    '<button class="import-file-btn" id="importTargetsBtn">Import .txt</button>' +
    '<input type="file" class="target-list-file-input" id="targetFileInput" accept=".txt">' +
    '</div>' +
    '</div>' +
    '</div>';
```

- [ ] **Step 2: Open `index.html`, navigate to Craft > nmap. Verify:**
  - "Single | List" toggle appears next to "Target" label
  - Default is Single mode — text input visible, textarea hidden
  - Clicking "List" shows textarea and hides single input
  - Clicking "Single" restores original view

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: render dual-mode target input (single/list) in craft section"
```

---

### Task 4: Wire Target List Handlers

**Files:**
- Modify: `index.html:4504-4604` (attachCraftHandlers function)

- [ ] **Step 1: Add helper function to parse target list**

Add this function immediately before `attachCraftHandlers` (before line 4504):

```javascript
function parseTargetList(text) {
  return text.split('\n')
    .map(function(line) { return line.trim(); })
    .filter(function(line) { return line.length > 0; })
    .filter(function(line, i, arr) { return arr.indexOf(line) === i; }); // dedupe
}
```

- [ ] **Step 2: Add mode toggle, textarea, and file import handlers inside `attachCraftHandlers`**

Inside `attachCraftHandlers`, after the target input handler block (after line 4536), add:

```javascript
  // Target mode toggle
  var modeBtns = container.querySelectorAll('.target-mode-btn');
  for (var mb = 0; mb < modeBtns.length; mb++) {
    (function(btn) {
      btn.addEventListener('click', function(e) {
        e.stopPropagation();
        state.targetMode = btn.getAttribute('data-mode');
        for (var m = 0; m < modeBtns.length; m++) {
          modeBtns[m].classList.toggle('active', modeBtns[m].getAttribute('data-mode') === state.targetMode);
        }
        var singleInput = container.querySelector('#craftTarget');
        var listArea = container.querySelector('.target-list-area');
        if (state.targetMode === 'list') {
          if (singleInput) singleInput.style.display = 'none';
          if (listArea) listArea.classList.add('visible');
        } else {
          if (singleInput) singleInput.style.display = '';
          if (listArea) listArea.classList.remove('visible');
          state.batchEnabled = false;
        }
        updateOutput();
      });
    })(modeBtns[mb]);
  }

  // Target list textarea
  var listTextarea = container.querySelector('#craftTargetList');
  if (listTextarea) {
    listTextarea.addEventListener('input', function() {
      state.targetList = parseTargetList(listTextarea.value);
      var hostCount = container.querySelector('#hostCount');
      if (hostCount) hostCount.textContent = state.targetList.length + ' hosts';
      updateOutput();
    });
  }

  // Import targets file
  var importBtn = container.querySelector('#importTargetsBtn');
  var targetFileInput = container.querySelector('#targetFileInput');
  if (importBtn && targetFileInput) {
    importBtn.addEventListener('click', function() { targetFileInput.click(); });
    targetFileInput.addEventListener('change', function() {
      if (targetFileInput.files.length === 0) return;
      var reader = new FileReader();
      reader.onload = function(e) {
        var existing = listTextarea ? listTextarea.value : '';
        var newText = existing ? existing + '\n' + e.target.result : e.target.result;
        if (listTextarea) listTextarea.value = newText;
        state.targetList = parseTargetList(newText);
        var hostCount = container.querySelector('#hostCount');
        if (hostCount) hostCount.textContent = state.targetList.length + ' hosts';
        updateOutput();
      };
      reader.readAsText(targetFileInput.files[0]);
    });
  }
```

- [ ] **Step 3: Test in browser:**
  - Switch to List mode, type hosts (one per line) — host count updates
  - Duplicate lines get deduped in count
  - Empty lines ignored
  - Import .txt file appends to textarea
  - Switching back to Single mode hides list area

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: wire target list handlers — mode toggle, textarea parsing, file import"
```

---

### Task 5: Batch Command Generation

**Files:**
- Modify: `index.html:4339-4413` (buildCraftedCommand and colorCodeCommand functions)

- [ ] **Step 1: Add batch command builder function**

Add this function immediately before `buildCraftedCommand` (before line 4339):

```javascript
function buildBatchCommand(toolKey) {
  var tool = craftTools[toolKey];
  var state = getCraftState(toolKey);
  if (!tool || !state) return '';

  var hosts = state.targetList;
  if (hosts.length < 2) return buildCraftedCommand(toolKey);

  // Build base command without target
  var parts = [tool.name || toolKey];
  (tool.groups || []).forEach(function(g) {
    var selIdx = state.groups[g.name];
    if (selIdx >= 0 && selIdx < g.options.length) {
      var opt = g.options[selIdx];
      if (opt.flag) {
        var flag = opt.flag;
        if (opt.customInput && state.customInputs[opt.flag]) {
          flag += ' ' + state.customInputs[opt.flag];
        }
        parts.push(flag);
      }
    }
  });
  (tool.toggles || []).forEach(function(t) {
    if (state.toggles[t.flag]) {
      var flag = t.flag;
      if (t.customInput && state.customInputs[t.flag]) {
        flag += ' ' + state.customInputs[t.flag];
      }
      parts.push(flag);
    }
  });
  var baseCmd = parts.join(' ');

  if (hosts.length < 10) {
    return '@echo off\nfor %%H in (' + hosts.join(' ') + ') do (\n    ' + baseCmd + ' %%H\n    echo ---\n)';
  } else {
    return '@echo off\nfor /F "tokens=*" %%H in (targets.txt) do (\n    ' + baseCmd + ' %%H\n    echo ---\n)';
  }
}
```

- [ ] **Step 2: Modify `buildCraftedCommand` to handle list mode with `-iL`**

In `buildCraftedCommand` (line 4339), replace the target section at the end (lines 4370-4374):

```javascript
  if (state.target) {
    parts.push(state.target);
  }

  return parts.join(' ');
```

with:

```javascript
  if (state.targetMode === 'list' && state.targetList.length >= 2 && !state.batchEnabled) {
    // -iL mode for nmap; for curl, use first host with a note
    if ((tool.name || toolKey) === 'nmap') {
      parts.push('-iL targets.txt');
    } else {
      parts.push(state.targetList[0] || '');
    }
  } else if (state.target) {
    parts.push(state.target);
  }

  return parts.join(' ');
```

- [ ] **Step 3: Modify `updateOutput` in `attachCraftHandlers` to handle batch mode**

In `attachCraftHandlers`, replace the `updateOutput` function (line 4522-4527):

```javascript
  function updateOutput() {
    var preview = document.getElementById('craftedPreview');
    var copyBtn = document.getElementById('craftedCopyBtn');
    if (preview) preview.innerHTML = colorCodeCommand(toolKey);
    if (copyBtn) copyBtn.setAttribute('data-copy', buildCraftedCommand(toolKey));
  }
```

with:

```javascript
  function updateOutput() {
    var preview = document.getElementById('craftedPreview');
    var copyBtn = document.getElementById('craftedCopyBtn');
    if (state.targetMode === 'list' && state.batchEnabled && state.targetList.length >= 2) {
      var batchCmd = buildBatchCommand(toolKey);
      if (preview) {
        preview.classList.add('batch-mode');
        preview.textContent = batchCmd;
      }
      if (copyBtn) copyBtn.setAttribute('data-copy', batchCmd);
      // Show note for file-based loop
      var note = container.querySelector('.batch-script-note');
      if (note) {
        if (state.targetList.length >= 10) {
          note.textContent = 'Save host list as targets.txt in the same directory before running.';
          note.style.display = '';
        } else {
          note.style.display = 'none';
        }
      }
    } else {
      if (preview) {
        preview.classList.remove('batch-mode');
        preview.innerHTML = colorCodeCommand(toolKey);
      }
      if (copyBtn) copyBtn.setAttribute('data-copy', buildCraftedCommand(toolKey));
      // Show -iL note in list mode without batch
      var note = container.querySelector('.batch-script-note');
      if (note) {
        if (state.targetMode === 'list' && state.targetList.length >= 2 && !state.batchEnabled && (craftTools[toolKey].name || toolKey) === 'nmap') {
          note.textContent = 'Save host list as targets.txt and use -iL flag.';
          note.style.display = '';
        } else {
          note.style.display = 'none';
        }
      }
    }
  }
```

- [ ] **Step 4: Add batch toggle to `renderCraftSection` — in the toggles area**

In `renderCraftSection`, find the closing of the craft-groups-grid (around line 4488-4490):

```javascript
    html += '</div></div></div>';
  }
  html += '</div>'; // .craft-groups-grid
```

Replace with:

```javascript
    html += '</div></div></div>';
  }

  // Batch toggle — visible only in list mode with 2+ hosts
  var showBatch = state.targetMode === 'list' && state.targetList.length >= 2;
  html += '<div class="option-group" id="batchToggleGroup" style="' + (showBatch ? '' : 'display:none;') + '">' +
    '<div class="option-group-header">' +
    '<span class="option-group-chevron">&#9654;</span>' +
    '<span class="option-group-label">Batch</span></div>' +
    '<div class="option-group-body"><div class="toggles-grid">' +
    '<div class="toggle-item' + (state.batchEnabled ? ' active' : '') + '" data-flag="__batch__">' +
    '<span class="toggle-flag">for %%H</span>' +
    '<span class="toggle-label">Batch (CMD loop)</span>' +
    '<input type="checkbox" data-flag="__batch__"' + (state.batchEnabled ? ' checked' : '') + ' style="display:none;">' +
    '</div>' +
    '</div></div></div>';

  html += '</div>'; // .craft-groups-grid
```

- [ ] **Step 5: Add batch toggle handler in `attachCraftHandlers`**

After the toggle custom inputs handler block (after line 4603), add:

```javascript
  // Batch toggle handler
  var batchItem = container.querySelector('.toggle-item[data-flag="__batch__"]');
  if (batchItem) {
    batchItem.addEventListener('click', function() {
      state.batchEnabled = !state.batchEnabled;
      batchItem.classList.toggle('active', state.batchEnabled);
      var chk = batchItem.querySelector('input[type="checkbox"]');
      if (chk) chk.checked = state.batchEnabled;
      updateOutput();
    });
  }

  // Update batch toggle visibility when target list changes
  var origUpdateOutput = updateOutput;
  updateOutput = function() {
    var batchGroup = container.querySelector('#batchToggleGroup');
    if (batchGroup) {
      var showBatch = state.targetMode === 'list' && state.targetList.length >= 2;
      batchGroup.style.display = showBatch ? '' : 'none';
      if (!showBatch) state.batchEnabled = false;
    }
    origUpdateOutput();
  };
```

- [ ] **Step 6: Add batch script note element in `renderCraftSection`**

In `renderCraftSection`, find the crafted output block (line 4492-4497):

```javascript
  // Crafted command output
  html += '<div class="crafted-output">' +
    '<div class="crafted-output-label">Crafted Command</div>' +
    '<div class="crafted-command" id="craftedPreview">' + colorCodeCommand(toolKey) + '</div>' +
    '<button class="crafted-copy-btn" id="craftedCopyBtn" data-copy="' + escapeHtml(buildCraftedCommand(toolKey)) + '">Copy Command</button>' +
    '</div>';
```

Replace with:

```javascript
  // Crafted command output
  html += '<div class="crafted-output">' +
    '<div class="crafted-output-label">Crafted Command</div>' +
    '<div class="crafted-command" id="craftedPreview">' + colorCodeCommand(toolKey) + '</div>' +
    '<div class="batch-script-note" style="display:none;"></div>' +
    '<button class="crafted-copy-btn" id="craftedCopyBtn" data-copy="' + escapeHtml(buildCraftedCommand(toolKey)) + '">Copy Command</button>' +
    '</div>';
```

- [ ] **Step 7: Test in browser:**
  - Nmap craft > switch to List mode > enter 3 hosts > Batch toggle appears
  - Enable batch > command preview shows `@echo off\nfor %%H in (...)`
  - Enter 10+ hosts > command switches to `for /F` mode with note about targets.txt
  - Disable batch > command shows `nmap ... -iL targets.txt`
  - Curl craft > same behavior but list mode without batch shows first host
  - Switch back to Single mode > batch group hides, normal command resumes

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: batch CMD loop generation — for %%H inline and for /F file-based"
```

---

### Task 6: CSS for Matrix Comparer

**Files:**
- Modify: `index.html` CSS section

- [ ] **Step 1: Add matrix table CSS**

Add these styles after the multi-target CSS added in Task 2:

```css
/* ===== Matrix Comparer ===== */
.matrix-container {
  overflow-x: auto;
  margin-top: 12px;
}
.matrix-toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
  font-size: 0.7rem;
}
.matrix-toolbar button {
  background: transparent;
  border: 1px solid var(--border);
  color: var(--text-muted);
  padding: 2px 8px;
  cursor: pointer;
  font-family: inherit;
  font-size: 0.7rem;
}
.matrix-toolbar button.active {
  color: var(--accent-green);
  border-color: var(--accent-green);
}
.matrix-table {
  border-collapse: collapse;
  font-size: 0.75rem;
  width: 100%;
  min-width: 500px;
}
.matrix-table th,
.matrix-table td {
  border: 1px solid var(--border);
  padding: 4px 8px;
  text-align: center;
  white-space: nowrap;
}
.matrix-table th {
  background: var(--bg-input);
  color: var(--accent-cyan);
  position: sticky;
  top: 0;
  z-index: 1;
  cursor: pointer;
}
.matrix-table th:hover {
  color: var(--accent-green);
}
.matrix-table th:first-child,
.matrix-table td:first-child {
  position: sticky;
  left: 0;
  background: var(--bg-input);
  text-align: left;
  z-index: 2;
  color: var(--accent-green);
  font-weight: bold;
}
.matrix-table th:first-child {
  z-index: 3;
}
.matrix-cell-open { color: var(--accent-green); }
.matrix-cell-closed { color: #ff5555; }
.matrix-cell-filtered { color: var(--accent-yellow); }
.matrix-cell-missing { color: var(--text-muted); }
.matrix-cell-fingerprint { color: var(--accent-yellow); }
.matrix-cell-insecure { color: #ff5555; }
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "style: add CSS for matrix comparer table"
```

---

### Task 7: Multi-Select in Scan Timeline

**Files:**
- Modify: `index.html:4780-4870` (renderRescanSection, renderScanTimeline)

- [ ] **Step 1: Change the compare button area to support matrix**

In `renderRescanSection` (line 4780), find the compare and export buttons block (lines 4815-4821):

```javascript
    // Compare & export buttons
    html += '<div style="display:flex;gap:8px;margin-top:14px;align-items:center;">';
    html += '<button class="compare-btn' + (rescanSelected.length === 2 ? ' visible' : '') +
      '" id="compareBtn">Compare Selected (' + rescanSelected.length + '/2)</button>';
    if (rescanEntries.some(function(e) { return e.environment === rescanEnv; })) {
      html += '<button class="audit-btn-clear" id="rescanExportBtn">Export JSON</button>';
    }
    html += '</div>';
```

Replace with:

```javascript
    // Compare, matrix & export buttons
    html += '<div style="display:flex;gap:8px;margin-top:14px;align-items:center;flex-wrap:wrap;">';
    html += '<button class="compare-btn' + (rescanSelected.length === 2 ? ' visible' : '') +
      '" id="compareBtn">Compare Selected (' + rescanSelected.length + '/2)</button>';
    var canMatrix = rescanSelected.length >= 2 && (function() {
      var types = {};
      rescanEntries.forEach(function(e) {
        if (rescanSelected.indexOf(e.id) >= 0) types[e.scanType] = true;
      });
      return Object.keys(types).length === 1;
    })();
    html += '<button class="compare-btn' + (canMatrix ? ' visible' : '') +
      '" id="matrixBtn"' + (!canMatrix && rescanSelected.length >= 2 ? ' title="Select same scan type for matrix view"' : '') +
      '>Matrix View (' + rescanSelected.length + ')</button>';
    if (rescanEntries.some(function(e) { return e.environment === rescanEnv; })) {
      html += '<button class="audit-btn-clear" id="rescanExportBtn">Export JSON</button>';
    }
    html += '</div>';
```

- [ ] **Step 2: Allow selecting more than 2 scans in timeline**

In `attachRescanHandlers` (line 4872), find the scan entry click handler. It currently limits selection to 2. Look for the block that toggles `rescanSelected`. The handler is inside the function — search for `scan-entry` click handling.

Read the handler code, then modify it to allow unlimited selection (remove the `length >= 2` cap). The existing comparison still requires exactly 2, but matrix works with 2+.

Find the scan entry handler that has logic like:

```javascript
if (rescanSelected.length >= 2) { ... }
```

Change the cap check: instead of blocking at 2, allow unlimited selection. The compare button already shows count and only activates at exactly 2. Matrix activates at 2+.

- [ ] **Step 3: Add matrix button handler in `attachRescanHandlers`**

Add after the compare button handler, inside `attachRescanHandlers`:

```javascript
  // Matrix view button
  var matrixBtn = container.querySelector('#matrixBtn');
  if (matrixBtn) {
    matrixBtn.addEventListener('click', function() {
      if (rescanSelected.length < 2) return;
      // Check all same scan type
      var types = {};
      rescanEntries.forEach(function(e) {
        if (rescanSelected.indexOf(e.id) >= 0) types[e.scanType] = true;
      });
      if (Object.keys(types).length !== 1) return;
      rescanComparing = 'matrix';
      renderRescanSection(container);
    });
  }
```

- [ ] **Step 4: Update the comparison routing in `renderRescanSection`**

In `renderRescanSection`, find the comparison view check (line 4792):

```javascript
  if (rescanComparing && rescanSelected.length === 2) {
    html += renderComparisonView();
  } else {
```

Replace with:

```javascript
  if (rescanComparing === 'matrix' && rescanSelected.length >= 2) {
    html += renderMatrixView();
  } else if (rescanComparing && rescanSelected.length === 2) {
    html += renderComparisonView();
  } else {
```

- [ ] **Step 5: Update the existing compare button handler** to set `rescanComparing = true` explicitly (not 'matrix').

Find the compare button handler in `attachRescanHandlers` and ensure it sets:

```javascript
rescanComparing = true;
```

(It likely already does this — verify and leave as-is if correct.)

- [ ] **Step 6: Test in browser:**
  - Load 3+ scans of same type
  - Select all 3 — Matrix View button becomes visible
  - Select scans of different types — Matrix View button hidden/disabled
  - Compare button still works at exactly 2 selected
  - Matrix button shows count

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: multi-select timeline and matrix view button routing"
```

---

### Task 8: Render Matrix View — Port Scans

**Files:**
- Modify: `index.html` — add new function after `renderComparisonView` (~line 5470)

- [ ] **Step 1: Add `renderMatrixView` and `renderPortMatrix` functions**

Add after `renderComparisonView` (after line 5469):

```javascript
function renderMatrixView() {
  var selected = rescanEntries.filter(function(e) { return rescanSelected.indexOf(e.id) >= 0; });
  if (selected.length < 2) return renderEmptyState('Select at least 2 scans');

  // Determine scan type (all same, validated by button logic)
  var scanType = selected[0].scanType;

  var html = '<span class="comparison-back">&larr; Back to timeline</span>';
  html += '<div class="comparison-view visible">';
  html += '<h3 style="color:var(--accent-cyan);margin:0 0 8px;">Matrix View &mdash; ' + escapeHtml(scanType) + ' (' + selected.length + ' hosts)</h3>';

  switch (scanType) {
    case 'nmap-port': html += renderPortMatrix(selected); break;
    case 'curl-headers': html += renderHeaderMatrix(selected); break;
    default: html += renderEmptyState('Matrix view not supported for ' + escapeHtml(scanType));
  }

  html += '</div>';
  return html;
}

function renderPortMatrix(scans) {
  // Collect all port keys across all scans
  var allPorts = {};
  scans.forEach(function(scan) {
    var ports = (scan.parsed && scan.parsed.ports) || [];
    ports.forEach(function(p) {
      allPorts[p.port + '/' + p.protocol] = true;
    });
  });
  var sortedPorts = Object.keys(allPorts).sort(function(a, b) { return parseInt(a) - parseInt(b); });

  if (sortedPorts.length === 0) return renderEmptyState('No port data found in selected scans');

  // Build lookup: scan.id -> { portKey -> port object }
  var lookup = {};
  scans.forEach(function(scan) {
    lookup[scan.id] = {};
    var ports = (scan.parsed && scan.parsed.ports) || [];
    ports.forEach(function(p) {
      lookup[scan.id][p.port + '/' + p.protocol] = p;
    });
  });

  // Filter toggle and export
  var html = '<div class="matrix-toolbar">' +
    '<button class="matrix-filter-btn" id="matrixFilterDiff">Show only differences</button>' +
    '<button class="matrix-export-btn" id="matrixExportCsv">Export CSV</button>' +
    '</div>';

  html += '<div class="matrix-container"><table class="matrix-table" id="matrixTable">';

  // Header row
  html += '<thead><tr><th>Host</th>';
  sortedPorts.forEach(function(portKey) {
    html += '<th data-sort="' + escapeHtml(portKey) + '">' + escapeHtml(portKey) + '</th>';
  });
  html += '</tr></thead>';

  // Data rows
  html += '<tbody>';
  scans.forEach(function(scan) {
    html += '<tr><td>' + escapeHtml(scan.target || scan.title) + '</td>';
    sortedPorts.forEach(function(portKey) {
      var p = lookup[scan.id][portKey];
      if (p) {
        var cls = 'matrix-cell-' + (p.state || 'open');
        var label = p.state || 'open';
        if (p.service) label += ' (' + p.service + ')';
        html += '<td class="' + cls + '">' + escapeHtml(label) + '</td>';
      } else {
        html += '<td class="matrix-cell-missing">&mdash;</td>';
      }
    });
    html += '</tr>';
  });
  html += '</tbody></table></div>';

  return html;
}
```

- [ ] **Step 2: Test in browser:**
  - Load 3+ nmap-port scans with different targets
  - Select all, click Matrix View
  - Matrix table shows hosts as rows, ports as columns
  - Color coding: green=open, red=closed, yellow=filtered
  - Missing ports show dash
  - Horizontal scroll works, first column sticky

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: render port scan matrix view — hosts x ports grid"
```

---

### Task 9: Render Matrix View — Header Scans

**Files:**
- Modify: `index.html` — add `renderHeaderMatrix` after `renderPortMatrix`

- [ ] **Step 1: Add `renderHeaderMatrix` function**

Add immediately after `renderPortMatrix`:

```javascript
function renderHeaderMatrix(scans) {
  // Collect all header keys across all scans
  var allHeaders = {};
  scans.forEach(function(scan) {
    var headers = (scan.parsed && scan.parsed.headers) || {};
    Object.keys(headers).forEach(function(key) {
      allHeaders[key] = true;
    });
  });
  var sortedHeaders = Object.keys(allHeaders).sort();

  if (sortedHeaders.length === 0) return renderEmptyState('No header data found in selected scans');

  // Build lookup: scan.id -> { headerKey -> value }
  var lookup = {};
  scans.forEach(function(scan) {
    lookup[scan.id] = {};
    var headers = (scan.parsed && scan.parsed.headers) || {};
    Object.keys(headers).forEach(function(key) {
      lookup[scan.id][key] = headers[key].value || headers[key];
    });
  });

  // Fingerprint headers for highlighting
  var fpHeaders = {};
  (HEADER_RULES.fingerprint || []).forEach(function(r) {
    fpHeaders[r.header.toLowerCase()] = true;
  });

  var html = '<div class="matrix-toolbar">' +
    '<button class="matrix-filter-btn" id="matrixFilterDiff">Show only differences</button>' +
    '<button class="matrix-export-btn" id="matrixExportCsv">Export CSV</button>' +
    '</div>';

  html += '<div class="matrix-container"><table class="matrix-table" id="matrixTable">';

  // Header row
  html += '<thead><tr><th>Host</th>';
  sortedHeaders.forEach(function(hdr) {
    html += '<th data-sort="' + escapeHtml(hdr) + '">' + escapeHtml(hdr) + '</th>';
  });
  html += '</tr></thead>';

  // Data rows
  html += '<tbody>';
  scans.forEach(function(scan) {
    html += '<tr><td>' + escapeHtml(scan.target || scan.title) + '</td>';
    sortedHeaders.forEach(function(hdr) {
      var val = lookup[scan.id][hdr];
      if (val) {
        var cls = '';
        if (fpHeaders[hdr.toLowerCase()]) cls = 'matrix-cell-fingerprint';
        html += '<td class="' + cls + '">' + escapeHtml(val) + '</td>';
      } else {
        html += '<td class="matrix-cell-missing">&mdash;</td>';
      }
    });
    html += '</tr>';
  });
  html += '</tbody></table></div>';

  return html;
}
```

- [ ] **Step 2: Test in browser with curl-headers scans. Verify:**
  - Headers as columns, hosts as rows
  - Missing headers show dash
  - Fingerprint headers (Server, X-Powered-By) highlighted

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: render header scan matrix view — hosts x headers grid"
```

---

### Task 10: Matrix Interactivity — Filter, Sort, Export CSV

**Files:**
- Modify: `index.html` — add handlers in `attachRescanHandlers`

- [ ] **Step 1: Add matrix interactivity handlers**

In `attachRescanHandlers`, add after the matrix button handler (added in Task 7):

```javascript
  // Matrix filter — show only differences
  var filterBtn = container.querySelector('#matrixFilterDiff');
  if (filterBtn) {
    filterBtn.addEventListener('click', function() {
      filterBtn.classList.toggle('active');
      var table = container.querySelector('#matrixTable');
      if (!table) return;
      var rows = table.querySelectorAll('tbody tr');
      var headerCells = table.querySelectorAll('thead th');
      var colCount = headerCells.length;

      // For each column (skip first = host), check if all values same
      for (var col = 1; col < colCount; col++) {
        var values = [];
        for (var r = 0; r < rows.length; r++) {
          var cell = rows[r].cells[col];
          if (cell) values.push(cell.textContent.trim());
        }
        var allSame = values.every(function(v) { return v === values[0]; });
        var hide = filterBtn.classList.contains('active') && allSame;
        headerCells[col].style.display = hide ? 'none' : '';
        for (var r2 = 0; r2 < rows.length; r2++) {
          if (rows[r2].cells[col]) rows[r2].cells[col].style.display = hide ? 'none' : '';
        }
      }
    });
  }

  // Matrix sort by column
  var matrixTable = container.querySelector('#matrixTable');
  if (matrixTable) {
    var thCells = matrixTable.querySelectorAll('thead th');
    for (var th = 1; th < thCells.length; th++) {
      (function(colIdx) {
        thCells[colIdx].addEventListener('click', function() {
          var tbody = matrixTable.querySelector('tbody');
          var rowsArr = Array.prototype.slice.call(tbody.querySelectorAll('tr'));
          rowsArr.sort(function(a, b) {
            var aVal = a.cells[colIdx] ? a.cells[colIdx].textContent : '';
            var bVal = b.cells[colIdx] ? b.cells[colIdx].textContent : '';
            return aVal.localeCompare(bVal);
          });
          rowsArr.forEach(function(row) { tbody.appendChild(row); });
        });
      })(th);
    }
  }

  // Matrix export CSV
  var exportCsvBtn = container.querySelector('#matrixExportCsv');
  if (exportCsvBtn && matrixTable) {
    exportCsvBtn.addEventListener('click', function() {
      var rows = matrixTable.querySelectorAll('tr');
      var csvLines = [];
      for (var r = 0; r < rows.length; r++) {
        var cells = rows[r].querySelectorAll('th, td');
        var line = [];
        for (var c = 0; c < cells.length; c++) {
          var text = cells[c].textContent.replace(/"/g, '""');
          line.push('"' + text + '"');
        }
        csvLines.push(line.join(','));
      }
      var blob = new Blob([csvLines.join('\n')], { type: 'text/csv' });
      var url = URL.createObjectURL(blob);
      var a = document.createElement('a');
      a.href = url;
      a.download = new Date().toISOString().slice(0, 10) + '_matrix-export.csv';
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    });
  }
```

- [ ] **Step 2: Test in browser:**
  - Click "Show only differences" — columns with identical values across all hosts hide
  - Click again — all columns reappear
  - Click column header — rows sort by that column's values
  - Click "Export CSV" — CSV file downloads with matrix data
  - Back button returns to timeline

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: matrix interactivity — filter differences, column sort, CSV export"
```

---

### Task 11: JSON Upload Guardrails

**Files:**
- Modify: `index.html:4974-4994` (handleFilesDrop) and `index.html:6568-6591` (handleAuditFileDrop)

- [ ] **Step 1: Add toast notification helper and CSS**

Add toast CSS to the CSS section:

```css
/* ===== Toast Notifications ===== */
.toast-notification {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background: rgba(0, 255, 65, 0.15);
  color: var(--accent-green);
  border: 1px solid var(--accent-green);
  padding: 8px 16px;
  font-family: inherit;
  font-size: 0.8rem;
  z-index: 9999;
  max-width: 400px;
}
.toast-notification.toast-error {
  background: rgba(255, 85, 85, 0.15);
  color: #ff5555;
  border-color: #ff5555;
}
.toast-notification.toast-warn {
  background: rgba(230, 219, 116, 0.15);
  color: var(--accent-yellow);
  border-color: var(--accent-yellow);
}
```

Add toast and file size check functions near the utility functions section (after `escapeHtml`, around line 3960):

```javascript
function showToast(message, type) {
  var existing = document.querySelector('.toast-notification');
  if (existing) existing.remove();
  var toast = document.createElement('div');
  toast.className = 'toast-notification' + (type === 'error' ? ' toast-error' : type === 'warn' ? ' toast-warn' : '');
  toast.textContent = message;
  document.body.appendChild(toast);
  setTimeout(function() { toast.remove(); }, 4000);
}

function checkFileSize(file) {
  var sizeMB = file.size / (1024 * 1024);
  if (sizeMB > 50) {
    showToast('File too large (' + sizeMB.toFixed(1) + 'MB). Max 50MB.', 'error');
    return false;
  }
  if (sizeMB > 10) {
    showToast('Large file (' + sizeMB.toFixed(1) + 'MB). Loading may be slow.', 'warn');
  }
  return true;
}
```

- [ ] **Step 2: Update `handleFilesDrop` (Rescan History)**

Replace `handleFilesDrop` (line 4974):

```javascript
function handleFilesDrop(files, container) {
  var fileArray = Array.prototype.slice.call(files);
  fileArray.forEach(function(file) {
    if (!checkFileSize(file)) return;
    var reader = new FileReader();
    reader.onload = function(e) {
      try {
        var data = JSON.parse(e.target.result);
        if (data.entries && Array.isArray(data.entries)) {
          if (data.entries.length > 5000) {
            showToast('File contains ' + data.entries.length + ' entries. Loading may be slow.', 'warn');
          }
          data.entries.forEach(function(entry) {
            var exists = rescanEntries.some(function(ex) { return ex.id === entry.id; });
            if (!exists) rescanEntries.push(entry);
          });
        }
        renderRescanSection(container);
      } catch (err) {
        showToast('Invalid JSON format: ' + err.message, 'error');
      }
    };
    reader.readAsText(file);
  });
}
```

- [ ] **Step 3: Update `handleAuditFileDrop` (Header Audit)**

Replace `handleAuditFileDrop` (line 6568):

```javascript
function handleAuditFileDrop(files, container) {
  var fileArray = Array.prototype.slice.call(files);
  fileArray.forEach(function(file) {
    if (!checkFileSize(file)) return;
    var reader = new FileReader();
    reader.onload = function(e) {
      var content = e.target.result;
      try {
        var data = JSON.parse(content);
        if (data.auditEntries && Array.isArray(data.auditEntries)) {
          if (data.auditEntries.length > 5000) {
            showToast('File contains ' + data.auditEntries.length + ' entries. Loading may be slow.', 'warn');
          }
          data.auditEntries.forEach(function(entry) {
            var exists = auditEntries.some(function(ex) { return ex.id === entry.id; });
            if (!exists) auditEntries.push(entry);
          });
          renderAuditSection(container);
          return;
        }
      } catch(err) {
        /* not JSON, treat as header text */
        if (file.name.endsWith('.json')) {
          showToast('Invalid JSON format: ' + err.message, 'error');
          return;
        }
      }
      var textarea = container.querySelector('#auditInput');
      if (textarea) {
        textarea.value = content;
      }
    };
    reader.readAsText(file);
  });
}
```

- [ ] **Step 4: Test in browser:**
  - Drop a valid JSON file — loads normally
  - Drop an invalid/corrupt JSON file — red toast shows "Invalid JSON format"
  - Verify toast appears bottom-right and auto-dismisses after 4 seconds

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: JSON upload guardrails — size limits, parse errors, entry count warnings"
```

---

### Task 12: Final Integration Test

**Files:**
- Test: `index.html` in browser

- [ ] **Step 1: Test complete multi-target -> batch -> scan -> matrix workflow**

1. Open `index.html` in browser
2. Navigate to Craft > nmap
3. Switch target to List mode
4. Enter 5 hosts, one per line
5. Verify host count shows "5 hosts"
6. Enable Batch toggle — verify CMD loop output
7. Copy command — verify clipboard contains full batch script
8. Switch to 12 hosts — verify it changes to `for /F` mode with targets.txt note
9. Switch back to Single mode — verify normal command, batch hidden

- [ ] **Step 2: Test matrix comparer**

1. Navigate to Rescan History
2. Load 3+ nmap-port scan results (paste or import JSON)
3. Select 3 scans of same type
4. Click Matrix View — verify hosts x ports grid
5. Click "Show only differences" — identical columns hide
6. Click a column header — rows sort
7. Click Export CSV — file downloads
8. Click Back — returns to timeline

- [ ] **Step 3: Test upload guardrails**

1. Create a malformed JSON file, drop it on Rescan History drop zone
2. Verify red error toast appears
3. Verify toast auto-dismisses
4. Drop valid JSON — loads normally

- [ ] **Step 4: Test curl craft with list mode**

1. Navigate to Craft > curl
2. Switch to List mode, enter 3 URLs
3. Enable batch — verify CMD loop wraps curl command
4. Verify all curl flags appear correctly in batch script

- [ ] **Step 5: Test mobile responsive**

1. Open DevTools, toggle responsive mode
2. Verify target list textarea responsive
3. Verify matrix table scrolls horizontally
4. Verify toast doesn't overflow viewport

- [ ] **Step 6: Commit final state if any fixups needed**

```bash
git add index.html
git commit -m "fix: integration test fixups for multi-target and matrix features"
```
