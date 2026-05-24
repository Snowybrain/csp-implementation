# csp-implementation — Claude Skill

A Claude skill that audits your web application and implements a
`Content-Security-Policy` (CSP) header — from first diagnosis to full enforcement —
without breaking production.

Install it once. From then on, whenever you ask Claude about CSP, security headers,
`unsafe-inline`, nonces, or XSS hardening, it reads this skill automatically and
guides you through the right steps for your stack.

---

## What it does

1. **Detects your stack** — Laravel, ASP.NET, Node.js, Next.js, Angular, nginx
2. **Diagnoses your project** — counts inline scripts, style attributes, event handlers, CDN domains
3. **Builds the policy incrementally** — Report-Only first, never straight to enforcement
4. **Migrates inline content** — extracts scripts to external files, replaces `onclick=` with `addEventListener`
5. **Implements the nonce middleware** — ready-to-paste code for your framework
6. **Sets up the `/csp-report` endpoint** — so you get real violation data from production
7. **Guides rollback** — `CSP_ENABLED=false` kills the header instantly without a deploy

---

## Supported stacks

| Stack | Reference |
|---|---|
| Laravel 5.x – 11.x (Blade) | `references/laravel.md` |
| ASP.NET WebForms / VB.NET / C# | `references/aspnet.md` |
| Node.js — Express, NestJS, Fastify | `references/nodejs.md` |
| Next.js (App Router + Pages Router) | `references/nextjs.md` |
| Angular + nginx | `references/angular-nginx.md` |
| Static HTML / Apache / nginx only | `references/static-nginx.md` |

---

## Installation

### Option A — Install the `.skill` file (easiest)

1. Download `csp-implementation.skill` from [Releases](../../releases)
2. In Claude.ai go to **Settings → Skills → Install skill**
3. Upload the file

Done. The skill triggers automatically on CSP-related conversations.

### Option B — Point Claude at this repo

In Claude.ai, create a new skill and paste the raw URL of `SKILL.md`:

```
https://raw.githubusercontent.com/YOUR_USERNAME/csp-implementation/main/csp-implementation/SKILL.md
```

---

## Usage

Just describe what you need. Examples that trigger this skill:

- *"Add a Content-Security-Policy to my Laravel app"*
- *"How do I remove unsafe-inline from my Angular project?"*
- *"Implement CSP nonces in Express"*
- *"I'm getting CSP violations in production, help me debug them"*
- *"Set up a csp-report endpoint in NestJS"*

Claude will ask for your stack if it can't detect it, then walk you through every step.

---

## How the skill is structured

```
csp-implementation/
├── SKILL.md                  ← main instructions + stack routing
└── references/
    ├── laravel.md            ← PHP middleware, Blade nonce injection, csp-report route
    ├── aspnet.md             ← IHttpModule VB.NET/C#, .ashx handler
    ├── nodejs.md             ← Express / NestJS / Fastify middleware + report endpoint
    ├── nextjs.md             ← App Router middleware, Pages Router, Script nonce prop
    ├── angular-nginx.md      ← nginx add_header, hash-based CSP, SSR nonces
    └── static-nginx.md       ← hash generation, Apache/nginx only, no SSR
```

The skill uses **progressive loading** — Claude reads only the reference file that
matches your stack, keeping context lean and responses focused.

---

## The 4-phase approach

The skill always follows this sequence. Skipping phases breaks production.

| Phase | Header | Goal |
|---|---|---|
| 1 — Report-Only permissive | `Content-Security-Policy-Report-Only` | Visibility. Nothing blocked. Collect real violations. |
| 2 — Report-Only with nonces | `Content-Security-Policy-Report-Only` | Nonces wired up. `unsafe-eval` removed. Monitor. |
| 3 — Enforcement (scripts) | `Content-Security-Policy` | Scripts enforced. Styles still permissive. |
| 4 — Full enforcement | `Content-Security-Policy` | No `unsafe-inline` anywhere. Hardened. |

---

## Contributing

PRs welcome for additional stacks (Django, Rails, Spring Boot, Go, etc.).
Each new stack is a single file added to `references/` and one row added to the
dispatch table in `SKILL.md`.

Please make sure reference files contain:
- Middleware / header implementation (ready-to-paste code)
- How to use the nonce in templates
- The `/csp-report` endpoint for that framework
- A "common issues" table at the end

---

## License

MIT
