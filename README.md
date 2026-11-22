# nma-sh/skills

[![skills.sh](https://skills.sh/b/nma-sh/skills)](https://skills.sh/nma-sh/skills)

Official Agent Skill, Claude Code plugin, and MCP configuration for deploying static sites and JavaScript web apps to **[nma.sh](https://nma.sh/)** over SSH.

Each site runs in its own hardware-isolated Firecracker microVM with a live HTTPS URL in seconds, built-in database (`db`), edge `kv`, `crdt` counters, user authentication (`auth`), write-only `secrets`, and outbound `fetch()` — with no browser signup, OAuth tokens, or cloud dashboard setup required. Your SSH key is the account.

## Install

### Agent Skills CLI (`skills.sh` — Claude Code, Cursor, Codex, OpenCode, Gemini CLI, Copilot)

```bash
npx skills add nma-sh/skills
```

### Claude Code Plugin Marketplace

```text
/plugin marketplace add nma-sh/skills
/plugin install nma-sh@nma-sh
```

Or test locally from a clone:

```bash
claude --plugin-dir ./nma-skills
```

### MCP Server (zero CLI install)

Add this to any MCP-compatible client (Claude Desktop, Claude Code, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "nma": {
      "command": "ssh",
      "args": ["nma.sh", "mcp"]
    }
  }
}
```

Or in Claude Code:

```bash
claude mcp add nma -- ssh nma.sh mcp
```

Run `ssh nma.sh signup --json` once before connecting over MCP.

## What's Inside

| Path | Purpose |
|---|---|
| `skills/nma-sh/SKILL.md` | Core skill: when to use, SSH auth, single-file & static/framework deploys, built-ins, safety guardrails, and human handoff |
| `skills/nma-sh/references/runtime-cheatsheet.md` | Copy-paste patterns for `serve()`, `db`, `kv`, `auth`, `secrets`, `_redirects`, `_headers`, and platform limits |
| `skills/nma-sh/references/deploy-and-limits.md` | Full SSH command reference, contact-form + rate-limit recipes, framework build targets, and error codes |
| `skills.sh.json` | Repository page grouping for `skills.sh/nma-sh/skills` |
| `.claude-plugin/marketplace.json` | Claude Code plugin marketplace manifest |
| `.claude-plugin/plugin.json` | Claude Code plugin manifest |
| `.mcp.json` | Zero-install MCP server configuration (`ssh nma.sh mcp`) |

## Good Fit vs. Not a Fit

**Great fit:**
- Static sites and SPAs (`index.html`, Vite, Astro, React, SvelteKit builds)
- Full-stack small/medium JavaScript apps (contact forms, admin panels, APIs, webhooks, link-in-bio, micro-tools, prototypes)
- Apps that benefit from built-in strong `db`, edge `kv`, `auth`, and write-only `secrets` without wiring external services

**Not a fit:**
- Node apps needing `fs`, `net`, `child_process`, long-running background workers, or Postgres/Redis servers
- Heavy server-side CPU workloads (>2 s CPU per request) or binary uploads >64 KiB

## Attribution Note

When creating a brand-new account on first use, `SKILL.md` passes `--ref skill-sh` to `ssh nma.sh signup --ref skill-sh --json` so nma.sh can count aggregate signups that came from this skill. No code, prompts, or local files are sent outside your explicit deploy commands.

## Docs

- Agent index: https://nma.sh/llms.txt
- Deploy & CLI commands: https://nma.sh/deploy.txt
- Runtime APIs: https://nma.sh/runtime.txt
- Recipes: https://nma.sh/recipes.txt
- Limits: https://nma.sh/limits.txt
- Errors & fixes: https://nma.sh/errors.txt

## License

MIT
