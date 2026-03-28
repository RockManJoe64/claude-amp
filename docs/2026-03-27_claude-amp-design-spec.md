# claude-amp — Design Spec

## Overview

A Claude Code status line visualizer that transforms hook payload text into a real-time Winamp-style frequency bar display. Text content is treated as a discrete signal, decomposed via DFT into frequency bins, and rendered as colored Unicode block bars with decay, peak dots, and a context-driven beat oscillator.

Packaged as a **Claude Code plugin** (`claude-amp`) for one-command installation and marketplace distribution.

---

## Architecture

Split architecture: **Python** (managed by Astral UV) for signal processing (async hooks), **TypeScript/Bun** for rendering (synchronous status line).

```
Hook events (Stop, UserPromptSubmit, PreToolUse, PostToolUse)
        │
        ▼
┌─────────────────────┐     atomic write     ┌──────────────────────────┐
│   Python processor   │ ──────────────────▶  │  amp-state.json          │
│   (uv run, async)    │   (os.replace)       │  ${CLAUDE_PLUGIN_DATA}/  │
└─────────────────────┘                       └──────────────────────────┘
                                                        │
                                                   reads (cached)
                                                        │
                                                        ▼
                                              ┌─────────────────────┐
                                              │  Bun renderer        │
                                              │  (statusLine tick)   │
                                              └─────────────────────┘
                                                        │
                                                        ▼
                                              Claude Code status line
```

---

## Component 1 — Python signal processor

### Trigger

Fires on `Stop`, `UserPromptSubmit`, `PreToolUse`, and `PostToolUse` hooks. All configured as `async: true`.

### Data source

Reads structured JSON from **hook stdin** (not transcript JSONL). Each hook type provides the relevant payload content directly. This eliminates transcript path resolution fragility and concurrent write risks.

Future expansion to additional hook types only requires mapping their stdin schema to the same processing pipeline.

### Plugin structure

The visualizer is packaged as a Claude Code plugin following the standard plugin directory layout. The plugin contains hooks (signal processing), scripts (processor and renderer executables), and configuration.

**Plugin directory layout (`${CLAUDE_PLUGIN_ROOT}`):**

```
claude-amp/
├── .claude-plugin/
│   ├── plugin.json              # Plugin manifest (name, version, description)
│   └── marketplace.json         # Self-hosted marketplace catalog (source: "./")
├── hooks/
│   └── hooks.json               # Hook event configuration (Stop, UserPromptSubmit, etc.)
├── scripts/
│   ├── amp_processor.py         # Python signal processing entry point
│   ├── amp_renderer.ts          # Bun renderer entry point
│   ├── session_start.sh         # SessionStart hook — writes wrapper script on first run
│   ├── pyproject.toml           # UV project definition — pins Python + numpy
│   └── uv.lock                  # Lockfile for reproducible installs
├── config/
│   └── amp-defaults.toml        # Read-only reference — all parameters with defaults and docs
├── README.md                    # Usage documentation, including manual statusLine setup
└── LICENSE                      # License file
```

**Persistent data directory (`${CLAUDE_PLUGIN_DATA}`):**

Claude Code provides `${CLAUDE_PLUGIN_DATA}` as the plugin's persistent data directory. Unlike `${CLAUDE_PLUGIN_ROOT}` (which changes on each plugin update), `${CLAUDE_PLUGIN_DATA}` survives plugin updates and is the correct location for runtime state, user configuration, and installed dependencies:

```
${CLAUDE_PLUGIN_DATA}/
├── amp-state.json               # Processor → renderer state (atomic writes)
├── amp-config.toml              # User overrides (optional, created by user)
└── .venv/                       # UV virtual environment (created on first run)
```

This separation means plugin updates never overwrite user configuration or force a numpy reinstall. The UV `.venv` lives here because `${CLAUDE_PLUGIN_ROOT}` changes on each update — if the venv lived inside the plugin root, every update would trigger a cold reinstall. Uninstalling the plugin via `/plugin uninstall` removes both the plugin root and the data directory.

**`.claude-plugin/plugin.json`:**

```json
{
  "name": "claude-amp",
  "version": "1.0.0",
  "description": "Winamp-style frequency bar visualizer for the Claude Code status line. Transforms hook payloads into real-time DFT-driven bar animations with heat color gradients and context-aware beat oscillation.",
  "author": {
    "name": "Trespass Studios"
  },
  "keywords": ["visualizer", "status-line", "hooks", "signal-processing"]
}
```

**`.claude-plugin/marketplace.json`:**

The plugin repo doubles as its own marketplace (following the Superpowers pattern), allowing both direct install and marketplace-based install from the same repo:

```json
{
  "description": "Claude Amp — Winamp-style status line visualizer for Claude Code",
  "owner": {
    "name": "Trespass Studios"
  },
  "plugins": [
    {
      "name": "claude-amp",
      "description": "Winamp-style frequency bar visualizer for the Claude Code status line",
      "version": "1.0.0",
      "source": "./",
      "author": {
        "name": "Trespass Studios"
      }
    }
  ]
}
```

### Installation

**Claude Code Official Marketplace** (once accepted):

```
/plugin install claude-amp@claude-plugins-official
```

**Via self-hosted marketplace** (the plugin repo itself):

```
/plugin marketplace add <your-org>/claude-amp
/plugin install claude-amp@claude-amp
```

**Local development:**

```
claude --plugin-dir ./claude-amp
```

**Update:**

```
/plugin update claude-amp
```

### Status line registration

The `statusLine` field is a user setting, not a plugin hook — plugins cannot auto-register status line scripts. The README documents this setup step. The plugin includes a `SessionStart` hook that writes a wrapper script to `~/.claude/amp_renderer.sh` on first run, with the correct `${CLAUDE_PLUGIN_ROOT}` and `${CLAUDE_PLUGIN_DATA}` paths baked in. The user then adds to their settings:

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/amp_renderer.sh",
    "padding": 0
  }
}
```

This stable path avoids the user needing to know the plugin's internal install location, which changes on each plugin update.

**Status line stdin data:** Claude Code sends JSON to the statusLine script on every tick via stdin, including the `context_window.used_percentage` field. The renderer extracts `contextPct` directly from this stdin data rather than relying on the Python processor to compute it. This means the context beat oscillator always reflects the live context window state, not a stale value from the last hook fire.

### Dependency management

The Python processor is managed by **Astral UV**, which handles Python version resolution, virtual environment creation, and dependency installation automatically. UV eliminates manual `pip install` steps, `--break-system-packages` workarounds, and version mismatch issues across macOS, Linux, and Windows.

The UV virtual environment is created in `${CLAUDE_PLUGIN_DATA}/.venv` rather than inside the plugin root, so it persists across plugin updates.

**`scripts/pyproject.toml`:**

```toml
[project]
name = "claude-amp-processor"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "numpy>=1.24",
]

[tool.uv]
python-preference = "managed"
```

Setting `python-preference = "managed"` tells UV to download and manage its own Python installation if the system Python doesn't meet the version requirement. This means the user doesn't need Python pre-installed at all — UV handles everything.

**First run behavior:** The first `uv run` invocation creates the `.venv`, resolves dependencies, and installs numpy. This takes 1–3 seconds on a cold cache. Every subsequent invocation reuses the cached venv with zero overhead beyond normal Python startup (~80–150ms, invisible since hooks are async).

**Lockfile:** `uv.lock` is committed alongside the project files and ensures identical dependency versions across machines. Running `uv lock` regenerates it after any `pyproject.toml` change.

### Signal processing pipeline

#### Step 1 — Character sampling with mean-centering

Convert the text payload into a normalized, mean-centered sample array:

```python
import numpy as np

raw = np.array([c / 128.0 for c in payload.encode('utf-8', errors='replace')])
samples = raw - np.mean(raw)  # mean-center to remove DC bias
```

Mean-centering eliminates the DC offset caused by ASCII code clustering (lowercase letters at 0.75–0.95, whitespace at 0.07–0.25) and improves spectral differentiation between content types.

#### Step 2 — Hann window

Apply a Hann window before the DFT to prevent spectral leakage at payload boundaries, particularly important for truncated or partial payloads:

```python
window = np.hanning(len(samples))
windowed = samples * window
```

Cost: one multiply per sample. No harm on clean-boundary payloads, prevents artifacts on truncated ones.

#### Step 3 — DFT via FFT

```python
magnitudes = np.abs(np.fft.rfft(windowed))
```

Produces `N/2 + 1` frequency bins where N is the sample count.

#### Step 4 — Logarithmic binning with interpolation

Compress DFT bins into `NBARS` display bars using logarithmic mapping:

```python
def log_bin(magnitudes, nbars):
    n = len(magnitudes)
    if n == 0:
        return np.zeros(nbars)

    bars = np.zeros(nbars)
    for i in range(nbars):
        lo = n ** (i / nbars)
        hi = n ** ((i + 1) / nbars)

        lo_idx = int(np.floor(lo))
        hi_idx = int(np.floor(hi))

        if lo_idx >= n:
            lo_idx = n - 1
        if hi_idx >= n:
            hi_idx = n - 1

        if lo_idx == hi_idx:
            # Interpolate: fractional position within this single bin
            frac = lo - np.floor(lo)
            if lo_idx + 1 < n:
                bars[i] = magnitudes[lo_idx] * (1 - frac) + magnitudes[lo_idx + 1] * frac
            else:
                bars[i] = magnitudes[lo_idx]
        else:
            # Take max magnitude across the bin range (preserves spikes)
            bars[i] = np.max(magnitudes[lo_idx:hi_idx + 1])

    # Normalize to 0–1
    peak = np.max(bars)
    if peak > 0:
        bars = bars / peak
    return bars
```

When FFT bins < NBARS (short payloads like bash commands), interpolation fills empty bars rather than producing duplicates or zeros.

#### Step 5 — Shannon entropy calculation

Byte-level Shannon entropy of the raw payload, used to drive the heat color gradient:

```python
from collections import Counter
import math

def shannon_entropy(data: bytes) -> float:
    if len(data) == 0:
        return 0.0
    counts = Counter(data)
    length = len(data)
    return -sum((c / length) * math.log2(c / length) for c in counts.values())
```

**Expected ranges by content type:**

| Content type | Entropy (bits/byte) |
|---|---|
| Repetitive JSON (repeated keys) | 3.5–4.0 |
| English prose | 4.0–4.5 |
| Well-structured Python/TS code | 4.2–4.8 |
| Minified/obfuscated code | 5.0–5.5 |
| High-symbol-diversity output | 5.5–6.5 |

**Heat ramp mapping:** Clamp to [3.0, 6.0] range. Below 3.0 pins to full blue (cold/repetitive). Above 6.0 pins to full red (hot/chaotic). This keeps the gradient visually active across typical payloads.

#### Step 6 — Character class distribution (8 classes)

The class schema is balanced for the full range of Claude Code usage — not just code, but planning documents, design specs, architecture decisions, documentation, and conversational sessions.

| Class | Characters | Signal |
|---|---|---|
| Whitespace | space, `\t` | Indentation density, formatting intent |
| Line breaks | `\n`, `\r` | Paragraph density — short lines (code, lists) vs long paragraphs (prose, plans) |
| Lowercase | a–z | Prose, identifiers, keywords (baseline) |
| Uppercase | A–Z | Headers, constants, acronyms, emphasis |
| Digits | 0–9 | Numeric data, versions, IDs |
| Markdown/structural | ``#*_->[]()!`` | Headers, bullets, links, emphasis — the skeleton of planning and documentation content |
| Code syntax | ``{}=<>;:&\|^~+/`` | Brackets, operators, delimiters — code-specific structure |
| Punctuation/other | `.,?!'"` and everything else | Sentence structure, conversational tone |

**Design rationale:** Claude Code sessions frequently involve non-code content — planning, design specs, implementation plans, architecture decision records, markdown documentation. The old schema (which grouped all punctuation together and gave operators/brackets their own classes) produced nearly identical fingerprints for all prose-like content. Splitting **markdown/structural** from **code syntax** is the key distinction: a design doc heavy in `# ## - * [] ()` looks fundamentally different from a TypeScript file heavy in `{} => ; : |`. Separating **line breaks** from **whitespace** captures the difference between bullet-heavy structured output (high line-break frequency, short lines) and long-form prose (low line-break frequency, long paragraphs).

**Fingerprints by content type:**

| Content type | Dominant classes | Secondary classes |
|---|---|---|
| Python/TS code | Code syntax, whitespace | Lowercase, line breaks |
| JSON/YAML config | Code syntax, digits | Line breaks, lowercase |
| Markdown design doc | Markdown/structural, lowercase | Line breaks, uppercase |
| Planning conversation | Lowercase, punctuation/other | Whitespace (minimal) |
| Implementation plan (bullets) | Markdown/structural, line breaks | Lowercase, digits |
| Architecture decision record | Uppercase, markdown/structural | Lowercase, punctuation/other |
| Bash commands | Lowercase, code syntax | Digits |
| Bash stdout (logs) | Line breaks, digits | Lowercase, punctuation/other |

Counts are normalized to frequencies. Char-class data is NOT blended into bar amplitude. Instead it is written to the state file as a separate array for the renderer to use as a **color saturation/brightness modifier**, preserving both signals without interference.

#### Step 7 — Write state file (atomic)

Write to the persistent data directory (`${CLAUDE_PLUGIN_DATA}/`), creating it if it doesn't exist. Use a temporary file then atomically replace:

```python
import os, json
from pathlib import Path

data_dir = Path(os.environ.get("CLAUDE_PLUGIN_DATA", Path.home() / ".claude" / "viz"))
data_dir.mkdir(parents=True, exist_ok=True)
state_path = data_dir / "amp-state.json"

state = { ... }
tmp_path = str(state_path) + '.tmp'
with open(tmp_path, 'w') as f:
    json.dump(state, f)
os.replace(tmp_path, str(state_path))  # atomic on POSIX
```

---

## State file contract

`${CLAUDE_PLUGIN_DATA}/amp-state.json`

```json
{
  "version": 1,
  "inputBars": [0.8, 0.4, 0.9, "...N floats"],
  "outputBars": [0.3, 0.7, 0.2, "...N floats"],
  "inputEntropy": 4.3,
  "outputEntropy": 4.7,
  "inputCharClass": [0.18, 0.45, 0.05, 0.03, 0.12, 0.08, 0.06, 0.03],
  "outputCharClass": [0.15, 0.40, 0.08, 0.07, 0.10, 0.10, 0.07, 0.03],
  "inputChars": 48230,
  "outputChars": 127540,
  "lastTimestamp": 1711540800000,
  "hookType": "Stop"
}
```

**Schema rules:**

- `version` — integer, must match renderer's expected version or renderer falls back to decay-only mode
- `lastTimestamp` — Unix epoch milliseconds for staleness comparison
- `inputCharClass` / `outputCharClass` — 8-element float arrays, one per character class, normalized to sum to 1.0
- `inputChars` / `outputChars` — cumulative character counts across all hook invocations in this session. The Python processor accumulates these across calls (persisted in the state file itself — read previous value, add current payload length, write back). The renderer uses successive snapshots to compute character velocity (chars/sec), which serves as a proxy for token velocity.
- `hookType` — string identifying which hook produced this state, for potential future per-hook rendering behavior

**Note:** `contextPct` is NOT in the state file. The renderer reads it directly from the statusLine stdin JSON (`context_window.used_percentage`), which Claude Code provides on every tick. This gives the context beat oscillator and the context usage display a live signal rather than a stale value from the last hook fire.

**Staleness threshold:** If `Date.now() - lastTimestamp > 30_000` (30 seconds), the renderer enters decay-only mode — bars decay toward the ambient noise floor with no new signal data, context beat continues. Token velocity displays zero.

---

## Component 2 — Bun/TypeScript renderer

### Dependencies

**One runtime dependency.** The renderer uses raw ANSI escape sequences and Bun native APIs, plus `smol-toml` for config parsing.

| Need | Solution |
|---|---|
| Color output | Pre-computed ANSI escape strings (24-bit or 256-color, detected at startup) |
| File reading | `Bun.file()` native API |
| Terminal width | `process.stdout.columns` |
| Unicode blocks | Hardcoded string `' ▁▂▃▄▅▆▇█'` indexed by amplitude |
| Config parsing | `smol-toml` — zero-dependency TOML parser, minimal startup cost |

### Terminal compatibility

The renderer must produce correct output across a wide range of terminals, fonts, and platforms. Capability detection happens once at startup and selects the appropriate rendering strategy.

#### Color capability detection

```typescript
function detectColorMode(): 'truecolor' | '256' | '16' {
  const ct = process.env.COLORTERM;
  if (ct === 'truecolor' || ct === '24bit') return 'truecolor';

  // WT_SESSION indicates Windows Terminal (truecolor capable)
  if (process.env.WT_SESSION) return 'truecolor';

  // TERM_PROGRAM detection for macOS
  const tp = process.env.TERM_PROGRAM;
  if (tp === 'iTerm.app' || tp === 'ghostty' || tp === 'Tabby') return 'truecolor';

  // Alacritty sets COLORTERM but also check TERM
  if (process.env.TERM?.includes('alacritty')) return 'truecolor';

  // Check for 256-color support
  const term = process.env.TERM || '';
  if (term.includes('256color')) return '256';

  // macOS Terminal.app — only supports 256 colors, does NOT set COLORTERM
  if (tp === 'Apple_Terminal') return '256';

  // Conservative fallback
  return '256';
}
```

**Terminal color support matrix:**

| Terminal | Platform | Color support | Detection method |
|---|---|---|---|
| iTerm2 | macOS | 24-bit | `TERM_PROGRAM=iTerm.app` |
| Terminal.app | macOS | 256-color only | `TERM_PROGRAM=Apple_Terminal` |
| Ghostty | macOS, Linux | 24-bit | `COLORTERM=truecolor` |
| Alacritty | macOS, Linux, Windows | 24-bit | `COLORTERM=truecolor` |
| Tabby | Cross-platform | 24-bit | `TERM_PROGRAM=Tabby` |
| GNOME Terminal | Linux | 24-bit (3.18+) | `COLORTERM=truecolor` |
| Konsole (KDE) | Linux | 24-bit | `COLORTERM=truecolor` |
| Windows Terminal | Windows | 24-bit | `WT_SESSION` set |

#### Palette construction by color mode

```typescript
function buildPalette(stops: Color[], size: number, mode: ColorMode): string[] {
  const interpolated = interpolateStops(stops, size);

  if (mode === 'truecolor') {
    // 24-bit: \x1b[38;2;R;G;Bm
    return interpolated.map(c => `\x1b[38;2;${c.r};${c.g};${c.b}m`);
  }

  // 256-color: map each RGB to nearest 6x6x6 cube entry
  // Cube index: 16 + 36*r + 6*g + b (where r,g,b are 0–5)
  return interpolated.map(c => {
    const r5 = Math.round(c.r / 255 * 5);
    const g5 = Math.round(c.g / 255 * 5);
    const b5 = Math.round(c.b / 255 * 5);
    const idx = 16 + 36 * r5 + 6 * g5 + b5;
    return `\x1b[38;5;${idx}m`;
  });
}
```

The 256-color cube has enough range for the blue→teal→amber→red heat gradient. Visual fidelity drops (banding is visible) but the gradient remains functional and readable.

#### Unicode block character safety

The bar characters `▁▂▃▄▅▆▇█` (U+2581–2588) are in the Block Elements Unicode block and are universally supported across all target terminals and monospace fonts, including Nerd Font variants.

**Ambiguous width handling:** Some terminals in CJK-aware locales may treat block elements as double-width characters, rendering each bar as two cells and breaking the layout. Detect this at startup:

```typescript
function detectAmbiguousWidth(): boolean {
  // If locale suggests CJK, check terminal-specific settings
  const lang = process.env.LANG || '';
  const cjk = /\.(UTF-8|utf8)/.test(lang) &&
               /^(ja|ko|zh)/.test(lang);

  // Ghostty and Alacritty default to narrow (safe)
  // GNOME Terminal respects locale (may be wide in CJK locales)
  // Windows Terminal defaults to narrow
  return cjk && !process.env.WT_SESSION &&
         process.env.TERM_PROGRAM !== 'ghostty';
}
```

If ambiguous width is detected, halve the bar count (`N = Math.floor(N / 2)`) so the output still fits the terminal. This is a conservative fallback — most users won't hit it.

**Font fallback and glyph gaps:** Some fonts without native block element glyphs fall back to a secondary font with different metrics, producing hairline gaps between adjacent bars. Nerd Fonts generally handle this correctly since they're patched to include block elements at consistent metrics. For non-Nerd-Font users, the visual effect is cosmetic (thin gaps between bars) rather than broken — no mitigation needed beyond documenting the recommended font setup.

#### Peak dot character selection

`▔` (U+2594, upper one-eighth block) has inconsistent rendering across terminals — some fonts render it narrower than the full block characters, producing misaligned output. Use a **color-only peak strategy** instead:

When `peakBars[i]` is significantly above `bars[i]`, render the bar at the **peak amplitude level** using a brightened version of the bar's color (1.4× RGB, clamped to 255). When `peakBars[i]` converges back to `bars[i]`, the brightness boost fades. This produces the Winamp peak-hold visual effect without relying on any glyph that might render inconsistently.

```typescript
function renderBar(amplitude: number, peakAmplitude: number, baseColor: string, brightColor: string): string {
  // If peak is significantly above current bar, render at peak height with bright color
  const showPeak = peakAmplitude - amplitude > 0.08;
  const displayAmp = showPeak ? peakAmplitude : amplitude;
  const color = showPeak ? brightColor : baseColor;
  const blockIdx = Math.min(8, Math.floor(displayAmp * 9));
  return color + BLOCKS[blockIdx];
}
```

#### Line clearing

Terminals handle partial line overwrites differently. Always emit `\x1b[K` (erase to end of line) after the bar output to prevent leftover characters from a previous wider render:

```typescript
const output = bars.map(renderBar).join('') + '\x1b[0m\x1b[K';
process.stdout.write('\r' + output);
```

### Startup initialization

Performed once when the status line process starts:

```typescript
// 0. Load configuration (defaults + user overrides)
const config = loadConfig();

// 1. Capability detection
const colorMode = detectColorMode();
const ambiguousWidth = detectAmbiguousWidth();

// 2. Terminal width detection — dynamic bar count
const RESERVED_COLS = config.display.reserved_cols;
let N = config.display.bar_count > 0
  ? config.display.bar_count
  : Math.min(config.display.max_bars, Math.max(config.display.min_bars,
      process.stdout.columns - RESERVED_COLS));
if (ambiguousWidth) N = Math.floor(N / 2);

// 3. Pre-computed heat palettes (base + bright for peak dots)
const HEAT_STOPS = config.color.stops.map(([r, g, b]) => ({ r, g, b }));
const PALETTE_SIZE = config.color.palette_size;
const PALETTE = buildPalette(HEAT_STOPS, PALETTE_SIZE, colorMode);
const BRIGHT_PALETTE = buildPalette(
  HEAT_STOPS.map(c => ({
    r: Math.min(255, Math.round(c.r * config.color.peak_brightness)),
    g: Math.min(255, Math.round(c.g * config.color.peak_brightness)),
    b: Math.min(255, Math.round(c.b * config.color.peak_brightness)),
  })),
  PALETTE_SIZE, colorMode
);

// 4. Block characters
const BLOCKS = ' ▁▂▃▄▅▆▇█';

// 5. State arrays
const bars = new Float64Array(N);
const peakBars = new Float64Array(N);
const peakDecay = new Float64Array(N);
let lastTickTime = Date.now();

// 6. Listen for terminal resize
process.stdout.on('resize', () => {
  let newN = Math.min(48, Math.max(12, process.stdout.columns - RESERVED_COLS));
  if (ambiguousWidth) newN = Math.floor(newN / 2);
  // Resize state arrays if N changes (reallocate Float64Arrays)
});
```

### Tick behavior

The status line is **event-driven, not frame-based**. Claude Code refreshes the status line whenever the conversation state changes — when a new message appears, when Claude starts or stops generating, when a tool fires, or when the context window updates. Updates are throttled to **at most every 300ms** (~3.3 ticks per second maximum).

On each tick, Claude Code spawns the status line command, pipes JSON to its stdin, and reads the first line of stdout as the rendered status line. The stdin JSON includes:

```json
{
  "hook_event_name": "Status",
  "session_id": "abc123",
  "cwd": "/current/working/directory",
  "model": { "id": "claude-opus-4-6", "display_name": "Opus" },
  "context_window": { "used_percentage": 34.2 },
  "cost": { "total_cost_usd": 0.45 }
}
```

The renderer extracts `context_window.used_percentage` to drive the context beat oscillator and context display, and `model.display_name` to show the current model. Both are live values on every tick, not stale snapshots from the last hook fire. The model name updates immediately if the user switches models via `/model`.

**During idle periods** (user reading, thinking, away from keyboard), there are zero ticks — the status line shows whatever was last rendered. This means tick intervals are variable: 300ms during active conversation, potentially seconds or minutes during idle. The decay and peak gravity calculations must account for this.

### Time-corrected decay

Because tick intervals are variable, all per-tick calculations are normalized to elapsed wall-clock time rather than assuming a fixed frame rate:

```typescript
let lastTickTime = Date.now();

// On each tick:
const now = Date.now();
const elapsed = now - lastTickTime;
lastTickTime = now;

const TICK_BASELINE = 300; // ms — expected tick interval
const tickScale = elapsed / TICK_BASELINE;

// Bar decay: normalize to baseline tick rate
const effectiveDecay = Math.pow(config.decay.rate, tickScale);
for (let i = 0; i < N; i++) {
  bars[i] *= effectiveDecay;
  bars[i] = Math.max(bars[i], config.decay.ambient_floor);
}

// Peak gravity: scale by elapsed time
const scaledGravity = config.decay.peak_gravity * tickScale;
for (let i = 0; i < N; i++) {
  if (bars[i] > peakBars[i]) {
    peakBars[i] = bars[i];
    peakDecay[i] = 0;
  } else {
    peakDecay[i] += scaledGravity;
    peakBars[i] = Math.max(0, peakBars[i] - peakDecay[i]);
  }
}

// Beat oscillator: uses wall-clock time directly (already correct)
const t = now / 1000; // seconds
const contextPct = stdinData.context_window.used_percentage / 100;
const beatFreq = config.beat.freq_base + contextPct * config.beat.freq_scale;
const beatAmp = config.beat.amp_base + contextPct * config.beat.amp_scale;
const multiplier = 1 + beatAmp * Math.sin(2 * Math.PI * beatFreq * t);
```

Without time correction, bars would decay much faster during active conversation (ticks every 300ms) than during idle periods (ticks separated by seconds), creating inconsistent visual behavior. The beat oscillator already uses wall-clock time (`t`) for its sin wave phase, so it needs no correction.

### Render loop (every statusLine tick)

Target budget: **< 5ms**

```
1. Parse stdin JSON (extract contextPct, model.display_name)
2. Compute elapsed time since last tick, calculate tickScale
3. Read amp-state.json (Bun.file, skip if mtime unchanged)
4. Check staleness (lastTimestamp > 30s ago → decay-only mode)
5. Check version (mismatch → decay-only mode)
6. If state file updated: compute velocity from char count deltas, apply EMA smoothing
7. Apply spatial decay weights (exponential left/right blend)
8. Apply time-corrected bar decay: bar[i] *= pow(decayRate, tickScale)
9. Apply ambient noise floor: bar[i] = max(bar[i], 0.03)
10. Update peak dots (time-corrected gravity):
    - If bar[i] > peakBars[i]: peakBars[i] = bar[i], peakDecay[i] = 0
    - Else: peakDecay[i] += gravity × tickScale, peakBars[i] -= peakDecay[i]
    - Clamp peakBars[i] >= 0
11. Apply context beat oscillator (wall-clock time, contextPct from stdin):
    beatFreq = 0.3 + contextPct × 3.0
    beatAmp  = 0.08 + contextPct × 0.45
    multiplier = 1 + beatAmp × sin(2π × beatFreq × t)
    bar[i] *= multiplier
12. Compute per-bar color:
    - Base color: entropy → palette index (clamped [3.0, 6.0] → [0, 63])
    - Spatial blend: inputEntropy drives left, outputEntropy drives right
    - Char-class modifier: adjust saturation/brightness based on
      dominant char class at this bar position
    - Select from PALETTE (base) or BRIGHT_PALETTE (peak) accordingly
13. Render viz bars: concatenate ANSI color + block char for each bar
14. Format right-side metrics: model name + ↑inputVel ↓outputVel + ctx XX%
15. Compose full line: bars + separator + metrics + reset + erase-to-EOL
16. Write single string to stdout (first line only — Claude Code reads one line)
```

### Spatial decay weights

Input signal weighted to left side, output signal weighted to right:

```typescript
const k = 3.0; // decay steepness

for (let i = 0; i < N; i++) {
  const pos = i / N;
  const inputWeight  = Math.exp(-k * pos);       // strong left
  const outputWeight = Math.exp(-k * (1 - pos));  // strong right
  bars[i] = inputWeight * inputBars[i] + outputWeight * outputBars[i];
}
```

### Character class as saturation/brightness modifier

The 8-element char-class array modifies the color of each bar without affecting its height. Map each bar position to a blended char-class value (interpolating input/output class arrays using the same spatial weights), then use the dominant class to shift color:

- **High markdown/structural** → cooler brightness, slightly blue-shifted (planning/design mode)
- **High code syntax** → warmer brightness, slightly amber-shifted (implementation mode)
- **High line breaks** → increased contrast (structured, list-heavy output)
- **High lowercase + punctuation/other balanced** → neutral (conversational baseline)
- **High uppercase** → slight brightness boost (headers, emphasis-heavy content)
- **High whitespace** → desaturate slightly (deep indentation, quiet structure)
- **High digits** → cool-shift the hue slightly (data-heavy output)

This gives the user a subtle ambient signal of what kind of work Claude is doing — the bar shifts hue character as the session moves between planning, design, implementation, and conversational phases. The effect is bounded at ±10-15% saturation/brightness adjustment, enough to differentiate content types without overpowering the entropy-driven heat gradient.

### Peak dots

Track a separate peak position per bar with gravity-based decay. Peaks are rendered as a **color brightness shift** rather than a separate glyph, ensuring consistent rendering across all terminals and fonts. Gravity is time-corrected (see Time-corrected decay above).

When `peakBars[i] - bars[i] > 0.08`, the bar renders at peak amplitude using the bright palette. As the peak decays back toward the current amplitude, the brightness boost fades. This produces the characteristic Winamp peak-hold-and-fall effect without any glyph compatibility risk.

### File I/O safety

```typescript
let lastState: AmpState | null = null;
let lastMtime = 0;

function readState(): AmpState | null {
  try {
    const file = Bun.file(STATE_PATH);
    const mtime = file.lastModified;
    if (mtime === lastMtime && lastState) return lastState; // cache hit
    
    const parsed = JSON.parse(file.textSync());
    if (parsed.version !== EXPECTED_VERSION) return null; // version mismatch
    
    lastState = parsed;
    lastMtime = mtime;
    return parsed;
  } catch {
    return lastState; // return last valid state on parse error
  }
}
```

Atomic writes from the Python side (`os.replace`) plus fallback to last valid state on the reader side eliminates partial-read corruption.

---

## Signal layers (compositing order)

| Layer | Source | Drives | Notes |
|---|---|---|---|
| 1. DFT signal | FFT of mean-centered, Hann-windowed character samples | Bar height | Log-binned with interpolation for short payloads |
| 2. Spatial blend | Exponential decay weights (k=3.0) | Left/right separation | Input signal fades left→right, output right→left |
| 3. Heat color | Shannon entropy of payload bytes | Bar color (blue→red) | Clamped to [3.0, 6.0] range, spatially blended |
| 4. Char-class texture | 8 semantic character class frequencies | Color saturation/brightness | ±10-15% modifier, does NOT affect bar height |
| 5. Context beat | sin wave scaled by contextPct | Global amplitude multiplier | 0.3Hz idle → 3.3Hz at full context |
| 6. Decay | Per-tick multiplicative decay (0.90) | Bar persistence | Decays toward ambient floor (0.03), not zero |
| 7. Peak dots | Per-bar max tracking with gravity | Peak indicator | Color brightness shift (1.4× RGB), no glyph dependency |

---

## Status line layout

The renderer outputs a single line combining the visualization bars and text metrics. The layout is:

```
▁▃▅▇█▆▄▂▁▃▅▇█▆▄▂▁▃▅▇█▆▄▂  Opus  ↑1.2k/s ↓3.8k/s  ctx 34%
╰──────── viz bars ────────╯ ╰mod╯ ╰─ velocity ──╯  ╰context╯
```

### Components

**Viz bars** (left) — the DFT-driven frequency bar visualization, dynamically sized to available terminal width after reserving space for the right-side metrics.

**Model** (after bars) — the current model's display name from the statusLine stdin JSON (`model.display_name`). Rendered in a dim/muted color to keep visual focus on the bars and velocity. Common values: `Opus`, `Sonnet`, `Haiku`. This updates live if the user switches models mid-session via `/model`.

**Token velocity** (center-right) — input and output character throughput displayed as characters per second, using `↑` for input (user/tool payloads flowing in) and `↓` for output (assistant responses flowing out). Displayed with SI suffixes for readability:

| Rate | Display |
|---|---|
| 0 chars/sec | `↑0` |
| 450 chars/sec | `↑450/s` |
| 1,200 chars/sec | `↑1.2k/s` |
| 15,800 chars/sec | `↑15.8k/s` |

Character velocity is used as a proxy for token velocity because hooks receive raw text, not tokenized content. The ratio of characters to tokens is roughly 4:1 for English text and code, but varies by content. The display uses characters (not estimated tokens) to avoid false precision.

**Context usage** (far right) — context window percentage from the statusLine stdin JSON, formatted as `ctx XX%`. Color-coded using the heat palette:

| Usage | Color |
|---|---|
| 0–40% | Default/dim (comfortable) |
| 40–70% | Amber (getting full) |
| 70–90% | Orange (consider compacting) |
| 90–100% | Red (near limit) |

### Width allocation

The renderer measures available terminal width and allocates space right-to-left:

```typescript
const stdinData = JSON.parse(readStdin());
const modelName = stdinData.model?.display_name || '?';

const VELOCITY_WIDTH = 18;    // "↑1.2k/s ↓3.8k/s"
const CONTEXT_WIDTH = 8;      // "ctx 34%"
const MODEL_WIDTH = modelName.length;
const SEPARATORS = 6;         // spaces between sections
const METRICS_WIDTH = MODEL_WIDTH + VELOCITY_WIDTH + CONTEXT_WIDTH + SEPARATORS;

const barCols = Math.max(config.display.min_bars,
  Math.min(config.display.max_bars, process.stdout.columns - METRICS_WIDTH));
```

Metrics always render at full width. The viz bars absorb whatever space remains. On very narrow terminals (< 40 columns), metrics are hidden and only the bars render.

### Velocity computation

The renderer tracks the previous state file snapshot to compute rates:

```typescript
let prevInputChars = 0;
let prevOutputChars = 0;
let prevStateTimestamp = 0;

function computeVelocity(state: AmpState): { inputVel: number; outputVel: number } {
  if (prevStateTimestamp === 0) {
    // First read — no delta available, show zero
    prevInputChars = state.inputChars;
    prevOutputChars = state.outputChars;
    prevStateTimestamp = state.lastTimestamp;
    return { inputVel: 0, outputVel: 0 };
  }

  const dtSec = (state.lastTimestamp - prevStateTimestamp) / 1000;
  if (dtSec <= 0) return { inputVel: 0, outputVel: 0 };

  const inputVel = (state.inputChars - prevInputChars) / dtSec;
  const outputVel = (state.outputChars - prevOutputChars) / dtSec;

  prevInputChars = state.inputChars;
  prevOutputChars = state.outputChars;
  prevStateTimestamp = state.lastTimestamp;

  return { inputVel: Math.max(0, inputVel), outputVel: Math.max(0, outputVel) };
}
```

Velocity is recalculated only when the state file updates (new hook fire). Between updates, the renderer displays the last computed velocity, which decays toward zero as the state file becomes stale.

### Velocity smoothing

Raw velocity is spiky — a single large hook payload produces a huge instantaneous rate that drops to zero until the next payload. Apply exponential moving average to smooth the display:

```typescript
const SMOOTHING = 0.3; // 0 = no smoothing, 1 = frozen
let smoothInputVel = 0;
let smoothOutputVel = 0;

// On each new velocity computation:
smoothInputVel = SMOOTHING * smoothInputVel + (1 - SMOOTHING) * rawInputVel;
smoothOutputVel = SMOOTHING * smoothOutputVel + (1 - SMOOTHING) * rawOutputVel;
```

When the state file is stale (>30s), smoothed velocity decays toward zero naturally through the EMA as new zero-valued samples are fed in.

### Rendering the metrics

```typescript
function formatVelocity(charsPerSec: number): string {
  if (charsPerSec < 1) return '0';
  if (charsPerSec < 1000) return `${Math.round(charsPerSec)}/s`;
  return `${(charsPerSec / 1000).toFixed(1)}k/s`;
}

function formatContext(pct: number): string {
  return `ctx ${Math.round(pct)}%`;
}

// Model name from stdin JSON — dim color to keep visual focus on bars
const modelStr = `\x1b[2m${modelName}\x1b[22m`; // dim on, dim off

// Compose the right-side metrics string with ANSI colors
const metrics = `  ${modelStr}  ↑${formatVelocity(smoothInputVel)} ↓${formatVelocity(smoothOutputVel)}  ${colorForContext(contextPct)}${formatContext(contextPct)}\x1b[0m`;
```

The full status line output is then:

```typescript
const output = renderedBars + metrics + '\x1b[K'; // erase to EOL
process.stdout.write(output);
```

---

## Hook configuration

`hooks/hooks.json` — uses the plugin wrapper format with `${CLAUDE_PLUGIN_ROOT}` for portable paths:

```json
{
  "description": "claude-amp — Winamp-style visualizer hooks for signal processing and renderer setup",
  "hooks": {
    "SessionStart": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "bash ${CLAUDE_PLUGIN_ROOT}/scripts/session_start.sh"
      }]
    }],
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "uv run --project ${CLAUDE_PLUGIN_ROOT}/scripts --python-preference managed --cache-dir ${CLAUDE_PLUGIN_DATA}/.uv-cache ${CLAUDE_PLUGIN_ROOT}/scripts/amp_processor.py",
        "async": true
      }]
    }],
    "UserPromptSubmit": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "uv run --project ${CLAUDE_PLUGIN_ROOT}/scripts --python-preference managed --cache-dir ${CLAUDE_PLUGIN_DATA}/.uv-cache ${CLAUDE_PLUGIN_ROOT}/scripts/amp_processor.py",
        "async": true
      }]
    }],
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "uv run --project ${CLAUDE_PLUGIN_ROOT}/scripts --python-preference managed --cache-dir ${CLAUDE_PLUGIN_DATA}/.uv-cache ${CLAUDE_PLUGIN_ROOT}/scripts/amp_processor.py",
        "async": true
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "uv run --project ${CLAUDE_PLUGIN_ROOT}/scripts --python-preference managed --cache-dir ${CLAUDE_PLUGIN_DATA}/.uv-cache ${CLAUDE_PLUGIN_ROOT}/scripts/amp_processor.py",
        "async": true
      }]
    }]
  }
}
```

The `SessionStart` hook runs `scripts/session_start.sh`, which writes `~/.claude/amp_renderer.sh` with the current `${CLAUDE_PLUGIN_ROOT}` and `${CLAUDE_PLUGIN_DATA}` paths baked in. This gives the user a stable path for their statusLine config that doesn't change on plugin updates.

---

## Graceful degradation

### Prerequisites

The visualizer requires two runtime tools to be installed:

| Dependency | Minimum version | Install |
|---|---|---|
| Astral UV | 0.4+ | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Bun | 1.0+ | `curl -fsSL https://bun.sh/install \| bash` |

UV manages Python and numpy automatically — no separate Python installation or `pip install` steps are needed. On first hook invocation, UV downloads the required Python version (if not already available) and installs numpy into an isolated `.venv`. This is transparent to the user.

The renderer checks for UV and Bun availability at startup. If either is missing, it outputs a static status line message indicating the missing prerequisite and exits cleanly.

### Runtime fallbacks

| Failure mode | Behavior |
|---|---|
| `amp-state.json` missing | Render ambient noise floor with idle beat |
| `amp-state.json` parse error | Use last valid cached state |
| State file version mismatch | Decay-only mode, no new signal |
| State file stale (>30s) | Decay-only mode, beat continues |
| Terminal too narrow (<12 cols) | Reduce N to fit, minimum 8 bars |
| No truecolor support (e.g. Terminal.app) | 256-color palette fallback (auto-detected) |
| CJK ambiguous width locale | Halve bar count to fit terminal |
| Terminal resize during session | Re-detect width, reallocate state arrays |
| Python processor crashes | State file simply doesn't update; renderer decays gracefully |
| UV not installed | Renderer displays prerequisite message and exits cleanly |
| UV first-run cold install | 1–3s delay on first hook fire (async, invisible to user); subsequent runs use cached `.venv` |
| `amp-defaults.toml` missing | Fatal error — required file, renderer exits with message |
| `amp-config.toml` missing | Normal operation — all defaults apply |
| `amp-config.toml` parse error | Ignore overrides, use defaults, log warning to stderr |

---

## Configuration

### Two-file merge-on-read

Parameters are managed through two TOML files with a defaults-plus-overrides pattern:

- **`${CLAUDE_PLUGIN_ROOT}/config/amp-defaults.toml`** — ships with the plugin, read-only. Documents every parameter with inline comments explaining what it does, its valid range, and its default value. This is the reference the user reads to understand what's available.
- **`${CLAUDE_PLUGIN_DATA}/amp-config.toml`** — user-created, optional, lives in the persistent data directory. Contains only the parameters the user wants to override. If this file doesn't exist, all defaults apply. If it exists, its values take precedence. Survives plugin updates.

Both the Python processor and the Bun renderer read the same two config files and apply the same merge logic: load defaults from the plugin's config directory, then overlay any values present in the persistent data directory. Config is loaded **once at startup**, not per-tick. Restarting Claude Code or running `/reload-plugins` picks up config changes.

### `amp-defaults.toml`

```toml
# Claude Code Visualizer — Default Configuration
# Do not edit this file. Create amp-config.toml with your overrides.

[display]
# Number of bars. Set to 0 for auto-detect from terminal width.
# Range: 8–48, or 0 for dynamic
bar_count = 0

# Columns reserved for status line padding and ANSI reset codes
reserved_cols = 4

# Minimum bar count if terminal is very narrow
min_bars = 8

# Maximum bar count even on wide terminals
max_bars = 48

[signal]
# How sharply input (left) and output (right) sides separate
# 2.0 = gradual blend, 6.0 = sharp separation
spatial_decay = 3.0

# Shannon entropy range mapped to the heat color gradient
# Values below entropy_min pin to full blue (cold)
# Values above entropy_max pin to full red (hot)
entropy_min = 3.0
entropy_max = 6.0

# Strength of character-class saturation/brightness modifier
# Higher = more visible difference between code, prose, markdown
# Range: 0.05–0.20
char_class_effect = 0.12

[decay]
# Per-tick multiplicative decay for bar amplitude
# Lower = faster fall, higher = longer trails
# Range: 0.85–0.95
rate = 0.90

# Minimum bar amplitude — prevents bars from reaching full zero
# Keeps the visualizer looking alive between hook events
# Range: 0.01–0.05
ambient_floor = 0.03

# Gravity acceleration for peak dot fall
# Lower = slower peak fall, higher = snappier peaks
# Range: 0.002–0.008
peak_gravity = 0.004

[beat]
# Context beat oscillator — pulse frequency and amplitude scale with
# context window usage. At low context, the beat is slow and subtle.
# At high context, it becomes fast and dramatic.

# Pulse frequency (Hz) at zero context usage
freq_base = 0.3

# How much faster the pulse gets as context fills
# Final frequency = freq_base + freq_scale × contextPct
freq_scale = 3.0

# Minimum beat amplitude (barely perceptible at low context)
amp_base = 0.08

# Maximum additional amplitude at full context
# Final amplitude = amp_base + amp_scale × contextPct
amp_scale = 0.45

[color]
# Number of pre-computed stops in the heat gradient palette
# Higher = smoother gradient, lower = more banding
# Range: 32–128
palette_size = 64

# Heat ramp color stops (blue → teal → amber → orange → red)
# Each stop is [R, G, B] with values 0–255
# Modify to change the overall color theme of the visualizer
stops = [
    [50, 100, 200],   # blue (cold, low entropy)
    [0, 180, 180],    # teal
    [220, 170, 50],   # amber
    [230, 100, 30],   # orange
    [220, 40, 40],    # red (hot, high entropy)
]

# Brightness multiplier for peak dot highlight
# Range: 1.1–1.8
peak_brightness = 1.4

[timing]
# Milliseconds before state file is considered stale
# Stale state triggers decay-only mode (no new signal, beat continues)
staleness_threshold = 30000

[metrics]
# Columns reserved for the right-side metrics display (velocity + context)
# Set to 0 to disable metrics and use full width for bars
reserved_width = 24

# Minimum terminal width required to show metrics alongside bars
# Below this, only bars render
min_width_for_metrics = 40

# Exponential moving average smoothing for velocity display
# 0.0 = no smoothing (raw, spiky), 1.0 = frozen (never updates)
# Range: 0.1–0.5
velocity_smoothing = 0.3

# Context usage color thresholds (percentage)
context_warn = 40
context_high = 70
context_critical = 90
```

### `amp-config.toml` (user overrides)

The user creates this file and includes only the values they want to change. Example:

```toml
# My visualizer tweaks

[decay]
rate = 0.88           # faster bar fall
ambient_floor = 0.05  # more visible idle state

[color]
stops = [
    [30, 60, 180],    # deeper blue
    [0, 200, 160],    # more green-teal
    [240, 180, 40],   # brighter amber
    [240, 80, 20],    # hotter orange
    [200, 30, 30],    # darker red
]
```

Everything not specified in `amp-config.toml` falls through to `amp-defaults.toml`.

### Config loading

Both components use the same merge logic. TOML parsing is stdlib in Python 3.11+ (`tomllib`) and handled by a lightweight parser in Bun.

**Python processor:**

```python
import tomllib
from pathlib import Path
import os

def load_config() -> dict:
    plugin_root = Path(os.environ["CLAUDE_PLUGIN_ROOT"])
    data_dir = Path(os.environ["CLAUDE_PLUGIN_DATA"])

    with open(plugin_root / "config" / "amp-defaults.toml", "rb") as f:
        config = tomllib.load(f)

    override_path = data_dir / "amp-config.toml"
    if override_path.exists():
        with open(override_path, "rb") as f:
            overrides = tomllib.load(f)
        deep_merge(config, overrides)

    return config

def deep_merge(base: dict, override: dict) -> None:
    for key, value in override.items():
        if isinstance(value, dict) and isinstance(base.get(key), dict):
            deep_merge(base[key], value)
        else:
            base[key] = value
```

**Bun renderer:**

```typescript
import { parse } from "smol-toml"; // lightweight, zero-dep TOML parser

function loadConfig(): Config {
  const pluginRoot = process.env.CLAUDE_PLUGIN_ROOT!;
  const dataDir = process.env.CLAUDE_PLUGIN_DATA!;

  const defaults = parse(Bun.file(`${pluginRoot}/config/amp-defaults.toml`).textSync());

  try {
    const overrides = parse(Bun.file(`${dataDir}/amp-config.toml`).textSync());
    return deepMerge(defaults, overrides) as Config;
  } catch {
    return defaults as Config; // no override file, use defaults
  }
}
```

`smol-toml` is a zero-dependency TOML parser that adds minimal startup cost. It's the only renderer dependency beyond Bun native APIs.

---

## Build order

1. **Scaffold plugin** — create `claude-amp/` with `.claude-plugin/plugin.json`, `hooks/hooks.json`, `scripts/`, `config/`
2. **UV project setup** — create `scripts/pyproject.toml` and `scripts/uv.lock`, verify `uv run` resolves Python + numpy
3. **Python processor** — FFT pipeline reading from stdin, writing valid state file to `${CLAUDE_PLUGIN_DATA}/`, invoked via `uv run`
4. **Bun renderer** — read state file + statusLine stdin JSON (for contextPct), render bars with decay and ambient floor
5. **SessionStart hook** — generate `~/.claude/amp_renderer.sh` wrapper with baked-in paths for statusLine setup
6. **Test locally** — `claude --plugin-dir ./claude-amp`, configure statusLine manually, validate end-to-end
7. **Add peak dots** — track per-bar maximums with gravity decay
8. **Add char-class saturation** — color modifier layer
9. **Add context beat** — oscillator driven by contextPct from statusLine stdin
10. **Wire remaining hooks** — PreToolUse, PostToolUse
11. **Tune parameters** — decay rate, beat curve, palette stops, spatial steepness
12. **Add graceful degradation** — prerequisite checks, terminal capability detection, fallback paths
13. **Publish** — push to GitHub for marketplace auto-discovery, document statusLine setup in README

---

## Future features (not in scope)

**Hot config reload via `FileChanged` hook** — Register a `FileChanged` hook watching `amp-config.toml` (matcher: `"amp-config.toml"`). When the user edits their config file, the hook fires and signals the renderer to reload parameters on the next statusLine tick without restarting Claude Code. The `CwdChanged` hook's `watchPaths` return value could dynamically register the config file path in `${CLAUDE_PLUGIN_DATA}/`. Currently, config changes require restarting Claude Code or running `/reload-plugins`.


