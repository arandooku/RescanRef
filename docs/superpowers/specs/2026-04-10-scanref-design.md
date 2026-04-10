# ScanRef — Design Spec

Single offline HTML file for shell/terminal command reference with interactive command crafting. Lives in OneDrive, opens in any browser, zero external dependencies.

## Constraints

- **Single `index.html` file** — all CSS, JS, and data inline
- **Fully offline** — no CDNs, no external fonts, no network requests
- **Data-driven** — commands and docs defined as JS arrays/objects at the top of the file; the rest is rendering logic
- **No localStorage** — no browser storage at all. Commands are edited directly in the HTML source. Scan history is persisted via a JSON file alongside the HTML
- **Target environment** — Windows laptops connected to Office 365. The `ScanRef/` folder lives in OneDrive and syncs across devices. User opens `index.html` directly in the browser (`file://` protocol). Some machines have limited/restricted internet access

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

## Rescan History Section

Tracks scan results over time and provides side-by-side comparison to see what's been fixed between scans.

### Sidebar Addition

A fourth sidebar group:

- **Reference** — static command cheatsheets
- **Craft** — interactive command builders
- **Docs** — verbose flag/option reference
- **Rescan History** — scan result tracking and comparison

### Storage: JSON Files in OneDrive Folder

No localStorage. All scan history persisted as JSON files in the same OneDrive folder as `index.html`. The folder syncs across Office 365 connected devices.

```
OneDrive/ScanRef/
  ├── index.html
  └── history/
      ├── 2026-04-10_prod-web-initial.json
      ├── 2026-04-15_prod-web-rescan.json
      ├── 2026-04-12_uat-api-initial.json
      └── archive-2026-q1.json          (optional: consolidated archive)
```

**Default mode: one file per scan session.** Each time the user imports scan results and saves, the app downloads a new JSON file with a descriptive filename (`YYYY-MM-DD_<env>-<target>-<title>.json`).

**Single-file mode (optional):** User can choose to work with a single consolidated JSON file instead. All new scan entries append to that file. The app detects whether a loaded file contains one entry or multiple and handles both.

**Archiving:** When a single file gets too large or the user wants to clean up, they can:
- Select scan entries in the history view
- Click "Export as Archive" — downloads selected entries as a new JSON file (e.g., `archive-2026-q1.json`)
- Optionally remove archived entries from the active working file

**Loading:**
- On app open, a prominent "Load History" drop zone is shown
- User can drag & drop one or multiple JSON files at once (or use file picker)
- App merges all loaded entries into memory, deduplicates by entry `id`
- Loaded files can be a mix of single-entry and multi-entry JSON files

**Saving:**
- After importing new scan results, user clicks "Save" which downloads the JSON file
- Filename auto-generated from metadata: `YYYY-MM-DD_<env>-<target>-<title-slug>.json`
- User can override the filename before saving
- If working in single-file mode, the save downloads the full consolidated file

**File format:** Both single-entry and multi-entry files use the same schema:

### Data Model

```js
// JSON file structure (works for single or multiple entries)
{
  "version": 1,
  "entries": [
    {
      "id": "uuid-string",
      "title": "Prod Web Server - Initial Scan",
      "environment": "Prod",         // "Prod" or "UAT"
      "target": "192.168.1.100",
      "timestamp": "2026-04-10T14:30:00Z",
      "scanType": "nmap-port",       // "nmap-port", "nmap-script", "curl-headers"
      "rawOutput": "...",            // original pasted/imported text
      "parsed": { ... }             // structured parsed data (see Parsing section)
    }
  ]
}
```

### User Flow

1. **Navigate to Rescan History** in sidebar
2. **Choose environment**: Prod or UAT (tab buttons at top)
3. **Import scan result**:
   - **Drag & drop** a `.txt`/`.nmap` output file onto the import area
   - **File picker** — click "Browse" to select a file
   - **Copy-paste** — paste raw output into a textarea, click "Import"
4. **Classify the import**:
   - App auto-detects scan type (nmap port scan, nmap script scan, curl headers) from the output format
   - User provides: **title** (required), confirms **environment** (Prod/UAT), confirms **target** (auto-detected from output)
   - Timestamp auto-set to current date/time, editable
5. **View parsed results** — app shows the structured parse (port table, script findings, headers)
6. **Save** — entry added to history, updated JSON auto-downloads

### Rescan Comparison Flow

1. User is in **Rescan History** → selects **Prod** or **UAT**
2. Sees a timeline/list of all scans for that environment, grouped by target
3. Selects two scans (or the app auto-selects latest vs. previous for same target)
4. **Side-by-side comparison view** opens:
   - Left: older scan (with timestamp + title)
   - Right: newer scan (with timestamp + title)
   - Differences highlighted with color coding:
     - **Green** — fixed/improved (port closed, vuln removed, header added)
     - **Red** — new issue (port opened, new vuln, header removed)
     - **Yellow** — changed but needs review (version changed, cipher changed)
     - **Gray** — unchanged

### Navigation Structure

```
Rescan History
  └─ Environment? → [Prod] [UAT]
       └─ Target list (grouped by IP/host)
            └─ Scan timeline (chronological list of scans for that target)
                 └─ Select two scans → Side-by-side comparison
```

### Parsing

The app parses three types of scan output into structured data.

#### nmap Port Scan Parser

Parses output from basic nmap scans (`-sS`, `-sT`, `-sU`, etc.).

**Input pattern:**
```
Nmap scan report for 192.168.1.100
Host is up (0.0045s latency).
Not shown: 995 closed ports

PORT     STATE    SERVICE     VERSION
22/tcp   open     ssh         OpenSSH 8.9p1
80/tcp   open     http        Apache httpd 2.4.52
443/tcp  open     ssl/http    nginx 1.18.0
3306/tcp closed   mysql
8080/tcp filtered http-proxy

OS details: Linux 5.4 - 5.15
```

**Parsed structure:**
```js
{
  type: "nmap-port",
  target: "192.168.1.100",
  hostStatus: "up",
  latency: "0.0045s",
  ports: [
    { port: 22, protocol: "tcp", state: "open", service: "ssh", version: "OpenSSH 8.9p1" },
    { port: 80, protocol: "tcp", state: "open", service: "http", version: "Apache httpd 2.4.52" },
    { port: 443, protocol: "tcp", state: "open", service: "ssl/http", version: "nginx 1.18.0" },
    { port: 3306, protocol: "tcp", state: "closed", service: "mysql", version: "" },
    { port: 8080, protocol: "tcp", state: "filtered", service: "http-proxy", version: "" }
  ],
  os: "Linux 5.4 - 5.15"
}
```

**Comparison logic:**
- Port state change: `open → closed` = green (fixed), `closed → open` = red (new exposure), `filtered → open` = red
- Service version change = yellow (review)
- Port not in new scan = green (removed)
- Port not in old scan = red (new)
- OS change = yellow

#### nmap Script Scan Parser

Parses output from NSE script scans (`--script ssl-cert`, `--script http-methods`, `--script vuln`, etc.).

**Supported script outputs:**

**`ssl-cert`:**
```
| ssl-cert: Subject: commonName=example.com
|   Issuer: commonName=R3/organizationName=Let's Encrypt
|   Not valid before: 2026-01-15
|   Not valid after:  2026-04-15
|   Subject Alternative Name: DNS:example.com, DNS:www.example.com
```

Parsed into:
```js
{
  script: "ssl-cert",
  port: 443,
  data: {
    subject: "example.com",
    issuer: "Let's Encrypt (R3)",
    validFrom: "2026-01-15",
    validTo: "2026-04-15",
    sans: ["example.com", "www.example.com"],
    expired: false,          // computed from validTo vs current date
    daysRemaining: 5         // computed
  }
}
```

Comparison: expired status changed, issuer changed, expiry date extended, SANs added/removed.

**`ssl-enum-ciphers`:**
```
| ssl-enum-ciphers:
|   TLSv1.0:
|     ciphers:
|       TLS_RSA_WITH_AES_128_CBC_SHA - weak
|   TLSv1.2:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 - strong
|   TLSv1.3:
|     ciphers:
|       TLS_AES_256_GCM_SHA384 - strong
```

Parsed into:
```js
{
  script: "ssl-enum-ciphers",
  port: 443,
  data: {
    versions: [
      { version: "TLSv1.0", ciphers: [{ name: "TLS_RSA_WITH_AES_128_CBC_SHA", strength: "weak" }] },
      { version: "TLSv1.2", ciphers: [{ name: "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384", strength: "strong" }] },
      { version: "TLSv1.3", ciphers: [{ name: "TLS_AES_256_GCM_SHA384", strength: "strong" }] }
    ]
  }
}
```

Comparison: TLS version added/removed (TLSv1.0 removed = green), weak cipher removed = green, new weak cipher = red.

**`http-methods`:**
```
| http-methods:
|   Supported Methods: GET HEAD POST OPTIONS TRACE
|   Potentially risky methods: TRACE
```

Parsed into:
```js
{
  script: "http-methods",
  port: 80,
  data: {
    methods: ["GET", "HEAD", "POST", "OPTIONS", "TRACE"],
    risky: ["TRACE"]
  }
}
```

Comparison: risky method removed = green, new risky method = red, method added/removed = yellow.

**`http-security-headers` / `http-headers`:**
```
| http-security-headers:
|   Strict-Transport-Security: max-age=31536000
|   X-Frame-Options: DENY
|   X-Content-Type-Options: NOSNIFF
|   MISSING: Content-Security-Policy
|   MISSING: X-XSS-Protection
```

Parsed into:
```js
{
  script: "http-security-headers",
  port: 443,
  data: {
    present: [
      { header: "Strict-Transport-Security", value: "max-age=31536000" },
      { header: "X-Frame-Options", value: "DENY" },
      { header: "X-Content-Type-Options", value: "NOSNIFF" }
    ],
    missing: ["Content-Security-Policy", "X-XSS-Protection"]
  }
}
```

Comparison: missing header now present = green, present header now missing = red, header value changed = yellow.

**`vulners` / `vuln`:**
```
| vulners:
|   cpe:/a:apache:http_server:2.4.52:
|     CVE-2024-12345  7.5  https://vulners.com/cve/CVE-2024-12345
|     CVE-2024-67890  5.0  https://vulners.com/cve/CVE-2024-67890
```

Parsed into:
```js
{
  script: "vulners",
  port: 80,
  data: {
    service: "apache:http_server:2.4.52",
    vulns: [
      { cve: "CVE-2024-12345", score: 7.5 },
      { cve: "CVE-2024-67890", score: 5.0 }
    ]
  }
}
```

Comparison: CVE no longer found = green (fixed), new CVE = red, score changed = yellow.

**Fallback for unrecognized scripts:** store as raw text blocks keyed by script name. Comparison uses line-level text diff with additions (red) and removals (green).

#### curl Headers Parser

Parses output from `curl -I` or `curl -v` (headers section).

**Input pattern:**
```
HTTP/2 200
date: Thu, 10 Apr 2026 14:30:00 GMT
content-type: text/html; charset=utf-8
server: nginx
strict-transport-security: max-age=31536000; includeSubDomains
x-frame-options: SAMEORIGIN
x-content-type-options: nosniff
content-security-policy: default-src 'self'
x-xss-protection: 1; mode=block
```

**Parsed structure:**
```js
{
  type: "curl-headers",
  target: "https://example.com",
  statusCode: 200,
  protocol: "HTTP/2",
  headers: [
    { key: "date", value: "Thu, 10 Apr 2026 14:30:00 GMT" },
    { key: "content-type", value: "text/html; charset=utf-8" },
    { key: "server", value: "nginx" },
    // ...
  ],
  securityHeaders: {
    present: [
      { header: "strict-transport-security", value: "max-age=31536000; includeSubDomains" },
      { header: "x-frame-options", value: "SAMEORIGIN" },
      { header: "x-content-type-options", value: "nosniff" },
      { header: "content-security-policy", value: "default-src 'self'" },
      { header: "x-xss-protection", value: "1; mode=block" }
    ],
    missing: []   // checked against known security header list
  }
}
```

**Known security headers checklist** (flagged as missing if absent):
- `Strict-Transport-Security`
- `X-Frame-Options`
- `X-Content-Type-Options`
- `Content-Security-Policy`
- `X-XSS-Protection`
- `Referrer-Policy`
- `Permissions-Policy`
- `X-Permitted-Cross-Domain-Policies`
- `Cross-Origin-Opener-Policy`
- `Cross-Origin-Resource-Policy`
- `Cross-Origin-Embedder-Policy`

**Comparison logic:**
- Security header missing → now present = green
- Security header present → now missing = red
- Header value changed = yellow (with old/new values shown)
- Status code changed = yellow
- Server header version disclosure removed = green
- New headers = neutral (gray)

### Comparison View UI

The side-by-side comparison view displays:

**Header bar:**
- Left: older scan title + timestamp
- Right: newer scan title + timestamp
- Summary badge: "3 fixed, 1 new issue, 2 unchanged"

**Comparison body (depends on scan type):**

**For port scans:**
- Table with columns: Port | Service | Old State | New State | Status
- Each row color-coded: green (fixed), red (new issue), yellow (changed), gray (same)

**For script scans:**
- Grouped by script name, then by finding
- Each finding shows old value → new value with color coding
- Expandable sections for verbose details

**For curl headers:**
- Table with columns: Header | Old Value | New Value | Status
- Security headers section at top with present/missing audit
- Color coding same as above

**Raw output tab:**
- Always available as a fallback
- Shows both raw outputs side by side with line-level diff highlighting

## Search

- Lives in the sidebar, always visible
- Filters in real-time as user types (debounced ~200ms)
- Matches against:
  - **Reference**: title, command text, notes, tags
  - **Craft**: tool name, flag labels, group names
  - **Docs**: flag, title, description text
  - **Rescan History**: scan title, target, environment
- When searching:
  - Sidebar categories show match counts (e.g., "Git (3)")
  - Main area shows matching items grouped by section across all three groups
  - Matching text is highlighted in yellow
- Clearing search returns to the previously active section
- Keyboard shortcut: `/` focuses the search bar, `Esc` clears and unfocuses

## Interactions

- **Copy to clipboard**: uses `navigator.clipboard.writeText()` with fallback to `document.execCommand('copy')` for older browsers. Shows "Copied!" tooltip for ~1.5s
- **Theme toggle**: switches CSS custom properties on `:root`. Defaults to dark mode on each load (no persistence)
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

- HTML structure + CSS + JS rendering logic: ~25-30 KB (includes parser + comparison engine)
- Starter reference commands (~30-40 entries): ~5 KB
- Craft tool definitions (nmap + curl): ~5 KB
- Docs content (verbose entries for ~50 flags): ~25-30 KB
- Rescan History UI + parsers + diff engine: ~15-20 KB
- **Total: ~75-100 KB** — well within single-file practicality
- **Scan history JSON files**: one per scan session (~1-5 KB each), or consolidated files that grow with usage
