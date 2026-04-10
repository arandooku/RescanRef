# ScanRef — Design Spec

Single offline HTML file for shell/terminal command reference with interactive command crafting. Lives in OneDrive, opens in any browser, zero external dependencies.

## Constraints

- **Single `index.html` file** — all CSS, JS, and data inline
- **Fully offline** — no CDNs, no external fonts, no network requests
- **Data-driven** — commands and docs defined as JS arrays/objects at the top of the file; the rest is rendering logic
- **No localStorage for data** — commands are edited directly in the HTML source. `localStorage` is only used for theme preference (dark/light)
- **Target environment** — Windows laptop with limited internet, opened in local browser from OneDrive folder

## Visual Design

- **Theme**: Monokai color palette
  - Dark mode (default): `#272822` sidebar, `#1e1f1c` main bg, `#f92672` accent (pink), `#a6e22e` code/success (green), `#66d9ef` links (cyan), `#e6db74` warnings (yellow), `#f8f8f2` text
  - Light mode: `#f8f8f2` sidebar, `#fafaf0` main bg, same accent hues (`#f92672`, `#a6e22e`, `#66d9ef`, `#e6db74`) darkened ~15% for readability on light backgrounds, `#272822` text, code blocks use `#f5f5e8` bg with `#272822` text
- **Typography**: System font stack for UI (`system-ui, -apple-system, Segoe UI, sans-serif`), monospace stack for code (`Cascadia Code, Fira Code, Consolas, monospace`)
- **Dark/light toggle**: bottom of sidebar, persists via `localStorage`

## Layout

Sidebar Tabs + Search pattern:

- **Sidebar** (fixed ~220px left):
  - App title ("ScanRef") and subtitle
  - Search bar — filters across ALL sections (Reference, Craft, Docs)
  - Category list grouped under three headings: Reference, Craft, Docs
  - Active category highlighted with Monokai pink
  - Dark/light toggle at bottom
- **Main area** (fills remaining width):
  - Section heading + description
  - Content area (cards, crafter UI, or doc entries depending on section type)
- **Responsive**: sidebar collapses to hamburger menu on narrow viewports (<768px)

## Sidebar Groups

### Reference (static command cheatsheets)

Sections organized by tool/technology. Starter sections (user will customize):
- Git
- Docker
- PowerShell
- SSH / Network
- npm / Node

Each section contains command cards.

### Craft (interactive command builders)

- nmap
- curl

Each crafter has its own UI with target input, toggles, and live command preview.

### Docs (verbose flag/option reference)

- nmap Reference
- curl Reference

Each doc page has detailed entries for every flag/option available in the corresponding Craft section.

## Data Structure

All data lives in JS objects at the top of the file for easy editing:

```js
// Reference commands
const commands = [
  {
    section: "Git",
    title: "Undo last commit (keep changes)",
    cmd: "git reset --soft HEAD~1",
    notes: "Moves HEAD back one commit but keeps changes staged",
    tags: ["undo", "reset"]
  },
  // ...
];

// Craft definitions — each tool's options, flags, and defaults
const craftTools = {
  nmap: {
    name: "nmap",
    description: "Network scanner and security auditing tool",
    targetPlaceholder: "192.168.1.1 or hostname",
    groups: [
      {
        name: "Scan Type",
        type: "single",  // single-select pill buttons
        options: [
          { flag: "-sS", label: "SYN Stealth", default: true, docId: "nmap-sS" },
          { flag: "-sT", label: "TCP Connect", docId: "nmap-sT" },
          { flag: "-sU", label: "UDP", docId: "nmap-sU" },
          { flag: "-sA", label: "ACK", docId: "nmap-sA" },
          { flag: "-sV", label: "Version", docId: "nmap-sV" },
          { flag: "-sn", label: "Ping Sweep", docId: "nmap-sn" }
        ]
      },
      {
        name: "Ports",
        type: "single",
        options: [
          { flag: "", label: "Top 1000 (default)", default: true, docId: "nmap-ports-default" },
          { flag: "-p-", label: "All Ports", docId: "nmap-p-all" },
          { flag: "--top-ports 100", label: "Top 100", docId: "nmap-top-ports" },
          { flag: "-p", label: "Custom range", customInput: true, placeholder: "80,443,8080-8090", docId: "nmap-p-custom" }
        ]
      },
      {
        name: "Timing",
        type: "single",
        options: [
          { flag: "-T0", label: "T0 Paranoid", docId: "nmap-T0" },
          { flag: "-T1", label: "T1 Sneaky", docId: "nmap-T1" },
          { flag: "-T2", label: "T2 Polite", docId: "nmap-T2" },
          { flag: "-T3", label: "T3 Normal", default: true, docId: "nmap-T3" },
          { flag: "-T4", label: "T4 Aggressive", docId: "nmap-T4" },
          { flag: "-T5", label: "T5 Insane", docId: "nmap-T5" }
        ]
      }
    ],
    toggles: [
      { flag: "-O", label: "OS Detection", docId: "nmap-O" },
      { flag: "-sV", label: "Service Version", docId: "nmap-sV-toggle" },
      { flag: "-A", label: "Aggressive", docId: "nmap-A" },
      { flag: "-v", label: "Verbose", docId: "nmap-v" },
      { flag: "-Pn", label: "Skip host discovery", docId: "nmap-Pn" },
      { flag: "--script vuln", label: "Vuln scripts", docId: "nmap-script-vuln" },
      { flag: "-oN", label: "Output to file", customInput: true, placeholder: "scan.txt", docId: "nmap-oN" }
    ]
  },
  curl: {
    // Same structure as nmap — populated from the curl Craft Options
    // section below (Method, Headers, Data/Body, Options groups)
  }
};

// Docs entries — verbose reference for each flag
const docs = {
  nmap: [
    {
      id: "nmap-sS",
      flag: "-sS",
      category: "Scan Type",
      title: "TCP SYN Stealth Scan",
      description: "The default and most popular scan type...",
      example: "nmap -sS 192.168.1.1",
      platforms: {
        linux: "Requires root: sudo nmap -sS ...",
        windows: "Run CMD/PowerShell as Administrator. No sudo prefix needed."
      },
      warnings: ["Requires root/admin privileges"],
      tips: ["Fastest scan type for large networks"]
    },
    // ...
  ],
  curl: [
    // ...
  ]
};
```

## Reference Section — Command Cards

Each card displays:
- **Title** — short description of what the command does
- **Tag badge** — section name (e.g., "git") in a small pill
- **Code block** — monospace command text on dark background
- **Copy button** — right side of code block, copies command to clipboard, shows "Copied!" for ~1.5 seconds
- **Notes** (optional) — brief explanation or parameter hints below the code block

## Craft Section — Command Builder UI

Each tool crafter contains (top to bottom):

1. **Section header** — tool name + description
2. **Target input** — text field with pink border, large monospace text. Updating the target live-updates the crafted command
3. **Option groups** — each group is a labeled card:
   - **Single-select groups** (Scan Type, Ports, Timing) — rendered as pill buttons. One active per group (pink bg). Clicking another deselects the current
   - **Custom input**: when a "Custom range" pill is selected, a text input appears below for the user to type a value (e.g., port range)
4. **Toggle switches** — two-column grid of on/off switches for boolean flags. Each toggle has:
   - Flag name and label
   - **ⓘ icon** — cyan link that navigates to the Docs entry for that flag
   - Toggle switch (green = on, gray = off)
   - Custom input variant: some toggles (like `-oN`) show a text field when enabled
5. **Crafted command output** — sticky/prominent card at the bottom:
   - Green border to stand out
   - Color-coded command preview: flags in pink, options in yellow, timing in cyan, target in green
   - Large prominent **Copy** button (green bg, bold)
   - Command updates in real-time as toggles/inputs change

### nmap Craft Options

**Scan Type** (single-select):
- `-sS` SYN Stealth (default)
- `-sT` TCP Connect
- `-sU` UDP
- `-sA` ACK
- `-sV` Version Detection
- `-sn` Ping Sweep (no port scan)

**Ports** (single-select):
- Top 1000 — default, no flag
- `-p-` All 65535 ports
- `--top-ports 100` Top 100
- `-p <custom>` Custom range (shows text input)

**Timing** (single-select):
- `-T0` Paranoid
- `-T1` Sneaky
- `-T2` Polite
- `-T3` Normal (default)
- `-T4` Aggressive
- `-T5` Insane

**Toggles** (multi-select on/off):
- `-O` OS Detection
- `-sV` Service/Version Detection
- `-A` Aggressive (OS + version + script + traceroute)
- `-v` Verbose output
- `-vv` Very verbose
- `-Pn` Skip host discovery (treat all hosts as online)
- `-n` No DNS resolution
- `--script vuln` Vulnerability scanning scripts
- `--script=default` Default NSE scripts
- `-oN <file>` Normal output to file (shows text input)
- `-oX <file>` XML output to file (shows text input)
- `--reason` Show reason for port state

### curl Craft Options

**Method** (single-select):
- `-X GET` (default)
- `-X POST`
- `-X PUT`
- `-X DELETE`
- `-X PATCH`
- `-X HEAD`

**Headers** (toggles):
- `-H "Content-Type: application/json"`
- `-H "Content-Type: application/x-www-form-urlencoded"`
- `-H "Authorization: Bearer <token>"` (shows token input)
- `-H "Accept: application/json"`
- Custom header (shows key:value input)

**Data/Body** (single-select, appears for POST/PUT/PATCH):
- `-d '<json>'` JSON body (shows textarea)
- `-d '<form>'` Form data (shows textarea)
- `-F 'file=@<path>'` File upload (shows file path input)

**Options** (toggles):
- `-v` Verbose
- `-s` Silent
- `-L` Follow redirects
- `-k` Insecure (skip TLS verification)
- `-o <file>` Output to file (shows input)
- `-I` Headers only
- `-w "%{http_code}"` Show status code
- `--connect-timeout <sec>` Connection timeout (shows input)
- `-x <proxy>` Use proxy (shows input)

## Docs Section — Verbose Reference

Each tool gets its own Docs page. Every flag/option from the Craft section has a corresponding doc entry.

### Doc Entry Structure

Each entry contains:
- **Flag** — large monospace, color-coded by category
- **Category badge** — Scan Type, Option, NSE Script, Header, etc.
- **Full title** — human-readable name
- **Description** — verbose multi-sentence explanation: what it does, how it works, when to use it, caveats
- **Platform behavior**:
  - **Linux** — how to run the command, any `sudo` requirements, path considerations, common shell behavior
  - **Windows (CMD)** — differences when running via `cmd.exe`: admin elevation (Run as Administrator instead of sudo), path format (`C:\Program Files\Nmap\nmap.exe` vs just `nmap` if in PATH), output encoding quirks, any flags that behave differently
  - **Windows (PowerShell)** — any PowerShell-specific notes (e.g., escaping special characters, `Invoke-WebRequest` as curl alias conflict)
- **Example command** — with target placeholder
- **Warnings** — root/admin requirements, potential for detection, speed impact
- **Tips** — what flags to combine it with, common patterns

### Craft → Docs Navigation

- Every **ⓘ** icon in Craft links to a `docId`
- Clicking **ⓘ** switches the sidebar active section to the corresponding Docs page and scrolls to the entry with matching `id`
- A **"← Back to Craft"** link at the top of Docs returns to the Craft section. All Craft selections and the target input are preserved (state lives in JS variables, not the DOM)
- Pill button groups in Craft also have a "Compare all →" link that jumps to the Docs page with all options in that group visible

## Search

- Lives in the sidebar, always visible
- Filters in real-time as user types (debounced ~200ms)
- Matches against:
  - **Reference**: title, command text, notes, tags
  - **Craft**: tool name, flag labels, group names
  - **Docs**: flag, title, description text
- When searching:
  - Sidebar categories show match counts (e.g., "Git (3)")
  - Main area shows matching items grouped by section across all three groups
  - Matching text is highlighted in yellow
- Clearing search returns to the previously active section
- Keyboard shortcut: `/` focuses the search bar, `Esc` clears and unfocuses

## Interactions

- **Copy to clipboard**: uses `navigator.clipboard.writeText()` with fallback to `document.execCommand('copy')` for older browsers. Shows "Copied!" tooltip for ~1.5s
- **Theme toggle**: switches CSS custom properties on `:root`. Stores preference in `localStorage` under key `scanref-theme`
- **Keyboard**: `/` to search, `Esc` to clear search, `1-9` to switch sidebar sections
- **Responsive**: below 768px width, sidebar becomes a slide-out menu triggered by hamburger icon

## How to Add Content

### Adding a reference command

Add an object to the `commands` array:
```js
{ section: "Git", title: "Cherry-pick a commit", cmd: "git cherry-pick <hash>", notes: "Applies a single commit to current branch", tags: ["cherry-pick"] }
```

### Adding a new reference section

Just use a new `section` value — the sidebar auto-generates from unique section names.

### Adding a craft toggle/option

Add to the relevant `craftTools` entry's `groups` or `toggles` array, and add a corresponding `docs` entry with matching `docId`.

### Adding a new craft tool

Add a new key to `craftTools` with the same structure. The sidebar and crafter UI auto-generate from this object.

## File Size Estimate

- HTML structure + CSS + JS rendering logic: ~15-20 KB
- Starter reference commands (~30-40 entries): ~5 KB
- Craft tool definitions (nmap + curl): ~5 KB
- Docs content (verbose entries for ~50 flags): ~25-30 KB
- **Total: ~50-75 KB** — well within single-file practicality
