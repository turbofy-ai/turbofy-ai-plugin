# Turbofy Plugin

This plugin connects your AI coding assistant — **Claude Code**, **Codex**, **Cursor**, or **OpenCode** — to Turbofy. Once installed, your assistant can help you build Turbofy apps, edit pages and blocks, and work with your data, all from inside the editor.

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
9. In a chat, run **`/turbofy-setup`** once (see [Fewer permission prompts](#fewer-permission-prompts-claude) below). This stops Claude from asking for permission every time it works on your Turbofy files.

That's it — from now on you can just use it.

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
      "type": "local",
      "command": ["npx", "-y", "@turbofy-ai/mcp@latest"],
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

- A direct connection from your assistant to your Turbofy workspace (the **Turbofy MCP server**), so it can list your organizations and workspaces, pull and push apps, edit your schema, and manage data.
- Four built-in **skills** that teach your assistant how Turbofy works. They turn on automatically when relevant:
  - **turbofy-platform** — the big picture: organizations, workspaces, environments, and the data schema.
  - **turbofy-apps** — building and editing Turbofy apps: pages, blocks, localization.
  - **turbofy-blocks** — writing the React components that power block types.
  - **turbofy-dynamic-fields** — writing the small server-side scripts that fill in dynamic content.
  - **turbofy-flows** - building and editing Turbofy flows.

You don't need to remember these names — your assistant picks the right one as you work.

---

## Fewer permission prompts (Claude)

Turbofy keeps your pulled workspaces and apps under `~/.turbofy/`. Because that
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

> This step is **Claude Code only** — it configures Claude's settings file.
> Cursor, Codex, and OpenCode manage permissions their own way.

---

## Troubleshooting

- **No custom icon in Claude Code.** Claude's plugin marketplace does not yet render custom plugin icons — all plugins show the same default placeholder ([anthropics/claude-code#28187](https://github.com/anthropics/claude-code/issues/28187)). The `icon` field is set in `.claude-plugin/` for when support lands.
- **Tool names look like `mcp__turbofy__<tool>`.** An installed plugin's MCP tools are named `mcp__{serverKey}__{tool}` from the `.mcp.json` server key (`turbofy`) — the plugin name no longer appears in tool names. Skills are namespaced `turbofy:<skill>`. In Codex the plugin is addressable as `@turbofy` (the plugin `name`, which must match its directory `plugins/turbofy/`). The UI shows "Turbofy MCP" via `displayName`.
- **Nothing happened after install.** Restart the app or reload plugins. In Claude Code you can also run `/reload-plugins`.
- **The assistant doesn't seem to see Turbofy.** Make sure you're signed in to Turbofy in your assistant, and that the `turbofy` MCP appears as connected (in Claude Code, run `/mcp`).
- **Claude keeps asking permission to touch `~/.turbofy`.** Run `/turbofy-setup` once (see [Fewer permission prompts](#fewer-permission-prompts-claude)).
- **I want a clean reinstall.** Remove the plugin from the plugin menu, then add the marketplace again and reinstall.

