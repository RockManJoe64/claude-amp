# claude-amp — Installation Guide

## Prerequisites

You need two tools installed on your system before installing claude-amp:

| Tool | Purpose | Install |
|---|---|---|
| **Astral UV** | Manages Python and numpy for signal processing | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| **Bun** | Runs the status line renderer | `curl -fsSL https://bun.sh/install \| bash` |

You do NOT need Python installed separately — UV downloads and manages its own Python automatically on first run.

Verify both are available:

```bash
uv --version    # should print 0.4.x or higher
bun --version   # should print 1.x or higher
```

---

## Step 1 — Install the plugin

Choose one of the following methods.

### From the official Claude Code marketplace

```
/plugin install claude-amp@claude-plugins-official
```

### From the claude-amp self-hosted marketplace

```
/plugin marketplace add <your-org>/claude-amp
/plugin install claude-amp@claude-amp
```

### For local development

Clone the repo and point Claude Code at it:

```bash
git clone https://github.com/<your-org>/claude-amp.git
claude --plugin-dir ./claude-amp
```

After installation, quit and restart Claude Code so the hooks load.

---

## Step 2 — Configure the status line

The plugin's hooks start processing signal data immediately, but the visualizer bars won't appear until you configure the status line. This is a one-time setup.

On the first session after install, claude-amp's `SessionStart` hook automatically writes a wrapper script to `~/.claude/amp_renderer.sh`. This script has the correct plugin paths baked in and stays stable across plugin updates.

Add the following to your settings file. Open `~/.claude/settings.json` (or `~/.claude/settings.local.json`) in your text editor and add:

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/amp_renderer.sh",
    "padding": 0
  }
}
```

If you already have other settings in this file, merge the `statusLine` key into the existing object — don't replace the whole file.

**If you already have a status line configured** (e.g., `ccstatusline`, `claude-statusline`, or a custom script), adding claude-amp's statusLine will replace it. Claude Code only supports one status line command at a time. If you want to keep your existing status line information alongside the visualizer, you can create a wrapper script that calls both and combines their output on one line.

---

## Step 3 — Restart and verify

Quit Claude Code and start a new session:

```bash
claude
```

You should see the visualizer bars appear at the bottom of your terminal. On first run, UV will spend 1–3 seconds downloading Python and installing numpy — this happens in the background (async hook) and is invisible to you. Subsequent sessions start instantly.

**To verify the hooks are loaded:**

Type `/hooks` in Claude Code. You should see `claude-amp` hooks registered for `SessionStart`, `Stop`, `UserPromptSubmit`, `PreToolUse`, and `PostToolUse`.

**To verify the status line:**

The bars should react when you send a message or when Claude responds. If you see a blank status line, check that `~/.claude/amp_renderer.sh` exists and is executable:

```bash
ls -la ~/.claude/amp_renderer.sh
```

If it doesn't exist, the `SessionStart` hook hasn't fired yet. Start a new Claude Code session and it will be created.

---

## Customizing parameters

claude-amp ships with sensible defaults. To customize the visualizer, create an override file at the plugin's persistent data directory. You can find this path by checking the wrapper script:

```bash
cat ~/.claude/amp_renderer.sh
# Look for the CLAUDE_PLUGIN_DATA export — that's your data directory
```

Then create `amp-config.toml` in that directory with only the values you want to change. For example:

```toml
# Faster bar decay and more visible idle state
[decay]
rate = 0.88
ambient_floor = 0.05

# Custom color theme
[color]
stops = [
    [30, 60, 180],
    [0, 200, 160],
    [240, 180, 40],
    [240, 80, 20],
    [200, 30, 30],
]
```

To see all available parameters and their defaults, read the reference file shipped with the plugin:

```bash
cat ~/.claude/amp_renderer.sh
# Find the CLAUDE_PLUGIN_ROOT path, then:
# cat <CLAUDE_PLUGIN_ROOT>/config/amp-defaults.toml
```

Restart Claude Code to pick up config changes.

---

## Updating

```
/plugin update claude-amp
```

Updates replace the plugin code but preserve your `amp-config.toml` and all runtime state — your customizations survive updates.

---

## Uninstalling

```
/plugin uninstall claude-amp
```

This removes both the plugin and its persistent data directory. You should also remove the `statusLine` entry from your `~/.claude/settings.json` and delete the wrapper script:

```bash
rm ~/.claude/amp_renderer.sh
```

---

## Troubleshooting

**Status line is blank:**
- Check that `~/.claude/amp_renderer.sh` exists and is executable (`chmod +x`)
- Check that Bun is installed and on your PATH
- Test the renderer directly: `echo '{}' | ~/.claude/amp_renderer.sh`

**Bars don't react to messages:**
- Run `/hooks` to confirm claude-amp hooks are registered
- Check that UV is installed: `uv --version`
- The first run installs numpy — wait a few seconds and try again

**"statusline skipped · restart to fix":**
- You need to accept the workspace trust dialog. Restart Claude Code and accept the prompt.

**Bars look garbled or misaligned:**
- Your terminal font may not include Unicode block elements. Try a Nerd Font or any monospace font with full Unicode support.
- On terminals with CJK locale, block characters may render as double-width. claude-amp auto-detects this and halves the bar count, but if it still looks wrong, set a fixed bar count in your `amp-config.toml`:

```toml
[display]
bar_count = 16
```

**UV takes a long time on first run:**
- The first hook invocation downloads Python and installs numpy. This takes 1–3 seconds on a fast connection and happens asynchronously — it won't block Claude Code. Subsequent runs use the cached environment.
