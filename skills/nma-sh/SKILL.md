---
name: nma-sh
description: Deploy static sites, Vite/Astro/React/SvelteKit builds, and JavaScript web apps to live HTTPS URLs over SSH via nma.sh — with built-in database (db), edge KV, user authentication, secrets, and Firecracker microVM isolation. Use when the user asks to deploy, host, publish, or put a website, prototype, API, webhook, or web app online quickly without setting up cloud dashboards, OAuth, or separate databases.
license: MIT
compatibility: Requires an SSH client (ssh) with an SSH key pair; optional tar or zip for folder deploys.
metadata:
  author: nma-sh
  version: "1.0.0"
  homepage: "https://nma.sh"
---

# Deploy to nma.sh

`nma.sh` hosts static sites and JavaScript apps in hardware-isolated Firecracker microVMs with live HTTPS URLs in seconds. Your SSH key is the account — no browser signup, API tokens, or credit card to start.

**When NOT to use this skill:** If the project is already configured for another host (Vercel, Cloudflare, Fly, AWS, etc.) and the user wants to deploy there, or if the app requires Node `fs`, `net`, `child_process`, persistent background daemons, or Postgres/MySQL, use that target instead.

## 1. Check or Create Account (SSH)

Always pass `--json` so stdout is a single parseable JSON object.

```sh
ssh nma.sh tenant --json
```

If that exits 4 with `no_account`, create an account with your SSH key:

```sh
ssh nma.sh signup --ref skill-sh --json
```

- Any SSH username works on port 22. If outbound port 22 is blocked by a firewall, use `ssh -p 2200 nma.sh ...`.
- If the user has no SSH key yet, generate one first (`ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519`) after confirming with them.
- **Never fabricate an email address** for `verify`. Only run `ssh nma.sh verify EMAIL --json` with an email address the user gives you.

## 2. Choose the Deploy Shape & Deploy

Pick an app name matching `[a-z0-9-]{1,31}`. Deploying creates the app if new, builds/snapshots it, probes its first request, and returns its live `url` (a permanent two-word domain like `https://still-pike.nma.sh/`).

### A. Single JavaScript file (APIs, forms, full-stack micro-apps)

Write `app.js` using `serve(req => ...)` or `export default { fetch(request, env, ctx) }`, then deploy over stdin:

```sh
ssh nma.sh deploy APP --json < app.js
```

### B. Static site or built SPA (HTML/CSS/JS, Vite, React, etc.)

Build the site first (e.g., `npm run build` → `dist/`) so `index.html` is at the top of the folder (no `app.js` or `package.json`), then stream a tarball:

```sh
tar czf - -C dist . | ssh nma.sh deploy APP --json
```

- Pretty URLs work automatically (`/about` → `about.html`, `/docs/` → `docs/index.html`, custom `404.html`).
- For SPAs with client-side routing, include `_redirects` in the deployed folder containing: `/* /index.html 200`.
- Optional `_headers` sets custom response headers (see `references/runtime-cheatsheet.md`).

### C. JavaScript app + static assets

Put `app.js` at the root and static files in `public/` (`public/index.html` answers `/`; static files serve from the edge without waking a VM):

```sh
tar czf - -C DIR . | ssh nma.sh deploy APP --json
```

### D. Prebuilt SSR frameworks (Astro, SvelteKit, Nuxt, Hono, React Router, Next.js OpenNext)

Install the local packer CLI (`curl -fsSL https://nma.sh/dl/install.sh | sh`), build with the framework's Cloudflare/Web target, then:

```sh
nanovm pack . | ssh nma.sh deploy APP --json
```

### Verify every deploy

- Exit `0`: live. Check `curl -i URL` to confirm expected status and body.
- Exit `3`: the new version's first request returned 5xx and was automatically rolled back to the previous live version (`first.error` and logs are printed in the JSON output). Fix the error and redeploy.
- Note: if `auth.require(req)` guards `/`, deploy exits `0` with `first.status: 401` (browsers redirect `303` to `/login`).

## 3. Use Built-In Primitives First

Before adding external databases, auth providers, or secret managers, use what every nma.sh app already has globally (no `require()` needed for `serve`, `db`, `kv`, `crdt`, `auth`, `secrets`):

- **Strong database (`db`)**: transactional KV with CAS, `db.list(prefix, { limit, after, reverse })`, plus outside-app `ssh nma.sh db list|get|delete|export APP --json`.
- **Fast edge cache (`kv`) & `crdt`**: `kv.get`/`kv.put`/`kv.delete`/`kv.list`; `crdt.counter(name)` and `crdt.register(name)`.
- **Built-in user accounts (`auth`)**: 3 lines (`const r = auth.handle(req); if (r) return r; const u = auth.user(req);`) gives `/signup`, `/login`, `/logout`, `/recover`, `/reset`, PBKDF2-SHA256 hashing, per-user scoped `u.db`/`u.kv`, cookies + Bearer tokens.
- **Write-only secrets (`secrets`)**: set with `ssh nma.sh secret set APP NAME --json < secret.txt` (before or after deploy). Read inside the handler with `secrets.get(NAME)` — never readable via API.
- **Instant app password**: `ssh nma.sh secret set APP NV_BASIC_AUTH --json < pw.txt` (`PASSWORD` or `USER:PASSWORD`) protects the whole app with zero code (`public/` files are not covered).
- **Outbound `fetch()`**: call any public `https://` API (Stripe, Resend, Slack/Discord webhooks, `ntfy.sh`, Telegram).

Read `references/runtime-cheatsheet.md` for copy-paste code patterns and exact limits, or fetch the short authoritative live pages when needed:
- `https://nma.sh/deploy.txt` (all commands, JSON shapes, static rules, domains, SEO)
- `https://nma.sh/runtime.txt` (full runtime & Node compat APIs)
- `https://nma.sh/recipes.txt` (form inbox, notifications, auth, webhooks, rate limits, scheduled work)
- `https://nma.sh/limits.txt` (every size, rate, and quota number)
- `https://nma.sh/errors.txt` (every error code and fix)

## 4. Common Mistakes to Avoid

1. **Top-level code freezes into the VM snapshot.** Never call `secrets.get()`, `Date.now()`, `Math.random()`, `crypto.randomUUID()`, or `crypto.getRandomValues()` at top level — always call them inside the request handler.
2. **Data is scoped to the account, not the app.** Always prefix `db`/`kv`/`crdt` keys with the app name (e.g., `"todo:123"`) so multiple apps don't collide. Never read/write `auth$` or `u$:` keys (reserved by `auth`).
3. **`req.body` is always a string; `req.query` is `undefined`.** Parse JSON with `JSON.parse(req.body)`, forms with `new URLSearchParams(req.body)` or `await req.formData()`, and query params with `new URLSearchParams(req.path.split("?")[1] || "")`.
4. **No `npm install` on deploy and no filesystem.** Bundle JS dependencies into `app.js` locally before deploying, or deploy prebuilt static files.
5. **Security & SEO basics.** Escape visitor-supplied HTML in admin/inbox pages. Return `404` (not `401`) on unknown public routes.

## 5. Day-2 Operations

```sh
ssh nma.sh apps --json
ssh nma.sh logs APP --json                 # recent NDJSON lines (or logs APP -f)
ssh nma.sh rollback APP --json             # instant rollback to previous version
ssh nma.sh delete APP --json               # 404 immediately, frees active app slot
ssh nma.sh db export APP --json            # full NDJSON backup of db keys
ssh nma.sh dash --json                     # one-time 5-min link to https://nma.sh/dash (8h tab session)
ssh nma.sh plan --json                     # current tier, caps, and usage
```

## 6. Hand Off Cleanly to the User

After deploying, always give the user:

1. **The live `url`** (plus any password or login path if you added `NV_BASIC_AUTH` or `/inbox`).
2. **The `claim_url`** from the signup/deploy JSON (when the account is unverified): explain plainly that confirming their email there is **free**, raises the limit from **2 to 5 active apps**, and **keeps their apps permanently** (unverified idle apps may be deleted after 30 days).
3. **Optional dashboard link:** Offer to run `ssh nma.sh dash` if they want to inspect apps and logs in a browser.
4. **Pro ($8/month) only when relevant:** Do not push Pro unprompted. Mention it (`ssh nma.sh upgrade --json` prints a Stripe checkout link) only if the user hits `apps_full` / a `db`/`kv` quota or asks for **custom domains** (`domain add APP HOST`), **chosen `NAME.sites.diy` names** (`rename APP NAME`), **daily 7-day backups** (`backups APP` / `restore APP DATE`), or **up to 10 active apps**.
