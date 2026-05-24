# CSP — Node.js (Express / NestJS / Fastify)

---

## Express

```js
// middleware/csp.js
const crypto = require('crypto');

function cspMiddleware(req, res, next) {
    if (process.env.CSP_ENABLED === 'false') return next();

    const nonce = crypto.randomBytes(16).toString('base64');
    res.locals.cspNonce = nonce; // available in EJS/Pug/Handlebars templates

    res.on('finish', () => {}); // hook — actual header set below

    const enforce = process.env.CSP_ENFORCE === 'true';
    const header  = enforce ? 'Content-Security-Policy' : 'Content-Security-Policy-Report-Only';
    const policy  = enforce ? enforcementPolicy(nonce) : reportOnlyPolicy(nonce);

    res.setHeader(header, policy);
    next();
}

function reportOnlyPolicy(nonce) {
    return [
        "default-src 'self'",
        `script-src 'self' 'nonce-${nonce}' 'strict-dynamic' 'unsafe-inline'`,
        "style-src 'self' 'unsafe-inline'",
        "font-src 'self' data:",
        "img-src 'self' data:",
        "connect-src 'self'",
        "object-src 'none'",
        "base-uri 'self'",
        "form-action 'self'",
        "report-uri /csp-report",
    ].join('; ');
}

function enforcementPolicy(nonce) {
    return [
        "default-src 'self'",
        `script-src 'nonce-${nonce}' 'strict-dynamic'`,
        `style-src-elem 'self' 'nonce-${nonce}'`,
        "style-src-attr 'unsafe-inline'",
        "font-src 'self' data:",
        "img-src 'self' data:",
        "connect-src 'self'",
        "frame-ancestors 'none'",
        "object-src 'none'",
        "base-uri 'self'",
        "form-action 'self'",
        "upgrade-insecure-requests",
        "report-uri /csp-report",
    ].join('; ');
}

module.exports = cspMiddleware;
```

```js
// app.js
const csp = require('./middleware/csp');
app.use(csp);

// CSP report endpoint — no CSRF needed
app.post('/csp-report', express.text({ type: 'application/csp-report' }), (req, res) => {
    const report = JSON.parse(req.body || '{}');
    const v = report['csp-report'] ?? report;
    const entry = `[${new Date().toISOString()}] directive=${v['violated-directive']} blocked=${v['blocked-uri']} doc=${v['document-uri']}\n`;
    require('fs').appendFileSync('logs/csp-violations.log', entry);
    res.status(204).end();
});
```

## NestJS

```ts
// middleware/csp.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { randomBytes } from 'crypto';
import { appendFileSync } from 'fs';

@Injectable()
export class CspMiddleware implements NestMiddleware {
    use(req: Request, res: Response, next: NextFunction) {
        if (process.env.CSP_ENABLED === 'false') return next();

        const nonce   = randomBytes(16).toString('base64');
        res.locals.cspNonce = nonce;

        const enforce = process.env.CSP_ENFORCE === 'true';
        const header  = enforce ? 'Content-Security-Policy' : 'Content-Security-Policy-Report-Only';
        res.setHeader(header, this.buildPolicy(nonce, enforce));
        next();
    }

    private buildPolicy(nonce: string, enforce: boolean): string {
        const directives = enforce
            ? [
                "default-src 'self'",
                `script-src 'nonce-${nonce}' 'strict-dynamic'`,
                `style-src-elem 'self' 'nonce-${nonce}'`,
                "style-src-attr 'unsafe-inline'",
                "object-src 'none'",
                "base-uri 'self'",
                "form-action 'self'",
                "report-uri /csp-report",
              ]
            : [
                "default-src 'self'",
                `script-src 'self' 'nonce-${nonce}' 'strict-dynamic' 'unsafe-inline'`,
                "style-src 'self' 'unsafe-inline'",
                "object-src 'none'",
                "base-uri 'self'",
                "report-uri /csp-report",
              ];
        return directives.join('; ');
    }
}
```

```ts
// app.module.ts — register middleware
export class AppModule implements NestModule {
    configure(consumer: MiddlewareConsumer) {
        consumer.apply(CspMiddleware).forRoutes('*');
    }
}
```

```ts
// csp-report.controller.ts
@Controller('csp-report')
export class CspReportController {
    @Post()
    @HttpCode(204)
    report(@Req() req: Request) {
        const v = req.body?.['csp-report'] ?? req.body ?? {};
        appendFileSync('logs/csp-violations.log',
            `[${new Date().toISOString()}] ${v['violated-directive']} | ${v['blocked-uri']}\n`
        );
    }
}
```

## Fastify

```ts
// plugins/csp.ts
import fp from 'fastify-plugin';
import { randomBytes } from 'crypto';

export default fp(async (fastify) => {
    fastify.addHook('onRequest', async (request, reply) => {
        if (process.env.CSP_ENABLED === 'false') return;

        const nonce = randomBytes(16).toString('base64');
        request.cspNonce = nonce; // extend FastifyRequest interface

        const enforce = process.env.CSP_ENFORCE === 'true';
        const header  = enforce ? 'Content-Security-Policy' : 'Content-Security-Policy-Report-Only';

        const policy = enforce
            ? `default-src 'self'; script-src 'nonce-${nonce}' 'strict-dynamic'; style-src-attr 'unsafe-inline'; object-src 'none'; base-uri 'self'; report-uri /csp-report`
            : `default-src 'self'; script-src 'self' 'nonce-${nonce}' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; object-src 'none'; report-uri /csp-report`;

        reply.header(header, policy);
    });

    // CSP report endpoint
    fastify.post('/csp-report', { config: { rawBody: true } }, async (request, reply) => {
        const v = (request.body as any)?.['csp-report'] ?? request.body ?? {};
        require('fs').appendFileSync('logs/csp-violations.log',
            `[${new Date().toISOString()}] ${v['violated-directive']} | ${v['blocked-uri']}\n`
        );
        reply.code(204).send();
    });
});
```

## Template usage (EJS example)

```html
<script nonce="<%= locals.cspNonce %>" src="/js/app.js"></script>
<script nonce="<%= locals.cspNonce %>">
    window.config = { userId: <%= user.id %> };
</script>
```
