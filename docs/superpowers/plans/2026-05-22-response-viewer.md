# Response Viewer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a standalone "Response Viewer" section to ScanRef that beautifies raw curl output — JSON, HTTP headers, HTML, and XML — pasted by the user.

**Architecture:** All code lives in `index.html` (single-file offline app). Pure functions detect the input type and produce an array of `{text, cls}` segments; a DOM renderer turns segments into `<span>`/text nodes inside a `<pre>`. The section, routing, nav, and keyboard shortcut follow the existing Header Audit patterns.

**Tech Stack:** Vanilla browser JS, native `JSON.parse`/`DOMParser`, no dependencies. Verification via the Playwright MCP browser tools against the `file://` URL.

---

## Conventions for this plan

- **No test runner exists.** ScanRef is a single HTML file. "Tests" = Playwright MCP: navigate to `file:///F:/Claude/RescanRef/index.html` (this reloads the file and any edits), then `browser_evaluate` a function and compare to the expected result. A function that does not yet exist throws `ReferenceError` — that is the RED state.
- **Build all new DOM with `document.createElement` / `textContent` / `appendChild`.** Do not assign HTML strings to elements: a repo security hook blocks edits containing that DOM HTML-injection property, and string injection would also weaken the app's XSS model. Existing HTML-string assignments elsewhere in the file stay untouched.
- All new JS goes in one block inserted immediately before `function renderAuditSection(container) {` (around line 7730).
- All new CSS goes in one block inserted immediately before the `/* ===== Header Audit ===== */` marker (around line 1368).
- Commit after every task.

---

## File Structure

Single file modified: `index.html`. New code, grouped:

- **CSS block** — `.viewer-*` layout + token color classes. Before `/* ===== Header Audit ===== */`.
- **JS block** — inserted before `renderAuditSection`. Contains, in order: `VIEWER_VOID_ELEMENTS` / `VIEWER_PRESERVE_CONTENT` constants, `currentViewerInput` state var, `vEl`, `detectResponseType`, `beautifyJson`, `tokenizeJson`, `parseHeaderBlock`, `headerSegments`, `serializeNode`, `beautifyMarkup`, `beautifyBodySegments`, `buildViewerOutput`, `renderSegments`, `renderViewerSection`, `attachViewerHandlers`.
- **Integration edits** — `renderMainContent` switch, `parseHash`, `getSectionLabel`, `buildSidebar`, keyboard `navMap`, help-overlay table.

---

## Task 1: CSS + DOM helper + state

**Files:**
- Modify: `index.html` — CSS before line ~1368; JS before line ~7730.

- [ ] **Step 1: Add the CSS block**

Edit `index.html`. Find the line:

```
/* ===== Header Audit ===== */
```

Replace it with (the new block, then the original marker):

```
/* ===== Response Viewer ===== */
.viewer-container { max-width: 1100px; }
.viewer-textarea {
  width: 100%;
  min-height: 120px;
  background: var(--input-bg);
  border: 1px solid var(--input-border);
  color: var(--text-primary);
  font-family: inherit;
  font-size: 0.8rem;
  line-height: 1.5;
  padding: 10px 12px;
  resize: none;
  outline: none;
  box-sizing: border-box;
}
.viewer-textarea:focus { border-color: var(--accent-green); }
.viewer-textarea::placeholder { color: var(--placeholder); }
.viewer-bar {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 12px 0 8px;
  min-height: 24px;
}
.viewer-badge {
  font-size: 0.62rem;
  text-transform: uppercase;
  letter-spacing: 1px;
  color: var(--accent-cyan);
  border: 1px solid var(--card-border);
  padding: 2px 8px;
}
.viewer-badge:empty { display: none; }
.viewer-output {
  background: var(--code-bg);
  border: 1px solid var(--card-border);
  color: var(--text-primary);
  font-family: inherit;
  font-size: 0.78rem;
  line-height: 1.55;
  padding: 12px;
  margin: 0;
  white-space: pre-wrap;
  word-break: break-word;
  overflow-x: auto;
}
.viewer-output:empty { display: none; }
.jv-key { color: var(--accent-cyan); }
.jv-str { color: var(--accent-yellow); }
.jv-num { color: var(--accent-purple); }
.jv-kw  { color: var(--accent-pink); }
.hv-status { color: var(--accent-green); font-weight: 700; }
.hv-name { color: var(--accent-cyan); }
.hv-sep  { color: var(--text-muted); }
.hv-val  { color: var(--text-primary); }
.viewer-err { color: var(--accent-pink); }

/* ===== Header Audit ===== */
```

- [ ] **Step 2: Add the DOM helper, constants, and state var**

Edit `index.html`. Find the line:

```
function renderAuditSection(container) {
```

Insert this block immediately BEFORE it:

```
/* ================================================================
   RESPONSE VIEWER
   ================================================================ */

// Persists the pasted input across section re-renders (navigation away/back).
var currentViewerInput = '';

// HTML elements that are void (no closing tag) / that keep raw text content.
var VIEWER_VOID_ELEMENTS = { area:1, base:1, br:1, col:1, embed:1, hr:1, img:1,
  input:1, link:1, meta:1, param:1, source:1, track:1, wbr:1 };
var VIEWER_PRESERVE_CONTENT = { script:1, style:1, pre:1, textarea:1 };

// Minimal DOM builder. opts: {cls, id, text, attrs}. kids: array of node|string.
function vEl(tag, opts, kids) {
  var e = document.createElement(tag);
  if (opts) {
    if (opts.cls) e.className = opts.cls;
    if (opts.id) e.id = opts.id;
    if (opts.text != null) e.textContent = opts.text;
    if (opts.attrs) {
      for (var k in opts.attrs) {
        if (opts.attrs.hasOwnProperty(k)) e.setAttribute(k, opts.attrs[k]);
      }
    }
  }
  if (kids) {
    for (var i = 0; i < kids.length; i++) {
      var c = kids[i];
      e.appendChild(typeof c === 'string' ? document.createTextNode(c) : c);
    }
  }
  return e;
}

```

- [ ] **Step 3: Verify the helper loads**

Run: navigate Playwright to `file:///F:/Claude/RescanRef/index.html`, then `browser_evaluate`:

```js
() => {
  const e = vEl('span', { cls: 'x', text: 'hi' }, []);
  return { tag: e.tagName, cls: e.className, text: e.textContent, viewerInput: typeof currentViewerInput };
}
```

Expected: `{ "tag": "SPAN", "cls": "x", "text": "hi", "viewerInput": "string" }`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: Response Viewer CSS, DOM helper, state scaffold"
```

---

## Task 2: Type detection

**Files:**
- Modify: `index.html` — append to the JS block from Task 1 (after `vEl`).

- [ ] **Step 1: Add `detectResponseType`**

Edit `index.html`. Find the closing of `vEl` followed by a blank line:

```
  return e;
}

```

Replace it with (keep `vEl` intact, add the new function after it):

```
  return e;
}

// Sniffs raw text and returns: 'empty' | 'http' | 'json' | 'html' | 'xml' | 'text'.
function detectResponseType(raw) {
  var t = String(raw == null ? '' : raw).replace(/^﻿/, '').replace(/^\s+/, '');
  if (t === '') return 'empty';
  if (/^HTTP\/\d/i.test(t)) return 'http';
  var c = t.charAt(0);
  if (c === '{' || c === '[') return 'json';
  if (c === '<') {
    if (/^<\?xml/i.test(t)) return 'xml';
    if (/<!doctype\s+html/i.test(t) || /<html[\s>]/i.test(t)) return 'html';
    return 'xml';
  }
  return 'text';
}

```

- [ ] **Step 2: Verify (RED then GREEN — function did not exist before this task)**

Run: navigate Playwright to `file:///F:/Claude/RescanRef/index.html`, then `browser_evaluate`:

```js
() => ({
  json_obj:  detectResponseType('{"a":1}'),
  json_arr:  detectResponseType('  [1,2]'),
  http:      detectResponseType('HTTP/2 200 OK\nServer: x'),
  html_doc:  detectResponseType('<!DOCTYPE html><html><body>x</body></html>'),
  html_tag:  detectResponseType('<html><body>x</body></html>'),
  xml_decl:  detectResponseType('<?xml version="1.0"?><r/>'),
  xml_bare:  detectResponseType('<root><a/></root>'),
  text:      detectResponseType('   just words'),
  empty:     detectResponseType('   \n  ')
})
```

Expected:

```json
{
  "json_obj": "json", "json_arr": "json", "http": "http",
  "html_doc": "html", "html_tag": "html", "xml_decl": "xml",
  "xml_bare": "xml", "text": "text", "empty": "empty"
}
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: Response Viewer type detection"
```

---

## Task 3: JSON beautify + tokenize

**Files:**
- Modify: `index.html` — append after `detectResponseType`.

- [ ] **Step 1: Add `beautifyJson` and `tokenizeJson`**

Edit `index.html`. Find the closing of `detectResponseType`:

```
  return 'text';
}

```

Replace it with (keep `detectResponseType` intact, add the two functions after it):

```
  return 'text';
}

// Pretty-prints JSON. Returns {ok, text, error}; on failure text is the raw input.
function beautifyJson(raw) {
  try {
    return { ok: true, text: JSON.stringify(JSON.parse(raw), null, 2), error: '' };
  } catch (e) {
    return { ok: false, text: String(raw), error: 'Invalid JSON: ' + e.message };
  }
}

// Splits well-formed (JSON.stringify) output into [{text, cls}] highlight segments.
function tokenizeJson(jsonText) {
  var re = /("(?:\\.|[^"\\])*")(\s*:)?|\b(true|false|null)\b|(-?\d+(?:\.\d+)?(?:[eE][+-]?\d+)?)/g;
  var segs = [];
  var last = 0;
  Array.from(jsonText.matchAll(re)).forEach(function(m) {
    if (m.index > last) segs.push({ text: jsonText.slice(last, m.index), cls: null });
    if (m[1] != null) {
      segs.push({ text: m[1], cls: m[2] ? 'jv-key' : 'jv-str' });
      if (m[2]) segs.push({ text: m[2], cls: null });
    } else if (m[3] != null) {
      segs.push({ text: m[3], cls: 'jv-kw' });
    } else if (m[4] != null) {
      segs.push({ text: m[4], cls: 'jv-num' });
    }
    last = m.index + m[0].length;
  });
  if (last < jsonText.length) segs.push({ text: jsonText.slice(last), cls: null });
  return segs;
}

```

- [ ] **Step 2: Verify**

Run: navigate Playwright to `file:///F:/Claude/RescanRef/index.html`, then `browser_evaluate`:

```js
() => {
  const good = beautifyJson('{"a":1,"b":[2,3]}');
  const bad  = beautifyJson('{not json}');
  const segs = tokenizeJson('{\n  "a": true,\n  "n": 5\n}');
  const classes = {};
  segs.forEach(s => { if (s.cls) classes[s.text] = s.cls; });
  return {
    good_ok: good.ok,
    good_text: good.text,
    bad_ok: bad.ok,
    bad_has_error: bad.error.indexOf('Invalid JSON') === 0,
    key_class: classes['"a"'],
    kw_class: classes['true'],
    num_class: classes['5']
  };
}
```

Expected:

```json
{
  "good_ok": true,
  "good_text": "{\n  \"a\": 1,\n  \"b\": [\n    2,\n    3\n  ]\n}",
  "bad_ok": false,
  "bad_has_error": true,
  "key_class": "jv-key",
  "kw_class": "jv-kw",
  "num_class": "jv-num"
}
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: Response Viewer JSON beautify and highlight"
```

---

## Task 4: HTTP header parsing

**Files:**
- Modify: `index.html` — append after `tokenizeJson`.

- [ ] **Step 1: Add `parseHeaderBlock` and `headerSegments`**

Edit `index.html`. Find the closing of `tokenizeJson`:

```
  if (last < jsonText.length) segs.push({ text: jsonText.slice(last), cls: null });
  return segs;
}

```

Replace it with (keep `tokenizeJson` intact, add the two functions after it):

```
  if (last < jsonText.length) segs.push({ text: jsonText.slice(last), cls: null });
  return segs;
}

// Parses an HTTP response: {statusLine, headers:[{name,value}], body}.
// The header block ends at the first blank line; everything after is body.
function parseHeaderBlock(raw) {
  var lines = String(raw).replace(/\r\n/g, '\n').split('\n');
  var statusLine = '';
  var headers = [];
  var i = 0;
  if (/^HTTP\/\d/i.test(lines[0] || '')) {
    statusLine = lines[0].trim();
    i = 1;
  }
  for (; i < lines.length; i++) {
    if (lines[i].trim() === '') { i++; break; }
    var idx = lines[i].indexOf(':');
    if (idx > 0) {
      headers.push({ name: lines[i].slice(0, idx).trim(), value: lines[i].slice(idx + 1).trim() });
    } else {
      headers.push({ name: '', value: lines[i].trim() });
    }
  }
  return { statusLine: statusLine, headers: headers, body: lines.slice(i).join('\n') };
}

// Turns a parsed header block into [{text, cls}] segments with aligned names.
function headerSegments(parsed) {
  var segs = [];
  if (parsed.statusLine) {
    segs.push({ text: parsed.statusLine, cls: 'hv-status' });
    segs.push({ text: '\n', cls: null });
  }
  var maxLen = 0;
  parsed.headers.forEach(function(h) { if (h.name.length > maxLen) maxLen = h.name.length; });
  parsed.headers.forEach(function(h) {
    if (h.name) {
      var pad = new Array(Math.max(0, maxLen - h.name.length) + 1).join(' ');
      segs.push({ text: h.name, cls: 'hv-name' });
      segs.push({ text: pad + ': ', cls: 'hv-sep' });
      segs.push({ text: h.value, cls: 'hv-val' });
    } else {
      segs.push({ text: h.value, cls: null });
    }
    segs.push({ text: '\n', cls: null });
  });
  return segs;
}

```

- [ ] **Step 2: Verify**

Run: navigate Playwright to `file:///F:/Claude/RescanRef/index.html`, then `browser_evaluate`:

```js
() => {
  const p = parseHeaderBlock('HTTP/2 200 OK\nContent-Type: text/html\nServer: nginx\n\n<html></html>');
  const segs = headerSegments(p);
  const onlyHeaders = parseHeaderBlock('HTTP/1.1 404 Not Found\nDate: today');
  return {
    status: p.statusLine,
    header_count: p.headers.length,
    first_header: p.headers[0],
    body: p.body,
    has_status_seg: segs.some(s => s.cls === 'hv-status'),
    has_name_seg: segs.some(s => s.cls === 'hv-name'),
    no_body_body: onlyHeaders.body
  };
}
```

Expected:

```json
{
  "status": "HTTP/2 200 OK",
  "header_count": 2,
  "first_header": { "name": "Content-Type", "value": "text/html" },
  "body": "<html></html>",
  "has_status_seg": true,
  "has_name_seg": true,
  "no_body_body": ""
}
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: Response Viewer HTTP header parsing"
```

---

## Task 5: HTML/XML beautify

**Files:**
- Modify: `index.html` — append after `headerSegments`.

- [ ] **Step 1: Add `serializeNode` and `beautifyMarkup`**

Edit `index.html`. Find the closing of `headerSegments`:

```
    segs.push({ text: '\n', cls: null });
  });
  return segs;
}

```

Replace it with (keep `headerSegments` intact, add the two functions after it):

```
    segs.push({ text: '\n', cls: null });
  });
  return segs;
}

// Recursively serializes a DOM node to an indented markup string (2-space indent).
function serializeNode(node, depth) {
  var pad = new Array(depth + 1).join('  ');
  var nt = node.nodeType;
  if (nt === 1) { // element
    var tag = node.nodeName.toLowerCase();
    var attrs = '';
    for (var a = 0; a < node.attributes.length; a++) {
      attrs += ' ' + node.attributes[a].name + '="' + node.attributes[a].value + '"';
    }
    var open = '<' + tag + attrs + '>';
    var close = '</' + tag + '>';
    if (VIEWER_VOID_ELEMENTS[tag]) return pad + '<' + tag + attrs + ' />';
    if (VIEWER_PRESERVE_CONTENT[tag]) {
      var rawText = node.textContent;
      if (rawText === '') return pad + open + close;
      if (rawText.indexOf('\n') === -1) return pad + open + rawText + close;
      return pad + open + '\n' + rawText + '\n' + pad + close;
    }
    if (node.childNodes.length === 1 && node.childNodes[0].nodeType === 3) {
      var inlineTxt = node.childNodes[0].nodeValue.replace(/\s+/g, ' ').trim();
      return pad + open + inlineTxt + close;
    }
    var childOut = [];
    for (var c = 0; c < node.childNodes.length; c++) {
      var s = serializeNode(node.childNodes[c], depth + 1);
      if (s !== null && s !== '') childOut.push(s);
    }
    if (childOut.length === 0) return pad + open + close;
    return pad + open + '\n' + childOut.join('\n') + '\n' + pad + close;
  }
  if (nt === 3) { // text
    var txt = node.nodeValue.replace(/\s+/g, ' ').trim();
    return txt === '' ? null : pad + txt;
  }
  if (nt === 8) return pad + '<!--' + node.nodeValue + '-->'; // comment
  if (nt === 4) return pad + '<![CDATA[' + node.nodeValue + ']]>'; // CDATA
  return null;
}

// Beautifies HTML/XML. Returns {ok, text, error}; tries the other parser on failure.
function beautifyMarkup(raw, isXml) {
  function tryParse(mime) {
    var doc = new DOMParser().parseFromString(raw, mime);
    return doc.querySelector('parsererror') ? null : doc;
  }
  var doc = isXml ? tryParse('application/xml') : tryParse('text/html');
  if (!doc) doc = isXml ? tryParse('text/html') : tryParse('application/xml');
  if (!doc || !doc.documentElement) {
    return { ok: false, text: String(raw), error: 'Could not parse markup' };
  }
  return { ok: true, text: serializeNode(doc.documentElement, 0), error: '' };
}

```

- [ ] **Step 2: Verify**

Run: navigate Playwright to `file:///F:/Claude/RescanRef/index.html`, then `browser_evaluate`:

```js
() => {
  const xml  = beautifyMarkup('<root><a>x</a><b/></root>', true);
  const html = beautifyMarkup('<html><body><div><p>hi</p></div></body></html>', false);
  return {
    xml_ok: xml.ok,
    xml_text: xml.text,
    html_ok: html.ok,
    html_has_indent: /\n {4}<p>hi<\/p>/.test(html.text)
  };
}
```

Expected:

```json
{
  "xml_ok": true,
  "xml_text": "<root>\n  <a>x</a>\n  <b></b>\n</root>",
  "html_ok": true,
  "html_has_indent": true
}
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: Response Viewer HTML/XML beautify"
```

---

## Task 6: Orchestrator + segment renderer

**Files:**
- Modify: `index.html` — append after `beautifyMarkup`.

- [ ] **Step 1: Add `beautifyBodySegments`, `buildViewerOutput`, `renderSegments`**

Edit `index.html`. Find the closing of `beautifyMarkup`:

```
  return { ok: true, text: serializeNode(doc.documentElement, 0), error: '' };
}

```

Replace it with (keep `beautifyMarkup` intact, add the three functions after it):

```
  return { ok: true, text: serializeNode(doc.documentElement, 0), error: '' };
}

// Beautifies a non-HTTP body of a known type. Returns {label, segments}.
function beautifyBodySegments(raw, type) {
  if (type === 'json') {
    var j = beautifyJson(raw);
    if (j.ok) return { label: 'JSON', segments: tokenizeJson(j.text) };
    return { label: 'JSON (invalid)', segments: [
      { text: j.error + '\n\n', cls: 'viewer-err' },
      { text: String(raw), cls: null }
    ] };
  }
  if (type === 'html' || type === 'xml') {
    var m = beautifyMarkup(raw, type === 'xml');
    if (m.ok) return { label: type.toUpperCase(), segments: [{ text: m.text, cls: null }] };
    return { label: type.toUpperCase() + ' (unparsed)', segments: [
      { text: m.error + '\n\n', cls: 'viewer-err' },
      { text: String(raw), cls: null }
    ] };
  }
  return { label: 'Plain text', segments: [{ text: String(raw), cls: null }] };
}

// Top-level: detects type, beautifies, returns {badge, segments}.
function buildViewerOutput(raw) {
  var type = detectResponseType(raw);
  if (type === 'empty') return { badge: '', segments: [] };
  if (type === 'http') {
    var parsed = parseHeaderBlock(raw);
    var segs = headerSegments(parsed);
    if (parsed.body && parsed.body.trim() !== '') {
      var bodyType = detectResponseType(parsed.body);
      var body = bodyType === 'http'
        ? { label: 'text', segments: [{ text: parsed.body, cls: null }] }
        : beautifyBodySegments(parsed.body, bodyType);
      segs.push({ text: '\n', cls: null });
      return { badge: 'HTTP response + ' + body.label, segments: segs.concat(body.segments) };
    }
    return { badge: 'HTTP headers', segments: segs };
  }
  var out = beautifyBodySegments(raw, type);
  return { badge: out.label, segments: out.segments };
}

// Renders [{text, cls}] segments into a <pre>, clearing it first.
function renderSegments(preEl, segments) {
  while (preEl.firstChild) preEl.removeChild(preEl.firstChild);
  segments.forEach(function(s) {
    if (s.cls) {
      preEl.appendChild(vEl('span', { cls: s.cls, text: s.text }));
    } else {
      preEl.appendChild(document.createTextNode(s.text));
    }
  });
}

```

- [ ] **Step 2: Verify**

Run: navigate Playwright to `file:///F:/Claude/RescanRef/index.html`, then `browser_evaluate`:

```js
() => {
  const json = buildViewerOutput('{"x":1}');
  const http = buildViewerOutput('HTTP/2 200 OK\nServer: nginx\n\n{"ok":true}');
  const bad  = buildViewerOutput('{broken');
  const pre = document.createElement('pre');
  renderSegments(pre, json.segments);
  return {
    json_badge: json.badge,
    http_badge: http.badge,
    bad_badge: bad.badge,
    bad_has_err: bad.segments.some(s => s.cls === 'viewer-err'),
    rendered_text: pre.textContent,
    rendered_has_span: pre.querySelector('span.jv-num') !== null
  };
}
```

Expected:

```json
{
  "json_badge": "JSON",
  "http_badge": "HTTP response + JSON",
  "bad_badge": "JSON (invalid)",
  "bad_has_err": true,
  "rendered_text": "{\n  \"x\": 1\n}",
  "rendered_has_span": true
}
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: Response Viewer output orchestrator and renderer"
```

---

## Task 7: Section render + handlers + routing

**Files:**
- Modify: `index.html` — append after `renderSegments`; edit `renderMainContent`, `parseHash`, `getSectionLabel`.

- [ ] **Step 1: Add `renderViewerSection` and `attachViewerHandlers`**

Edit `index.html`. Find the closing of `renderSegments`:

```
      preEl.appendChild(document.createTextNode(s.text));
    }
  });
}

```

Replace it with (keep `renderSegments` intact, add the two functions after it):

```
      preEl.appendChild(document.createTextNode(s.text));
    }
  });
}

// Renders the Response Viewer section into `container`. Built with DOM APIs
// only, so pasted markup is never live-injected into the page.
function renderViewerSection(container) {
  var header = vEl('div', { cls: 'section-header' }, [
    vEl('div', { cls: 'section-prompt' }, [
      '# ',
      vEl('span', { cls: 'cmd', text: 'scanref' }),
      ' ',
      vEl('span', { cls: 'flag', text: '--viewer' })
    ]),
    vEl('h2', { text: 'Response Viewer' }),
    vEl('p', { text: 'Paste raw curl output to beautify JSON, headers, HTML, or XML' })
  ]);

  var textarea = vEl('textarea', {
    cls: 'viewer-textarea',
    id: 'viewerInput',
    text: currentViewerInput,
    attrs: {
      'data-autosize': '',
      spellcheck: 'false',
      placeholder: 'Paste raw curl output here...\n\nJSON, HTTP headers, HTML, and XML are auto-detected.'
    }
  });

  var badge = vEl('span', { cls: 'viewer-badge', id: 'viewerBadge' });
  var copyBtn = vEl('button', { cls: 'crafted-copy-btn', id: 'viewerCopyBtn', text: 'Copy Output' });
  copyBtn.style.display = 'none';
  var bar = vEl('div', { cls: 'viewer-bar' }, [badge, copyBtn]);

  var output = vEl('pre', { cls: 'viewer-output', id: 'viewerOutput' });

  var wrap = vEl('div', { cls: 'viewer-container' }, [header, textarea, bar, output]);

  while (container.firstChild) container.removeChild(container.firstChild);
  container.appendChild(wrap);

  attachViewerHandlers(container);
}

function attachViewerHandlers(container) {
  var input = container.querySelector('#viewerInput');
  var output = container.querySelector('#viewerOutput');
  var badge = container.querySelector('#viewerBadge');
  var copyBtn = container.querySelector('#viewerCopyBtn');
  if (!input || !output || !badge || !copyBtn) return;

  function update() {
    currentViewerInput = input.value;
    var result = buildViewerOutput(input.value);
    renderSegments(output, result.segments);
    badge.textContent = result.badge;
    if (result.segments.length > 0) {
      copyBtn.style.display = '';
      copyBtn.setAttribute('data-copy', output.textContent);
    } else {
      copyBtn.style.display = 'none';
      copyBtn.removeAttribute('data-copy');
    }
  }

  input.addEventListener('input', update);
  copyBtn.addEventListener('click', function() {
    copyToClipboard(copyBtn.getAttribute('data-copy') || '', copyBtn);
  });

  update();
  autoSizeTextarea(input);
}

```

- [ ] **Step 2: Wire `renderMainContent`**

Edit `index.html`. Find:

```
    case 'audit':
      renderAuditSection(main);
      break;
    default:
```

Replace with:

```
    case 'audit':
      renderAuditSection(main);
      break;
    case 'viewer':
      renderViewerSection(main);
      break;
    default:
```

- [ ] **Step 3: Wire `parseHash`**

Edit `index.html`. Find:

```
  } else if (type === 'audit') {
    valid = true; key = key || 'audit';
  }
```

Replace with:

```
  } else if (type === 'audit') {
    valid = true; key = key || 'audit';
  } else if (type === 'viewer') {
    valid = true; key = key || 'viewer';
  }
```

- [ ] **Step 4: Wire `getSectionLabel`**

Edit `index.html`. Find:

```
    case 'audit':     return 'ScanRef · Header Audit';
```

Replace with:

```
    case 'audit':     return 'ScanRef · Header Audit';
    case 'viewer':    return 'ScanRef · Response Viewer';
```

- [ ] **Step 5: Verify routing**

Run: navigate Playwright to `file:///F:/Claude/RescanRef/index.html#viewer`, then `browser_evaluate`:

```js
() => {
  const sec = document.querySelector('.viewer-container');
  const ta = document.getElementById('viewerInput');
  ta.value = '{"hello":"world","n":42}';
  ta.dispatchEvent(new Event('input', { bubbles: true }));
  const out = document.getElementById('viewerOutput');
  const badge = document.getElementById('viewerBadge');
  return {
    section_present: sec !== null,
    title: document.title,
    badge: badge.textContent,
    output_text: out.textContent,
    has_key_span: out.querySelector('span.jv-key') !== null,
    copy_visible: document.getElementById('viewerCopyBtn').style.display !== 'none'
  };
}
```

Expected:

```json
{
  "section_present": true,
  "title": "ScanRef · Response Viewer",
  "badge": "JSON",
  "output_text": "{\n  \"hello\": \"world\",\n  \"n\": 42\n}",
  "has_key_span": true,
  "copy_visible": true
}
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: Response Viewer section render, handlers, routing"
```

---

## Task 8: Sidebar nav + keyboard shortcut + help overlay

**Files:**
- Modify: `index.html` — `buildSidebar`, keyboard `navMap`, help-overlay table.

- [ ] **Step 1: Add the sidebar nav item**

Edit `index.html`. Find:

```
  // Header Audit
  html += '<div class="nav-group-label">Audit</div>';
  html += '<div class="nav-item" data-type="audit" data-key="audit">' +
    '<span class="nav-item-label">Header Audit</span></div>';
```

Replace with:

```
  // Header Audit + Response Viewer
  html += '<div class="nav-group-label">Audit</div>';
  html += '<div class="nav-item" data-type="audit" data-key="audit">' +
    '<span class="nav-item-label">Header Audit</span></div>';
  html += '<div class="nav-item" data-type="viewer" data-key="viewer">' +
    '<span class="nav-item-label">Response Viewer</span></div>';
```

- [ ] **Step 2: Add the keyboard shortcut**

Edit `index.html`. Find:

```
    r: function() { navigateTo({ type: 'rescan', key: 'rescan' }); },
    h: function() { navigateTo({ type: 'audit', key: 'audit' }); }
  };
```

Replace with:

```
    r: function() { navigateTo({ type: 'rescan', key: 'rescan' }); },
    h: function() { navigateTo({ type: 'audit', key: 'audit' }); },
    v: function() { navigateTo({ type: 'viewer', key: 'viewer' }); }
  };
```

- [ ] **Step 3: Add the help-overlay row**

Edit `index.html`. Find:

```
      <tr><td><kbd>h</kbd></td><td>Header audit</td></tr>
```

Replace with:

```
      <tr><td><kbd>h</kbd></td><td>Header audit</td></tr>
      <tr><td><kbd>v</kbd></td><td>Response viewer</td></tr>
```

- [ ] **Step 4: Verify nav + shortcut**

Run: navigate Playwright to `file:///F:/Claude/RescanRef/index.html`, then `browser_evaluate`:

```js
() => {
  const navItem = document.querySelector('.nav-item[data-type="viewer"]');
  document.dispatchEvent(new KeyboardEvent('keydown', { key: 'v', bubbles: true }));
  const helpHasV = !!Array.from(document.querySelectorAll('#helpOverlay kbd'))
    .find(k => k.textContent === 'v');
  return {
    nav_item_present: navItem !== null,
    nav_label: navItem ? navItem.textContent.trim() : null,
    section_after_v: !!document.querySelector('.viewer-container'),
    hash_after_v: location.hash,
    help_has_v: helpHasV
  };
}
```

Expected:

```json
{
  "nav_item_present": true,
  "nav_label": "Response Viewer",
  "section_after_v": true,
  "hash_after_v": "#viewer/viewer",
  "help_has_v": true
}
```

- [ ] **Step 5: Final integration test — all four input types**

Run: navigate Playwright to `file:///F:/Claude/RescanRef/index.html#viewer`, then `browser_evaluate`:

```js
() => {
  const ta = document.getElementById('viewerInput');
  const out = document.getElementById('viewerOutput');
  const badge = document.getElementById('viewerBadge');
  function feed(v) {
    ta.value = v;
    ta.dispatchEvent(new Event('input', { bubbles: true }));
    return { badge: badge.textContent, text: out.textContent };
  }
  const json = feed('{"a":[1,2]}');
  const headers = feed('HTTP/2 200 OK\nContent-Type: text/html\nServer: nginx');
  const xml = feed('<root><child>v</child></root>');
  const html = feed('<html><body><h1>Hi</h1></body></html>');
  const bad = feed('{not valid');
  const empty = feed('   ');
  return {
    json_badge: json.badge,
    json_indented: json.text.indexOf('\n  ') !== -1,
    headers_badge: headers.badge,
    headers_aligned: headers.text.indexOf('HTTP/2 200 OK') === 0,
    xml_badge: xml.badge,
    xml_indented: xml.text.indexOf('<root>\n  <child>') === 0,
    html_badge: html.badge,
    bad_badge: bad.badge,
    empty_badge: empty.badge,
    empty_output: out.textContent
  };
}
```

Expected:

```json
{
  "json_badge": "JSON",
  "json_indented": true,
  "headers_badge": "HTTP headers",
  "headers_aligned": true,
  "xml_badge": "XML",
  "xml_indented": true,
  "html_badge": "HTML",
  "bad_badge": "JSON (invalid)",
  "empty_badge": "",
  "empty_output": ""
}
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: Response Viewer nav item, v shortcut, help entry"
```

---

## Task 9: Update project docs

**Files:**
- Modify: `CLAUDE.md` — add the Response Viewer to the "Current state" notes.

- [ ] **Step 1: Note the feature in `CLAUDE.md`**

Edit `CLAUDE.md`. Find:

```
- Architecture codemaps in `docs/CODEMAPS/` (architecture, frontend, backend, data, dependencies)
```

Replace with:

```
- Response Viewer: standalone `#viewer` section — paste raw curl output, auto-detects and beautifies JSON / HTTP headers / HTML / XML (native `JSON.parse` + `DOMParser`, no deps)
- Architecture codemaps in `docs/CODEMAPS/` (architecture, frontend, backend, data, dependencies)
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: note Response Viewer in CLAUDE.md"
```

---

## Self-Review

**Spec coverage:**
- Beautify JSON / headers / HTML / XML — Tasks 3, 4, 5; orchestrated in Task 6.
- Offline, no dependencies — only `JSON.parse`, `DOMParser`, DOM APIs used.
- Terminal aesthetic — CSS in Task 1 uses existing `--accent-*` / `--code-bg` vars, no radius/shadow.
- Never blank on bad input — `beautifyJson`/`beautifyMarkup` return `ok:false` with the raw text; `beautifyBodySegments` emits a `viewer-err` segment plus raw (Tasks 3, 5, 6).
- Standalone `#viewer` section + nav item — Tasks 7, 8.
- `v` keyboard shortcut, free of collisions — Task 8.
- Type detection table — `detectResponseType`, Task 2.
- HTTP response = headers above beautified body — `buildViewerOutput`, Task 6.
- Security: no HTML-string injection — section and output built via `vEl`/`textContent`/`createTextNode`; `DOMParser` output is read, never injected. Tasks 1, 6, 7.
- Copy button — Task 7.
- All in `index.html`, section markers present — Tasks 1, 2.

**Placeholder scan:** No TBD/TODO; every code step shows complete code; every test step shows the exact eval and expected JSON.

**Type consistency:** Segment shape `{text, cls}` is uniform across `tokenizeJson`, `headerSegments`, `beautifyBodySegments`, `buildViewerOutput`, `renderSegments`. Beautifier result shape `{ok, text, error}` is uniform across `beautifyJson` and `beautifyMarkup`. `buildViewerOutput` returns `{badge, segments}`; `beautifyBodySegments` returns `{label, segments}` — distinct on purpose (badge is derived from label).

**Known v1 limitations (acceptable, documented):**
- A redirect chain (`curl -i -L`, multiple `HTTP/` blocks) shows the first block's headers; the remainder renders as plain text.
- `<!doctype>` is omitted from beautified HTML (the walker starts at `documentElement`).
- An HTML fragment with no `<html>`/doctype is detected as XML; `beautifyMarkup` retries with the HTML parser, so it still renders.
