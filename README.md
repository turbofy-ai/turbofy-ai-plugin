# Turbofy Plugin Marketplace

This repository is a **plugin marketplace** for AI coding assistants — **Claude Code**, **Codex**, **Cursor**, or **OpenCode**. It ships the **Turbofy HTTP** (`turbofy-http`) plugin: a hosted MCP with typed schema, app, flow, and block workflows in a persistent remote session tree, edited with `fs_*` tools.

---

## How to install

For **Claude Code**, **Codex**, and **Cursor**, the installation is the same two-step process:

1. Add this GitHub repository as a **plugin marketplace**.
2. Pick **Turbofy HTTP** from that marketplace and install.

The repository URL is the same in all three apps:

```
https://github.com/turbofy-ai/turbofy-ai-plugin
```

To test a branch in Claude Code (CLI / slash command), pin a ref:

```
turbofy-ai/turbofy-ai-plugin@story/http-mcp-skills-rewrite
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
7. Click on **Select repository** and paste `https://github.com/turbofy-ai/turbofy-ai-plugin` (or `turbofy-ai/turbofy-ai-plugin@<branch>`) and click **Sync**.
8. Select **Turbofy HTTP MCP** and click **Install** (`+` button).

That's it — from now on you can just use it.

### Codex

1. Open Codex.
2. Open the **Plugins** panel.
3. Next to the search input, click **Built by OpenAI**.
4. Click **Add more** — this opens the **Add marketplace** dialog.
5. Paste `https://github.com/turbofy-ai/turbofy-ai-plugin` as the source.
6. Click **Save**.
7. Click **Built by OpenAI** next to the plugin search input again.
8. Select **Turbofy HTTP**.
9. Click the **+** button next to the plugin in the search results, then **Install**.
10. In a chat window, click the **+** button → **Plugins** → the plugin name, or type `@turbofy-http`.
11. The first time you use it, a browser window opens to authenticate with your Turbofy credentials.
12. Sign in — from then on you can work on your apps directly from Codex.

### Cursor (Agent window mode)

1. Open Cursor and switch to the **Agent window** mode.
2. Go to **Settings** → **Plugins**.
3. Paste `https://github.com/turbofy-ai/turbofy-ai-plugin` into the **Search or Paste Link** input.
4. Select **Turbofy HTTP** from the results to install.

### OpenCode

OpenCode doesn't yet support installing this kind of plugin in one click, so you need to add the pieces by hand.

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "Turbofy": {
      "type": "remote",
      "url": "https://mcp.turbofy.com/mcp",
      "enabled": true
    }
  }
}
```

Copy [`plugins/turbofy-http/skills/`](plugins/turbofy-http/skills/) into `.opencode/skills/` or `~/.config/opencode/skills/`.

---

## What you get after installing

- HTTP MCP — typed schema, app, flow, and block sources in `workspaces/<environment>/<workspaceId>/`.
- Skills under `plugins/turbofy-http/skills/` covering the hosted pull/edit/push workflow.

You don't need to remember skill names — your assistant picks the right one as you work.

---

## Troubleshooting

- **No custom icon in Claude Code.** Claude's plugin marketplace does not yet render custom plugin icons — all plugins show the same default placeholder ([anthropics/claude-code#28187](https://github.com/anthropics/claude-code/issues/28187)). The `icon` field is set in `.claude-plugin/` for when support lands.
- **Tool names.** HTTP tools look like `mcp__Turbofy__<tool>` (the plugin's MCP server key is `Turbofy`). Skills are namespaced per plugin. In Codex address them as `@turbofy-http` (plugin `name` must match the directory under `plugins/`).
- **Nothing happened after install.** Restart the app or reload plugins. In Claude Code you can also run `/reload-plugins`.
- **The assistant doesn't seem to see Turbofy.** Make sure you're signed in, and that the expected MCP appears as connected (in Claude Code, run `/mcp`).
- **I want a clean reinstall.** Remove the plugin from the plugin menu, then add the marketplace again and reinstall.
