# nma-sh/skills — Free Hosting via Remote MCP & SSH

[![skills.sh](https://skills.sh/b/nma-sh/skills)](https://skills.sh/nma-sh/skills)

Official **Free Hosting** Agent Skill, [Agent Plugins v1.0.0](https://agent-plugins.org/) portable plugin pack, and Claude Code plugin (with bundled Remote Streamable HTTP MCP at `https://nma.sh/mcp`) for deploying static sites and JavaScript web apps to **[nma.sh](https://nma.sh/)**.

Each site runs in its own hardware-isolated Firecracker microVM with a live HTTPS URL in seconds, built-in database (`db`), edge `kv`, `crdt` counters, user authentication (`auth`), write-only `secrets`, and outbound `fetch()` — **free to start** (2 apps with no email/card, 5 apps kept permanently with a free verified email; flat $8/month Pro for 10 apps, custom domains, and daily backups).

## Install

### Claude Code Plugin Marketplace (Skills + Remote MCP)

```text
/plugin marketplace add nma-sh/skills
/plugin install nma-sh@nma-sh
```

Or test locally from a clone:

```bash
claude --plugin-dir ./nma-skills
```

### Remote MCP Server (Zero Local Install — Claude.ai, ChatGPT, Cursor, VS Code, Claude Code)

Connect directly to the hosted Streamable HTTP MCP endpoint with OAuth 2.1 + PKCE:

- **URL:** `https://nma.sh/mcp`
- **Email sign-in / sign-up:** enter your email on the OAuth page for a one-time sign-in link and 6-digit code (new accounts start verified with 5 permanent apps, or you can permanently link an existing SSH-key account).
- **Direct SSH connect:** run `ssh nma.sh mcp-connect` and paste the one-time 5-minute code into **Use my account**.
- **Revoke anytime:** run `ssh nma.sh mcp revoke` (owner key) to disconnect every remote MCP client immediately.

CLI shortcut for Claude Code:

```bash
claude mcp add --transport http nma-sh https://nma.sh/mcp
```

### Agent Skills CLI (`skills.sh` — Claude Code, Cursor, Codex, OpenCode, Gemini CLI, Copilot)

```bash
npx skills add nma-sh/skills
```

### Optional Local SSH Stdio MCP

If you prefer local SSH transport over the hosted endpoint, run `ssh nma.sh signup --json` once and register `ssh nma.sh mcp` in your MCP client (`claude mcp add nma -- ssh nma.sh mcp`).

## What's Inside

| Path | Purpose |
|---|---|
| `plugin.json` | [Agent Plugins v1.0.0](https://agent-plugins.org/) portable plugin manifest (`nma-sh`) |
| `mcp.json` | Agent Plugins v1.0.0 remote `streamable-http` MCP config (`https://nma.sh/mcp`) |
| `.mcp.json` | Claude Code plugin remote `http` MCP config (`https://nma.sh/mcp`) |
| `skills/nma-sh/SKILL.md` | Full `nma.sh` deployment & hosting skill (static sites, JS apps, DB/KV/auth/secrets, logs, rollback, domains) |
| `skills/free-hosting/SKILL.md` | Free Hosting skill targeting instant free static & JS app hosting |
| `skills/*/references/` | Local bundled cheat sheets (`runtime-cheatsheet.md`, `deploy-and-limits.md`) |
| `skills.sh.json` | Repository page grouping for `skills.sh/nma-sh/skills` |
| `.claude-plugin/` | Claude Code plugin and marketplace manifests |

## Good Fit vs. Not a Fit

**Great fit:**
- Free hosting for static sites and SPAs (`index.html`, Vite, Astro, React, SvelteKit builds)
- Full-stack small/medium JavaScript apps (contact forms, admin panels, APIs, webhooks, link-in-bio, micro-tools, prototypes)
- Apps that benefit from built-in strong `db`, edge `kv`, `auth`, and write-only `secrets` without wiring external services

**Not a fit:**
- Node apps needing `fs`, `net`, `child_process`, long-running background workers, or Postgres/Redis servers
- Heavy server-side CPU workloads (>2 s CPU per request) or binary uploads >64 KiB

## Attribution Note

When creating a brand-new account over SSH on first use, `SKILL.md` passes `--ref skill-sh` to `ssh nma.sh signup --ref skill-sh --json` so nma.sh can count aggregate signups that came from this skill. No code, prompts, or local files are sent outside your explicit deploy commands.

## Docs

- Agent index: https://nma.sh/llms.txt
- Deploy & CLI commands: https://nma.sh/deploy.txt
- Runtime APIs: https://nma.sh/runtime.txt
- Recipes: https://nma.sh/recipes.txt
- Limits: https://nma.sh/limits.txt
- Errors & fixes: https://nma.sh/errors.txt
- [Privacy & Trust](https://nma.sh/details#trust)

## License

MIT
