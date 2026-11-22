# nma.sh Runtime & Limits Cheat Sheet

Local reference bundled with the `free-hosting` skill. See `deploy-and-limits.md` for SSH commands, recipes, and error codes.

## Request & Response

```js
serve(async req => {
  const path = req.path.split("?")[0];
  const query = new URLSearchParams(req.path.split("?")[1] || "");
  // req.method, req.url, req.headers (lowercase keys), req.body (always string)
  // req.headers["x-real-ip"] is set by nma.sh to the visitor's IP
  return {
    status: 200,
    contentType: "application/json",
    headers: { "cache-control": "no-store" }, // multiple Set-Cookie: [["set-cookie", a], ["set-cookie", b]]
    body: JSON.stringify({ path, q: query.get("q") }),
  };
});
```

Workers format also works (pick one form, not both):

```js
export default {
  async fetch(request, env, ctx) {
    // env.NAME reads secret NAME; ctx.waitUntil(promise) runs before response ends
    return new Response("hello");
  },
};
```

## Safe Secret Lookup (always inside the handler)

```js
let token;
try { token = secrets.get("API_TOKEN"); } catch (e) { token = null; }
```

## Built-in `db`, `kv`, and `crdt`

Values are strings (`JSON.stringify` objects/numbers yourself). Always prefix keys with your app name.

```js
// Strong transactional db with CAS
db.put("app:item:" + Date.now() + ":" + crypto.randomUUID(), JSON.stringify(obj));
const val = db.get("app:key"); // string | null; optional { consistency: "session"|"eventual"|"strong" }
const row = db.getv("app:key"); // { value, v } | null
const nextV = db.cas("app:key", 0, "created"); // 0 = create-if-absent; returns new v or null on conflict
db.delete("app:key"); // true | false
const { keys, next } = db.list("app:item:", { limit: 50, reverse: true }); // pass { after: next } to page

// Edge-cached kv (eventual list; session/eventual get)
kv.put("app:cache:home", html);
const cached = kv.get("app:cache:home", { consistency: "session" });

// Conflict-free counters & registers
const views = crdt.counter("app:views");
views.add(1);
const total = views.value();
```

## Built-in User Auth (3 lines, zero setup)

```js
serve(req => {
  const r = auth.handle(req);
  if (r) return r; // handles /signup /login /logout /recover /reset (HTML forms or JSON + Bearer token)

  const u = auth.require(req);
  if (!u.id) return u; // 401 for API clients, 303 to /login for browsers

  // u = { id, user, db, kv } — u.db and u.kv are isolated to this user alone
  return { contentType: "application/json", body: JSON.stringify({ user: u.user }) };
});
```

- `auth.user(req)` returns `null` or `{ id, user, db, kv }` without forcing a redirect.
- Never touch `auth$` or `u$:` keys in `db`/`kv`.

## Static Site `_redirects` and `_headers`

Only apply to static sites (`index.html` at top, no `app.js`). Never served as public files.

`_redirects` (up to 200 rules; existing files win over rules):
```text
/old-path /new-path 301
/blog/* /posts/:splat 301
/* /index.html 200
```

`_headers` (up to 100 rules, 8 KiB; cannot override `Content-Length`, `Set-Cookie`, `ETag`, `Location`):
```text
/assets/*
  Cache-Control: public, max-age=31536000, immutable
```

## Key Platform Limits (nma.sh)

- **Apps:** 2 active unverified (idle apps may be deleted after 30 days) · 5 active with verified email (kept for good) · 10 active on Pro ($8/mo). 16 lifetime app numbers per account (deleting frees an active slot immediately).
- **App / secret names:** app `[a-z0-9-]{1,31}`, secret `[A-Z0-9_]{1,32}`, secret value <= 1024 bytes.
- **Data:** `db` 10,000 keys & 40 MiB values per account (5,000 keys & 20 MiB per group across 2 groups); `kv` 600 keys per account; key <= 512 bytes, value <= 64 KiB.
- **HTTP:** request body <= 64 KiB (`413`), request headers <= 8 KiB (`431`), path+query <= 255 bytes (`414`), response <= 1 MiB (`502`).
- **Execution:** 2 s CPU per request (`504` if exceeded); 8 concurrent requests per account (64 on Pro).
- **Outbound `fetch()`:** `https://` on port 443 to public hosts only; 10 req/s per app (burst 20), 1.5 s timeout, 256 KiB response body, 48 KiB request body, max 5 redirects.
- **Deploys:** single JS file ~6 MB; folder/archive <= 1,000 files, <= 5 MiB per file, <= 25 MiB total (27 MiB archive).
- **Node compat:** `crypto`, `buffer`, `stream`, `url`, `path`, `events`, `util`, `querystring`, `assert`, `timers`, `string_decoder`, `async_hooks`, `os`, `perf_hooks` — no `fs`, `net`, or `child_process`.
