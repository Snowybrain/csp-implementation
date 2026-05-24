# CSP — Static HTML / Apache / nginx only

For sites with no server-side rendering. Nonces are not viable — use hashes.

---

## nginx

```nginx
server {
    # Phase 1 — Report-Only
    add_header Content-Security-Policy-Report-Only
        "default-src 'self'; script-src 'self' [HASH_OF_INLINE_SCRIPTS]; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; object-src 'none'; base-uri 'self'"
        always;

    # Phase 4 — Enforcement
    # add_header Content-Security-Policy "..." always;
}
```

## Generating hashes for inline scripts

```bash
# For each inline <script> block, generate its hash:
printf '%s' 'console.log("hello")' | openssl dgst -sha256 -binary | openssl base64

# Result goes in script-src as: 'sha256-BASE64RESULT'
# Example: script-src 'self' 'sha256-qznLcsROx4GACP2dm0UCKCzCG+HiZ1guq6ZZDob/Tng='
```

**Important**: The hash must match the **exact bytes** of the script content, including whitespace.
Any change to the script requires regenerating the hash.

## Apache

```apache
# .htaccess or VirtualHost
Header always set Content-Security-Policy-Report-Only \
    "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; object-src 'none'"
```

## Recommendation

Static sites have the simplest CSP path:
1. Eliminate all inline scripts (move to `.js` files)
2. Eliminate all inline event handlers (use `addEventListener`)
3. Apply a strict `script-src 'self'` — no nonces, no hashes, no `unsafe-inline`

The only exception is `style-src-attr 'unsafe-inline'` if using UI libraries that inject `style=""` attributes.
