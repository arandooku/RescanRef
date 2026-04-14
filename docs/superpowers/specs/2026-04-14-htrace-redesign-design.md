# ScanRef htrace.sh Terminal Redesign

## Summary

Redesign the entire ScanRef app to match the htrace.sh terminal aesthetic. Approach B: full CSS rewrite + targeted HTML changes. JS untouched.

## Decisions

- **Font**: OCR A Extended (system font, no CDN), fallback chain: `'OCR A Extended', 'OCR A Std', Consolas, monospace`
- **Accent color**: Green-dominant. Green (#a6e22e) is the primary accent for nav, headers, buttons, code. Pink (#f92672) demoted to errors/critical indicators only.
- **Light theme**: Removed entirely. Single dark theme only.
- **Approach**: CSS rewrite + targeted HTML changes (Approach B). No JS changes.

## Color System

Single dark palette. Remove `:root[data-theme="light"]` block and `data-theme` attribute from `<html>`.

| Variable | New Value | Purpose |
|---|---|---|
| --main-bg | #0a0c08 | Deep black background |
| --sidebar-bg | #0e100c | Slightly lighter sidebar |
| --card-bg | #0e100c | Flat, same as sidebar |
| --code-bg | #0a0c08 | Same as main bg |
| --card-border | #1e201a | Subtle borders |
| --input-bg | #0a0c08 | Same as main bg |
| --input-border | #1e201a | Same as card border |
| --accent-pink | #f92672 | Errors, critical, failures only |
| --accent-green | #a6e22e | Primary accent (nav, headers, buttons, code) |
| --accent-cyan | #66d9ef | Header names, URLs, secondary info |
| --accent-yellow | #e6db74 | Strings, arguments, highlights |
| --accent-purple | #ae81ff | Tags, numbers, metadata |
| --accent-orange | #fd971f | Warnings |
| --text-primary | #e0e0d8 | Main text, slightly muted |
| --text-secondary | #8a8c84 | Secondary text |
| --text-muted | #4a4c44 | Muted/dimmed text |
| --card-hover-border | #3e403a | Hover state borders |
| --scrollbar-thumb | #2a2c24 | Scrollbar |
| --mark-bg | rgba(230, 219, 116, 0.2) | Search highlight |
| --overlay-bg | rgba(0,0,0,0.8) | Modal overlays |
| --shadow | none | No shadows - flat terminal aesthetic |

Remove all glow variables (--glow-pink, --glow-green, --glow-cyan).

## Typography

- **All text**: `'OCR A Extended', 'OCR A Std', Consolas, monospace`
- No sans-serif anywhere. The entire app is monospace.
- Remove the separate mono font-family declarations on code blocks (they inherit).
- Base size: 13px body, adjust from there.
- Line-height: 1.6

## Geometry

- **border-radius: 0 everywhere**. No rounded corners on anything - cards, buttons, inputs, badges, modals.
- **No box-shadows** except colored status dot glows.
- **No ambient gradients** (remove .main-content::before radial gradient).
- **No hover transforms** (remove translateY(-1px) on buttons).
- **No entrance animations** (remove fadeInUp, fadeIn, slideInLeft, pulseGlow keyframes and their usage). Terminal output appears instantly.

## Sidebar Changes

### CSS
- Active nav item: green accent instead of pink
  - border-left-color: --accent-green
  - background: rgba(166, 226, 46, 0.08)
  - color: --accent-green
- Hover: green tint instead of pink
- Brand name "SCANREF": stays #f92672 (pink) as the sole pink UI element - it's the logo/signature color. Everything else green.
- Match count badges: green-tinted instead of pink-tinted
- Search input: green focus border instead of pink

### HTML
- Remove theme toggle from sidebar footer entirely (the .sidebar-footer .theme-toggle block)
- Sidebar footer can show version info or be removed

## Section Headers - Terminal Prompt Style

### HTML Changes
Each section header currently looks like:
```html
<div class="section-header">
  <h2>Nmap Basics</h2>
  <p>24 commands</p>
</div>
```

Change to terminal-prompt style:
```html
<div class="section-header">
  <div class="section-prompt"># scanref --section nmap-basics --count 24</div>
  <h2>Nmap Basics</h2>
</div>
```

The `.section-prompt` line is styled as a dim terminal command. The h2 becomes a bordered inline box (green border, green text).

## Cards (Reference Section)

### CSS
- Sharp edges (border-radius: 0)
- Card header bullet: green dot instead of pink chevron
- Tags: keep purple (#ae81ff) background tint
- Code blocks: green text on #0a0c08, with syntax coloring:
  - Commands: #a6e22e (green)
  - Flags: #66d9ef (cyan)
  - Strings/args: #e6db74 (yellow)
  - Numbers: #ae81ff (purple)
  - Comments: #75715e (gray)
- Copy button: green border/text on hover, uppercase "COPY" text
- No card hover elevation/transform

## Craft Section

### CSS
- Toggle buttons: transparent bg with colored border, no border-radius
- Active toggle: subtle bg tint + brighter border
- Input fields: sharp edges, green focus border
- Output box: #0a0c08 bg, green command text with `$ ` prompt prefix
- Buttons: bordered, uppercase, letter-spacing

## Header Audit Section

### CSS
- Textarea: sharp, green focus border
- Buttons: bordered terminal style (analyze=green, clear=muted, save=green, compare=cyan)
- Audit results: htrace.sh output style
  - Status dots with colored glow (green=pass, orange=warn, red=fail)
  - Header names in cyan
  - Values in primary text color
  - Status words colored (pass=green, weak=orange, fail=red)
  - Dash separators between sections
  - `>>` prefix for raw header display
  - Summary line: "verification: N pass . N warn . N fail"
- Drop overlay: dashed border, green on dragover instead of pink

## Rescan History Section

### CSS
- List items: tabular, no border-radius
- Scores: colored (green=good, orange=mid, red=bad)
- Comparison view: diff-style with + (green) and - (red) prefixes
- Date/target in muted/cyan colors

## Buttons (Global)

All buttons become terminal-style:
- transparent background
- 1px solid border (color varies by purpose)
- no border-radius
- uppercase text with letter-spacing
- hover: brighter border + subtle bg tint
- no translateY transforms
- Primary action: green border/text
- Secondary/clear: muted border/text
- Danger/critical: pink border/text
- Info/compare: cyan border/text

Exception: the "analyze" button in header audit can have a subtle green bg to stand out as the primary CTA.

## Inputs & Textareas (Global)

- border-radius: 0
- background: --main-bg (#0a0c08)
- border: 1px solid --input-border (#1e201a)
- focus: border-color --accent-green
- no focus box-shadow (or very subtle green one)
- placeholder: #3a3c34

## Modals & Overlays

- Sharp corners
- Darker overlay (#000 at 0.8 opacity)
- No shadows
- Green accent for confirm actions

## Scrollbar

- Thinner (4px width)
- thumb: #2a2c24
- No border-radius on thumb

## What Stays the Same

- Overall layout (sidebar + main content flex)
- All JavaScript logic
- Data structures
- Sidebar navigation hierarchy and tree nav
- Card expand/collapse behavior
- All functional behavior
- Mobile hamburger menu (just restyled)

## What Gets Removed

- Light theme CSS block (~30 lines)
- Theme toggle HTML + JS
- data-theme attribute on <html>
- Entrance animations (fadeInUp, fadeIn, slideInLeft, pulseGlow)
- Ambient gradient (.main-content::before)
- All glow CSS variables
- All box-shadows except status dot glows
- All border-radius values
- All translateY hover transforms
