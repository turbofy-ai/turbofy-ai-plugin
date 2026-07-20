# Turbofy Plugin Marketplace

This repository is a **plugin marketplace** for AI coding assistants — **Claude Code**, **Codex**, **Cursor**, or **OpenCode**. It currently ships **two** plugins you can install side by side:

| Plugin | What it is |
|---|---|
| **Turbofy** (`turbofy`) | Classic local MCP (`npx @turbofy-ai/mcp`) — pull/push apps and schema to `~/.turbofy`, TypeScript DSL workflows. |
| **Turbofy HTTP** (`turbofy-http`) | New HTTP MCP — schema/apps/flows as JSON, remote `block_type_*` sessions. No local `~/.turbofy` checkout. |

Install one or both. Their MCP server keys differ (`turbofy` vs `turbofy-http`), so they can run in parallel without clobbering each other.

---

## How to install

For **Claude Code**, **Codex**, and **Cursor**, the installation is the same two-step process:

1. Add this GitHub repository as a **plugin marketplace**.
2. Pick **Turbofy** and/or **Turbofy HTTP** from that marketplace and install.

The repository URL is the same in all three apps:

```
https://github.com/graphapi-io/turbofy-ai-plugin
```

To test a branch in Claude Code (CLI / slash command), pin a ref:

```
graphapi-io/turbofy-ai-plugin@story/http-mcp-skills-rewrite
```

(Do not use `#branch` in the desktop “Add marketplace” URL field — fragments get stripped. Prefer `owner/repo@branch`.)

Pick your app below for the exact clicks.

### Claude (desktop app)

1. Open the **Claude** desktop app.
2. Go to **Claude Code**.
3. Click **Customize**.
4. Click the **+** icon next to **Personal Plugins**.
5. Choose **Create Plugin** → **Add marketplace**.
6. Choose **Add from a repository**.
7. Click on **Select repository** and paste `https://github.com/graphapi-io/turbofy-ai-plugin` (or `graphapi-io/turbofy-ai-plugin@<branch>`) and click **Sync**.
8. Select **Turbofy** and/or **Turbofy HTTP MCP** and click **Install** (`+` button).
9. If you installed classic **Turbofy**, run **`/turbofy-setup`** once (see [Fewer permission prompts](#fewer-permission-prompts-claude) below). **Turbofy HTTP** does not need this.

That's it — from now on you can just use it.

### Codex

1. Open Codex.
2. Open the **Plugins** panel.
3. Next to the search input, click **Built by OpenAI**.
4. Click **Add more** — this opens the **Add marketplace** dialog.
5. Paste `https://github.com/graphapi-io/turbofy-ai-plugin` as the source.
6. Click **Save**.
7. Click **Built by OpenAI** next to the plugin search input again.
8. Select **Turbofy** and/or **Turbofy HTTP**.
9. Click the **+** button next to the plugin in the search results, then **Install**.
10. In a chat window, click the **+** button → **Plugins** → the plugin name, or type `@turbofy` / `@turbofy-http`.
11. The first time you use it, a browser window opens to authenticate with your Turbofy credentials.
12. Sign in — from then on you can work on your apps directly from Codex.

### Cursor (Agent window mode)

1. Open Cursor and switch to the **Agent window** mode.
2. Go to **Settings** → **Plugins**.
3. Paste `https://github.com/graphapi-io/turbofy-ai-plugin` into the **Search or Paste Link** input.
4. Select **Turbofy** and/or **Turbofy HTTP** from the results to install.

### OpenCode

OpenCode doesn't yet support installing this kind of plugin in one click, so you need to add the pieces by hand.

**Classic Turbofy (local MCP)**

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "turbofy": {
      "type": "local",
      "command": ["npx", "-y", "@turbofy-ai/mcp@latest"],
      "enabled": true
    }
  }
}
```

Copy [`plugins/turbofy/skills/`](plugins/turbofy/skills/) into `.opencode/skills/` or `~/.config/opencode/skills/`.

**Turbofy HTTP**

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "turbofy-http": {
      "type": "remote",
      "url": "https://mcp.alpha.turbofy.com/mcp",
      "enabled": true
    }
  }
}
```

Copy [`plugins/turbofy-http/skills/`](plugins/turbofy-http/skills/) into your OpenCode skills directory (use distinct folder names if you also keep the classic skills).

---

## What you get after installing

### Turbofy (classic)

- Local MCP via `npx @turbofy-ai/mcp` — pull/push apps and schema under `~/.turbofy`.
- Skills: **turbofy-platform**, **turbofy-apps**, **turbofy-blocks**, **turbofy-dynamic-fields**, **turbofy-flows**, plus **turbofy-setup** (Claude permissions for `~/.turbofy`).

### Turbofy HTTP

- HTTP MCP — JSON schema/apps/flows, `app_get` → `app_push`, remote `block_type_*` sessions.
- Skills under `plugins/turbofy-http/skills/` covering the same areas for the HTTP workflow (no `turbofy-setup`).

You don't need to remember skill names — your assistant picks the right one as you work. Prefer **one** plugin’s skill set for a given task so the agent doesn’t mix local-DSL and HTTP-JSON workflows.

---

## Fewer permission prompts (Claude — classic Turbofy only)

Classic Turbofy keeps pulled workspaces and apps under `~/.turbofy/`. Because that
folder lives outside your current project, Claude asks for permission every time
it reads or edits a file there — which gets noisy fast.

To fix it once, type `/` in a Claude chat and run the **`/turbofy-setup`** skill:

```
/turbofy-setup
```

It adds a small block to your `~/.claude/settings.json` that trusts the
`~/.turbofy` folder, so Claude can work on your Turbofy files without asking each
time. It's safe to run more than once — it won't duplicate anything. After it
finishes, run `/reload-plugins` or restart the app.

If you'd rather not run it, you can get the same effect by switching the session
to **Auto accept edits** mode from the selector in the message box — but you'd
need to do that each session, whereas `/turbofy-setup` is permanent.

> This step is **Claude Code only**, and only needed for classic **Turbofy**.
> **Turbofy HTTP** does not write under `~/.turbofy`.

---

## Troubleshooting

- **No custom icon in Claude Code.** Claude's plugin marketplace does not yet render custom plugin icons — all plugins show the same default placeholder ([anthropics/claude-code#28187](https://github.com/anthropics/claude-code/issues/28187)). The `icon` field is set in `.claude-plugin/` for when support lands.
- **Tool names.** Classic tools look like `mcp__turbofy__<tool>`; HTTP tools look like `mcp__turbofy-http__<tool>`. Skills are namespaced per plugin. In Codex address them as `@turbofy` / `@turbofy-http` (plugin `name` must match the directory under `plugins/`).
- **Nothing happened after install.** Restart the app or reload plugins. In Claude Code you can also run `/reload-plugins`.
- **The assistant doesn't seem to see Turbofy.** Make sure you're signed in, and that the expected MCP appears as connected (in Claude Code, run `/mcp`).
- **Claude keeps asking permission to touch `~/.turbofy`.** Run `/turbofy-setup` once (classic plugin only).
- **I want a clean reinstall.** Remove the plugin(s) from the plugin menu, then add the marketplace again and reinstall.
