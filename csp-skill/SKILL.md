---
name: csp-implementation
description: >
  Audit a web application and implement a Content-Security-Policy (CSP) header —
  from diagnosis to enforcement. Use this skill whenever the user mentions CSP,
  Content-Security-Policy, security headers, inline scripts, unsafe-inline,
  script-src, nonces, XSS hardening, or asks to secure their web app headers.
  Also trigger when the user uploads a project and asks to "add CSP", "fix security
  headers", "implement nonces", or "remove unsafe-inline". Supports Laravel/PHP,
  ASP.NET/VB.NET, Angular, Node.js/Express/NestJS/Fastify, Next.js, and nginx/Apache.
  Always use this skill for multi-step CSP workflows — do not attempt them without it.
---

# CSP Implementation Skill

Audit a codebase and implement Content-Security-Policy headers correctly, incrementally,
and without breaking production. Works on any web stack.

---

## Step 0 — Identify the stack

Before anything else, detect the project's stack from uploaded files, file extensions,
or the user's description. This determines which reference file to read next.

| Stack | Reference file |
|---|---|
| Laravel / PHP (Blade) | `references/laravel.md` |
| ASP.NET WebForms / VB.NET | `references/aspnet.md` |
| Angular + nginx | `references/angular-nginx.md` |
| Node.js (Express / NestJS / Fastify) | `references/nodejs.md` |
| Next.js | `references/nextjs.md` |
| Static HTML / Apache / nginx only | `references/static-nginx.md` |

**Read the relevant reference file immediately after identifying the stack.**
If the stack is unclear, ask. Do not guess.

---

## Step 1 — Diagnose the project

Run (or instruct the user to run) the diagnostic commands for their stack.
Collect the results before writing any code.

### Universal diagnostics (apply to all stacks)

```bash
# Inline <script> blocks without src
grep -rn "<script>" resources/views --include="*.blade.php" | grep -v 'src='
# (adjust path/extension for non-Laravel stacks)

# Inline style attributes
grep -rn 'style=' resources/views --include="*.blade.php" | wc -l

# Inline event handlers (onclick, onsubmit, etc.)
grep -rn 'on[a-z]*=' resources/views --include="*.blade.php"

# Dangerous JS patterns in own code (not vendor)
grep -rn "eval\|new Function\|document\.write" public/js --include="*.js"

# innerHTML in own JS files
grep -rn "innerHTML" public/js --include="*.js"
```

### What to look for

Build a findings table with these categories:

| Category | Count | Files | Criticality |
|---|---|---|---|
| Inline `<script>` blocks | N | ... | HIGH |
| Inline `style=` attributes | N | ... | MEDIUM |
| Inline event handlers (`onclick` etc.) | N | ... | HIGH |
| `eval` / `new Function` in own code | N | ... | HIGH |
| `innerHTML` in own JS | N | ... | MEDIUM |
| External CDN domains | N | list | HIGH |
| CSP header present | yes/no | — | CRITICAL |

---

## Step 2 — Classify inline content

For each inline script found, classify it:

- **Dynamic** (contains server-side variables like `{{ locale }}`, `{{ userId }}`) → needs **nonce**
- **Static** (pure JS, no server data) → can be **extracted to external file** OR use **hash**
- **Event handler** (`onclick=`, `onsubmit=`) → must be **removed and replaced with `addEventListener`**; nonces and hashes do NOT work on event handlers

For external resources, map every domain to its CSP directive:

| Domain | Directive |
|---|---|
| `cdn.jsdelivr.net` | `script-src` + `style-src` |
| `fonts.googleapis.com` | `style-src` |
| `fonts.gstatic.com` | `font-src` |
| `www.googletagmanager.com` | `script-src` |
| `www.google-analytics.com` | `script-src` + `connect-src` |

---

## Step 3 — Build the policy incrementally

**Never go straight to enforcement.** Always start with Report-Only.

### Phase 1 — Report-Only permissive (Week 1)

Goal: visibility. Nothing is blocked. Deploy to production and collect real violations.

```
Content-Security-Policy-Report-Only:
  default-src 'self';
  script-src 'self' 'unsafe-inline' 'unsafe-eval' [ALL_CDN_DOMAINS];
  style-src 'self' 'unsafe-inline' [STYLE_CDN_DOMAINS];
  font-src 'self' [FONT_CDN_DOMAINS];
  img-src 'self' data: [IMG_DOMAINS];
  connect-src 'self' [API_DOMAINS];
  frame-src [FRAME_DOMAINS];
  object-src 'none';
  base-uri 'self';
  report-uri /csp-report
```

### Phase 2 — Report-Only with nonces (Week 2–3)

Goal: nonces implemented, `unsafe-eval` removed, `unsafe-inline` removed from `script-src`.

```
Content-Security-Policy-Report-Only:
  script-src 'nonce-{NONCE}' 'strict-dynamic' [CDN_DOMAINS];
  style-src 'self' 'unsafe-inline' [STYLE_CDNS];   ← unsafe-inline stays until styles extracted
  ...
  report-uri /csp-report
```

### Phase 3 — Enforcement script-src (Week 4–5)

Goal: block unauthorized scripts. Style-src stays permissive.

```
Content-Security-Policy:
  script-src 'nonce-{NONCE}' 'strict-dynamic';
  style-src 'self' 'unsafe-inline' [STYLE_CDNS];   ← still permissive
  ...
```

### Phase 4 — Full enforcement (Week 6–8)

Goal: eliminate all `unsafe-inline`. Requires 171+ inline styles extracted.

```
Content-Security-Policy:
  script-src 'nonce-{NONCE}' 'strict-dynamic';
  style-src-elem 'nonce-{NONCE}' 'self' [STYLE_CDNS];
  style-src-attr 'unsafe-inline';   ← Bootstrap/DataTables inject this, keep temporarily
  ...
```

> **Key rule**: `style-src-elem` controls `<link>` and `<style>` tags.
> `style-src-attr` controls `style=""` attributes. Splitting them is more surgical than a single `style-src`.

---

## Step 4 — Migrate inline content

### Inline scripts → external files

Extract to `public/js/[name].js` and load with `<script nonce="{{ $cspNonce }}" src="...">`.

**Exception**: Scripts with dynamic server data (e.g., `var locale = '{{ locale }}'`)
→ Use a `data-attribute` on `<html>` instead: `<html data-locale="{{ locale }}">`, then read in JS:
```js
const locale = document.documentElement.dataset.locale;
```

### Event handlers → addEventListener

```html
<!-- BEFORE (violates CSP) -->
<a onclick="irParaTopo()">Top</a>

<!-- AFTER (CSP-compliant) -->
<a id="scroll-to-top">Top</a>
```
```js
// In external JS file loaded with nonce
document.getElementById('scroll-to-top')?.addEventListener('click', function(e) {
    e.preventDefault();
    irParaTopo();
});
```

**Critical**: Test logout, form submit, and any navigation handler before deploying to production.

### Email templates — permanent exception

Mail templates (Blade, Razor, Jinja, etc.) **must keep inline styles**.
Email clients (Outlook, Gmail) do not load external CSS. Document this as an intentional exception.

---

## Step 5 — Implement the report endpoint

Every stack needs a `/csp-report` endpoint that:
1. Accepts `POST` with `Content-Type: application/csp-report`
2. Parses the JSON body (the report is in `body['csp-report']` or directly in `body`)
3. Logs: `violated-directive`, `blocked-uri`, `document-uri`, `source-file`, `line-number`
4. Returns `204 No Content`
5. **Excludes CSRF validation** — browsers don't send CSRF tokens in CSP reports

Monitor with: `tail -f storage/logs/csp-violations.log`

---

## Step 6 — Nonce implementation by stack

Read the stack-specific reference file for exact implementation code.
All stacks follow the same pattern:

1. **Generate** nonce per request: `base64(cryptographic_random_bytes(16))` — minimum 128 bits
2. **Share** nonce with templates globally (middleware layer, not controller)
3. **Apply** to every inline `<script nonce="...">` and `<style nonce="...">`
4. **Include** in the CSP header: `script-src 'nonce-{VALUE}'`

Nonces must be **different on every HTTP request**. Never reuse or cache.

---

## Step 7 — Rollback strategy

Always implement a killswitch before deploying:

```
# .env / environment variable
CSP_ENABLED=false    → removes the header entirely, zero impact on users
CSP_ENFORCE=false    → switches to Report-Only without code deploy
```

**Commit discipline**: One atomic commit per file changed. Never mix CSP middleware
changes with inline script extractions. This allows `git revert` on individual steps.

---

## Output format

Deliver all generated files with:
- Exact destination path (e.g., `app/Http/Middleware/CspMiddleware.php`)
- Inline comments explaining every non-obvious decision
- A summary table of what was changed and why

---

## Edge cases and gotchas

- **Browser extensions** inject scripts that generate false CSP violations — filter `chrome-extension://` and `moz-extension://` from the report endpoint log
- **`strict-dynamic`** ignores host allowlists in modern browsers — CDN domains in `script-src` are for older browser fallback only
- **Bootstrap / DataTables** inject `style=""` attributes dynamically via JS — `style-src-attr 'unsafe-inline'` is required until those libraries are replaced
- **`document.write`** in vendor/minified libraries — not fixable, but `unsafe-eval` is not required for it; it's blocked separately by `script-src`
- **IE11** does not support nonces — if IE11 users exist, nonce-based CSP will silently fail for them (all scripts blocked)
- **SRI (Subresource Integrity)**: Add `integrity="sha384-..."` to CDN script tags in hardening phase — this allows removing CDN domains from `script-src` allowlist

---

## References

- `references/laravel.md` — Laravel 5.x–11.x middleware, Blade nonce injection, csp-report route
- `references/aspnet.md` — VB.NET/C# IHttpModule, WebForms ScriptManager nonce, HttpResponse headers
- `references/angular-nginx.md` — Angular CSP meta tag, nginx `add_header`, nonce via SSR or index.html transform
- `references/nodejs.md` — Express/NestJS/Fastify helmet integration, middleware nonce pattern
- `references/nextjs.md` — Next.js middleware, `next.config.js` headers, App Router vs Pages Router
- `references/static-nginx.md` — nginx `add_header` only, hash-based CSP for static sites

