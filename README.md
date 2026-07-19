# Turbofy Plugin

This plugin connects your AI coding assistant — **Claude Code**, **Codex**, **Cursor**, or **OpenCode** — to Turbofy. Once installed, your assistant can help you build Turbofy apps, edit pages and blocks, and work with your data, all from inside the editor.

The plugin ships the **Turbofy HTTP MCP** (currently the alpha endpoint) plus skills that describe the JSON / remote-session workflows. There is no local workspace checkout under `~/.turbofy`.

---

## How to install

For **Claude Code**, **Codex**, and **Cursor**, the installation is the same two-step process:

1. Add this GitHub repository as a **plugin marketplace**.
2. Pick the **Turbofy** plugin from that marketplace and install it.

The repository URL is the same in all three apps:

```
https://github.com/graphapi-io/turbofy-ai-plugin
```

Pick your app below for the exact clicks.

### Claude (desktop app)

1. Open the **Claude** desktop app.
2. Go to **Claude Code**.
3. Click **Customize**.
4. Click the **+** icon next to **Personal Plugins**.
5. Choose **Create Plugin** → **Add marketplace**.
6. Choose **Add from a repository**.
7. Click on **Select repository** and paste `https://github.com/graphapi-io/turbofy-ai-plugin` and click **Sync**.
8. Select the **Turbofy** plugin and click **Install** (`+` button).

That's it — from now on you can just use it. The first time tools run, authenticate with your Turbofy credentials when prompted.

### Codex

1. Open Codex.
2. Open the **Plugins** panel.
3. Next to the search input, click **Built by OpenAI**.
4. Click **Add more** — this opens the **Add marketplace** dialog.
5. Paste `https://github.com/graphapi-io/turbofy-ai-plugin` as the source.
6. Click **Save**.
7. Click **Built by OpenAI** next to the plugin search input again.
8. Select **Turbofy**.
9. Click the **+** button next to the Turbofy plugin in the search results.
10. Click **Install Turbofy**.
11. In a chat window, click the **+** button → **Plugins** → **Turbofy**, or simply type `@Turbofy` to use the plugin.
12. The first time you use it, a browser window opens to authenticate with your Turbofy credentials.
13. Sign in — from then on you can work on your apps directly from Codex.

### Cursor (Agent window mode)

1. Open Cursor and switch to the **Agent window** mode.
2. Go to **Settings** → **Plugins**.
3. Paste `https://github.com/graphapi-io/turbofy-ai-plugin` into the **Search or Paste Link** input.
4. Select **Turbofy** from the results to install it.

### OpenCode

OpenCode doesn't yet support installing this kind of plugin in one click, so you need to add the two pieces by hand.

**1. Add the Turbofy connection (MCP server)**

Open the OpenCode config file:

- For a single project: `opencode.json` in that project's root.
- For all your projects: `~/.config/opencode/opencode.json`.

Add (or merge) this block:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "turbofy": {
      "type": "remote",
      "url": "https://tdsbhitua7.execute-api.eu-central-1.amazonaws.com/mcp",
      "enabled": true
    }
  }
}
```

**2. Add the Turbofy skills**

Copy the folders from this repo's [`skills/`](skills/) into one of these locations:

- Project-only: `.opencode/skills/`
- All projects: `~/.config/opencode/skills/`

For example:

```bash
mkdir -p .opencode/skills
cp -R /path/to/turbofy-ai-plugin/skills/* .opencode/skills/
```

Restart OpenCode and the `turbofy-*` skills will show up as available skills.

---

## What you get after installing

- A direct HTTP connection from your assistant to Turbofy (the **Turbofy MCP**), so it can list organizations and workspaces, read/write schema and flows as JSON, inspect and push apps, manage data, and edit block React sources in a remote build session.
- Built-in **skills** that teach your assistant how Turbofy works. They turn on automatically when relevant:
  - **turbofy-platform** — organizations, workspaces, schema JSON, data CRUD.
  - **turbofy-apps** — apps as JSON manifests via `app_get` → `app_push`.
  - **turbofy-blocks** — remote `block_type_*` session + React component rules.
  - **turbofy-dynamic-fields** — server-side `$$std` scripts for dynamic content.
  - **turbofy-flows** — flow JSON declarations via `flow_upsert`.

You don't need to remember these names — your assistant picks the right one as you work.

---

## Troubleshooting

- **No custom icon in Claude Code.** Claude's plugin marketplace does not yet render custom plugin icons — all plugins show the same default placeholder ([anthropics/claude-code#28187](https://github.com/anthropics/claude-code/issues/28187)). The `icon` field is set in `.claude-plugin/` for when support lands.
- **Tool names look like `mcp__turbofy__<tool>`.** An installed plugin's MCP tools are named `mcp__{serverKey}__{tool}` from the `.mcp.json` server key (`turbofy`) — the plugin name no longer appears in tool names. Skills are namespaced `turbofy:<skill>`. In Codex the plugin is addressable as `@turbofy` (the plugin `name`, which must match its directory `plugins/turbofy/`). The UI shows "Turbofy MCP" via `displayName`.
- **Nothing happened after install.** Restart the app or reload plugins. In Claude Code you can also run `/reload-plugins`.
- **The assistant doesn't seem to see Turbofy.** Make sure you're signed in to Turbofy in your assistant, and that the `turbofy` MCP appears as connected (in Claude Code, run `/mcp`).
- **I want a clean reinstall.** Remove the plugin from the plugin menu, then add the marketplace again and reinstall.
- **Still seeing references to `~/.turbofy` or `npx @turbofy-ai/mcp`.** That was the old local MCP. This plugin uses the HTTP MCP URL in `mcp.json` — update/reinstall the plugin if an older copy is cached.
