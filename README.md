# ZeroVoice — KokoroTTS ONNX Workbench

A deterministic NPC voice synthesis workbench. Every NPC receives a unique, reproducible voice derived purely from their spawn coordinates — zero stored voice state, zero lookups, zero iteration. Built as a Tauri desktop application with KokoroTTS running locally via ONNX Runtime.

![Windows](https://img.shields.io/badge/platform-Windows-blue) ![Tauri](https://img.shields.io/badge/Tauri-v2-orange) ![KokoroTTS](https://img.shields.io/badge/KokoroTTS-v1.0-green) ![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## How It Works

ZeroVoice uses the **ZeroByte Systems** methodology — position-is-seed procedural generation where spawn coordinates are the only input. Three integers (X, Y, Z) are hashed with xxHash64 using five independent salts to derive five voice properties in O(1):

| Property | Derivation |
|----------|-----------|
| **Voice A** | Hash mod 26 Kokoro anchor voices |
| **Voice B** | Hash mod 25 remaining voices (B != A guaranteed) |
| **Slerp t** | Interpolation scalar [0.0, 1.0] along the arc between A and B |
| **Pitch scale** | [0.85, 1.15] range |
| **Energy scale** | [0.80, 1.20] range |

The 256-dimensional style vector at the interpolated point between voices A and B is computed via linear interpolation and passed directly to the ONNX inference session. The same coordinates on any machine at any time produce identical voice parameters.

### Voice Capacity

| Property | Value |
|----------|-------|
| Kokoro named anchor voices | 26 |
| Unique unordered voice pair arcs | 325 |
| Interpolation resolution (float32) | ~16.7 million steps per arc |
| Pitch variations | ~5 million distinct values |
| Energy variations | ~6.7 million distinct values |
| **Effective distinct voice identities** | **~5.4 billion** |
| Per-NPC storage required | **0 bytes** |
| Model disk footprint | 88 MB (int8 ONNX) |

With 325 voice arcs, ~16.7M interpolation steps per arc, and independent pitch/energy scaling, the combinatorial space exceeds 5.4 billion perceptually distinct voice identities — all derivable from three integers with no stored state.

---

## Features

- **Three-column workbench UI** — Coordinate Input, Voice Spec, and Synthesis panels
- **Instant voice preview** — browser-side xxHash (WASM) gives immediate VoiceSpec feedback on every keystroke
- **26 Kokoro anchor voices** — American/British, male/female, blended via linear interpolation
- **Oscilloscope waveform display** — real-time PCM visualization during playback
- **WAV export** — 16-bit mono 24 kHz WAV download
- **Parameter overrides** — lock and manually adjust any derived parameter (slerp t, pitch, energy, voice selection)
- **Preset slots (P1-P4)** — bookmark coordinates for quick recall
- **Region bias mode** — optional coherent noise constrains voice selection by world position, creating geographic "dialects"
- **Session history** — scrollable log of synthesis attempts with one-click restore
- **History export** — download full session log as JSON
- **World seed** — editable 64-bit hex seed that changes every derived voice globally
- **First-run auto-download** — model assets (~113 MB) are downloaded automatically on first launch
- **Debug console (Ctrl+F9)** — hidden diagnostic panel with copy-all for troubleshooting
- **Dark industrial aesthetic** — CRT scanline overlay, amber accent palette, IBM Plex Mono typography
- **Fully offline** — no internet required after first-run asset download

---

## Installation

### Download the MSI (Recommended)

1. Go to [Releases](https://github.com/MushroomFleet/ZeroVoice-KokoroTTS/releases)
2. Download the latest `ZeroVoice.Workbench_x.x.x_x64_en-US.msi`
3. Run the installer
4. Launch **ZeroVoice Workbench** from the Start Menu
5. On first launch, the app will automatically download the required model assets (~113 MB):
   - KokoroTTS ONNX model (88 MB)
   - Voice style vectors (27 MB)
   - eSpeak-NG phonemizer (~2 MB + data)

### Build from Source

Requires: Rust toolchain, Node.js 18+, npm

```bash
git clone https://github.com/MushroomFleet/ZeroVoice-KokorotTTS.git
cd ZeroVoice-KokorotTTS
npm install
npm run tauri dev      # Development (hot-reload)
npm run tauri build    # Production MSI
```

---

## Usage

### 1. Enter Spawn Coordinates

Type X, Y, Z integer values (-32768 to 32767) representing an NPC's spawn point. Use arrow keys to step (Shift+Arrow = 10, Ctrl+Arrow = 100), or click **RANDOMISE** for a random coordinate.

### 2. Review the Voice Spec

The center panel instantly shows the derived voice: which two Kokoro voices are blended, the interpolation point, pitch, and energy. All values are deterministic — the same coordinates always produce the same voice.

### 3. Override Parameters (Optional)

Check any **LOCK** checkbox to manually adjust that parameter. Drag the arc visualiser to set the slerp point. When overrides are active, the panel shows **[OVERRIDE ACTIVE]** in amber.

### 4. Synthesise

Type dialogue text (up to 512 characters) and click **SYNTHESISE**. The waveform canvas shows a scanning bar during generation, then renders the full PCM oscilloscope with a moving playhead during playback.

### 5. Export

Click **EXPORT WAV** to download the audio as a 16-bit 24 kHz mono WAV file. Use **Export JSON** in the history bar to save the full session log.

---

## ZeroByte Systems — Position-as-Seed Determinism

ZeroVoice implements the [ZeroByte Systems](https://github.com/MushroomFleet/ZeroByte-Systems) methodology for procedural content generation:

- **O(1) access** — `voice_from_spawn(x, y, z)` is five xxHash64 calls. No loops, no tables, no iteration.
- **Zero storage** — NPC voices are never stored. They are recomputed on demand from coordinates alone.
- **Parallelism** — each NPC hashes independently with no shared mutable state.
- **Determinism** — xxHash64 is platform-independent. The same world seed and coordinates produce identical results on any machine, any OS, any time.
- **Hierarchy** — `WORLD_SEED -> WORLD_SEED ^ SALT -> position_hash -> hash_to_float -> property`
- **Coherence** — optional regional bias adds smooth geographic variation via bilinear-interpolated noise across the X/Z plane.

### How 5.4 Billion Voices Are Derived from 3 Integers

```
Spawn (X, Y, Z)
    |
    +-- xxHash64(seed ^ SALT_A) --> voice_a index (0-25)
    +-- xxHash64(seed ^ SALT_B) --> voice_b index (0-25, != A)
    +-- xxHash64(seed ^ SALT_T) --> slerp t (0.0 - 1.0)
    +-- xxHash64(seed ^ SALT_P) --> pitch (0.85 - 1.15)
    +-- xxHash64(seed ^ SALT_E) --> energy (0.80 - 1.20)
    |
    +--> style_vector = lerp(voice_table[A][token_len], voice_table[B][token_len], t)
    |
    +--> KokoroTTS ONNX inference --> 24 kHz PCM audio
```

Each hash output is 64 bits. The lower 32 bits are mapped to floats, giving ~4.29 billion distinct values per property. The five properties are statistically independent (different salts), producing a combinatorial voice space of approximately:

> 325 arcs x 16.7M slerp steps x 5M pitch values x 6.7M energy values = **~5.4 x 10^9 distinct voices**

All from `0 bytes` of per-NPC storage.

---

## Architecture

```
src-tauri/src/
  zerovoice.rs      — ZeroByte hash layer, VoiceSpec derivation, regional bias
  slerp.rs          — Voice table loading (NPZ), style vector interpolation
  kokoro_tts.rs     — ONNX session management, inference
  phonemize.rs      — eSpeak-NG IPA phonemization, Kokoro token mapping
  setup.rs          — First-run asset download with progress events
  commands/
    voice.rs        — Tauri IPC: synthesize_npc_speech, preview_voice_spec
    setup.rs        — Tauri IPC: check_setup, run_setup, init_session

src/
  App.tsx           — Main state management, setup flow
  components/
    workbench/      — WorkbenchShell, CoordinatePanel, VoiceSpecPanel, SynthesisPanel
    controls/       — CoordInput, ParamSlider, WaveformCanvas, VoiceSelector, ArcVisualiser
    ui/             — StatusBar, DebugConsole, SegmentNumber
  lib/
    zerovoice-preview.ts  — Browser-side xxHash (WASM) for instant VoiceSpec preview
    voice.ts              — Tauri invoke wrappers
    waveform.ts           — WAV encoding and download
```

---

## Requirements

- Windows 10/11 (x64)
- ~200 MB disk space (app + model assets)
- No GPU required — runs on CPU via ONNX Runtime
- No internet required after first launch

---

## Credits

- [KokoroTTS](https://huggingface.co/hexgrad/Kokoro-82M) by hexgrad — 82M parameter TTS model
- [kokoro-onnx](https://github.com/thewh1teagle/kokoro-onnx) by thewh1teagle — ONNX model files and voice vectors
- [eSpeak-NG](https://github.com/espeak-ng/espeak-ng) — open-source speech synthesizer used for phonemization
- [ONNX Runtime](https://onnxruntime.ai/) by Microsoft — cross-platform ML inference
- [Tauri](https://tauri.app/) — lightweight desktop application framework
- [ZeroByte Systems](https://github.com/MushroomFleet/ZeroByte-Systems) — position-as-seed procedural generation methodology

---

## License

MIT

---

## Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{zerovoice_kokorotts,
  title = {ZeroVoice: Deterministic NPC Voice Synthesis via Position-as-Seed Procedural Generation with KokoroTTS},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/ZeroVoice-KokoroTTS},
  version = {0.1.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
