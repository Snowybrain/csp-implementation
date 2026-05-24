# CSP — Next.js (App Router and Pages Router)

---

## App Router (Next.js 13+) — Middleware approach

```ts
// middleware.ts (root of project)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
    const response = NextResponse.next();

    // Generate nonce
    const nonce   = Buffer.from(crypto.randomUUID()).toString('base64');
    const enforce = process.env.CSP_ENFORCE === 'true';
    const header  = enforce ? 'Content-Security-Policy' : 'Content-Security-Policy-Report-Only';

    const policy = enforce
        ? [
            "default-src 'self'",
            `script-src 'nonce-${nonce}' 'strict-dynamic'`,
            `style-src-elem 'self' 'nonce-${nonce}'`,
            "style-src-attr 'unsafe-inline'",
            "object-src 'none'",
            "base-uri 'self'",
            "report-uri /api/csp-report",
          ].join('; ')
        : [
            "default-src 'self'",
            `script-src 'self' 'nonce-${nonce}' 'strict-dynamic' 'unsafe-inline'`,
            "style-src 'self' 'unsafe-inline'",
            "object-src 'none'",
            "base-uri 'self'",
            "report-uri /api/csp-report",
          ].join('; ');

    response.headers.set(header, policy);

    // Pass nonce to the page via request header (read in layout.tsx)
    const requestHeaders = new Headers(request.headers);
    requestHeaders.set('x-nonce', nonce);
    return NextResponse.next({ request: { headers: requestHeaders } });
}

export const config = {
    matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

```tsx
// app/layout.tsx — read nonce from request headers
import { headers } from 'next/headers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
    const nonce = headers().get('x-nonce') ?? '';

    return (
        <html>
            <head>
                <script nonce={nonce} src="/js/analytics.js" />
            </head>
            <body>{children}</body>
        </html>
    );
}
```

## CSP report API route

```ts
// app/api/csp-report/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { appendFileSync } from 'fs';

export async function POST(req: NextRequest) {
    const body = await req.json();
    const v    = body['csp-report'] ?? body;
    appendFileSync('logs/csp-violations.log',
        `[${new Date().toISOString()}] ${v['violated-directive']} | ${v['blocked-uri']}\n`
    );
    return new NextResponse(null, { status: 204 });
}
```

## Pages Router (Next.js 12 and below)

```js
// next.config.js
const crypto = require('crypto');

const securityHeaders = (nonce) => [
    {
        key: 'Content-Security-Policy-Report-Only',
        value: `default-src 'self'; script-src 'self' 'nonce-${nonce}' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; object-src 'none'; report-uri /api/csp-report`,
    },
];

module.exports = {
    async headers() {
        // Static nonce is NOT secure — use middleware approach above for Pages Router too
        // This is only for illustration; use Next.js middleware for per-request nonces
        return [{ source: '/(.*)', headers: securityHeaders('') }];
    },
};
```

## Next.js gotchas

| Issue | Cause | Fix |
|---|---|---|
| `<Script strategy="afterInteractive">` blocked | Next.js injects inline loader | Apply nonce via `nonce` prop on `<Script>` |
| `next/image` blocked | Uses `blob:` for optimization | Add `img-src 'self' blob: data:` |
| HMR in dev broken | WebSocket to `localhost:3000` | Add `ws://localhost:3000` to `connect-src` in dev |
| API routes generate CSP header | Middleware runs on all routes | Exclude `/api/` from middleware matcher if needed |
