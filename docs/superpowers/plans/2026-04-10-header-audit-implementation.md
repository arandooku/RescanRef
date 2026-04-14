# Header Audit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a "Header Audit" section to RescanRef that analyzes HTTP response headers for security issues (missing, deprecated, insecure values, fingerprinting) with OWASP-recommended fixes, saveable/comparable over time.

**Architecture:** All new code goes into the existing `index.html` single-file app. New JS data constants (`HEADER_RULES`, `auditEntries[]`), analysis engine functions, UI renderer functions, and CSS styles are added in clearly marked sections. Reuses existing parsers (`parseCurlHeaders`, `parseNmapScript`), copy-to-clipboard, modal pattern, drop zone pattern, and comparison engine color scheme. All dynamic content is passed through the existing `escapeHtml()` function before rendering, consistent with the existing security pattern throughout the codebase.

**Tech Stack:** Vanilla HTML/CSS/JS (same as existing app). No new dependencies.

---

## File Structure

Single file: `index.html` (currently ~4,667 lines). New code adds ~1,800 lines in these logical sections:

```
index.html
  <style>
    ... existing CSS ...
    /* ===== Header Audit ===== */        <- NEW: ~120 lines of CSS
  </style>
  <body>
    ... existing HTML ...
    <!-- Audit Save Modal -->              <- NEW: modal HTML (~20 lines)
  </body>
  <script>
    ... existing data ...
    // === HEADER AUDIT: DATA ===          <- NEW: ~1,400 lines (mostly fingerprint data)
    ... existing functions ...
    // === HEADER AUDIT: PARSER ===        <- NEW: ~60 lines
    // === HEADER AUDIT: ENGINE ===        <- NEW: ~120 lines
    // === HEADER AUDIT: UI ===            <- NEW: ~300 lines
    ... existing init() ...
  </script>
```

---

## Task 1: Add Header Audit CSS Styles

**Files:**
- Modify: `index.html:1165` (before closing `</style>` tag)

Add CSS for the audit report sections (collapsible panels, color indicators, count badges, audit cards).

- [ ] **Step 1: Add Header Audit CSS before the closing `</style>` tag**

Insert before line 1166 (`</style>`), after the existing `.hidden` rule on line 1166:

```css
/* ===== Header Audit ===== */
.audit-container { max-width: 900px; }
.audit-input-zone { margin-bottom: 20px; }
.audit-textarea {
  width: 100%;
  min-height: 160px;
  padding: 12px;
  background: var(--input-bg);
  color: var(--text-primary);
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-family: 'Cascadia Code', 'Fira Code', 'JetBrains Mono', Consolas, monospace;
  font-size: 0.8rem;
  resize: vertical;
  outline: none;
  transition: border-color 0.2s;
}
.audit-textarea:focus { border-color: var(--accent-pink); }
.audit-textarea::placeholder { color: var(--text-muted); }
.audit-actions { display: flex; gap: 8px; margin-top: 10px; flex-wrap: wrap; }
.audit-btn-analyze {
  padding: 9px 22px;
  background: var(--accent-pink);
  color: #fff;
  font-weight: 700;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
}
.audit-btn-analyze:hover { transform: translateY(-1px); box-shadow: 0 4px 12px rgba(249, 38, 114, 0.3); }
.audit-btn-clear {
  padding: 9px 18px;
  background: transparent;
  border: 1px solid var(--card-border);
  border-radius: 6px;
  color: var(--text-secondary);
  cursor: pointer;
  transition: border-color 0.2s;
}
.audit-btn-clear:hover { border-color: var(--text-muted); }
.audit-btn-save {
  padding: 9px 18px;
  background: var(--accent-green);
  border: none;
  border-radius: 6px;
  color: #0d0e0a;
  font-weight: 700;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
}
.audit-btn-save:hover { transform: translateY(-1px); box-shadow: 0 4px 12px rgba(166, 226, 46, 0.3); }
.audit-btn-compare {
  display: none;
  padding: 9px 22px;
  background: var(--accent-cyan);
  color: #0d0e0a;
  font-weight: 700;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
}
.audit-btn-compare:hover { transform: translateY(-1px); box-shadow: 0 4px 12px rgba(102, 217, 239, 0.3); }
.audit-btn-compare.visible { display: inline-block; }

.audit-drop-overlay {
  border: 2px dashed var(--card-border);
  border-radius: 8px;
  padding: 18px;
  text-align: center;
  color: var(--text-muted);
  font-size: 0.8rem;
  margin-bottom: 10px;
  cursor: pointer;
  transition: border-color 0.2s, background 0.2s;
}
.audit-drop-overlay.dragover {
  border-color: var(--accent-pink);
  background: rgba(249, 38, 114, 0.06);
}
.audit-drop-overlay input[type="file"] { display: none; }

.audit-report { margin-top: 20px; }
.audit-section {
  border: 1px solid var(--card-border);
  border-radius: 6px;
  margin-bottom: 8px;
  overflow: hidden;
  transition: border-color 0.2s;
}
.audit-section:hover { border-color: var(--card-hover-border); }
.audit-section-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  cursor: pointer;
  background: var(--card-bg);
  user-select: none;
  transition: background 0.2s;
}
.audit-section-header:hover { background: var(--card-hover-border); }
.audit-section-arrow {
  font-size: 0.7rem;
  color: var(--text-muted);
  transition: transform 0.2s;
  flex-shrink: 0;
}
.audit-section.expanded .audit-section-arrow { transform: rotate(90deg); }
.audit-section-title {
  flex: 1;
  font-size: 0.85rem;
  font-weight: 600;
}
.audit-section-count {
  font-size: 0.7rem;
  padding: 2px 8px;
  border-radius: 10px;
  font-weight: 700;
}
.audit-section-count.red { background: rgba(249, 38, 114, 0.15); color: var(--accent-pink); }
.audit-section-count.yellow { background: rgba(230, 219, 116, 0.15); color: var(--accent-yellow); }
.audit-section-count.orange { background: rgba(253, 151, 31, 0.15); color: #fd971f; }
.audit-section-count.gray { background: rgba(117, 113, 94, 0.15); color: var(--text-muted); }
.audit-section-count.green { background: rgba(166, 226, 46, 0.15); color: var(--accent-green); }
.audit-section-body {
  display: none;
  padding: 0 14px 12px;
}
.audit-section.expanded .audit-section-body { display: block; }
.audit-finding {
  padding: 8px 0;
  border-bottom: 1px solid var(--card-border);
  font-size: 0.82rem;
}
.audit-finding:last-child { border-bottom: none; }
.audit-finding-header {
  font-weight: 600;
  color: var(--accent-cyan);
  font-family: 'Cascadia Code', 'Fira Code', 'JetBrains Mono', Consolas, monospace;
  font-size: 0.8rem;
}
.audit-finding-detail {
  color: var(--text-secondary);
  font-size: 0.78rem;
  margin-top: 3px;
}
.audit-finding-recommended {
  margin-top: 5px;
}
.audit-finding-recommended .code-block-wrapper {
  margin-top: 4px;
}
.audit-finding-recommended .code-block {
  font-size: 0.75rem;
  padding: 6px 10px;
}
.audit-finding-value {
  font-family: 'Cascadia Code', 'Fira Code', 'JetBrains Mono', Consolas, monospace;
  font-size: 0.75rem;
  color: var(--text-muted);
  margin-top: 2px;
}

/* Audit timeline (saved audits) */
.audit-timeline { margin-top: 20px; }
.audit-entry {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 14px;
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: 6px;
  margin-bottom: 6px;
  cursor: pointer;
  transition: all 0.2s;
}
.audit-entry:hover { border-color: var(--card-hover-border); transform: translateX(2px); }
.audit-entry.selected { border-color: var(--accent-pink); background: rgba(249, 38, 114, 0.06); }
.audit-entry-select {
  width: 18px; height: 18px;
  border: 2px solid var(--card-border);
  border-radius: 3px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}
.audit-entry.selected .audit-entry-select { border-color: var(--accent-pink); background: var(--accent-pink); }
.audit-entry.selected .audit-entry-select::after { content: '\2713'; color: #fff; font-size: 0.7rem; }
.audit-entry-title { flex: 1; font-size: 0.85rem; }
.audit-entry-summary {
  font-size: 0.7rem;
  color: var(--text-muted);
  font-family: 'Cascadia Code', 'Fira Code', 'JetBrains Mono', Consolas, monospace;
}
.audit-entry-date { font-size: 0.75rem; color: var(--text-muted); }
```

- [ ] **Step 2: Verify in browser**

Open `index.html` in browser. Verify the app still loads correctly -- no CSS errors in console, existing sections render normally.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "style: add Header Audit CSS styles"
```

---

## Task 2: Add Audit Save Modal HTML

**Files:**
- Modify: `index.html:1238` (after the existing classify modal closing `</div>`, before `<script>`)

- [ ] **Step 1: Add the audit save modal HTML**

Insert after line 1238 (closing `</div>` of classifyModal), before line 1240 (`<script>`):

```html
<!-- Audit Save Modal -->
<div class="modal-overlay" id="auditSaveModal">
  <div class="modal">
    <h3>Save Audit Result</h3>
    <div class="modal-field">
      <label for="auditModalTitle">Title (required)</label>
      <input type="text" id="auditModalTitle" placeholder="e.g. example.com - Q1 Audit">
    </div>
    <div class="modal-field">
      <label for="auditModalTarget">Target</label>
      <input type="text" id="auditModalTarget" placeholder="Auto-detected from headers">
    </div>
    <div class="modal-actions">
      <button class="modal-btn-cancel" id="auditModalCancel">Cancel</button>
      <button class="modal-btn-save" id="auditModalSave">Save &amp; Export</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Verify in browser**

Open `index.html`. Verify no rendering issues. The modal should be hidden (overlay not visible).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add audit save modal HTML"
```

---

## Task 3: Add Header Audit Data Constants

**Files:**
- Modify: `index.html` -- insert after the `docs` object (after ~line 2980), before the utility functions

This is the largest task. It adds the `HEADER_RULES` constant and `auditEntries`/`auditSelected`/`auditComparing` state variables.

- [ ] **Step 1: Add the state variables and HEADER_RULES constant**

Insert after the closing of the `docs` object (after ~line 2980, before `function generateId()`), with a section marker:

```js
/* ================================================================
   HEADER AUDIT: DATA
   ================================================================ */

var auditEntries = [];
var auditSelected = [];
var auditComparing = false;
var pendingAuditResult = null;

var HEADER_RULES = {
  // -- Missing: 15 critical security headers --
  missing: [
    { header: 'Cache-Control', description: 'Controls caching behavior; prevents sensitive data from being cached', recommended: 'no-store, max-age=0' },
    { header: 'Clear-Site-Data', description: 'Clears browsing data (cookies, storage, cache) associated with the site', recommended: '"cache","cookies","storage"' },
    { header: 'Content-Type', description: 'Prevents MIME type confusion attacks', recommended: 'text/html; charset=UTF-8' },
    { header: 'Content-Security-Policy', description: 'Mitigates XSS and data injection attacks by controlling resource loading', recommended: "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self'; font-src 'self'; connect-src 'self'; frame-ancestors 'self'; base-uri 'self'; form-action 'self'" },
    { header: 'Cross-Origin-Embedder-Policy', description: 'Controls cross-origin resource embedding; required for SharedArrayBuffer', recommended: 'require-corp' },
    { header: 'Cross-Origin-Opener-Policy', description: 'Isolates browsing context to prevent cross-origin attacks', recommended: 'same-origin' },
    { header: 'Cross-Origin-Resource-Policy', description: 'Controls which origins can load this resource', recommended: 'same-origin' },
    { header: 'NEL', description: 'Network Error Logging; reports network errors to a specified endpoint', recommended: '{"report_to":"default","max_age":31536000,"include_subdomains":true}' },
    { header: 'Permissions-Policy', description: 'Controls which browser features the page can use', recommended: 'accelerometer=(), camera=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=()' },
    { header: 'Referrer-Policy', description: 'Controls how much referrer information is sent with requests', recommended: 'strict-origin-when-cross-origin' },
    { header: 'Strict-Transport-Security', description: 'Forces HTTPS connections; prevents protocol downgrade attacks', recommended: 'max-age=31536000; includeSubDomains; preload' },
    { header: 'X-Content-Type-Options', description: 'Prevents MIME type sniffing', recommended: 'nosniff' },
    { header: 'X-Frame-Options', description: 'Prevents clickjacking by controlling iframe embedding', recommended: 'DENY' },
    { header: 'X-Permitted-Cross-Domain-Policies', description: 'Controls Adobe Flash/Acrobat cross-domain policy files', recommended: 'none' },
    { header: 'Integrity-Policy', description: 'Enforces subresource integrity checks', recommended: 'require-sri-for script style' }
  ],

  // -- Deprecated: headers that should not be present --
  deprecated: [
    { header: 'Accept-CH-Lifetime', reason: 'Deprecated; removed from Client Hints specification' },
    { header: 'Content-DPR', reason: 'Deprecated; removed from Client Hints specification' },
    { header: 'Digest', reason: 'Deprecated; superseded by Content-Digest and Repr-Digest' },
    { header: 'Expect-CT', reason: 'Deprecated since June 2021; Certificate Transparency is now enforced by default' },
    { header: 'Feature-Policy', reason: 'Deprecated; renamed to Permissions-Policy' },
    { header: 'Large-Allocation', reason: 'Deprecated; non-standard Firefox-only header' },
    { header: 'P3P', reason: 'Deprecated; Platform for Privacy Preferences is obsolete' },
    { header: 'Pragma', reason: 'Deprecated; use Cache-Control instead' },
    { header: 'Public-Key-Pins', reason: 'Deprecated; risk of bricking sites, replaced by Certificate Transparency' },
    { header: 'Public-Key-Pins-Report-Only', reason: 'Deprecated along with Public-Key-Pins' },
    { header: 'Report-To', reason: 'Deprecated; superseded by Reporting-Endpoints' },
    { header: 'Tk', reason: 'Deprecated; Do Not Track header is obsolete' },
    { header: 'Warning', reason: 'Deprecated in HTTP/1.1; removed in HTTP semantics (RFC 9110)' },
    { header: 'X-Content-Security-Policy', reason: 'Deprecated; use standard Content-Security-Policy instead' },
    { header: 'X-Download-Options', reason: 'Deprecated; IE-only header, no longer relevant' },
    { header: 'X-Pad', reason: 'Deprecated; was a Netscape workaround for browser bugs' },
    { header: 'X-SourceMap', reason: 'Deprecated; use standard SourceMap header instead' },
    { header: 'X-UA-Compatible', reason: 'Deprecated; IE compatibility mode is obsolete' },
    { header: 'X-Webkit-CSP', reason: 'Deprecated; use standard Content-Security-Policy instead' },
    { header: 'X-XSS-Protection', reason: 'Deprecated; can introduce XSS vulnerabilities in older browsers. Use CSP instead' }
  ],

  // -- Fingerprint: headers that leak server/technology information --
  // Each entry: { header, pattern, technology }
  // pattern: '*' means mere presence is a fingerprint; otherwise substring match on value
  fingerprint: [
    { header: 'Server', pattern: 'Apache', technology: 'Apache HTTP Server' },
    { header: 'Server', pattern: 'nginx', technology: 'nginx' },
    { header: 'Server', pattern: 'Microsoft-IIS', technology: 'Microsoft IIS' },
    { header: 'Server', pattern: 'LiteSpeed', technology: 'LiteSpeed Web Server' },
    { header: 'Server', pattern: 'Caddy', technology: 'Caddy Web Server' },
    { header: 'Server', pattern: 'Kestrel', technology: 'ASP.NET Core Kestrel' },
    { header: 'Server', pattern: 'Cowboy', technology: 'Erlang/Cowboy' },
    { header: 'Server', pattern: 'gunicorn', technology: 'Python Gunicorn' },
    { header: 'Server', pattern: 'Jetty', technology: 'Eclipse Jetty' },
    { header: 'Server', pattern: 'Tomcat', technology: 'Apache Tomcat' },
    { header: 'Server', pattern: 'Express', technology: 'Node.js Express' },
    { header: 'Server', pattern: 'Werkzeug', technology: 'Python Werkzeug/Flask' },
    { header: 'Server', pattern: 'Phusion Passenger', technology: 'Phusion Passenger' },
    { header: 'Server', pattern: 'Tengine', technology: 'Tengine (Alibaba)' },
    { header: 'Server', pattern: 'openresty', technology: 'OpenResty (nginx + Lua)' },
    { header: 'Server', pattern: 'AmazonS3', technology: 'Amazon S3' },
    { header: 'Server', pattern: 'CloudFront', technology: 'Amazon CloudFront' },
    { header: 'Server', pattern: 'gws', technology: 'Google Web Server' },
    { header: 'Server', pattern: 'ESF', technology: 'Google ESF' },
    { header: 'Server', pattern: 'Varnish', technology: 'Varnish Cache' },
    { header: 'X-Powered-By', pattern: '*', technology: 'Technology stack disclosure' },
    { header: 'X-AspNet-Version', pattern: '*', technology: 'ASP.NET version disclosure' },
    { header: 'X-AspNetMvc-Version', pattern: '*', technology: 'ASP.NET MVC version disclosure' },
    { header: 'X-Drupal-Cache', pattern: '*', technology: 'Drupal CMS' },
    { header: 'X-Drupal-Dynamic-Cache', pattern: '*', technology: 'Drupal CMS' },
    { header: 'X-Generator', pattern: '*', technology: 'Site generator disclosure' },
    { header: 'X-Litespeed-Cache', pattern: '*', technology: 'LiteSpeed Cache' },
    { header: 'X-Mod-Pagespeed', pattern: '*', technology: 'Google PageSpeed Module' },
    { header: 'X-Page-Speed', pattern: '*', technology: 'Google PageSpeed' },
    { header: 'X-Turbo-Charged-By', pattern: '*', technology: 'LiteSpeed Web Server' },
    { header: 'X-Varnish', pattern: '*', technology: 'Varnish Cache' },
    { header: 'X-OWA-Version', pattern: '*', technology: 'Microsoft Outlook Web Access' },
    { header: 'X-SharePointHealthScore', pattern: '*', technology: 'Microsoft SharePoint' },
    { header: 'SPRequestGuid', pattern: '*', technology: 'Microsoft SharePoint' },
    { header: 'MicrosoftSharePointTeamServices', pattern: '*', technology: 'Microsoft SharePoint' },
    { header: 'X-MS-InvokeApp', pattern: '*', technology: 'Microsoft service' },
    { header: 'X-FEServer', pattern: '*', technology: 'Microsoft Exchange' },
    { header: 'X-DiagInfo', pattern: '*', technology: 'Microsoft Exchange' },
    { header: 'X-Cocoon-Version', pattern: '*', technology: 'Apache Cocoon' },
    { header: 'X-Content-Powered-By', pattern: '*', technology: 'CMS disclosure' },
    { header: 'X-Envoy-Upstream-Service-Time', pattern: '*', technology: 'Envoy Proxy' },
    { header: 'X-Runtime', pattern: '*', technology: 'Ruby/Rails runtime disclosure' },
    { header: 'X-Rack-Cache', pattern: '*', technology: 'Ruby Rack Cache' },
    { header: 'X-Request-Id', pattern: '*', technology: 'Application framework disclosure' },
    { header: 'X-Shopify-Stage', pattern: '*', technology: 'Shopify' },
    { header: 'X-ShopId', pattern: '*', technology: 'Shopify' },
    { header: 'X-StorageUsed', pattern: '*', technology: 'Shopify' },
    { header: 'CF-RAY', pattern: '*', technology: 'Cloudflare CDN' },
    { header: 'CF-Cache-Status', pattern: '*', technology: 'Cloudflare CDN' },
    { header: 'CF-Request-ID', pattern: '*', technology: 'Cloudflare CDN' },
    { header: 'X-Sucuri-ID', pattern: '*', technology: 'Sucuri WAF' },
    { header: 'X-Sucuri-Cache', pattern: '*', technology: 'Sucuri CDN' },
    { header: 'X-Iinfo', pattern: '*', technology: 'Incapsula/Imperva WAF' },
    { header: 'X-CDN', pattern: '*', technology: 'CDN disclosure' },
    { header: 'X-CDN-Pop', pattern: '*', technology: 'CDN Point of Presence disclosure' },
    { header: 'X-Served-By', pattern: '*', technology: 'Server/CDN node disclosure' },
    { header: 'X-Cache', pattern: '*', technology: 'Cache layer disclosure' },
    { header: 'X-Cache-Hits', pattern: '*', technology: 'Cache layer disclosure' },
    { header: 'X-Timer', pattern: '*', technology: 'Fastly CDN' },
    { header: 'Fastly-Debug-Digest', pattern: '*', technology: 'Fastly CDN' },
    { header: 'X-Fastly-Request-ID', pattern: '*', technology: 'Fastly CDN' },
    { header: 'X-Akamai-Transformed', pattern: '*', technology: 'Akamai CDN' },
    { header: 'X-Azure-Ref', pattern: '*', technology: 'Microsoft Azure' },
    { header: 'X-MSEdge-Ref', pattern: '*', technology: 'Microsoft Azure CDN' },
    { header: 'X-Amz-Cf-Id', pattern: '*', technology: 'Amazon CloudFront' },
    { header: 'X-Amz-Cf-Pop', pattern: '*', technology: 'Amazon CloudFront' },
    { header: 'X-Amz-Request-Id', pattern: '*', technology: 'Amazon Web Services' },
    { header: 'X-Amz-Id-2', pattern: '*', technology: 'Amazon S3' },
    { header: 'X-Served-By-Varnish', pattern: '*', technology: 'Varnish Cache' },
    { header: 'Via', pattern: 'varnish', technology: 'Varnish Cache' },
    { header: 'Via', pattern: 'CloudFront', technology: 'Amazon CloudFront' },
    { header: 'Via', pattern: 'vegur', technology: 'Heroku' },
    { header: 'X-Heroku-Queue-Depth', pattern: '*', technology: 'Heroku' },
    { header: 'X-Heroku-Queue-Wait-Time', pattern: '*', technology: 'Heroku' },
    { header: 'X-Vercel-Cache', pattern: '*', technology: 'Vercel' },
    { header: 'X-Vercel-Id', pattern: '*', technology: 'Vercel' },
    { header: 'X-Netlify-Request-Id', pattern: '*', technology: 'Netlify' },
    { header: 'X-NF-Request-Id', pattern: '*', technology: 'Netlify' },
    { header: 'X-GitHub-Request-Id', pattern: '*', technology: 'GitHub' },
    { header: 'X-Served-By', pattern: 'cache-', technology: 'Fastly CDN' },
    { header: 'X-WordPress-Cache', pattern: '*', technology: 'WordPress' },
    { header: 'X-Redirect-By', pattern: 'WordPress', technology: 'WordPress' },
    { header: 'X-Pingback', pattern: '*', technology: 'WordPress' },
    { header: 'X-Wix-Request-Id', pattern: '*', technology: 'Wix' },
    { header: 'X-Cloud-Trace-Context', pattern: '*', technology: 'Google Cloud Platform' },
    { header: 'X-GCP-Route', pattern: '*', technology: 'Google Cloud Platform' },
    { header: 'X-Appengine-Resource-Usage', pattern: '*', technology: 'Google App Engine' },
    { header: 'X-Cloud-Functions', pattern: '*', technology: 'Google Cloud Functions' },
    { header: 'X-Firebase-Auth', pattern: '*', technology: 'Google Firebase' },
    { header: 'X-Debug-Token', pattern: '*', technology: 'Symfony PHP framework' },
    { header: 'X-Symfony-Cache', pattern: '*', technology: 'Symfony PHP framework' },
    { header: 'X-Laravel-Cache', pattern: '*', technology: 'Laravel PHP framework' },
    { header: 'X-Django-Cache', pattern: '*', technology: 'Django Python framework' },
    { header: 'X-Craft-CSRF', pattern: '*', technology: 'Craft CMS' },
    { header: 'X-Magento-Cache-Control', pattern: '*', technology: 'Magento eCommerce' },
    { header: 'X-Magento-Cache-Debug', pattern: '*', technology: 'Magento eCommerce' },
    { header: 'X-Joomla-Token', pattern: '*', technology: 'Joomla CMS' },
    { header: 'Liferay-Portal', pattern: '*', technology: 'Liferay Portal' },
    { header: 'X-Confluence-Request-Time', pattern: '*', technology: 'Atlassian Confluence' },
    { header: 'X-JIRA-Request-Time', pattern: '*', technology: 'Atlassian JIRA' },
    { header: 'X-Umbraco-Version', pattern: '*', technology: 'Umbraco CMS' },
    { header: 'X-Kentico-Version', pattern: '*', technology: 'Kentico CMS' },
    { header: 'X-EPiServer-PageId', pattern: '*', technology: 'Episerver CMS' },
    { header: 'X-Sitecore-Cluster', pattern: '*', technology: 'Sitecore CMS' }
  ],

  // -- Security: known security-relevant headers --
  security: [
    'accept-ch', 'accept-patch', 'accept-ranges', 'access-control-allow-credentials',
    'access-control-allow-headers', 'access-control-allow-methods', 'access-control-allow-origin',
    'access-control-expose-headers', 'access-control-max-age', 'cache-control',
    'clear-site-data', 'content-disposition', 'content-security-policy',
    'content-security-policy-report-only', 'content-type', 'cross-origin-embedder-policy',
    'cross-origin-opener-policy', 'cross-origin-resource-policy', 'document-policy',
    'expect-ct', 'feature-policy', 'integrity-policy', 'nel',
    'no-vary-search', 'origin-agent-cluster', 'permissions-policy',
    'permissions-policy-report-only', 'pragma', 'public-key-pins',
    'public-key-pins-report-only', 'referrer-policy', 'report-to',
    'reporting-endpoints', 'sec-fetch-dest', 'sec-fetch-mode', 'sec-fetch-site',
    'sec-fetch-user', 'sec-websocket-accept', 'server-timing',
    'set-cookie', 'strict-transport-security', 'supports-loading-mode',
    'timing-allow-origin', 'tk', 'vary', 'warning',
    'www-authenticate', 'x-content-security-policy', 'x-content-type-options',
    'x-dns-prefetch-control', 'x-download-options', 'x-frame-options',
    'x-pad', 'x-permitted-cross-domain-policies', 'x-ua-compatible',
    'x-webkit-csp', 'x-xss-protection'
  ],

  // -- Value checks: per-header validation rules --
  valueChecks: {
    'strict-transport-security': {
      checks: [
        {
          test: function(value) {
            var match = value.match(/max-age\s*=\s*(\d+)/i);
            return match && parseInt(match[1]) >= 31536000;
          },
          issue: 'max-age should be at least 31536000 (1 year)',
          recommended: 'max-age=31536000; includeSubDomains'
        },
        {
          test: function(value) {
            return /includeSubDomains/i.test(value);
          },
          issue: 'Should include includeSubDomains directive',
          recommended: 'max-age=31536000; includeSubDomains'
        }
      ]
    },
    'referrer-policy': {
      checks: [
        {
          test: function(value) {
            var insecure = ['unsafe-url', 'no-referrer-when-downgrade', ''];
            return insecure.indexOf(value.toLowerCase().trim()) === -1;
          },
          issue: 'Insecure referrer policy; may leak full URL to external sites',
          recommended: 'strict-origin-when-cross-origin'
        }
      ]
    },
    'x-frame-options': {
      checks: [
        {
          test: function(value) {
            var valid = ['deny', 'sameorigin'];
            return valid.indexOf(value.toLowerCase().trim()) >= 0;
          },
          issue: 'Invalid or deprecated value; ALLOW-FROM is deprecated',
          recommended: 'DENY'
        }
      ]
    },
    'x-content-type-options': {
      checks: [
        {
          test: function(value) {
            return value.toLowerCase().trim() === 'nosniff';
          },
          issue: 'Must be set to "nosniff"',
          recommended: 'nosniff'
        }
      ]
    },
    'permissions-policy': {
      checks: [
        {
          test: function(value) {
            var deprecated = ['speaker', 'vibrate', 'vr'];
            var found = [];
            deprecated.forEach(function(feat) {
              if (value.toLowerCase().indexOf(feat) >= 0) found.push(feat);
            });
            return found.length === 0;
          },
          issue: 'Contains deprecated feature directives',
          recommended: 'Remove deprecated features (speaker, vibrate, vr)'
        },
        {
          test: function(value) {
            return value.indexOf('=*') === -1;
          },
          issue: 'Overly permissive; using wildcard (*) grants access to all origins',
          recommended: 'Restrict features to specific origins or self'
        }
      ]
    }
  }
};
```

- [ ] **Step 2: Verify in browser**

Open `index.html`. Verify the app still loads -- open browser console, type `HEADER_RULES.missing.length` and confirm it returns `15`. Type `HEADER_RULES.fingerprint.length` and confirm it returns a number > 90. Type `auditEntries` and confirm it's an empty array.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Header Audit rules knowledge base (missing, deprecated, fingerprint, value checks)"
```

---

## Task 4: Add Header Audit Sidebar Entry and Router

**Files:**
- Modify: `index.html:3099-3104` (buildSidebar -- add Header Audit nav item)
- Modify: `index.html:3172-3187` (renderMainContent -- add audit case)

- [ ] **Step 1: Add Header Audit sidebar entry in `buildSidebar()`**

Find the Rescan History sidebar block (lines 3099-3102) and add the Header Audit entry after it:

Replace:
```js
  // Rescan History
  html += '<div class="nav-group-label">History</div>';
  html += '<div class="nav-item" data-type="rescan" data-key="rescan">' +
    '<span class="nav-item-label">Rescan History</span></div>';

  nav.innerHTML = html;
```

With:
```js
  // Rescan History
  html += '<div class="nav-group-label">History</div>';
  html += '<div class="nav-item" data-type="rescan" data-key="rescan">' +
    '<span class="nav-item-label">Rescan History</span></div>';

  // Header Audit
  html += '<div class="nav-group-label">Audit</div>';
  html += '<div class="nav-item" data-type="audit" data-key="audit">' +
    '<span class="nav-item-label">Header Audit</span></div>';

  nav.innerHTML = html;
```

- [ ] **Step 2: Add audit case in `renderMainContent()`**

Find the switch statement in `renderMainContent()` (line 3172) and add the audit case:

Replace:
```js
    case 'rescan':
      renderRescanSection(main);
      break;
    default:
      main.innerHTML = renderEmptyState('Unknown section type');
```

With:
```js
    case 'rescan':
      renderRescanSection(main);
      break;
    case 'audit':
      renderAuditSection(main);
      break;
    default:
      main.innerHTML = renderEmptyState('Unknown section type');
```

- [ ] **Step 3: Add stub `renderAuditSection()` function**

Insert before the `init()` function (before the line with `function init() {`), with a section marker:

```js
/* ================================================================
   HEADER AUDIT: UI RENDERER
   ================================================================ */

function renderAuditSection(container) {
  container.innerHTML = '<div class="audit-container">' +
    '<div class="section-header"><h2>Header Audit</h2>' +
    '<p>Analyze HTTP response headers for security issues</p></div>' +
    renderEmptyState('Paste headers below to analyze', 'Supports curl -I output, raw headers, and nmap http-security-headers') +
    '</div>';
}
```

- [ ] **Step 4: Verify in browser**

Open `index.html`. Verify "Header Audit" appears in the sidebar under an "Audit" label. Click it -- should show the stub empty state with "Paste headers below to analyze".

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add Header Audit sidebar entry and content router"
```

---

## Task 5: Add Header Audit Input Parsing

**Files:**
- Modify: `index.html` -- insert after the existing parsers section (~line 4090), before the comparison engine

This adds `detectHeaderInputType()`, `parseRawHeaders()`, and `normalizeHeaderInput()`.

- [ ] **Step 1: Add the parser functions**

Insert after `parseCurlHeaders()` (after ~line 4089), before the comparison engine section marker (~line 4091):

```js
/* ================================================================
   HEADER AUDIT: PARSER
   ================================================================ */

function detectHeaderInputType(raw) {
  var trimmed = raw.trim();
  if (/\|\s*http-security-headers:/i.test(trimmed)) return 'nmap-script';
  if (/^HTTP\/[12]/m.test(trimmed)) return 'curl-headers';
  if (/^[\w-]+\s*:/m.test(trimmed)) return 'raw-headers';
  return 'raw-headers';
}

function parseRawHeaders(raw) {
  var headers = {};
  var lines = raw.trim().split('\n');
  for (var i = 0; i < lines.length; i++) {
    var line = lines[i].trim();
    if (!line) continue;
    var colonIdx = line.indexOf(':');
    if (colonIdx > 0) {
      var key = line.substring(0, colonIdx).trim();
      var value = line.substring(colonIdx + 1).trim();
      headers[key.toLowerCase()] = { original: key, value: value };
    }
  }
  return { headers: headers, meta: { statusCode: null, protocol: null, source: 'raw-headers' } };
}

function normalizeHeaderInput(raw) {
  var inputType = detectHeaderInputType(raw);
  var result = { headers: {}, meta: { statusCode: null, protocol: null, source: inputType, target: '' } };

  if (inputType === 'curl-headers') {
    var parsed = parseCurlHeaders(raw);
    result.meta.statusCode = parsed.statusCode;
    result.meta.protocol = parsed.protocol;
    result.meta.target = parsed.target || '';
    parsed.headers.forEach(function(h) {
      result.headers[h.key] = { original: h.key, value: h.value };
    });
  } else if (inputType === 'nmap-script') {
    var scriptParsed = parseNmapScript(raw);
    result.meta.target = scriptParsed.target || '';
    // Extract headers from http-security-headers script results
    (scriptParsed.scripts || []).forEach(function(s) {
      if (s.script === 'http-security-headers' && s.data) {
        (s.data.present || []).forEach(function(h) {
          result.headers[h.header.toLowerCase()] = { original: h.header, value: h.value || '' };
        });
      }
    });
  } else {
    var rawParsed = parseRawHeaders(raw);
    result.headers = rawParsed.headers;
  }

  // Auto-detect target from host header if not set
  if (!result.meta.target && result.headers['host']) {
    result.meta.target = result.headers['host'].value;
  }

  return result;
}
```

- [ ] **Step 2: Verify in browser console**

Open `index.html`, open browser console, test:

```js
// Test curl input
var r1 = normalizeHeaderInput('HTTP/2 200\nStrict-Transport-Security: max-age=31536000\nX-Frame-Options: DENY\nServer: nginx/1.18');
console.log(Object.keys(r1.headers).length); // Should be 3
console.log(r1.meta.source); // Should be 'curl-headers'

// Test raw input
var r2 = normalizeHeaderInput('Content-Type: text/html\nX-Powered-By: Express');
console.log(Object.keys(r2.headers).length); // Should be 2
console.log(r2.meta.source); // Should be 'raw-headers'
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Header Audit input parser with curl, raw, and nmap support"
```

---

## Task 6: Add Header Audit Analysis Engine

**Files:**
- Modify: `index.html` -- insert after the parser functions from Task 5

- [ ] **Step 1: Add the analysis engine**

Insert immediately after the parser functions (after `normalizeHeaderInput`):

```js
/* ================================================================
   HEADER AUDIT: ANALYSIS ENGINE
   ================================================================ */

function analyzeHeaders(normalized) {
  var headers = normalized.headers;
  var headerKeysLower = Object.keys(headers);

  var findings = {
    enabled: [],
    missing: [],
    deprecated: [],
    fingerprint: [],
    insecure: [],
    empty: []
  };

  // 1. Enabled security headers
  HEADER_RULES.security.forEach(function(secHeader) {
    if (headers[secHeader]) {
      findings.enabled.push({ header: headers[secHeader].original || secHeader, value: headers[secHeader].value });
    }
  });

  // 2. Missing critical headers
  var hasFrameAncestors = false;
  if (headers['content-security-policy']) {
    hasFrameAncestors = /frame-ancestors/i.test(headers['content-security-policy'].value);
  }

  HEADER_RULES.missing.forEach(function(rule) {
    var key = rule.header.toLowerCase();
    if (!headers[key]) {
      // Skip X-Frame-Options if CSP frame-ancestors is present
      if (key === 'x-frame-options' && hasFrameAncestors) return;
      findings.missing.push({ header: rule.header, description: rule.description, recommended: rule.recommended });
    }
  });

  // 3. Fingerprint headers
  HEADER_RULES.fingerprint.forEach(function(rule) {
    var key = rule.header.toLowerCase();
    if (headers[key]) {
      var value = headers[key].value;
      if (rule.pattern === '*' || value.toLowerCase().indexOf(rule.pattern.toLowerCase()) >= 0) {
        // Avoid duplicate entries for same header+technology
        var alreadyFound = findings.fingerprint.some(function(f) {
          return f.header.toLowerCase() === key && f.technology === rule.technology;
        });
        if (!alreadyFound) {
          findings.fingerprint.push({ header: headers[key].original || rule.header, value: value, technology: rule.technology });
        }
      }
    }
  });

  // 4. Deprecated headers
  HEADER_RULES.deprecated.forEach(function(rule) {
    var key = rule.header.toLowerCase();
    if (headers[key]) {
      findings.deprecated.push({ header: headers[key].original || rule.header, value: headers[key].value, reason: rule.reason });
    }
  });

  // 5. Insecure values
  Object.keys(HEADER_RULES.valueChecks).forEach(function(headerKey) {
    if (headers[headerKey]) {
      var value = headers[headerKey].value;
      var rules = HEADER_RULES.valueChecks[headerKey];
      (rules.checks || []).forEach(function(check) {
        if (!check.test(value)) {
          findings.insecure.push({
            header: headers[headerKey].original || headerKey,
            value: value,
            issue: check.issue,
            recommended: check.recommended
          });
        }
      });
    }
  });

  // 6. Empty headers
  headerKeysLower.forEach(function(key) {
    if (!headers[key].value || !headers[key].value.trim()) {
      findings.empty.push({ header: headers[key].original || key });
    }
  });

  var counts = {
    enabled: findings.enabled.length,
    missing: findings.missing.length,
    deprecated: findings.deprecated.length,
    fingerprint: findings.fingerprint.length,
    insecure: findings.insecure.length,
    empty: findings.empty.length
  };

  return { findings: findings, counts: counts };
}
```

- [ ] **Step 2: Verify in browser console**

```js
var input = normalizeHeaderInput('HTTP/2 200\nStrict-Transport-Security: max-age=300\nX-XSS-Protection: 1; mode=block\nServer: nginx/1.18\nX-Powered-By: Express\nReferrer-Policy: unsafe-url');
var result = analyzeHeaders(input);
console.log('Missing:', result.counts.missing);       // Should be > 0 (many missing headers)
console.log('Deprecated:', result.counts.deprecated);  // Should be 1 (X-XSS-Protection)
console.log('Fingerprint:', result.counts.fingerprint); // Should be >= 2 (Server nginx, X-Powered-By)
console.log('Insecure:', result.counts.insecure);      // Should be >= 2 (HSTS max-age too low, Referrer-Policy unsafe)
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Header Audit analysis engine (missing, deprecated, fingerprint, insecure, empty checks)"
```

---

## Task 7: Build Full Audit Section UI

**Files:**
- Modify: `index.html` -- replace the stub `renderAuditSection()` from Task 4

This is the largest UI task. It replaces the stub with the full input zone, report renderer, timeline, and event handlers.

- [ ] **Step 1: Replace `renderAuditSection()` with the full implementation**

Replace the entire stub `renderAuditSection()` function (and the section marker above it) with:

```js
/* ================================================================
   HEADER AUDIT: UI RENDERER
   ================================================================ */

var currentAuditResult = null;

function renderAuditSection(container) {
  var html = '<div class="audit-container">';
  html += '<div class="section-header"><h2>Header Audit</h2>' +
    '<p>Analyze HTTP response headers for security issues</p></div>';

  if (auditComparing && auditSelected.length === 2) {
    html += renderAuditComparisonView();
  } else {
    // Input zone
    html += '<div class="audit-input-zone">';
    html += '<div class="audit-drop-overlay" id="auditDropZone">' +
      'Drop a .txt file with headers here, or click to browse' +
      '<input type="file" id="auditFileInput" accept=".txt,.text,.json">' +
      '</div>';
    html += '<textarea class="audit-textarea" id="auditInput" placeholder="Paste headers here...\n\nSupported formats:\n  curl -I https://example.com\n  HTTP/2 200\n  Strict-Transport-Security: max-age=31536000\n  ...\n\n  Raw Header: value lines\n  nmap http-security-headers script output">' +
      escapeHtml(currentAuditResult ? currentAuditResult.rawInput : '') + '</textarea>';
    html += '<div class="audit-actions">';
    html += '<button class="audit-btn-analyze" id="auditAnalyzeBtn">Analyze</button>';
    if (currentAuditResult) {
      html += '<button class="audit-btn-clear" id="auditClearBtn">Clear</button>';
      html += '<button class="audit-btn-save" id="auditSaveBtn">Save Audit</button>';
    }
    html += '</div>';
    html += '</div>';

    // Report
    if (currentAuditResult) {
      html += renderAuditReport(currentAuditResult);
    }

    // Saved audits timeline
    if (auditEntries.length > 0) {
      html += renderAuditTimeline();
    }
  }

  html += '</div>';
  container.innerHTML = html;
  attachAuditHandlers(container);
}

function renderAuditReport(result) {
  var f = result.findings;
  var c = result.counts;
  var html = '<div class="audit-report">';

  // Summary line
  var summaryParts = [];
  if (c.missing > 0) summaryParts.push(c.missing + ' missing');
  if (c.deprecated > 0) summaryParts.push(c.deprecated + ' deprecated');
  if (c.insecure > 0) summaryParts.push(c.insecure + ' insecure');
  if (c.fingerprint > 0) summaryParts.push(c.fingerprint + ' fingerprint');
  if (c.empty > 0) summaryParts.push(c.empty + ' empty');
  summaryParts.push(c.enabled + ' enabled');

  html += '<div style="margin-bottom:14px;font-size:0.82rem;color:var(--text-secondary);">' +
    (result.meta && result.meta.target ? '<strong style="color:var(--accent-cyan);">' + escapeHtml(result.meta.target) + '</strong> &mdash; ' : '') +
    summaryParts.join(' &middot; ') + '</div>';

  // Sections in severity order
  html += renderAuditFindingSection('Missing Security Headers', f.missing, c.missing, 'red', function(item) {
    return '<div class="audit-finding">' +
      '<div class="audit-finding-header">' + escapeHtml(item.header) + '</div>' +
      '<div class="audit-finding-detail">' + escapeHtml(item.description) + '</div>' +
      '<div class="audit-finding-recommended">Recommended: ' +
      '<div class="code-block-wrapper"><pre class="code-block">' + escapeHtml(item.header) + ': ' + escapeHtml(item.recommended) + '</pre>' +
      '<button class="copy-btn" data-copy="' + escapeHtml(item.header + ': ' + item.recommended) + '">Copy</button></div></div></div>';
  });

  html += renderAuditFindingSection('Deprecated Headers', f.deprecated, c.deprecated, 'yellow', function(item) {
    return '<div class="audit-finding">' +
      '<div class="audit-finding-header">' + escapeHtml(item.header) + '</div>' +
      '<div class="audit-finding-value">Current: ' + escapeHtml(item.value) + '</div>' +
      '<div class="audit-finding-detail">' + escapeHtml(item.reason) + '</div></div>';
  });

  html += renderAuditFindingSection('Insecure Values', f.insecure, c.insecure, 'orange', function(item) {
    return '<div class="audit-finding">' +
      '<div class="audit-finding-header">' + escapeHtml(item.header) + '</div>' +
      '<div class="audit-finding-value">Current: ' + escapeHtml(item.value) + '</div>' +
      '<div class="audit-finding-detail">' + escapeHtml(item.issue) + '</div>' +
      '<div class="audit-finding-recommended">Recommended: ' +
      '<div class="code-block-wrapper"><pre class="code-block">' + escapeHtml(item.header) + ': ' + escapeHtml(item.recommended) + '</pre>' +
      '<button class="copy-btn" data-copy="' + escapeHtml(item.header + ': ' + item.recommended) + '">Copy</button></div></div></div>';
  });

  html += renderAuditFindingSection('Fingerprint Headers', f.fingerprint, c.fingerprint, 'yellow', function(item) {
    return '<div class="audit-finding">' +
      '<div class="audit-finding-header">' + escapeHtml(item.header) + '</div>' +
      '<div class="audit-finding-value">' + escapeHtml(item.value) + '</div>' +
      '<div class="audit-finding-detail">Leaks: ' + escapeHtml(item.technology) + '</div></div>';
  });

  html += renderAuditFindingSection('Empty Headers', f.empty, c.empty, 'gray', function(item) {
    return '<div class="audit-finding">' +
      '<div class="audit-finding-header">' + escapeHtml(item.header) + '</div>' +
      '<div class="audit-finding-detail">Header present but value is empty</div></div>';
  });

  html += renderAuditFindingSection('Enabled Security Headers', f.enabled, c.enabled, 'green', function(item) {
    return '<div class="audit-finding">' +
      '<div class="audit-finding-header">' + escapeHtml(item.header) + '</div>' +
      '<div class="audit-finding-value">' + escapeHtml(item.value) + '</div></div>';
  });

  html += '</div>';
  return html;
}

function renderAuditFindingSection(title, items, count, colorClass, renderItem) {
  var expanded = count > 0 && colorClass !== 'green';
  var html = '<div class="audit-section' + (expanded ? ' expanded' : '') + '">';
  html += '<div class="audit-section-header">' +
    '<span class="audit-section-arrow">&#9654;</span>' +
    '<span class="audit-section-title">' + escapeHtml(title) + '</span>' +
    '<span class="audit-section-count ' + colorClass + '">' + count + '</span>' +
    '</div>';
  html += '<div class="audit-section-body">';
  items.forEach(function(item) {
    html += renderItem(item);
  });
  if (count === 0) {
    html += '<div class="audit-finding"><div class="audit-finding-detail" style="color:var(--accent-green);">None found</div></div>';
  }
  html += '</div></div>';
  return html;
}

function renderAuditTimeline() {
  var html = '<div class="audit-timeline">';
  html += '<h3 style="color:var(--accent-cyan);font-size:0.9rem;margin:20px 0 10px;padding-bottom:6px;border-bottom:1px solid var(--card-border);">Saved Audits</h3>';

  // Group by target
  var groups = {};
  auditEntries.forEach(function(e) {
    var target = e.target || 'Unknown';
    if (!groups[target]) groups[target] = [];
    groups[target].push(e);
  });

  Object.keys(groups).forEach(function(target) {
    var audits = groups[target];
    audits.sort(function(a, b) { return new Date(a.timestamp) - new Date(b.timestamp); });
    html += '<div class="target-group"><div class="target-group-header">' + escapeHtml(target) + '</div>';
    audits.forEach(function(audit) {
      var selected = auditSelected.indexOf(audit.id) >= 0;
      var summary = audit.counts.missing + 'M ' + audit.counts.deprecated + 'D ' +
        audit.counts.insecure + 'I ' + audit.counts.fingerprint + 'F';
      html += '<div class="audit-entry' + (selected ? ' selected' : '') + '" data-id="' + escapeHtml(audit.id) + '">' +
        '<div class="audit-entry-select"></div>' +
        '<span class="audit-entry-title">' + escapeHtml(audit.title) + '</span>' +
        '<span class="audit-entry-summary">' + summary + '</span>' +
        '<span class="audit-entry-date">' + formatDate(audit.timestamp) + '</span>' +
        '</div>';
    });
    html += '</div>';
  });

  // Compare button
  html += '<button class="audit-btn-compare' + (auditSelected.length === 2 ? ' visible' : '') +
    '" id="auditCompareBtn">Compare Selected (' + auditSelected.length + '/2)</button>';

  html += '</div>';
  return html;
}

function attachAuditHandlers(container) {
  // Analyze button
  var analyzeBtn = container.querySelector('#auditAnalyzeBtn');
  if (analyzeBtn) {
    analyzeBtn.addEventListener('click', function() {
      var raw = container.querySelector('#auditInput').value.trim();
      if (!raw) return;
      var normalized = normalizeHeaderInput(raw);
      var result = analyzeHeaders(normalized);
      currentAuditResult = {
        rawInput: raw,
        inputType: normalized.meta.source,
        meta: normalized.meta,
        findings: result.findings,
        counts: result.counts
      };
      renderAuditSection(container);
    });
  }

  // Clear button
  var clearBtn = container.querySelector('#auditClearBtn');
  if (clearBtn) {
    clearBtn.addEventListener('click', function() {
      currentAuditResult = null;
      renderAuditSection(container);
    });
  }

  // Save button
  var saveBtn = container.querySelector('#auditSaveBtn');
  if (saveBtn) {
    saveBtn.addEventListener('click', function() {
      if (!currentAuditResult) return;
      document.getElementById('auditModalTitle').value = '';
      document.getElementById('auditModalTarget').value = currentAuditResult.meta.target || '';
      document.getElementById('auditSaveModal').classList.add('open');
    });
  }

  // Drop zone
  var dropZone = container.querySelector('#auditDropZone');
  var fileInput = container.querySelector('#auditFileInput');
  if (dropZone && fileInput) {
    dropZone.addEventListener('click', function() { fileInput.click(); });
    dropZone.addEventListener('dragover', function(e) { e.preventDefault(); dropZone.classList.add('dragover'); });
    dropZone.addEventListener('dragleave', function() { dropZone.classList.remove('dragover'); });
    dropZone.addEventListener('drop', function(e) {
      e.preventDefault();
      dropZone.classList.remove('dragover');
      handleAuditFileDrop(e.dataTransfer.files, container);
    });
    fileInput.addEventListener('change', function() { handleAuditFileDrop(fileInput.files, container); });
  }

  // Collapsible sections
  var sectionHeaders = container.querySelectorAll('.audit-section-header');
  for (var i = 0; i < sectionHeaders.length; i++) {
    (function(header) {
      header.addEventListener('click', function() {
        header.parentElement.classList.toggle('expanded');
      });
    })(sectionHeaders[i]);
  }

  // Audit entry selection
  var entries = container.querySelectorAll('.audit-entry');
  for (var j = 0; j < entries.length; j++) {
    (function(entry) {
      entry.addEventListener('click', function() {
        var id = entry.getAttribute('data-id');
        var idx = auditSelected.indexOf(id);
        if (idx >= 0) {
          auditSelected.splice(idx, 1);
        } else if (auditSelected.length < 2) {
          auditSelected.push(id);
        } else {
          auditSelected.shift();
          auditSelected.push(id);
        }
        renderAuditSection(container);
      });
    })(entries[j]);
  }

  // Compare button
  var compareBtn = container.querySelector('#auditCompareBtn');
  if (compareBtn) {
    compareBtn.addEventListener('click', function() {
      if (auditSelected.length === 2) {
        auditComparing = true;
        renderAuditSection(container);
      }
    });
  }

  // Comparison back button
  var compBack = container.querySelector('.comparison-back');
  if (compBack) {
    compBack.addEventListener('click', function() {
      auditComparing = false;
      renderAuditSection(container);
    });
  }

  attachCopyHandlers(container);
}

function handleAuditFileDrop(files, container) {
  var fileArray = Array.prototype.slice.call(files);
  fileArray.forEach(function(file) {
    var reader = new FileReader();
    reader.onload = function(e) {
      var content = e.target.result;
      // Check if it's a JSON audit export
      try {
        var data = JSON.parse(content);
        if (data.auditEntries && Array.isArray(data.auditEntries)) {
          data.auditEntries.forEach(function(entry) {
            var exists = auditEntries.some(function(ex) { return ex.id === entry.id; });
            if (!exists) auditEntries.push(entry);
          });
          renderAuditSection(container);
          return;
        }
      } catch(err) { /* not JSON, treat as header text */ }
      // Set as textarea content
      var textarea = container.querySelector('#auditInput');
      if (textarea) {
        textarea.value = content;
      }
    };
    reader.readAsText(file);
  });
}
```

- [ ] **Step 2: Verify in browser**

Open `index.html`. Click "Header Audit" in sidebar. Paste the following into the textarea:

```
HTTP/2 200
Strict-Transport-Security: max-age=300
X-XSS-Protection: 1; mode=block
Server: nginx/1.18
X-Powered-By: Express
Referrer-Policy: unsafe-url
X-Content-Type-Options: nosniff
```

Click "Analyze". Verify:
- Missing section shows multiple missing headers (red, expanded)
- Deprecated section shows X-XSS-Protection (yellow, expanded)
- Insecure Values section shows HSTS max-age too low + Referrer-Policy unsafe (orange, expanded)
- Fingerprint section shows Server nginx + X-Powered-By Express (yellow, expanded)
- Enabled section shows the present security headers (green, collapsed)
- Sections are collapsible (click headers to toggle)
- Copy buttons work on recommended values
- Clear button resets the view
- Save button opens the modal (will wire up in next task)

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Header Audit UI with report renderer, input handling, and timeline"
```

---

## Task 8: Wire Up Audit Save Modal and JSON Export

**Files:**
- Modify: `index.html` -- add event listeners for the audit save modal (after the existing classify modal handlers)

- [ ] **Step 1: Add audit modal event handlers**

Insert after the existing classify modal `modalSave` handler block (after the line `});` that closes the `modalSave` click handler, around line 4442), before the next section comment:

```js
/* ================================================================
   AUDIT SAVE MODAL
   ================================================================ */

document.getElementById('auditModalCancel').addEventListener('click', function() {
  document.getElementById('auditSaveModal').classList.remove('open');
});

document.getElementById('auditModalSave').addEventListener('click', function() {
  var title = document.getElementById('auditModalTitle').value.trim();
  if (!title) {
    document.getElementById('auditModalTitle').style.borderColor = 'var(--accent-pink)';
    return;
  }

  var target = document.getElementById('auditModalTarget').value.trim();
  var now = new Date().toISOString();

  var entry = {
    id: generateId(),
    title: title,
    target: target || (currentAuditResult && currentAuditResult.meta ? currentAuditResult.meta.target : '') || 'unknown',
    timestamp: now,
    rawInput: currentAuditResult ? currentAuditResult.rawInput : '',
    inputType: currentAuditResult ? currentAuditResult.inputType : '',
    findings: currentAuditResult ? currentAuditResult.findings : {},
    counts: currentAuditResult ? currentAuditResult.counts : {}
  };

  auditEntries.push(entry);

  // Download JSON file
  var jsonData = { version: 1, auditEntries: [entry] };
  var blob = new Blob([JSON.stringify(jsonData, null, 2)], { type: 'application/json' });
  var url = URL.createObjectURL(blob);
  var a = document.createElement('a');
  var dateStr = now.slice(0, 10);
  var fileSlug = slugify('audit-' + (target || 'scan') + '-' + title);
  a.href = url;
  a.download = dateStr + '_' + fileSlug + '.json';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);

  document.getElementById('auditSaveModal').classList.remove('open');

  if (activeSection && activeSection.type === 'audit') {
    renderAuditSection(document.getElementById('mainContent'));
  }
});
```

- [ ] **Step 2: Verify in browser**

1. Go to Header Audit, paste headers, click Analyze
2. Click "Save Audit" -- modal should open
3. Enter a title, click "Save & Export"
4. A JSON file should download
5. The saved audit should appear in the timeline below the report
6. Verify the downloaded JSON contains the audit data with `version: 1` and `auditEntries` array

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add audit save modal with JSON export"
```

---

## Task 9: Add Audit Comparison View

**Files:**
- Modify: `index.html` -- add `renderAuditComparisonView()` and `compareAuditCategory()` in the audit UI section

- [ ] **Step 1: Add the comparison view functions**

Insert after the `handleAuditFileDrop()` function, before the audit save modal section:

```js
function renderAuditComparisonView() {
  var older = null;
  var newer = null;
  auditEntries.forEach(function(e) {
    if (e.id === auditSelected[0]) older = e;
    if (e.id === auditSelected[1]) newer = e;
  });
  if (!older || !newer) return renderEmptyState('Could not find selected audits');

  // Ensure older is actually older
  if (new Date(older.timestamp) > new Date(newer.timestamp)) {
    var tmp = older; older = newer; newer = tmp;
  }

  var html = '<div class="comparison-back">&larr; Back to Audit</div>';
  html += '<div class="comparison-header">' +
    '<div class="comparison-scan-info">Older: <strong>' + escapeHtml(older.title) + '</strong> (' + formatDate(older.timestamp) + ')</div>' +
    '<div class="comparison-scan-info">Newer: <strong>' + escapeHtml(newer.title) + '</strong> (' + formatDate(newer.timestamp) + ')</div>' +
    '</div>';

  // Compare missing headers
  html += '<h4 style="color:var(--accent-cyan);margin:12px 0 8px;font-size:0.9rem;">Missing Headers</h4>';
  html += compareAuditCategory(
    older.findings.missing || [], newer.findings.missing || [],
    function(item) { return item.header.toLowerCase(); },
    'header'
  );

  // Compare deprecated headers
  html += '<h4 style="color:var(--accent-cyan);margin:12px 0 8px;font-size:0.9rem;">Deprecated Headers</h4>';
  html += compareAuditCategory(
    older.findings.deprecated || [], newer.findings.deprecated || [],
    function(item) { return item.header.toLowerCase(); },
    'header'
  );

  // Compare insecure values
  html += '<h4 style="color:var(--accent-cyan);margin:12px 0 8px;font-size:0.9rem;">Insecure Values</h4>';
  html += compareAuditCategory(
    older.findings.insecure || [], newer.findings.insecure || [],
    function(item) { return item.header.toLowerCase() + ':' + item.issue; },
    'header'
  );

  // Compare fingerprint headers
  html += '<h4 style="color:var(--accent-cyan);margin:12px 0 8px;font-size:0.9rem;">Fingerprint Headers</h4>';
  html += compareAuditCategory(
    older.findings.fingerprint || [], newer.findings.fingerprint || [],
    function(item) { return item.header.toLowerCase(); },
    'header'
  );

  // Summary counts
  var fixed = 0;
  var regressed = 0;
  var categories = ['missing', 'deprecated', 'insecure', 'fingerprint'];
  categories.forEach(function(cat) {
    var oldCount = older.counts[cat] || 0;
    var newCount = newer.counts[cat] || 0;
    if (newCount < oldCount) fixed += (oldCount - newCount);
    if (newCount > oldCount) regressed += (newCount - oldCount);
  });

  html += '<div class="comparison-summary" style="margin-top:16px;">' +
    (fixed > 0 ? '<span style="color:var(--accent-green);">' + fixed + ' fixed</span> &middot; ' : '') +
    (regressed > 0 ? '<span style="color:var(--accent-pink);">' + regressed + ' regressed</span> &middot; ' : '') +
    'Older: ' + (older.counts.missing + older.counts.deprecated + older.counts.insecure + older.counts.fingerprint) + ' issues &rarr; ' +
    'Newer: ' + (newer.counts.missing + newer.counts.deprecated + newer.counts.insecure + newer.counts.fingerprint) + ' issues' +
    '</div>';

  return html;
}

function compareAuditCategory(olderItems, newerItems, keyFn, labelProp) {
  var olderKeys = {};
  var newerKeys = {};
  olderItems.forEach(function(item) { olderKeys[keyFn(item)] = item; });
  newerItems.forEach(function(item) { newerKeys[keyFn(item)] = item; });

  var allKeys = {};
  Object.keys(olderKeys).forEach(function(k) { allKeys[k] = true; });
  Object.keys(newerKeys).forEach(function(k) { allKeys[k] = true; });

  if (Object.keys(allKeys).length === 0) {
    return '<div style="color:var(--text-muted);font-size:0.8rem;margin-bottom:12px;">None in either audit</div>';
  }

  var rows = '';
  Object.keys(allKeys).forEach(function(key) {
    var inOld = !!olderKeys[key];
    var inNew = !!newerKeys[key];
    var cls, status;
    if (inOld && !inNew) {
      cls = 'diff-fixed';
      status = 'Fixed';
    } else if (!inOld && inNew) {
      cls = 'diff-new-issue';
      status = 'New';
    } else {
      cls = 'diff-same';
      status = 'Same';
    }
    var item = newerKeys[key] || olderKeys[key];
    rows += '<tr class="' + cls + '"><td>' + escapeHtml(item[labelProp]) + '</td>' +
      '<td>' + (inOld ? 'Yes' : '-') + '</td>' +
      '<td>' + (inNew ? 'Yes' : '-') + '</td>' +
      '<td>' + status + '</td></tr>';
  });

  return '<table class="diff-table"><thead><tr><th>Item</th><th>Older</th><th>Newer</th><th>Status</th></tr></thead><tbody>' + rows + '</tbody></table>';
}
```

- [ ] **Step 2: Verify in browser**

1. Go to Header Audit, run two different audits and save both
2. Select both in the timeline (click each card)
3. Click "Compare Selected (2/2)"
4. Verify the comparison view shows:
   - Back link works
   - Side-by-side comparison tables for each category
   - Green rows for fixed issues, red for new, gray for same
   - Summary counts at the bottom
5. Click "Back to Audit" to return to normal view

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Header Audit comparison view with diff tables"
```

---

## Task 10: Search Integration and Final Polish

**Files:**
- Modify: `index.html` -- update search filtering to include audit section

- [ ] **Step 1: Add audit section to the search filter**

Find the search handling code that builds match counts for `updateSidebarMatchCounts`. The search system filters across sections and shows counts. Add a count for the audit section.

Locate the function that handles search (it references `searchQuery` and calls `updateSidebarMatchCounts`). Add this in the section where counts are computed:

```js
// Add audit match counting
counts['audit:audit'] = 0;
auditEntries.forEach(function(e) {
  var q = searchQuery.toLowerCase();
  if ((e.title && e.title.toLowerCase().indexOf(q) >= 0) ||
      (e.target && e.target.toLowerCase().indexOf(q) >= 0)) {
    counts['audit:audit']++;
  }
});
```

- [ ] **Step 2: Verify in browser**

1. Open app, save at least one audit with title "example.com test"
2. Press `/` to search, type "example"
3. Verify the "Header Audit" sidebar item shows a match count badge

- [ ] **Step 3: Full end-to-end verification**

Complete walkthrough:
1. Click "Header Audit" in sidebar
2. Paste `curl -I` output with mixed good/bad headers
3. Click Analyze -- report renders with all 6 sections
4. Verify collapsible sections toggle on click
5. Verify copy buttons work on recommended values
6. Click "Save Audit" -- modal opens, enter title, click "Save & Export"
7. JSON file downloads with correct naming format
8. Saved audit appears in timeline with summary counts
9. Paste different headers, analyze, save second audit
10. Select both audits in timeline
11. Click "Compare Selected" -- comparison view shows diffs
12. Verify green/red/gray color coding in diff tables
13. Click "Back to Audit" -- returns to normal view
14. Drop a previously exported `.json` file -- audit loads into timeline
15. Drop a `.txt` file with raw headers -- content appears in textarea

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: integrate Header Audit with search and complete end-to-end flow"
```
