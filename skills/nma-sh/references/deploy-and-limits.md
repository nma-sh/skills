# nma.sh Commands, Recipes & Error Codes

Local reference bundled with the `nma-sh` skill. All commands support `--json` anywhere in the arguments (or `-- --json ...` if placed first).

## SSH Commands Reference

| Command | Purpose |
|---|---|
| `ssh nma.sh signup [--ref TAG] --json` | Create an account bound to your SSH key |
| `ssh nma.sh tenant --json` | Current tenant id, role, apps_used/apps_cap, email status, plan, `claim_url` |
| `ssh nma.sh apps --json` | List all apps, states (`live`, `warned`, `deleting`, `deleted`), and URLs |
| `ssh nma.sh deploy APP --json < app.js` | Deploy single JS file (or pipe `.tar`, `.tgz`, `.zip` for folders) |
| `ssh nma.sh deployments APP --json` | List version history and digests |
| `ssh nma.sh rollback APP [VERSION] --json` | Roll back to previous (or named) version |
| `ssh nma.sh delete APP --json` | Delete an app (404 immediately; frees active app slot; data stays) |
| `ssh nma.sh logs APP [--limit N] --json` | Read recent NDJSON log lines (`1..1000`; `-f` to follow) |
| `ssh nma.sh secret set APP NAME --json < file` | Write-only secret (works before or after first deploy) |
| `ssh nma.sh secret delete APP NAME --json` | Delete an app secret |
| `ssh nma.sh db list APP [PREFIX] [--after K] [--limit N] --json` | List strong `db` keys outside the app |
| `ssh nma.sh db get APP KEY --json` | Read a `db` key (`{app, v, value}`) |
| `ssh nma.sh db delete APP KEY --json` | Delete a `db` key |
| `ssh nma.sh db export APP --json` | Export every `db` key as NDJSON (up to 8 MiB over MCP) |
| `ssh nma.sh verify EMAIL --json` | Email a 24h confirmation link (raises free cap 2 → 5 apps, kept for good) |
| `ssh nma.sh plan --json` | Plan limits, Pro pricing ($8/mo), and current usage |
| `ssh nma.sh upgrade --json` | Generate a Stripe Pro checkout link for the user |
| `ssh nma.sh billing --json` | Generate a Stripe billing portal link |
| `ssh nma.sh dash --json` | One-time 5-minute link to `https://nma.sh/dash` (8h browser tab session) |
| `ssh nma.sh dash signout --json` | Sign out all active browser dashboard sessions |
| `ssh nma.sh domain add APP HOST --json` | Pro: attach custom domain over TLS (prints CNAME or A record) |
| `ssh nma.sh domain ls --json` / `domain rm HOST --json` | Pro: list or remove custom domains |
| `ssh nma.sh rename APP NAME --json` | Pro: claim chosen `NAME.sites.diy` hostname (5–30 chars `[a-z0-9-]`) |
| `ssh nma.sh backups APP --json` / `restore APP DATE --json` | Pro: list or restore last 7 UTC days of `db` + `kv` |
| `ssh nma.sh keys --json` / `keys create ID ROLE --json` | List keys or create `owner`/`deployer`/`reader` key (3 keys max per account) |
| `ssh nma.sh retry --json` | Finish a mutation whose outcome was unknown |
| `ssh nma.sh help --json` | Full command catalog |

## Common App Patterns

### Contact Form + Password-Protected `/inbox`

```js
const esc = s => String(s ?? "").replace(/[&<>"]/g, c => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;" })[c]);

serve(async req => {
  const path = req.path.split("?")[0];
  if (path === "/inbox") {
    let pw; try { pw = secrets.get("INBOX_PASSWORD"); } catch (e) { pw = null; }
    if (!pw || req.headers["authorization"] !== "Basic " + btoa("me:" + pw)) {
      return { status: 401, headers: { "www-authenticate": 'Basic realm="inbox"' }, body: "password needed" };
    }
    const { keys } = db.list("msg:", { limit: 100 });
    const rows = keys.reverse().map(k => {
      const m = JSON.parse(db.get(k));
      return `<p><b>${esc(m.name)}</b> ${esc(m.at)}<br>${esc(m.text)}</p>`;
    }).join("");
    return { contentType: "text/html; charset=utf-8", body: `<h1>Inbox</h1>${rows}` };
  }
  if (req.method === "POST") {
    const f = new URLSearchParams(req.body);
    db.put("msg:" + Date.now() + ":" + crypto.randomUUID(),
      JSON.stringify({ name: f.get("name"), text: f.get("text"), at: new Date().toISOString() }));
    return { status: 303, headers: { location: "/thanks" }, body: "" };
  }
  if (path === "/thanks") return { contentType: "text/html", body: "<p>Thanks, received.</p>" };
  if (path !== "/") return { status: 404, body: "not found" };
  return {
    contentType: "text/html; charset=utf-8",
    body: '<form method="post"><input name="name" required><textarea name="text" required></textarea><button>Send</button></form>',
  };
});
```

### Per-IP Rate Limit in 64 Fixed `kv` Buckets

```js
function hit(ip, limitPerMin) {
  let h = 0;
  for (const c of ip) h = (h * 31 + c.charCodeAt(0)) >>> 0;
  const key = "rl:" + (h % 64), win = Math.floor(Date.now() / 60000);
  let b = JSON.parse(kv.get(key, { consistency: "session" }) || "null");
  if (!b || b.w !== win) b = { w: win, n: {} };
  b.n[ip] = (b.n[ip] || 0) + 1;
  kv.put(key, JSON.stringify(b));
  return b.n[ip] <= limitPerMin;
}
```

### Scheduled Work (No Built-In Cron)

Apps run only on HTTP requests (2 s CPU max). Either trigger a secret-guarded `POST /cron` from GitHub Actions `schedule:`, or run lazily on incoming requests using a timestamp in `kv`.

## Key Exit Codes & Error Fixes

- **Exit `0`:** success.
- **Exit `3`:** deploy's first request returned 5xx; automatically rolled back. Inspect `first.error` and `logs`, fix, and redeploy.
- **Exit `4`:** 4xx refusal — fix the request (`no_account` → `signup`; `compile_error` → fix JS; `apps_full` → delete an unused app, `verify EMAIL` for 5, or `upgrade` for 10; `pro_only` → Pro feature; `too_large` → shrink bundle under 6 MB JS / 25 MiB static; `slow_down` → wait `retry_after_s`).
- **Exit `75`:** timeout or `outcome_unknown` — run `ssh nma.sh retry --json`.
