# Response Viewer — Design Spec

**Date:** 2026-05-22
**Status:** Approved
**Component:** ScanRef (`index.html`, single-file offline app)

## Purpose

Give users a way to beautify raw curl output inside ScanRef. The app crafts
curl commands but cannot execute them, so the viewer is a paste-and-format
tool: the user runs curl elsewhere, pastes the raw output, and sees a
readable, syntax-highlighted result.

This solves the readability gap surfaced by the bulk-scan work — `curl -s`
output is an undifferentiated blob, and JSON/HTML/XML bodies are unreadable
without formatting.

## Goals

- Beautify JSON, HTTP headers, HTML, and XML pasted as raw text.
- Fully offline — no dependencies, no CDNs, no fetch. Native browser APIs only.
- Match the existing terminal aesthetic (dark, OCR A Extended, green accent,
  no border-radius / shadows / animations).
- Never fail to blank: malformed input always shows a message plus the raw text.

## Non-Goals (YAGNI)

- No minify / compact mode — beautify only.
- No indent-size configuration — fixed at 2 spaces.
- No download button — Copy button only.
- No live execution of curl.

## Placement

- New standalone section, hash route `#viewer`, mirroring the Header Audit
  section (`#audit`).
- New sidebar nav item labelled "Response Viewer", placed next to Header Audit.
- Keyboard jump shortcut `v` (consistent with existing `c/d/r/h` section
  jumps). `v` is unused by current bindings (`/ ? c d r h`, `1-9`, `Esc`).

## Layout

Mirrors the Header Audit section structure:

1. Section header (terminal-prompt style, e.g. `# scanref --viewer`).
2. Auto-resizing paste `<textarea>` for raw input (reuse the existing
   auto-resize paste-textarea behaviour).
3. A detected-type badge showing what the input was recognised as
   (JSON / Headers / HTML / XML / HTTP response / Plain text).
4. Beautified output in a `<pre>` code block, terminal-styled.
5. A Copy button for the beautified output.

Beautification runs on input (no explicit "Beautify" button needed).

## Type Detection

Sniff the trimmed input:

| Condition | Detected type |
|-----------|---------------|
| Starts with `HTTP/` | HTTP response — split headers from body at the first blank line, then beautify the body by its own type |
| First non-whitespace char is `{` or `[` | JSON |
| First non-whitespace char is `<` and (`<?xml` present, or no `<html`/`<!doctype` marker) | XML |
| First non-whitespace char is `<` otherwise | HTML |
| None of the above | Plain text — shown unchanged |

## Beautifiers

### JSON
- `JSON.parse(raw)` then `JSON.stringify(obj, null, 2)`.
- Render with `<span>` token colouring: keys, string values, numbers,
  booleans/null distinguished by class.
- On `JSON.parse` failure: show an error line plus the raw text.

### HTTP headers
- Parse the status line (`HTTP/x.x NNN Reason`) and each `Key: Value` pair.
- Render aligned, with the header name and value colour-distinguished.
- For a full HTTP response, beautify the body separately by its detected type
  and show headers above body.

### HTML / XML
- `new DOMParser().parseFromString(raw, 'text/html' | 'application/xml')`.
- Recursively walk the resulting (detached) document and re-serialise to a
  string with 2-space indent per depth level.
- Handle: element nodes, text nodes (trim/collapse insignificant whitespace),
  comment nodes, void/self-closing elements.
- On a parser error (`<parsererror>` element present, or empty result):
  fall back to showing the raw text with a notice.

## Error Handling

Every beautifier path has a fallback. Malformed JSON, unparseable XML, or an
HTML parser error must produce a clear inline message and still display the
original raw text. The output panel is never left blank.

## Security

The app's XSS model is `escapeHtml()` on every untrusted value before it
reaches the DOM, backed by a CSP. The viewer must not weaken this:

- Beautified output is always assembled as an **escaped string** wrapped in
  styled `<span>`s — never raw `innerHTML` of untrusted content.
- HTML input is parsed with `DOMParser` into a **detached document**, which
  does not execute scripts or load resources. The walker reads node names and
  text only and produces a plain string. That string is `escapeHtml()`-ed
  before display.
- No pasted markup is ever inserted into the live document.

## Constraints

- All changes within `index.html`. No new runtime files.
- New code under a `/* ===== Response Viewer ===== */` section comment marker.
- Follow existing patterns: section rendering, hash routing (`navigateTo`),
  nav registration, Copy-button wiring, auto-resize textareas.

## Acceptance Criteria

- Pasting a JSON object/array shows it indented with coloured tokens.
- Pasting a `-i`/`-I` header dump shows aligned, coloured headers.
- Pasting an HTML or XML document shows it indented per nesting depth.
- Pasting a full `HTTP/...` response shows beautified headers above a
  beautified body.
- Malformed input of any type shows a message and the raw text, never blank.
- `#viewer` deep-links to the section; the nav item navigates to it.
- No console errors; no external requests; pasted markup never executes.
