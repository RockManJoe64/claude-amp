# Winamp Visualization Strategy

The following strategy was created with Claude.ai.

Here's the full conceptual map we built up:

---

**How Winamp does it**

Audio comes in as a continuous stream of PCM samples — each one a pressure amplitude value between -1 and +1, captured ~44,000 times per second. Every ~16ms, Winamp grabs a buffer of 1024–2048 of those samples and runs the DFT on it.

The DFT decomposes that buffer into N/2 frequency bins. Each bin answers the question: *how much energy is present at this frequency?* Bin 0 is the DC offset (average value). Bin 1 is the slowest oscillation that fits in the buffer. Bin N/2 is the fastest. The magnitude of each bin is the amplitude of that frequency — that's the number that drives bar height.

The critical non-obvious step is that those N/2 bins don't map one-to-one to bars. They get logarithmically compressed first. Human hearing is logarithmic — the perceptual difference between 100Hz and 200Hz feels the same as between 1000Hz and 2000Hz. So the first display bar might cover only bins 1–2 (deep bass), the middle bars cover moderate ranges, and the last bars average over hundreds of bins (high treble). This is why bass hits look dramatic and treble looks fine-grained. The Hann window applied before the DFT prevents spectral leakage — without it, sharp buffer edges create spurious high-frequency artefacts.

After the DFT and binning, decay is applied every frame: each bar multiplies by ~0.88–0.92, so it falls with gravity rather than snapping to zero. A separate slower decay drives the floating peak dot.

---

**The text analogy**

Instead of PCM samples, we have characters. Each character becomes a normalized sample value: `charCode / 128`, giving a float between 0 and 1. The sequence of characters in a hook payload — the content of a file Claude just wrote, the bash output it received, the user's prompt — becomes a discrete signal exactly like an audio buffer.

The DFT of that character sequence decomposes it into frequency bins with the same interpretation, just reframed:

- Low-frequency bins capture **long repeating patterns** — indentation rhythm in Python, repeated JSON keys, paragraph-level structural cadence
- High-frequency bins capture **short repeating patterns** — alternating punctuation, dense identifiers, character-level noise

This maps to audio because text structure is also hierarchical and logarithmic. A document has a few paragraph-level patterns but thousands of character-level variations, so log binning gives the interesting structural patterns visual weight and compresses character noise to the right edge — exactly the same reason Winamp uses it for bass vs treble.

The fingerprint is content-specific and deterministic. Code with heavy indentation produces low-frequency energy spikes. Minified JSON produces flat high-frequency noise. Prose produces a smooth mid-range curve. A bash command is short and produces a sparse, spiky spectrum. That uniqueness is what makes it a more honest signal than pure velocity — the bar shape is literally derived from what Claude wrote, not just how fast tokens arrived.

---

**Where the analogy breaks down — and how we handle it**

Audio updates 60 times per second. Hook payloads fire maybe once every few seconds. So the DFT produces a static snapshot per turn, not a live stream. Decay becomes even more important here than in Winamp — it's the only thing keeping the bar alive between hook events, slowly returning it to silence while the context beat oscillator breathes rhythm into the whole bar at a rate proportional to context window usage.

The character class mode (whitespace / lowercase / uppercase / digits / punctuation / brackets / operators) is something audio has no equivalent for — it's a purely textual fingerprint that shifts visibly between tool types. A `Write` hook payload looks different from a `Bash` response just from its syntactic distribution. We proposed blending 70% DFT log-binned signal with 30% character-class texture to get both the organic frequency-domain feel and a stable content-type indicator in a single bar.
