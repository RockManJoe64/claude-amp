# Winamp-style token visualizer — tech stack recommendation

## Summary

Use a split architecture: **Python** for signal processing and **TypeScript/Bun** for rendering. Each language handles the component it is best suited for, communicating through a single shared JSON state file.

---

## Architecture overview

```
Stop / UserPromptSubmit hooks
        │
        ▼
┌─────────────────────┐        writes        ┌─────────────────────┐
│   Python processor  │ ──────────────────▶  │  viz-state.json     │
│   (async hook)      │                       │  ~/.claude/         │
└─────────────────────┘                       └─────────────────────┘
                                                        │
                                                     reads
                                                        │
                                                        ▼
                                              ┌─────────────────────┐
                                              │  TypeScript renderer │
                                              │  (statusLine tick)  │
                                              └─────────────────────┘
                                                        │
                                                        ▼
                                              Claude Code status line
```

---

## Component 1 — Python signal processor

**Fires on:** `Stop` and `UserPromptSubmit` hooks (both `async: true`)

**Responsibilities:**

- Read the last user and assistant entries from the transcript JSONL
- Extract character arrays from `tool_input` / `tool_response` content
- Run `numpy.fft.rfft()` on the character value array
- Log-bin the FFT magnitudes into N display bars
- Compute the character-class texture layer (8 semantic classes → N bars)
- Blend: `0.7 × DFT log-binned + 0.3 × char-class texture`
- Compute token deltas from cumulative usage for velocity
- Write `~/.claude/viz-state.json`

**Why Python:**

`numpy.fft.rfft()` is two lines of code and battle-hardened. The TypeScript equivalent requires a manual Goertzel implementation (~30 lines) that is slower on long payloads. Since this hook runs async, Python's 80–150ms cold start is completely invisible — Claude does not wait for it.

**Key libraries:**

| Library | Purpose |
|---|---|
| `numpy` | `rfft()` for DFT, array math for log binning |
| `json` | stdin parsing and state file writing |
| `pathlib` | transcript path resolution |

**No virtual environment needed** — numpy is the only non-stdlib dependency.

---

## Component 2 — TypeScript/Bun renderer

**Fires on:** every `statusLine` tick

**Responsibilities:**

- Read `viz-state.json` (cached, ~1ms)
- Apply per-frame decay: `bar[i] *= decayRate` (default 0.88–0.92)
- Apply context beat oscillator: `amp *= 1 + beatAmp × sin(2π × beatFreq × t)`
- Compute per-bar color from input/output heat gradient with exponential spatial decay
- Render N Unicode block chars with ANSI 24-bit color to stdout
- Total budget: under 15ms

**Why TypeScript/Bun:**

The status line fires synchronously — Claude Code waits for it. Bun starts in ~5–15ms vs Python's 80–150ms. TypeScript is also the primary language at Slalom, the Claude Code status line ecosystem (ccstatusline, ccusage) is Bun-first, and chalk gives ergonomic 24-bit ANSI color interpolation across the heat gradient.

**Key libraries:**

| Library | Purpose |
|---|---|
| `bun` (runtime) | Native file API, sub-15ms startup |
| `chalk` | 24-bit ANSI color for heat gradient |
| `ccstatusline` | Optional — drop viz in as a custom widget |

---

## State file contract

`~/.claude/viz-state.json` is the interface between the two components.

```json
{
  "inputBars":        [0.8, 0.4, 0.9, ...],
  "outputBars":       [0.3, 0.7, 0.2, ...],
  "inputEntropy":     3.2,
  "outputEntropy":    4.7,
  "contextPct":       0.34,
  "lastTimestamp":    "2026-03-27T10:00:00.000Z",
  "prevInputTokens":  15000,
  "prevOutputTokens": 4000
}
```

The renderer only reads `inputBars`, `outputBars`, and `contextPct`. All heavy computation happens in the Python processor and is never repeated at render time.

---

## Hook configuration

```json
{
  "hooks": {
    "Stop": [{
      "hooks": [{
        "type": "command",
        "command": "python ~/.claude/hooks/viz_processor.py",
        "async": true
      }]
    }],
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "command",
        "command": "python ~/.claude/hooks/viz_processor.py",
        "async": true
      }]
    }]
  },
  "statusLine": {
    "type": "command",
    "command": "bun ~/.claude/hooks/viz_renderer.ts",
    "padding": 0
  }
}
```

---

## Algorithm layers (render order)

1. **DFT signal** — character array → FFT → log-binned magnitudes → left/right exponential spatial weighting
2. **Char-class texture** — 8 semantic class counts (whitespace, lowercase, uppercase, digits, punctuation, brackets, operators, other) → blended 30% over DFT signal
3. **Heat color gradient** — each bar's color interpolates between input velocity heat (left-weighted) and output velocity heat (right-weighted) using the same exponential decay
4. **Context beat oscillator** — global amplitude multiplier: `1 + beatAmp × sin(2π × beatFreq × t)` where both `beatAmp` and `beatFreq` scale with context window percentage
5. **Decay** — applied every status line tick: `bar[i] *= 0.88–0.92`, keeps the bar alive between hook events

---

## Decision matrix

| Component | Python | TypeScript/Bun | Winner |
|---|---|---|---|
| Hook startup latency | 80–150ms | 5–15ms | TS/Bun |
| FFT / DFT math | `numpy.fft` — 2 lines | Manual Goertzel — 30 lines | Python |
| JSONL transcript parsing | Simple | Simple | Tie |
| ANSI terminal rendering | Limited, dated libs | chalk, ink, ecosystem | TS/Bun |
| Status line integration | Extra overhead | Native, ccstatusline-compatible | TS/Bun |
| Your primary language | Secondary | Daily driver | TS/Bun |
| Async hook blocking risk | None (runs async) | N/A | Tie |
| Dependency management | pip/venv fragility | bun install, lockfile | TS/Bun |

---

## Suggested build order

1. **Python processor first** — get the FFT pipeline working and writing a valid state file using a hardcoded sample payload
2. **TypeScript renderer second** — read the state file and render the bar with decay, validate it fits the status line width
3. **Wire the hooks** — connect `Stop` and `UserPromptSubmit` to the processor, validate state file updates on each turn
4. **Tune parameters** — decay rate, beat frequency curve, log-bin count, color heat stops
