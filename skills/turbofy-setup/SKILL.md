---
name: turbofy-setup
description: "Use when Claude Code keeps asking permission to read or edit Turbofy files. One-time setup: adds ~/.turbofy to allowed paths in ~/.claude/settings.json. Triggers: '/turbofy-setup', 'stop permission prompts', 'configure Turbofy permissions'. Claude Code only."
disable-model-invocation: false
---

# Turbofy permission setup

Your goal is to stop Claude Code from prompting the user for permission every
time it reads or edits files under `~/.turbofy/` (where the Turbofy MCP keeps
your pulled workspaces and apps). You do this by adding a small block to the
user's **user-level** settings file at `~/.claude/settings.json`.

This is a **Claude Code** action — it edits `~/.claude/settings.json`. If the
user is running a different assistant (Cursor, Codex, OpenCode), tell them this
setup doesn't apply and stop.

If you reached this skill on your own (the user didn't explicitly run it),
briefly confirm with the user before editing their settings file.

This skill is **idempotent** — running it again must not create duplicates.

Follow these steps exactly:

## 1. Resolve paths

Run:

```bash
echo "HOME=$HOME"; ls -la "$HOME/.claude/settings.json" 2>/dev/null || echo "NO_SETTINGS_FILE"
```

The two paths you care about are:

- Settings file: `$HOME/.claude/settings.json`
- Turbofy directory: `$HOME/.turbofy`

Make sure the Turbofy directory exists so the path is valid (harmless if it
already does):

```bash
mkdir -p "$HOME/.turbofy"
```

## 2. Read the current settings

- If `~/.claude/settings.json` exists, **Read** it and parse it as JSON.
- If it does **not** exist, treat the current settings as an empty object `{}`.
- If the file exists but is **not valid JSON**, STOP. Do not overwrite it.
  Tell the user their `~/.claude/settings.json` has a syntax error they need to
  fix first, show them the parse error, and end the skill.

## 3. Merge in the Turbofy permissions

You must end up with these entries present under `permissions`. **Preserve every
existing key and value** — only add what is missing, and never add an entry that
is already there (compare as exact strings):

- `permissions.additionalDirectories` must contain `"~/.turbofy"`.
- `permissions.allow` must contain all three of:
  - `"Read(~/.turbofy/**)"`
  - `"Edit(~/.turbofy/**)"`
  - `"Write(~/.turbofy/**)"`

If `permissions`, `additionalDirectories`, or `allow` don't exist yet, create
them as needed. The result should look like this once merged (other existing
keys stay untouched):

```json
{
  "permissions": {
    "additionalDirectories": ["~/.turbofy"],
    "allow": [
      "Read(~/.turbofy/**)",
      "Edit(~/.turbofy/**)",
      "Write(~/.turbofy/**)"
    ]
  }
}
```

## 4. Write it back

Write the merged JSON back to `~/.claude/settings.json` with 2-space
indentation. If nothing was missing (everything already present), do **not**
rewrite the file — just report that setup was already complete.

## 5. Report

Tell the user, briefly:

- What you added (or that it was already configured).
- That they should **reload plugins** (`/reload-plugins`) or restart the app for
  the change to take effect.
- That Claude will no longer ask for permission when working in their Turbofy
  files. If they still see occasional prompts, they can also switch the session
  to **Auto accept edits** mode from the selector in the message box.

Keep the final summary short and friendly — this is a one-time setup step.
