# CSP — Angular + nginx

Angular apps are static SPA bundles served by nginx (or CloudFront/S3).
CSP is applied at the **server level**, not in the Angular app itself.

---

## Option A — nginx `add_header` (no nonce, hash-based)

For SPAs without SSR, nonces are not feasible (the server doesn't render HTML per-request).
Use hashes for any unavoidable inline scripts, and eliminate all inline scripts from `index.html`.

```nginx
# /etc/nginx/conf.d/app.conf

server {
    listen 443 ssl;
    server_name app.example.com;

    root /var/www/app/dist;
    index index.html;

    # Phase 1 — Report-Only (safe to deploy immediately)
    add_header Content-Security-Policy-Report-Only
        "default-src 'self'; script-src 'self' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data:; connect-src 'self' https://api.example.com; object-src 'none'; base-uri 'self'; report-uri /csp-report"
        always;

    # Phase 4 — Enforcement (replace the above when ready)
    # add_header Content-Security-Policy
    #     "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; ..."
    #     always;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # CSP report endpoint — proxy to your backend
    location /csp-report {
        proxy_pass http://backend:3000/csp-report;
    }
}
```

**Eliminating inline scripts from Angular's index.html**:

Angular CLI sometimes injects inline scripts (e.g., `zone.js` polyfills, runtime chunk).
Check `dist/index.html` after build. If inline scripts exist:

```json
// angular.json — move runtime to external file
"optimization": {
    "scripts": true,
    "styles": { "minify": true, "inlineCritical": false },
    "fonts": false
},
"inlineStyleLanguage": "scss"
```

Or add a hash for each remaining inline script:

```bash
# Generate hash for an inline script block
echo -n 'window["__env"] = {}' | openssl dgst -sha256 -binary | openssl enc -base64
# → Add 'sha256-RESULT' to script-src
```

## Option B — Angular SSR (with nonces)

If using Angular Universal or SSR:

```ts
// server.ts — Express SSR server
import { randomBytes } from 'crypto';

app.use((req, res, next) => {
    const nonce = randomBytes(16).toString('base64');
    res.locals.cspNonce = nonce;

    const enforce = process.env.CSP_ENFORCE === 'true';
    const header  = enforce ? 'Content-Security-Policy' : 'Content-Security-Policy-Report-Only';

    res.setHeader(header,
        `default-src 'self'; script-src 'nonce-${nonce}' 'strict-dynamic'; style-src 'self' 'unsafe-inline'; object-src 'none'; base-uri 'self'; report-uri /csp-report`
    );
    next();
});
```

```ts
// In AppServerModule — pass nonce to Angular renderer
providers: [
    {
        provide: 'CSP_NONCE',
        useFactory: (req: Request) => req.res?.locals.cspNonce ?? '',
        deps: [REQUEST],
    },
],
```

## Angular-specific gotchas

| Issue | Cause | Fix |
|---|---|---|
| Router not working | `<base href="/">` blocked | `base-uri 'self'` covers it |
| Lazy loaded modules blocked | Chunks loaded dynamically | `'strict-dynamic'` propagates trust — required |
| Angular animations broken | Adds `style=""` via Renderer2 | `style-src-attr 'unsafe-inline'` required |
| Material components broken | Mat injects `<style>` | `style-src-elem 'nonce-...'` or `'unsafe-inline'` |
| Google Fonts blocked | `fonts.googleapis.com` | Add to `style-src`, `fonts.gstatic.com` to `font-src` |
| API calls blocked | `connect-src` missing domain | Add your API domain to `connect-src` |
| `ng serve` CSP errors | Dev server uses `webpack-dev-server` with HMR | Use `--no-hmr` or add `ws://localhost:4200` to `connect-src` |
