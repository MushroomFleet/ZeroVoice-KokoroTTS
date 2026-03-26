<!-- TINS Specification v1.0 -->
<!-- ZS:COMPLEXITY:HIGH -->
<!-- ZS:PRIORITY:HIGH -->
<!-- ZS:PLATFORM:DESKTOP -->
<!-- ZS:LANGUAGE:RUST,TYPESCRIPT -->

# ZeroVoice — KokoroTTS ONNX Workbench

A deterministic NPC voice synthesis system and developer workbench. Spawn-point coordinates are the seed — every NPC receives a unique, reproducible voice derived from where they were born in the world, with zero stored voice state per NPC. Built as a Tauri desktop application with a Vite/React/TypeScript frontend and a Rust backend driving KokoroTTS via ONNX Runtime.

The workbench UI allows audio designers, systems programmers, and QA to enter any spawn coordinate, instantly preview the derived voice parameters, type dialogue text, synthesise audio, hear playback through an oscilloscope waveform display, and export WAV files — all from a single dark-industrial instrument-style interface.

The system ships as a Windows MSI distributable containing the KokoroTTS int8 ONNX model (~88 MB), the voice style vectors bin (~3 MB), and the espeak-ng phonemizer binary (~17 MB total with data). No internet connection is required at runtime.

---

## Functionality

### Core Concept — Position-as-Seed Voice Derivation

Every NPC voice is derived from three integers: their spawn X, Y, and Z coordinates. These coordinates are hashed with xxHash64 using five independent salts to produce five independent properties: which two Kokoro anchor voices to blend (A and B), the slerp scalar `t` along the arc between them, a pitch scale, and an energy scale. The 256-dimensional Kokoro style vector at the interpolated point between A and B is computed via spherical linear interpolation (slerp) and passed directly to the ONNX inference session.

Voice derivation is O(1) — no lookup tables, no stored state, no iteration. The same coordinates on any machine at any time produce identical voice parameters. NPC movement never changes the voice because only spawn coordinates are ever passed to the hash function.

### Voice Capacity

| Property | Value |
|---|---|
| Kokoro named anchor voices | 26 |
| Unique unordered voice pair arcs | 325 |
| Slerp `t` resolution (float32) | ~16.7 million steps |
| Effective distinct voice identities | ~5.4 billion |
| Per-NPC storage required | 0 bytes |
| Model disk footprint | 88 MB (int8 ONNX) |
| Voice style vectors | ~3 MB |

### Workbench UI — Three-Column Layout

```
┌──────────────────────────────────────────────────────────────────────────┐
│  ZER0VOICE WORKBENCH              v0.1.0 · KokoroTTS ONNX int8 · 88 MB  │
├───────────────────┬──────────────────────────────┬───────────────────────┤
│  COORDINATE INPUT │   VOICE SPEC                 │  SYNTHESIS            │
│                   │                              │                       │
│  X  [ -32768 ▲▼]  │  Voice A  [af_bella    ▼]    │ ┌─────────────────┐  │
│  Y  [      0 ▲▼]  │  Voice B  [bm_george   ▼]    │ │ ~ oscilloscope~ │  │
│  Z  [    312 ▲▼]  │                              │ └─────────────────┘  │
│                   │  Slerp t  [▓▓▓▓░░░] 0.6142   │                       │
│  [RANDOMISE    ]  │  Pitch    [▓▓▓░░░░] 1.0230   │  ┌─────────────────┐ │
│  [COPY HASH    ]  │  Energy   [▓░░░░░░] 0.8870   │  │ Enter dialogue… │ │
│                   │  □ LOCK t    □ LOCK Pitch     │  └─────────────────┘ │
│  World Seed       │  □ LOCK Energy               │                       │
│  [0xDEADBEEFC0FFE]│                              │  [▶  SYNTHESISE   ]   │
│                   │  Hash  0xa3f2c1d8...          │  [■  STOP         ]   │
│  [■] Region Bias  │  Arc   182 / 325              │                       │
│  [region map 2D ] │  □ Enable Region Bias         │  Duration:   0.00 s   │
│                   │                              │  Samples:       0     │
│  [P1][P2][P3][P4] │  [RESET OVERRIDES]           │  Rate:      24 kHz   │
│                   │                              │  [↓ EXPORT WAV    ]   │
├───────────────────┴──────────────────────────────┴───────────────────────┤
│ HISTORY  spawn(312,0,-88) · af_bella→bm_george · t=0.61 · "Stay out…" · 1.2s │
└──────────────────────────────────────────────────────────────────────────┘
```

### Panel 1 — Coordinate Input

The user enters integer X, Y, Z values representing an NPC spawn point. All downstream state derives from these three values.

- X, Y, Z inputs accept integers in range −32768 to 32767. Input type is `text` with manual parse and clamp — not `type="number"` (avoids browser spinner and negative-value UX issues).
- Arrow keys on a focused coordinate input increment/decrement by 1. Shift+Arrow = ±10. Ctrl+Arrow = ±100.
- Up/down stepper buttons flank each input.
- Input is debounced 80 ms before triggering rehash. The VoiceSpec panel updates immediately in the browser (via WASM hash) with no Tauri round-trip.
- **Randomise** button fills X/Y/Z using `crypto.getRandomValues()` — cryptographically uniform, no `Math.random()`.
- **Copy Hash** copies the hex representation of `position_hash(x, y, z, SALT_VOICE_A)` to clipboard.
- **World Seed** is a hex input field (default `0xDEADBEEFC0FFE`). Editing it triggers full rehash. It is a 64-bit unsigned integer stored as a `bigint` in TS and `u64` in Rust. Both sides must use the identical value.
- **Preset Slots P1–P4**: four bookmark buttons. Each slot stores the current X/Y/Z. Clicking a filled slot restores its coordinates. Long-press or right-click clears a slot. Slots persist in `localStorage` under keys `zv_preset_0` through `zv_preset_3`.

### Panel 2 — Voice Spec

Displays every derived property from the current coordinate hash. When a parameter is locked, its slider becomes interactive and the locked value (not the hash-derived value) is passed to synthesis.

**Readout rows:**

| Parameter | Display | Override |
|---|---|---|
| Voice A | Name badge, gender dot (● female, ○ male), index 0–25 | Dropdown — any of 26 Kokoro voices |
| Voice B | Same; B ≠ A enforced (dropdown filters out current A) | Dropdown |
| Slerp t | Amber fill-track slider [0.000–1.000], 4-decimal readout | Enabled when "LOCK t" checkbox is checked |
| Pitch scale | Slider [0.850–1.150], 3-decimal readout | Enabled when "LOCK Pitch" checked |
| Energy scale | Slider [0.800–1.200], 3-decimal readout | Enabled when "LOCK Energy" checked |
| Hash | Hex string, truncated, with copy icon | Read-only always |
| Arc index | `182 / 325` format | Read-only always |

When any parameter is locked, the panel title shows `[OVERRIDE ACTIVE]` in amber. **Reset Overrides** button unchecks all locks and restores all sliders to hash-derived values.

Voice pair arc visualiser: a small inline canvas (280 × 32 px) draws a semicircular arc in amber, with named endpoints at each end and a draggable dot at position `t`. Dragging the dot sets `t` and automatically checks the "LOCK t" checkbox.

**Region Bias toggle**: checkbox enables the optional regional coherent noise bias, which constrains which voice indices are selected based on the X/Z position in world space. When enabled, the 2D bias map in the Coordinate Panel becomes interactive.

### Panel 3 — Synthesis

- **Textarea**: multi-line text input, max 512 characters, character counter `n / 512` in the bottom-right corner of the textarea. Placeholder: `"Enter NPC dialogue…"`. Disabled during synthesis.
- **Synthesise button**: triggers the Tauri `synthesize_npc_speech` command. Shows a spinner (`⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏`) with text `"GENERATING…"` during synthesis. Disabled during synthesis and if textarea is empty.
- **Stop button**: cancels in-flight Web Audio playback. Does not cancel ONNX inference (which runs in Rust and cannot be interrupted mid-computation).
- **Waveform canvas** (480 × 80 px, `imageRendering: pixelated`):
  - *Idle state*: flat green line at midpoint.
  - *Synthesising state*: animated amber scanning bar sweeping left to right (driven by `requestAnimationFrame`).
  - *Playing state*: oscilloscope — full PCM waveform drawn in amber, white playhead line advancing in real time.
  - *Stopped/done*: waveform frozen on last frame.
- **Duration**, **Samples**, **Rate** readouts update after each successful synthesis.
- **Export WAV** button: encodes the last PCM buffer as a 16-bit mono 24 kHz WAV and triggers a browser download. Disabled if no PCM is loaded.

**Synthesis state machine:**
```
IDLE ──[synthesise click]──► SYNTHESISING ──[ort returns]──► PLAYING ──[ended]──► IDLE
                                    │                             │
                              [ort error]                   [stop click]
                                    │                             │
                                  ERROR                        IDLE
```

### History Bar

A horizontally scrolling bar at the bottom of the window. Each synthesis attempt appends an entry:

```
spawn(312, 0, -88) · af_bella → bm_george · t=0.6142 · "Stay out of the forest after dark." · 1.24 s
```

Maximum 50 entries. Oldest are removed when limit is reached. Clicking an entry restores its exact coordinate and lock state. A **Clear** button empties the log. An **Export JSON** button downloads the full log as `zerovoice-history.json`. History does not persist across app restarts (session-only).

### Aesthetic Specification

**Theme**: dark industrial workstation — audio firmware meets game debug tool. Hard edges, no border-radius on panels, 1 px borders throughout.

**Fonts**: `IBM Plex Mono` (400, 500) for all numbers, values, labels, code. `IBM Plex Sans Condensed` (400, 600) for panel headings only. Both loaded from Google Fonts.

**Colour palette (CSS custom properties)**:
```
--bg-void:       #0d0d0f   (window background)
--bg-panel:      #111114   (panel backgrounds)
--bg-control:    #18181c   (input/control backgrounds)
--bg-hover:      #1e1e24   (hover state)
--border-dim:    #2a2a32   (default borders)
--border-mid:    #3a3a46   (elevated borders)
--border-bright: #505060   (active/focus borders)
--amber:         #e8a020   (primary accent — numbers, active controls, waveform)
--amber-dim:     #7a5010   (disabled amber elements)
--amber-glow:    #e8a02033 (amber background tint)
--slate:         #4a7fa5   (voice pair indicators, male badge)
--slate-dim:     #243d52   (male badge background)
--text-primary:  #d4d0c8
--text-secondary:#7a7870
--text-amber:    #e8a020
--text-slate:    #6aafd4
--female-border: #7a4a8a
--female-bg:     #2a1a35
--female-text:   #c49ad4
--green-ready:   #2a6040   (waveform idle state)
```

**Scanline texture**: a `::after` pseudo-element on `body` with `position: fixed; inset: 0; pointer-events: none; z-index: 9999` applies `repeating-linear-gradient(0deg, transparent 0px, transparent 2px, rgba(0,0,0,0.06) 2px, rgba(0,0,0,0.06) 4px)` — this creates a subtle CRT scanline effect over the entire window with zero performance cost.

**Slider appearance**: custom `input[type="range"]` — 2 px track height, 10 × 18 px amber thumb, no default appearance. Track fill is achieved with a CSS custom property `--fill-pct` updated by React on every value change and applied via `linear-gradient` on the track.

**Buttons**: two variants: `.btn-primary` (amber background, black text) and `.btn-secondary` (transparent, `--border-mid` border, `--text-secondary` text). No border-radius. Monospace font.

**Inputs**: `--bg-control` background, `--border-mid` border, amber caret, amber text for coordinate values. Focus state changes border to `--amber`.

---

## Technical Implementation

### Repository Layout

```
zerov0ice/
├── src-tauri/
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   ├── resources/
│   │   ├── kokoro-v1.0.int8.onnx       (download separately — not in repo)
│   │   ├── voices-v1.0.bin             (download separately — not in repo)
│   │   ├── espeak-ng.exe               (download separately — not in repo)
│   │   └── espeak-ng-data/             (download separately — not in repo)
│   └── src/
│       ├── main.rs
│       ├── lib.rs
│       ├── zerovoice.rs                (hash layer + VoiceSpec)
│       ├── slerp.rs                    (voice table loading + slerp fn)
│       ├── kokoro_tts.rs               (ONNX session + synthesize)
│       ├── phonemize.rs                (espeak-ng subprocess wrapper)
│       └── commands/
│           └── voice.rs               (Tauri command handlers)
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── components/
│   │   ├── workbench/
│   │   │   ├── WorkbenchShell.tsx
│   │   │   ├── CoordinatePanel.tsx
│   │   │   ├── VoiceSpecPanel.tsx
│   │   │   ├── SynthesisPanel.tsx
│   │   │   ├── RegionBiasMap.tsx
│   │   │   └── HistoryBar.tsx
│   │   ├── controls/
│   │   │   ├── CoordInput.tsx
│   │   │   ├── ParamSlider.tsx
│   │   │   ├── VoiceSelector.tsx
│   │   │   ├── WaveformCanvas.tsx
│   │   │   ├── HashDisplay.tsx
│   │   │   └── ArcVisualiser.tsx
│   │   └── ui/
│   │       ├── SegmentNumber.tsx
│   │       └── StatusBar.tsx
│   ├── hooks/
│   │   ├── useVoiceSpec.ts
│   │   ├── useSynthesis.ts
│   │   └── useRegionBias.ts
│   ├── lib/
│   │   ├── zerovoice-preview.ts
│   │   ├── voice.ts
│   │   ├── waveform.ts
│   │   └── history.ts
│   ├── workers/
│   │   └── regionBias.worker.ts
│   └── styles/
│       ├── workbench.css
│       └── controls.css
├── package.json
├── tsconfig.json
└── vite.config.ts
```

### Data Models

```typescript
// Coordinates — the only input to the hash layer
interface Coords {
  x: number;  // i32, clamped −32768..32767
  y: number;
  z: number;
}

// Derived from coords — never stored, always recomputed
interface VoiceSpec {
  voiceA:      number;   // 0–25, index into KOKORO_VOICES
  voiceB:      number;   // 0–25, always ≠ voiceA
  voiceAName:  string;   // e.g. "af_bella"
  voiceBName:  string;   // e.g. "bm_george"
  t:           number;   // [0.0, 1.0] — slerp scalar
  pitchScale:  number;   // [0.85, 1.15]
  energyScale: number;   // [0.80, 1.20]
  hashHex:     string;   // 16-char hex string for display
  arcIndex:    number;   // 0–324, which of 325 unordered arcs
}

// Override state — which params are manually locked
interface Overrides {
  t:      { locked: boolean; value: number };
  pitch:  { locked: boolean; value: number };
  energy: { locked: boolean; value: number };
  voiceA: { locked: boolean; value: number };
  voiceB: { locked: boolean; value: number };
}

// One entry in the history bar
interface HistoryEntry {
  coords:    Coords;
  spec:      VoiceSpec;
  text:      string;
  durationS: number;
  timestamp: number;  // Date.now()
}

// Preset slot (persisted to localStorage)
interface PresetSlot {
  coords: Coords | null;
  label:  string;        // e.g. "P1"
}

// Synthesis state machine
type SynthesisState = "idle" | "synthesising" | "playing" | "error";
```

```rust
// src-tauri/src/zerovoice.rs — Rust-side data model
pub struct VoiceSpec {
    pub voice_a:      usize,   // 0–25
    pub voice_b:      usize,   // 0–25, always ≠ voice_a
    pub t:            f32,     // [0.0, 1.0]
    pub pitch_scale:  f32,     // [0.85, 1.15]
    pub energy_scale: f32,     // [0.80, 1.20]
}

// Serialisable DTO for preview_voice_spec command
#[derive(serde::Serialize)]
pub struct VoiceSpecDto {
    pub voice_a_idx:  usize,
    pub voice_a_name: String,
    pub voice_b_idx:  usize,
    pub voice_b_name: String,
    pub t:            f32,
    pub pitch_scale:  f32,
    pub energy_scale: f32,
    pub hash_hex:     String,
    pub arc_index:    usize,
}
```

### Kokoro Voice Name Table

The 26 Kokoro v1.0 voices in index order (this ordering must be identical in both Rust and TypeScript):

```rust
// src-tauri/src/zerovoice.rs
pub const KOKORO_VOICES: [&str; 26] = [
    "af_alloy", "af_aoede", "af_bella", "af_jessica", "af_kore",
    "af_nicole", "af_nova", "af_river", "af_sarah", "af_sky",
    "am_adam", "am_echo", "am_eric", "am_fenrir", "am_liam",
    "am_michael", "am_onyx", "am_puck", "bf_alice", "bf_emma",
    "bf_isabella", "bf_lily", "bm_daniel", "bm_fable", "bm_george",
    "bm_lewis",
];
// Prefix key: a=American, b=British; f=female, m=male
```

```typescript
// src/lib/zerovoice-preview.ts
export const KOKORO_VOICES: string[] = [
  "af_alloy", "af_aoede", "af_bella", "af_jessica", "af_kore",
  "af_nicole", "af_nova", "af_river", "af_sarah", "af_sky",
  "am_adam", "am_echo", "am_eric", "am_fenrir", "am_liam",
  "am_michael", "am_onyx", "am_puck", "bf_alice", "bf_emma",
  "bf_isabella", "bf_lily", "bm_daniel", "bm_fable", "bm_george",
  "bm_lewis",
];
export const isFemale = (name: string) => name[1] === "f";
```

---

## Implementation Steps

### Step 1 — Scaffold the Tauri + Vite Project

```bash
npm create tauri-app@latest zerov0ice -- \
  --template react-ts \
  --manager npm

cd zerov0ice
npm install
```

Verify `src-tauri/Cargo.toml` was created, then proceed to Step 2 before running.

---

### Step 2 — Configure `Cargo.toml`

Replace the contents of `src-tauri/Cargo.toml`:

```toml
[package]
name    = "zerov0ice"
version = "0.1.0"
edition = "2021"

[lib]
name         = "zerov0ice_lib"
crate-type   = ["staticlib", "cdylib", "rlib"]

[build-dependencies]
tauri-build = { version = "2", features = [] }

[dependencies]
tauri       = { version = "2", features = ["macos-private-api"] }
tauri-plugin-shell = { version = "2" }
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
anyhow      = "1"
ort         = { version = "2", features = ["load-dynamic"] }
ndarray     = "0.15"
xxhash-rust = { version = "0.8", features = ["xxh64"] }
npyz        = "0.8"
```

---

### Step 3 — Configure `tauri.conf.json`

Replace `src-tauri/tauri.conf.json`:

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "ZeroVoice Workbench",
  "version": "0.1.0",
  "identifier": "dev.zerovoice.workbench",
  "build": {
    "frontendDist": "../dist",
    "devUrl": "http://localhost:1420",
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build"
  },
  "app": {
    "windows": [
      {
        "title": "ZER0VOICE WORKBENCH",
        "width": 1140,
        "height": 700,
        "minWidth": 900,
        "minHeight": 560,
        "resizable": true,
        "decorations": true,
        "theme": "Dark"
      }
    ],
    "security": { "csp": null }
  },
  "bundle": {
    "active": true,
    "targets": "all",
    "resources": {
      "resources/kokoro-v1.0.int8.onnx": "resources/kokoro-v1.0.int8.onnx",
      "resources/voices-v1.0.bin":        "resources/voices-v1.0.bin",
      "resources/espeak-ng.exe":          "resources/espeak-ng.exe",
      "resources/espeak-ng-data":         "resources/espeak-ng-data"
    },
    "windows": {
      "wix": {},
      "nsis": {}
    }
  },
  "plugins": {
    "shell": {
      "open": false,
      "sidecar": false,
      "scope": [
        { "name": "resources/espeak-ng", "cmd": "resources/espeak-ng.exe", "args": true }
      ]
    }
  }
}
```

---

### Step 4 — Install Frontend Dependencies

```bash
npm install xxhash-wasm
npm install --save-dev @types/react @types/react-dom
```

Update `package.json` to confirm:

```json
{
  "name": "zerov0ice",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev":   "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@tauri-apps/api":    "^2",
    "@tauri-apps/plugin-shell": "^2",
    "react":             "^18",
    "react-dom":         "^18",
    "xxhash-wasm":       "^1.0.1"
  },
  "devDependencies": {
    "@types/react":      "^18",
    "@types/react-dom":  "^18",
    "@vitejs/plugin-react": "^4",
    "typescript":        "^5",
    "vite":              "^5"
  }
}
```

---

### Step 5 — Rust: ZeroBytes Hash Layer (`src-tauri/src/zerovoice.rs`)

This file is the ZeroBytes core. All five voice properties are derived from a single call to `position_hash` per property, each with a unique salt. The salt constants are named and typed to prevent accidental collision.

```rust
use xxhash_rust::xxh64::Xxh64;

pub const KOKORO_VOICES: [&str; 26] = [
    "af_alloy", "af_aoede", "af_bella", "af_jessica", "af_kore",
    "af_nicole", "af_nova", "af_river", "af_sarah", "af_sky",
    "am_adam",  "am_echo",  "am_eric",  "am_fenrir", "am_liam",
    "am_michael","am_onyx", "am_puck",  "bf_alice",  "bf_emma",
    "bf_isabella","bf_lily","bm_daniel","bm_fable",  "bm_george",
    "bm_lewis",
];

pub const NUM_VOICES: u64 = 26;

// Salt constants — each property gets a unique, collision-free salt
const SALT_VOICE_A: u64 = 0x0001;
const SALT_VOICE_B: u64 = 0x0002;
const SALT_SLERP_T: u64 = 0x0003;
const SALT_PITCH:   u64 = 0x0004;
const SALT_ENERGY:  u64 = 0x0005;
const SALT_REGION:  u64 = 0x0006;

/// The world seed. Changing this changes every NPC voice — treat as a
/// save-file breaking change and increment the save schema version.
pub const WORLD_SEED: u64 = 0xDEAD_BEEF_C0FFE;

pub struct VoiceSpec {
    pub voice_a:      usize,
    pub voice_b:      usize,
    pub t:            f32,
    pub pitch_scale:  f32,
    pub energy_scale: f32,
}

#[derive(serde::Serialize)]
pub struct VoiceSpecDto {
    pub voice_a_idx:  usize,
    pub voice_a_name: String,
    pub voice_b_idx:  usize,
    pub voice_b_name: String,
    pub t:            f32,
    pub pitch_scale:  f32,
    pub energy_scale: f32,
    pub hash_hex:     String,
    pub arc_index:    usize,
}

/// Hash spawn coordinates with an independent salt per property.
/// Uses spawn point only — current position MUST NEVER be passed here.
/// The xxh64 output is platform-independent (same result on all OSes).
pub fn position_hash(sx: i32, sy: i32, sz: i32, salt: u64) -> u64 {
    let mut h = Xxh64::new(WORLD_SEED ^ salt);
    h.update(&sx.to_le_bytes());
    h.update(&sy.to_le_bytes());
    h.update(&sz.to_le_bytes());
    h.digest()
}

/// Map the lower 32 bits of a hash to a float in [0.0, 1.0).
pub fn hash_to_float(h: u64) -> f32 {
    (h & 0xFFFF_FFFF) as f32 / 0x1_0000_0000_u64 as f32
}

/// Derive a VoiceSpec from spawn coordinates using the world seed.
/// This is O(1) — no iteration, no stored state.
pub fn voice_from_spawn(sx: i32, sy: i32, sz: i32) -> VoiceSpec {
    let h_a   = position_hash(sx, sy, sz, SALT_VOICE_A);
    let h_b   = position_hash(sx, sy, sz, SALT_VOICE_B);
    let h_t   = position_hash(sx, sy, sz, SALT_SLERP_T);
    let h_p   = position_hash(sx, sy, sz, SALT_PITCH);
    let h_e   = position_hash(sx, sy, sz, SALT_ENERGY);

    let idx_a = (h_a % NUM_VOICES) as usize;
    // Ensure B ≠ A: map into (NUM_VOICES - 1) remaining slots, then offset
    let idx_b = ((h_b % (NUM_VOICES - 1)) as usize + idx_a + 1) % NUM_VOICES as usize;

    VoiceSpec {
        voice_a:      idx_a,
        voice_b:      idx_b,
        t:            hash_to_float(h_t),
        pitch_scale:  0.85 + hash_to_float(h_p) * 0.30,
        energy_scale: 0.80 + hash_to_float(h_e) * 0.40,
    }
}

/// Convert VoiceSpec to a serialisable DTO for the preview Tauri command.
pub fn spec_to_dto(sx: i32, sy: i32, sz: i32, spec: &VoiceSpec) -> VoiceSpecDto {
    let hash_hex = format!("{:016x}", position_hash(sx, sy, sz, SALT_VOICE_A));
    let arc_index = {
        let a = spec.voice_a;
        let b = spec.voice_b;
        let (lo, hi) = if a < b { (a, b) } else { (b, a) };
        // Map unordered pair to 0..324
        lo * (NUM_VOICES as usize - 1) - (lo * lo.saturating_sub(1)) / 2 + hi - lo - 1
    };
    VoiceSpecDto {
        voice_a_idx:  spec.voice_a,
        voice_a_name: KOKORO_VOICES[spec.voice_a].to_string(),
        voice_b_idx:  spec.voice_b,
        voice_b_name: KOKORO_VOICES[spec.voice_b].to_string(),
        t:            spec.t,
        pitch_scale:  spec.pitch_scale,
        energy_scale: spec.energy_scale,
        hash_hex,
        arc_index,
    }
}

/// Optional: regional coherent noise bias. Constrains voice index selection
/// to a sub-range based on slowly-varying noise across the X/Z plane.
/// Scale factor 0.005 → voice "dialects" change roughly every 200 world units.
pub fn regional_voice_bias(sx: i32, sz: i32) -> f32 {
    let freq = 0.005_f32;
    let x0 = (sx as f32 * freq) as i32;
    let z0 = (sz as f32 * freq) as i32;
    let sx_f = smooth_step((sx as f32 * freq).fract());
    let sz_f = smooth_step((sz as f32 * freq).fract());

    let n00 = hash_to_float(position_hash(x0,     z0,     0, SALT_REGION));
    let n10 = hash_to_float(position_hash(x0 + 1, z0,     0, SALT_REGION));
    let n01 = hash_to_float(position_hash(x0,     z0 + 1, 0, SALT_REGION));
    let n11 = hash_to_float(position_hash(x0 + 1, z0 + 1, 0, SALT_REGION));

    let nx0 = lerp(n00, n10, sx_f);
    let nx1 = lerp(n01, n11, sx_f);
    lerp(nx0, nx1, sz_f)
}

fn smooth_step(t: f32) -> f32 { t * t * (3.0 - 2.0 * t) }
fn lerp(a: f32, b: f32, t: f32) -> f32 { a + (b - a) * t }

/// voice_from_spawn with regional bias applied.
/// bias < 0.33  → voice pair drawn from indices 0–8   (American female / light)
/// bias 0.33..0.66 → indices 9–17 (American male / mid)
/// bias > 0.66  → indices 18–25 (British / deep)
pub fn voice_from_spawn_biased(sx: i32, sy: i32, sz: i32) -> VoiceSpec {
    let bias = regional_voice_bias(sx, sz);
    let h_a  = position_hash(sx, sy, sz, SALT_VOICE_A);
    let h_b  = position_hash(sx, sy, sz, SALT_VOICE_B);
    let h_t  = position_hash(sx, sy, sz, SALT_SLERP_T);
    let h_p  = position_hash(sx, sy, sz, SALT_PITCH);
    let h_e  = position_hash(sx, sy, sz, SALT_ENERGY);

    let (range_start, range_len): (u64, u64) = if bias < 0.33 {
        (0, 9)
    } else if bias < 0.66 {
        (9, 9)
    } else {
        (18, 8)
    };

    let idx_a = (range_start + h_a % range_len) as usize;
    let idx_b = ((h_b % (NUM_VOICES - 1)) as usize + idx_a + 1) % NUM_VOICES as usize;

    VoiceSpec {
        voice_a:      idx_a,
        voice_b:      idx_b,
        t:            hash_to_float(h_t),
        pitch_scale:  0.85 + hash_to_float(h_p) * 0.30,
        energy_scale: 0.80 + hash_to_float(h_e) * 0.40,
    }
}
```

---

### Step 6 — Rust: Voice Table + Slerp (`src-tauri/src/slerp.rs`)

```rust
use anyhow::{Context, Result};
use npyz::NpyFile;
use std::collections::HashMap;
use crate::zerovoice::{KOKORO_VOICES, VoiceSpec};

pub type VoiceTable = Vec<Vec<f32>>;

/// Load all 26 voice style vectors from Kokoro's voices-v1.0.bin (NPZ format).
/// Each voice has shape (N_styles, 1, 256) float32.
/// We mean-pool the N_styles axis to produce one 256-dim representative vector.
pub fn load_voice_table(bin_path: &str) -> Result<VoiceTable> {
    let data = std::fs::read(bin_path)
        .with_context(|| format!("cannot read voice bin: {bin_path}"))?;

    // NPZ is a ZIP archive containing one .npy file per voice name.
    let mut zip = zip::ZipArchive::new(std::io::Cursor::new(&data))
        .context("voices-v1.0.bin is not a valid ZIP/NPZ archive")?;

    let mut raw: HashMap<String, Vec<f32>> = HashMap::new();
    for i in 0..zip.len() {
        let mut file = zip.by_index(i)?;
        let name = file.name().trim_end_matches(".npy").to_string();
        let npy = NpyFile::new(&mut file)?;
        let flat: Vec<f32> = npy.into_vec()?;
        raw.insert(name, flat);
    }

    // For each voice, mean-pool over the N_styles axis to get shape [256].
    // Shape is (N, 1, 256) → stride is 256 per style row.
    let mut table: VoiceTable = Vec::with_capacity(KOKORO_VOICES.len());
    for &voice_name in &KOKORO_VOICES {
        let flat = raw.get(voice_name)
            .with_context(|| format!("voice '{voice_name}' missing from voices-v1.0.bin"))?;
        let dim = 256_usize;
        assert!(flat.len() % dim == 0, "unexpected shape for {voice_name}");
        let n_styles = flat.len() / dim;

        // Mean-pool
        let mut mean = vec![0.0_f32; dim];
        for row in 0..n_styles {
            for d in 0..dim {
                mean[d] += flat[row * dim + d];
            }
        }
        let n = n_styles as f32;
        mean.iter_mut().for_each(|v| *v /= n);

        // L2-normalise so slerp operates on unit vectors
        let norm: f32 = mean.iter().map(|v| v * v).sum::<f32>().sqrt();
        if norm > 1e-8 {
            mean.iter_mut().for_each(|v| *v /= norm);
        }

        table.push(mean);
    }

    Ok(table)
}

/// Spherical linear interpolation between two L2-normalised 256-dim vectors.
/// Preserves vector magnitude — prevents the energy collapse that lerp causes
/// toward the midpoint of two style vectors.
pub fn slerp(a: &[f32], b: &[f32], t: f32) -> Vec<f32> {
    debug_assert_eq!(a.len(), b.len());
    let dot: f32 = a.iter().zip(b).map(|(x, y)| x * y).sum::<f32>().clamp(-1.0, 1.0);
    let theta = dot.acos();

    if theta.abs() < 1e-6 {
        // Nearly identical vectors — fall back to lerp (avoid divide by ~0)
        return a.iter().zip(b).map(|(x, y)| x * (1.0 - t) + y * t).collect();
    }

    let sin_theta = theta.sin();
    let sa = ((1.0 - t) * theta).sin() / sin_theta;
    let sb = (t * theta).sin()         / sin_theta;

    a.iter().zip(b).map(|(x, y)| sa * x + sb * y).collect()
}

/// Resolve the final 256-dim style vector for a VoiceSpec.
pub fn resolve_style(table: &VoiceTable, spec: &VoiceSpec) -> Vec<f32> {
    slerp(&table[spec.voice_a], &table[spec.voice_b], spec.t)
}
```

`Cargo.toml` needs the `zip` crate for NPZ parsing:

```toml
zip  = { version = "2", default-features = false, features = ["deflate"] }
npyz = "0.8"
```

---

### Step 7 — Rust: Phonemization (`src-tauri/src/phonemize.rs`)

```rust
use anyhow::{Context, Result};
use std::collections::HashMap;

/// Run espeak-ng as a subprocess to convert raw text to IPA phonemes,
/// then map IPA symbols to Kokoro's integer token vocabulary.
pub fn phonemize(text: &str, espeak_path: &str, data_dir: &str) -> Result<Vec<i64>> {
    let output = std::process::Command::new(espeak_path)
        .args([
            "-q",           // quiet — no audio
            "--ipa",        // output IPA
            "-v", "en-us",  // American English
            "--stdin",
        ])
        .env("ESPEAK_DATA_PATH", data_dir)
        .stdin(std::process::Stdio::piped())
        .stdout(std::process::Stdio::piped())
        .stderr(std::process::Stdio::null())
        .spawn()
        .context("failed to spawn espeak-ng")?;

    use std::io::Write;
    output.stdin.as_ref().unwrap().write_all(text.as_bytes())?;
    let result = output.wait_with_output()?;
    let ipa = String::from_utf8_lossy(&result.stdout).into_owned();

    ipa_to_tokens(&ipa)
}

/// Map IPA string to Kokoro token IDs.
/// Kokoro uses a fixed phoneme vocabulary; unknown symbols are skipped.
fn ipa_to_tokens(ipa: &str) -> Result<Vec<i64>> {
    // Kokoro token map — each entry is (IPA symbol, token id)
    // This is the subset of the full IPA that Kokoro-82M was trained on.
    static TOKEN_MAP: std::sync::OnceLock<HashMap<&'static str, i64>> = std::sync::OnceLock::new();
    let map = TOKEN_MAP.get_or_init(|| {
        HashMap::from([
            (" ", 16), (".", 4),  (",", 7),  ("!", 30), ("?", 44),
            ("ə", 43), ("ɪ", 61), ("ʊ", 86), ("ɛ", 46), ("æ", 35),
            ("ɑ", 34), ("ɔ", 65), ("ʌ", 85), ("i", 53), ("u", 81),
            ("e", 43), ("o", 62), ("a", 35), ("p", 68), ("b", 36),
            ("t", 76), ("d", 41), ("k", 55), ("ɡ", 50), ("f", 47),
            ("v", 82), ("θ", 79), ("ð", 42), ("s", 73), ("z", 90),
            ("ʃ", 75), ("ʒ", 91), ("h", 51), ("tʃ", 78),("dʒ", 44),
            ("m", 60), ("n", 62), ("ŋ", 64), ("l", 57), ("r", 71),
            ("w", 83), ("j", 54), ("ˈ", 135),("ˌ", 138),("ː", 158),
        ])
    });

    let mut tokens: Vec<i64> = Vec::new();
    let chars: Vec<char> = ipa.chars().collect();
    let mut i = 0;
    while i < chars.len() {
        // Try two-character digraphs first
        if i + 1 < chars.len() {
            let digraph: String = [chars[i], chars[i + 1]].iter().collect();
            if let Some(&tok) = map.get(digraph.as_str()) {
                tokens.push(tok);
                i += 2;
                continue;
            }
        }
        let ch: String = chars[i].to_string();
        if let Some(&tok) = map.get(ch.as_str()) {
            tokens.push(tok);
        }
        i += 1;
    }

    // Kokoro requires pad token 0 at start and end; max context 510 tokens
    tokens.truncate(510);
    let mut padded = vec![0_i64];
    padded.extend(tokens);
    padded.push(0);

    Ok(padded)
}
```

---

### Step 8 — Rust: ONNX Session (`src-tauri/src/kokoro_tts.rs`)

```rust
use anyhow::{Context, Result};
use ndarray::Array;
use ort::{Session, SessionBuilder};
use crate::slerp::{VoiceTable, load_voice_table, resolve_style};
use crate::zerovoice::VoiceSpec;

pub struct KokoroSession {
    session:     Session,
    voice_table: VoiceTable,
}

impl KokoroSession {
    /// Initialise the ONNX session and load the voice style table.
    /// Call once at app startup; store in Tauri managed state.
    pub fn new(model_path: &str, voices_path: &str) -> Result<Self> {
        let session = SessionBuilder::new()
            .context("failed to create ORT SessionBuilder")?
            .with_intra_threads(2)
            .context("failed to set intra_threads")?
            .commit_from_file(model_path)
            .with_context(|| format!("failed to load ONNX model: {model_path}"))?;

        let voice_table = load_voice_table(voices_path)?;

        Ok(Self { session, voice_table })
    }

    /// Synthesise speech from token IDs and a VoiceSpec (with optional overrides).
    /// Returns raw float32 PCM samples at 24 kHz, mono.
    pub fn synthesize(
        &self,
        tokens:    &[i64],
        spec:      &VoiceSpec,
        override_t:      Option<f32>,
        override_pitch:  Option<f32>,
        override_energy: Option<f32>,
    ) -> Result<Vec<f32>> {
        // Apply overrides
        let mut working = VoiceSpec {
            voice_a:      spec.voice_a,
            voice_b:      spec.voice_b,
            t:            override_t.unwrap_or(spec.t),
            pitch_scale:  override_pitch.unwrap_or(spec.pitch_scale),
            energy_scale: override_energy.unwrap_or(spec.energy_scale),
        };

        let style_vec = resolve_style(&self.voice_table, &working);

        // Build input tensors
        let n = tokens.len();
        let tokens_arr = Array::from_shape_vec((1, n), tokens.to_vec())
            .context("failed to build tokens array")?;
        let style_arr  = Array::from_shape_vec((1, 256), style_vec)
            .context("failed to build style array")?;
        let speed_arr  = Array::from_elem((1,), working.pitch_scale);

        let outputs = self.session.run(ort::inputs![
            "tokens" => tokens_arr,
            "style"  => style_arr,
            "speed"  => speed_arr,
        ]?)?;

        // Extract waveform — shape [1, N_samples] or [N_samples]
        let waveform = outputs["waveform"]
            .try_extract_tensor::<f32>()
            .context("failed to extract waveform tensor")?;

        Ok(waveform.view().iter().copied().collect())
    }
}
```

---

### Step 9 — Rust: Tauri Command Layer (`src-tauri/src/commands/voice.rs`)

```rust
use tauri::State;
use crate::kokoro_tts::KokoroSession;
use crate::zerovoice::{voice_from_spawn, voice_from_spawn_biased, spec_to_dto, VoiceSpecDto};
use crate::phonemize::phonemize;

/// Main synthesis command. Called by the frontend Synthesise button.
/// Takes raw dialogue text (backend handles phonemization).
/// Accepts optional override values for any locked parameters.
#[tauri::command]
pub async fn synthesize_npc_speech(
    state:           State<'_, KokoroSession>,
    app:             tauri::AppHandle,
    spawn_x:         i32,
    spawn_y:         i32,
    spawn_z:         i32,
    text:            String,
    use_region_bias: bool,
    override_t:      Option<f32>,
    override_pitch:  Option<f32>,
    override_energy: Option<f32>,
    override_voice_a: Option<usize>,
    override_voice_b: Option<usize>,
) -> Result<Vec<f32>, String> {
    // Resolve resource paths from the app bundle
    let model_dir = app.path().resource_dir()
        .map_err(|e| e.to_string())?
        .join("resources");

    let espeak_exe = model_dir.join("espeak-ng.exe");
    let espeak_data = model_dir.join("espeak-ng-data");

    // Derive the VoiceSpec from spawn coords
    let mut spec = if use_region_bias {
        voice_from_spawn_biased(spawn_x, spawn_y, spawn_z)
    } else {
        voice_from_spawn(spawn_x, spawn_y, spawn_z)
    };

    // Apply voice index overrides
    if let Some(a) = override_voice_a { spec.voice_a = a; }
    if let Some(b) = override_voice_b { spec.voice_b = b; }

    // Phonemize
    let tokens = phonemize(
        &text,
        espeak_exe.to_str().unwrap(),
        espeak_data.to_str().unwrap(),
    ).map_err(|e| e.to_string())?;

    // Synthesise
    state.synthesize(&tokens, &spec, override_t, override_pitch, override_energy)
        .map_err(|e| e.to_string())
}

/// Lightweight preview command — derives VoiceSpec and returns DTO without synthesis.
/// Used by the frontend to cross-check the WASM browser-side hash preview.
#[tauri::command]
pub fn preview_voice_spec(
    spawn_x:         i32,
    spawn_y:         i32,
    spawn_z:         i32,
    use_region_bias: bool,
) -> VoiceSpecDto {
    let spec = if use_region_bias {
        voice_from_spawn_biased(spawn_x, spawn_y, spawn_z)
    } else {
        voice_from_spawn(spawn_x, spawn_y, spawn_z)
    };
    spec_to_dto(spawn_x, spawn_y, spawn_z, &spec)
}
```

---

### Step 10 — Rust: `main.rs` and `lib.rs`

```rust
// src-tauri/src/lib.rs
mod zerovoice;
mod slerp;
mod kokoro_tts;
mod phonemize;
pub mod commands;

use tauri::Manager;
use kokoro_tts::KokoroSession;

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_shell::init())
        .setup(|app| {
            let resource_dir = app.path().resource_dir()
                .expect("failed to get resource dir")
                .join("resources");

            let model_path  = resource_dir.join("kokoro-v1.0.int8.onnx");
            let voices_path = resource_dir.join("voices-v1.0.bin");

            let session = KokoroSession::new(
                model_path.to_str().unwrap(),
                voices_path.to_str().unwrap(),
            ).expect("failed to initialise KokoroTTS ONNX session");

            app.manage(session);
            Ok(())
        })
        .invoke_handler(tauri::generate_handler![
            commands::voice::synthesize_npc_speech,
            commands::voice::preview_voice_spec,
        ])
        .run(tauri::generate_context!())
        .expect("error running Tauri application");
}
```

```rust
// src-tauri/src/main.rs
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]
fn main() { zerov0ice_lib::run(); }
```

```rust
// src-tauri/src/commands/mod.rs
pub mod voice;
```

---

### Step 11 — TypeScript: Browser-Side Hash Preview (`src/lib/zerovoice-preview.ts`)

This module mirrors `zerovoice.rs` exactly using `xxhash-wasm`. It runs in the browser with no Tauri round-trip, giving instant VoiceSpec feedback on every keystroke. The Tauri `preview_voice_spec` command exists as a cross-check tool during development.

```typescript
import { createXXHash64 } from "xxhash-wasm";
import type { VoiceSpec } from "./types";

export const KOKORO_VOICES: readonly string[] = [
  "af_alloy", "af_aoede", "af_bella",   "af_jessica", "af_kore",
  "af_nicole", "af_nova",  "af_river",   "af_sarah",   "af_sky",
  "am_adam",  "am_echo",  "am_eric",    "am_fenrir",  "am_liam",
  "am_michael","am_onyx", "am_puck",    "bf_alice",   "bf_emma",
  "bf_isabella","bf_lily","bm_daniel",  "bm_fable",   "bm_george",
  "bm_lewis",
] as const;

export const isFemale = (name: string): boolean => name[1] === "f";

const NUM_VOICES = BigInt(KOKORO_VOICES.length); // 26n
const WORLD_SEED = 0xDEADBEEFC0FFEn;

// Salts — must match Rust constants exactly
const SALT_VOICE_A = 0x0001n;
const SALT_VOICE_B = 0x0002n;
const SALT_SLERP_T = 0x0003n;
const SALT_PITCH   = 0x0004n;
const SALT_ENERGY  = 0x0005n;

let _xxh64: Awaited<ReturnType<typeof createXXHash64>> | null = null;

async function getHasher() {
  if (!_xxh64) _xxh64 = await createXXHash64();
  return _xxh64;
}

function hashCoords(
  xxh64: Awaited<ReturnType<typeof createXXHash64>>,
  sx: number, sy: number, sz: number,
  salt: bigint
): bigint {
  const seed = WORLD_SEED ^ salt;
  // xxhash-wasm accepts a Uint8Array and a seed (as number — seed truncated to 32 bits
  // for the wasm api; we XOR the full u64 salt into seed manually)
  const buf = new ArrayBuffer(12);
  const view = new DataView(buf);
  view.setInt32(0, sx, true);
  view.setInt32(4, sy, true);
  view.setInt32(8, sz, true);
  // xxhash-wasm createXXHash64 returns a hasher that takes (data, seed_lo, seed_hi)
  const seedLo = Number(seed & 0xFFFFFFFFn);
  const seedHi = Number((seed >> 32n) & 0xFFFFFFFFn);
  const result = xxh64.h64Raw(new Uint8Array(buf), BigInt(seedLo) | (BigInt(seedHi) << 32n));
  return result;
}

function toFloat(h: bigint): number {
  return Number(h & 0xFFFFFFFFn) / 0x100000000;
}

function arcIndex(a: number, b: number): number {
  const [lo, hi] = a < b ? [a, b] : [b, a];
  return lo * (KOKORO_VOICES.length - 1) - Math.floor(lo * (lo - 1) / 2) + hi - lo - 1;
}

export async function deriveVoiceSpec(
  sx: number, sy: number, sz: number,
  worldSeedOverride?: bigint
): Promise<VoiceSpec> {
  const xxh64 = await getHasher();
  const h = (salt: bigint) => hashCoords(xxh64, sx, sy, sz, salt);

  const hA = h(SALT_VOICE_A);
  const hB = h(SALT_VOICE_B);
  const hT = h(SALT_SLERP_T);
  const hP = h(SALT_PITCH);
  const hE = h(SALT_ENERGY);

  const idxA = Number(hA % NUM_VOICES);
  const idxB = (Number(hB % (NUM_VOICES - 1n)) + idxA + 1) % KOKORO_VOICES.length;

  return {
    voiceA:      idxA,
    voiceB:      idxB,
    voiceAName:  KOKORO_VOICES[idxA],
    voiceBName:  KOKORO_VOICES[idxB],
    t:           toFloat(hT),
    pitchScale:  0.85 + toFloat(hP) * 0.30,
    energyScale: 0.80 + toFloat(hE) * 0.40,
    hashHex:     hA.toString(16).padStart(16, "0"),
    arcIndex:    arcIndex(idxA, idxB),
  };
}
```

---

### Step 12 — TypeScript: Types (`src/lib/types.ts`)

```typescript
export interface Coords {
  x: number;
  y: number;
  z: number;
}

export interface VoiceSpec {
  voiceA:      number;
  voiceB:      number;
  voiceAName:  string;
  voiceBName:  string;
  t:           number;
  pitchScale:  number;
  energyScale: number;
  hashHex:     string;
  arcIndex:    number;
}

export interface Overrides {
  t:      { locked: boolean; value: number };
  pitch:  { locked: boolean; value: number };
  energy: { locked: boolean; value: number };
  voiceA: { locked: boolean; value: number };
  voiceB: { locked: boolean; value: number };
}

export interface HistoryEntry {
  id:        string;
  coords:    Coords;
  spec:      VoiceSpec;
  overrides: Overrides;
  text:      string;
  durationS: number;
  timestamp: number;
}

export interface PresetSlot {
  coords: Coords | null;
}

export type SynthesisState = "idle" | "synthesising" | "playing" | "error";
```

---

### Step 13 — TypeScript: Tauri Invoke Wrappers (`src/lib/voice.ts`)

```typescript
import { invoke } from "@tauri-apps/api/core";
import type { Overrides, VoiceSpec } from "./types";

export interface SynthesizeArgs {
  spawnX:        number;
  spawnY:        number;
  spawnZ:        number;
  text:          string;
  useRegionBias: boolean;
  overrideT:     number | null;
  overridePitch: number | null;
  overrideEnergy:number | null;
  overrideVoiceA:number | null;
  overrideVoiceB:number | null;
}

export async function synthesizeNpcSpeech(args: SynthesizeArgs): Promise<Float32Array> {
  const raw = await invoke<number[]>("synthesize_npc_speech", {
    spawnX:         args.spawnX,
    spawnY:         args.spawnY,
    spawnZ:         args.spawnZ,
    text:           args.text,
    useRegionBias:  args.useRegionBias,
    overrideT:      args.overrideT,
    overridePitch:  args.overridePitch,
    overrideEnergy: args.overrideEnergy,
    overrideVoiceA: args.overrideVoiceA,
    overrideVoiceB: args.overrideVoiceB,
  });
  return new Float32Array(raw);
}

export async function previewVoiceSpec(
  x: number, y: number, z: number,
  useRegionBias: boolean
): Promise<VoiceSpec> {
  return invoke<VoiceSpec>("preview_voice_spec", {
    spawnX: x, spawnY: y, spawnZ: z, useRegionBias,
  });
}

export function buildSynthArgs(
  x: number, y: number, z: number,
  text: string,
  overrides: Overrides,
  useRegionBias: boolean
): SynthesizeArgs {
  return {
    spawnX: x, spawnY: y, spawnZ: z, text, useRegionBias,
    overrideT:      overrides.t.locked      ? overrides.t.value      : null,
    overridePitch:  overrides.pitch.locked  ? overrides.pitch.value  : null,
    overrideEnergy: overrides.energy.locked ? overrides.energy.value : null,
    overrideVoiceA: overrides.voiceA.locked ? overrides.voiceA.value : null,
    overrideVoiceB: overrides.voiceB.locked ? overrides.voiceB.value : null,
  };
}
```

---

### Step 14 — TypeScript: WAV Export (`src/lib/waveform.ts`)

```typescript
export function encodePcmAsWav(pcm: Float32Array, sampleRate = 24000): Blob {
  const numSamples = pcm.length;
  const buffer = new ArrayBuffer(44 + numSamples * 2);
  const view = new DataView(buffer);

  const str = (off: number, s: string) =>
    [...s].forEach((c, i) => view.setUint8(off + i, c.charCodeAt(0)));

  str(0,  "RIFF");
  view.setUint32( 4, 36 + numSamples * 2,  true);
  str(8,  "WAVE");
  str(12, "fmt ");
  view.setUint32(16, 16,               true); // subchunk size
  view.setUint16(20,  1,               true); // PCM
  view.setUint16(22,  1,               true); // mono
  view.setUint32(24, sampleRate,       true);
  view.setUint32(28, sampleRate * 2,   true); // byte rate
  view.setUint16(32,  2,               true); // block align
  view.setUint16(34, 16,               true); // bits per sample
  str(36, "data");
  view.setUint32(40, numSamples * 2,   true);

  for (let i = 0; i < numSamples; i++) {
    const s = Math.max(-1, Math.min(1, pcm[i]));
    view.setInt16(44 + i * 2, s < 0 ? s * 0x8000 : s * 0x7FFF, true);
  }

  return new Blob([buffer], { type: "audio/wav" });
}

export function downloadWav(pcm: Float32Array, filename = "zerovoice-output.wav"): void {
  const url = URL.createObjectURL(encodePcmAsWav(pcm));
  const a = Object.assign(document.createElement("a"), { href: url, download: filename });
  a.click();
  setTimeout(() => URL.revokeObjectURL(url), 10_000);
}
```

---

### Step 15 — CSS Design System (`src/styles/workbench.css`)

```css
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;500&family=IBM+Plex+Sans+Condensed:wght@400;600&display=swap');

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --bg-void:        #0d0d0f;
  --bg-panel:       #111114;
  --bg-control:     #18181c;
  --bg-hover:       #1e1e24;
  --border-dim:     #2a2a32;
  --border-mid:     #3a3a46;
  --border-bright:  #505060;
  --amber:          #e8a020;
  --amber-dim:      #7a5010;
  --amber-glow:     #e8a02033;
  --slate:          #4a7fa5;
  --slate-dim:      #243d52;
  --text-primary:   #d4d0c8;
  --text-secondary: #7a7870;
  --text-amber:     #e8a020;
  --text-slate:     #6aafd4;
  --female-border:  #7a4a8a;
  --female-bg:      #2a1a35;
  --female-text:    #c49ad4;
  --green-ready:    #2a6040;
  --font-mono:      'IBM Plex Mono', monospace;
  --font-ui:        'IBM Plex Sans Condensed', sans-serif;
}

html, body, #root { height: 100%; overflow: hidden; }

body {
  background: var(--bg-void);
  color: var(--text-primary);
  font-family: var(--font-mono);
  font-size: 13px;
  -webkit-font-smoothing: antialiased;
}

/* Scanline overlay — zero performance cost */
body::after {
  content: '';
  position: fixed;
  inset: 0;
  background: repeating-linear-gradient(
    0deg,
    transparent 0px, transparent 2px,
    rgba(0,0,0,0.06) 2px, rgba(0,0,0,0.06) 4px
  );
  pointer-events: none;
  z-index: 9999;
}

.workbench {
  display: grid;
  grid-template-rows: 32px 1fr 28px 32px; /* header | columns | history | status */
  grid-template-columns: 220px 1fr 260px;
  height: 100vh;
  gap: 1px;
  background: var(--border-dim); /* 1px gap colour */
}

.workbench-header {
  grid-column: 1 / -1;
  background: var(--bg-panel);
  display: flex;
  align-items: center;
  padding: 0 14px;
  gap: 16px;
  border-bottom: 1px solid var(--border-dim);
}

.workbench-header .app-title {
  font-family: var(--font-mono);
  font-size: 12px;
  font-weight: 500;
  color: var(--amber);
  letter-spacing: 0.12em;
}

.workbench-header .app-meta {
  font-size: 11px;
  color: var(--text-secondary);
  letter-spacing: 0.05em;
}

.panel {
  background: var(--bg-panel);
  padding: 10px 12px;
  overflow-y: auto;
  overflow-x: hidden;
}

.panel-title {
  font-family: var(--font-ui);
  font-size: 9px;
  font-weight: 600;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  color: var(--text-secondary);
  border-bottom: 1px solid var(--border-dim);
  padding-bottom: 5px;
  margin-bottom: 10px;
}

.history-bar {
  grid-column: 1 / -1;
  background: var(--bg-panel);
  border-top: 1px solid var(--border-dim);
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 10px;
  overflow-x: auto;
  white-space: nowrap;
  font-size: 11px;
  color: var(--text-secondary);
}

.history-bar .history-label {
  font-size: 9px;
  letter-spacing: 0.12em;
  color: var(--text-secondary);
  text-transform: uppercase;
  flex-shrink: 0;
}

.history-entry {
  cursor: pointer;
  padding: 2px 6px;
  border: 1px solid var(--border-dim);
  color: var(--text-secondary);
  transition: color 0.1s, border-color 0.1s;
  flex-shrink: 0;
}
.history-entry:hover { color: var(--text-primary); border-color: var(--border-mid); }
.history-entry.latest { color: var(--amber); border-color: var(--amber-dim); }
```

---

### Step 16 — CSS Controls (`src/styles/controls.css`)

```css
/* Coordinate input */
.coord-input {
  font-family: var(--font-mono);
  font-size: 17px;
  font-weight: 500;
  color: var(--amber);
  background: var(--bg-control);
  border: 1px solid var(--border-mid);
  width: 88px;
  text-align: right;
  padding: 3px 6px;
  outline: none;
  caret-color: var(--amber);
  border-radius: 0;
}
.coord-input:focus { border-color: var(--amber); box-shadow: 0 0 0 1px var(--amber-glow); }
.coord-input.error  { border-color: #8a2020; }

/* Amber fill-track range slider */
input[type="range"].param-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 2px;
  background: linear-gradient(
    to right,
    var(--amber) 0%,
    var(--amber) var(--fill-pct, 0%),
    var(--border-mid) var(--fill-pct, 0%),
    var(--border-mid) 100%
  );
  outline: none;
  cursor: pointer;
}
input[type="range"].param-slider:disabled {
  opacity: 0.35;
  cursor: not-allowed;
}
input[type="range"].param-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 10px;
  height: 18px;
  background: var(--amber);
  cursor: pointer;
  border: none;
  border-radius: 0;
}
input[type="range"].param-slider:disabled::-webkit-slider-thumb {
  background: var(--amber-dim);
}

/* Buttons */
.btn-primary {
  font-family: var(--font-mono);
  font-size: 12px;
  font-weight: 500;
  letter-spacing: 0.06em;
  background: var(--amber);
  color: #0d0d0f;
  border: none;
  border-radius: 0;
  padding: 8px 16px;
  cursor: pointer;
  width: 100%;
  transition: opacity 0.1s;
}
.btn-primary:hover    { opacity: 0.85; }
.btn-primary:active   { opacity: 0.70; }
.btn-primary:disabled { background: var(--amber-dim); color: #4a3010; cursor: not-allowed; opacity: 1; }

.btn-secondary {
  font-family: var(--font-mono);
  font-size: 11px;
  background: transparent;
  color: var(--text-secondary);
  border: 1px solid var(--border-mid);
  border-radius: 0;
  padding: 5px 10px;
  cursor: pointer;
  transition: color 0.1s, border-color 0.1s;
}
.btn-secondary:hover { color: var(--text-primary); border-color: var(--border-bright); }

/* Voice badge */
.voice-badge {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-family: var(--font-mono);
  font-size: 12px;
  font-weight: 500;
  padding: 2px 8px;
  border: 1px solid;
  border-radius: 0;
}
.voice-badge.female { background: var(--female-bg); color: var(--female-text); border-color: var(--female-border); }
.voice-badge.male   { background: var(--slate-dim); color: var(--text-slate);   border-color: var(--slate); }

/* Hash display */
.hash-display {
  font-family: var(--font-mono);
  font-size: 11px;
  color: var(--text-secondary);
  display: flex;
  align-items: center;
  gap: 6px;
}
.hash-display .hash-value { color: var(--text-amber); letter-spacing: 0.04em; }

/* Lock checkbox */
.lock-toggle {
  display: flex;
  align-items: center;
  gap: 5px;
  cursor: pointer;
  font-size: 10px;
  color: var(--text-secondary);
  user-select: none;
}
.lock-toggle input[type="checkbox"] {
  accent-color: var(--amber);
  width: 11px;
  height: 11px;
  cursor: pointer;
}
.lock-toggle.active { color: var(--amber); }
```

---

### Step 17 — React: Root and App Shell (`src/main.tsx`, `src/App.tsx`)

```tsx
// src/main.tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import "./styles/workbench.css";
import "./styles/controls.css";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode><App /></React.StrictMode>
);
```

```tsx
// src/App.tsx
import { useState, useEffect, useCallback, useRef } from "react";
import { WorkbenchShell } from "./components/workbench/WorkbenchShell";
import { deriveVoiceSpec } from "./lib/zerovoice-preview";
import type { Coords, VoiceSpec, Overrides, HistoryEntry, SynthesisState, PresetSlot } from "./lib/types";
import { synthesizeNpcSpeech, buildSynthArgs } from "./lib/voice";
import { downloadWav } from "./lib/waveform";

const DEFAULT_COORDS: Coords  = { x: 0, y: 0, z: 0 };
const DEFAULT_OVERRIDES: Overrides = {
  t:      { locked: false, value: 0.5 },
  pitch:  { locked: false, value: 1.0 },
  energy: { locked: false, value: 1.0 },
  voiceA: { locked: false, value: 0   },
  voiceB: { locked: false, value: 1   },
};

export default function App() {
  const [coords,         setCoords]        = useState<Coords>(DEFAULT_COORDS);
  const [spec,           setSpec]          = useState<VoiceSpec | null>(null);
  const [overrides,      setOverrides]     = useState<Overrides>(DEFAULT_OVERRIDES);
  const [useRegionBias,  setUseRegionBias] = useState(false);
  const [text,           setText]          = useState("");
  const [synthState,     setSynthState]    = useState<SynthesisState>("idle");
  const [pcm,            setPcm]           = useState<Float32Array | null>(null);
  const [currentSample,  setCurrentSample] = useState(0);
  const [history,        setHistory]       = useState<HistoryEntry[]>([]);
  const [presets,        setPresets]       = useState<PresetSlot[]>(() => {
    try {
      return JSON.parse(localStorage.getItem("zv_presets") ?? "null")
        ?? [{ coords: null }, { coords: null }, { coords: null }, { coords: null }];
    } catch { return [{ coords: null }, { coords: null }, { coords: null }, { coords: null }]; }
  });

  const audioCtxRef    = useRef<AudioContext | null>(null);
  const sourceRef      = useRef<AudioBufferSourceNode | null>(null);
  const rafRef         = useRef<number>(0);

  // Derive VoiceSpec whenever coords change (browser-side, instant)
  useEffect(() => {
    deriveVoiceSpec(coords.x, coords.y, coords.z).then(setSpec);
  }, [coords]);

  const stopPlayback = useCallback(() => {
    sourceRef.current?.stop();
    sourceRef.current = null;
    cancelAnimationFrame(rafRef.current);
    setSynthState("idle");
    setCurrentSample(0);
  }, []);

  const handleSynthesise = useCallback(async () => {
    if (!spec || !text.trim()) return;
    setSynthState("synthesising");
    setPcm(null);
    setCurrentSample(0);

    try {
      const args = buildSynthArgs(
        coords.x, coords.y, coords.z,
        text, overrides, useRegionBias
      );
      const audio = await synthesizeNpcSpeech(args);
      setPcm(audio);
      setSynthState("playing");

      // Playback via Web Audio API
      if (!audioCtxRef.current) audioCtxRef.current = new AudioContext();
      const ctx = audioCtxRef.current;
      const buffer = ctx.createBuffer(1, audio.length, 24000);
      buffer.copyToChannel(audio, 0);
      const src = ctx.createBufferSource();
      src.buffer = buffer;
      src.connect(ctx.destination);
      sourceRef.current = src;

      const startTime = ctx.currentTime;
      const tick = () => {
        const elapsed = ctx.currentTime - startTime;
        setCurrentSample(Math.min(Math.floor(elapsed * 24000), audio.length));
        if (elapsed < buffer.duration) rafRef.current = requestAnimationFrame(tick);
      };
      rafRef.current = requestAnimationFrame(tick);

      src.onended = () => { setSynthState("idle"); setCurrentSample(0); };
      src.start();

      // Append to history
      const entry: HistoryEntry = {
        id: crypto.randomUUID(),
        coords: { ...coords },
        spec: { ...spec },
        overrides: JSON.parse(JSON.stringify(overrides)),
        text,
        durationS: audio.length / 24000,
        timestamp: Date.now(),
      };
      setHistory(h => [entry, ...h].slice(0, 50));

    } catch (err) {
      console.error("Synthesis failed:", err);
      setSynthState("error");
    }
  }, [spec, text, coords, overrides, useRegionBias]);

  const savePreset = (idx: number) => {
    const updated = presets.map((p, i) => i === idx ? { coords: { ...coords } } : p);
    setPresets(updated);
    localStorage.setItem("zv_presets", JSON.stringify(updated));
  };

  const loadPreset = (idx: number) => {
    const slot = presets[idx];
    if (slot.coords) setCoords({ ...slot.coords });
  };

  const clearPreset = (idx: number) => {
    const updated = presets.map((p, i) => i === idx ? { coords: null } : p);
    setPresets(updated);
    localStorage.setItem("zv_presets", JSON.stringify(updated));
  };

  const restoreHistory = (entry: HistoryEntry) => {
    setCoords(entry.coords);
    setOverrides(entry.overrides);
    setText(entry.text);
  };

  return (
    <WorkbenchShell
      coords={coords}           onCoordsChange={setCoords}
      spec={spec}
      overrides={overrides}     onOverridesChange={setOverrides}
      useRegionBias={useRegionBias} onRegionBiasChange={setUseRegionBias}
      text={text}               onTextChange={setText}
      synthState={synthState}
      pcm={pcm}
      currentSample={currentSample}
      history={history}
      presets={presets}
      onSynthesise={handleSynthesise}
      onStop={stopPlayback}
      onExportWav={() => pcm && downloadWav(pcm)}
      onSavePreset={savePreset}
      onLoadPreset={loadPreset}
      onClearPreset={clearPreset}
      onRestoreHistory={restoreHistory}
      onClearHistory={() => setHistory([])}
    />
  );
}
```

---

### Step 18 — React: Key Control Components

#### `src/components/controls/CoordInput.tsx`

```tsx
import { useRef, KeyboardEvent } from "react";

interface Props {
  axis:     "X" | "Y" | "Z";
  value:    number;
  onChange: (v: number) => void;
}

const clamp = (v: number) => Math.max(-32768, Math.min(32767, Math.round(v)));

export function CoordInput({ axis, value, onChange }: Props) {
  const inputRef = useRef<HTMLInputElement>(null);

  const onKey = (e: KeyboardEvent<HTMLInputElement>) => {
    if (e.key !== "ArrowUp" && e.key !== "ArrowDown") return;
    e.preventDefault();
    const step = e.ctrlKey ? 100 : e.shiftKey ? 10 : 1;
    onChange(clamp(value + (e.key === "ArrowUp" ? step : -step)));
  };

  const onBlur = (s: string) => {
    const n = parseInt(s, 10);
    onChange(isNaN(n) ? 0 : clamp(n));
  };

  return (
    <div style={{ display: "flex", alignItems: "center", gap: 6, marginBottom: 6 }}>
      <span style={{ fontFamily: "var(--font-mono)", fontSize: 11, color: "var(--text-secondary)", width: 10 }}>{axis}</span>
      <input
        ref={inputRef}
        className="coord-input"
        type="text"
        inputMode="numeric"
        defaultValue={value}
        key={value}
        onKeyDown={onKey}
        onBlur={e => onBlur(e.target.value)}
        onFocus={e => e.target.select()}
      />
      <div style={{ display: "flex", flexDirection: "column", gap: 1 }}>
        {(["▲","▼"] as const).map((arrow, i) => (
          <button key={arrow} onClick={() => onChange(clamp(value + (i === 0 ? 1 : -1)))}
            style={{ fontFamily: "var(--font-mono)", fontSize: 7, background: "var(--bg-control)",
              border: "1px solid var(--border-dim)", color: "var(--text-secondary)",
              width: 16, height: 11, cursor: "pointer", lineHeight: 1, padding: 0 }}>
            {arrow}
          </button>
        ))}
      </div>
    </div>
  );
}
```

#### `src/components/controls/ParamSlider.tsx`

```tsx
interface Props {
  label:    string;
  value:    number;
  min:      number;
  max:      number;
  step:     number;
  decimals?: number;
  locked:   boolean;
  onLock:   (locked: boolean) => void;
  onChange: (v: number) => void;
}

export function ParamSlider({ label, value, min, max, step, decimals = 3, locked, onLock, onChange }: Props) {
  const fillPct = `${((value - min) / (max - min)) * 100}%`;
  return (
    <div style={{ marginBottom: 10 }}>
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 3 }}>
        <span style={{ fontFamily: "var(--font-mono)", fontSize: 11, color: "var(--text-secondary)" }}>{label}</span>
        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
          <span style={{ fontFamily: "var(--font-mono)", fontSize: 13,
            color: locked ? "var(--amber)" : "var(--text-primary)", minWidth: 54, textAlign: "right" }}>
            {value.toFixed(decimals)}
          </span>
          <label className={`lock-toggle${locked ? " active" : ""}`}>
            <input type="checkbox" checked={locked} onChange={e => onLock(e.target.checked)} />
            LOCK
          </label>
        </div>
      </div>
      <input type="range" className="param-slider"
        min={min} max={max} step={step} value={value}
        disabled={!locked}
        onChange={e => onChange(parseFloat(e.target.value))}
        style={{ "--fill-pct": fillPct } as React.CSSProperties}
      />
    </div>
  );
}
```

#### `src/components/controls/WaveformCanvas.tsx`

```tsx
import { useRef, useEffect } from "react";
import type { SynthesisState } from "../../lib/types";

interface Props {
  pcm:           Float32Array | null;
  synthState:    SynthesisState;
  currentSample: number;
}

export function WaveformCanvas({ pcm, synthState, currentSample }: Props) {
  const ref = useRef<HTMLCanvasElement>(null);
  const rafRef = useRef<number>(0);

  useEffect(() => {
    const canvas = ref.current;
    if (!canvas) return;
    const ctx = canvas.getContext("2d")!;
    const W = canvas.width, H = canvas.height, mid = H / 2;

    const draw = () => {
      ctx.fillStyle = "#0d0d0f";
      ctx.fillRect(0, 0, W, H);

      // Grid
      ctx.strokeStyle = "#1e1e24";
      ctx.lineWidth = 0.5;
      [0.25, 0.5, 0.75].forEach(r => {
        ctx.beginPath(); ctx.moveTo(0, H * r); ctx.lineTo(W, H * r); ctx.stroke();
      });

      if (synthState === "synthesising") {
        const bar = (Date.now() % 1400) / 1400 * W;
        ctx.strokeStyle = "#e8a02055";
        ctx.lineWidth = 1.5;
        ctx.beginPath(); ctx.moveTo(bar, 0); ctx.lineTo(bar, H); ctx.stroke();
        ctx.strokeStyle = "#e8a020";
        ctx.lineWidth = 1;
        ctx.beginPath(); ctx.moveTo(0, mid); ctx.lineTo(bar, mid); ctx.stroke();
        rafRef.current = requestAnimationFrame(draw);
        return;
      }

      if (!pcm || pcm.length === 0) {
        ctx.strokeStyle = "#2a6040";
        ctx.lineWidth = 1;
        ctx.beginPath(); ctx.moveTo(0, mid); ctx.lineTo(W, mid); ctx.stroke();
        return;
      }

      const spp = pcm.length / W;
      ctx.strokeStyle = "#e8a020";
      ctx.lineWidth = 1;
      ctx.beginPath();
      for (let px = 0; px < W; px++) {
        const s = pcm[Math.floor(px * spp)] ?? 0;
        const y = mid - s * mid * 0.88;
        px === 0 ? ctx.moveTo(px, y) : ctx.lineTo(px, y);
      }
      ctx.stroke();

      if (synthState === "playing" && pcm.length > 0) {
        const px = (currentSample / pcm.length) * W;
        ctx.strokeStyle = "#ffffff44";
        ctx.lineWidth = 1;
        ctx.beginPath(); ctx.moveTo(px, 0); ctx.lineTo(px, H); ctx.stroke();
      }
    };

    cancelAnimationFrame(rafRef.current);
    draw();
    if (synthState === "synthesising") {
      const loop = () => { draw(); rafRef.current = requestAnimationFrame(loop); };
    }

    return () => cancelAnimationFrame(rafRef.current);
  }, [pcm, synthState, currentSample]);

  return (
    <canvas ref={ref} width={460} height={80}
      style={{ width: "100%", height: 80, display: "block", imageRendering: "pixelated",
        border: "1px solid var(--border-dim)" }} />
  );
}
```

---

### Step 19 — Determinism Verification Test

Add this to `src-tauri/src/zerovoice.rs` under `#[cfg(test)]`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn voice_spec_is_deterministic() {
        let coords = [(0,0,0), (312,0,-88), (-999,1024,5000), (32767,-32768,1)];
        for (x, y, z) in coords {
            let a = voice_from_spawn(x, y, z);
            let b = voice_from_spawn(x, y, z);
            let c = voice_from_spawn(x, y, z);
            assert_eq!(a.voice_a, b.voice_a);
            assert_eq!(b.voice_a, c.voice_a);
            assert_eq!(a.voice_b, b.voice_b);
            assert_eq!(a.t.to_bits(), b.t.to_bits());     // exact float equality
            assert_eq!(a.pitch_scale.to_bits(), c.pitch_scale.to_bits());
        }
    }

    #[test]
    fn voice_b_never_equals_voice_a() {
        for x in -10..10_i32 {
            for z in -10..10_i32 {
                let spec = voice_from_spawn(x, 0, z);
                assert_ne!(spec.voice_a, spec.voice_b,
                    "A == B at ({x}, 0, {z})");
            }
        }
    }

    #[test]
    fn slerp_preserves_magnitude() {
        use crate::slerp::slerp;
        let a: Vec<f32> = (0..256).map(|i| (i as f32 * 0.1).sin()).collect();
        let b: Vec<f32> = (0..256).map(|i| (i as f32 * 0.2).cos()).collect();
        let norm = |v: &[f32]| v.iter().map(|x| x * x).sum::<f32>().sqrt();
        let na = norm(&a); let nb = norm(&b);
        // Normalise inputs
        let a: Vec<f32> = a.iter().map(|x| x / na).collect();
        let b: Vec<f32> = b.iter().map(|x| x / nb).collect();
        for t in [0.0, 0.25, 0.5, 0.75, 1.0_f32] {
            let out = slerp(&a, &b, t);
            let n = norm(&out);
            assert!((n - 1.0).abs() < 1e-5, "slerp magnitude {n} at t={t}");
        }
    }
}
```

Run with `cargo test -p zerov0ice-lib`.

---

### Step 20 — Download Model Assets

The ONNX model and voice files are not included in the repository. Download them before first run:

```bash
# From the project root:
mkdir -p src-tauri/resources

# Kokoro int8 ONNX model (~88 MB)
curl -L -o src-tauri/resources/kokoro-v1.0.int8.onnx \
  https://github.com/thewh1teagle/kokoro-onnx/releases/download/model-files-v1.0/kokoro-v1.0.int8.onnx

# Voice style vectors (~3 MB)
curl -L -o src-tauri/resources/voices-v1.0.bin \
  https://github.com/thewh1teagle/kokoro-onnx/releases/download/model-files-v1.0/voices-v1.0.bin

# espeak-ng for Windows (download and extract into resources/)
# Download from: https://github.com/espeak-ng/espeak-ng/releases
# Place espeak-ng.exe and espeak-ng-data/ inside src-tauri/resources/
```

Also download `onnxruntime.dll` for Windows and place in `src-tauri/` (required by `ort` with `load-dynamic` feature):

```bash
# ONNX Runtime 1.18 Windows x64
curl -L -o ort-win.zip \
  https://github.com/microsoft/onnxruntime/releases/download/v1.18.1/onnxruntime-win-x64-1.18.1.zip
unzip ort-win.zip -d ort-tmp
cp ort-tmp/onnxruntime-win-x64-1.18.1/lib/onnxruntime.dll src-tauri/
rm -rf ort-tmp ort-win.zip
```

---

### Step 21 — Build and Run

```bash
# Development (hot-reload frontend, Tauri dev window)
npm run tauri dev

# Production MSI build
npm run tauri build
# Output: src-tauri/target/release/bundle/msi/ZeroVoice Workbench_0.1.0_x64_en-US.msi
```

---

## MSI Bundle Inventory

| File | Size | Purpose |
|---|---|---|
| `kokoro-v1.0.int8.onnx` | ~88 MB | KokoroTTS ONNX model (int8 quantised) |
| `voices-v1.0.bin` | ~3 MB | 26 voice style vectors (NPZ format) |
| `espeak-ng.exe` | ~2 MB | G2P phonemizer |
| `espeak-ng-data/` | ~15 MB | espeak-ng language data |
| `onnxruntime.dll` | ~7 MB | ONNX Runtime (load-dynamic) |
| Tauri app binary | ~8 MB | Compiled Rust + embedded webview |
| **Total** | **~123 MB** | Within 150 MB target |

---

## ZeroBytes Laws Compliance

| Law | Implementation |
|---|---|
| O(1) access | `voice_from_spawn` is five xxHash64 calls — no loops, no iteration |
| Parallelism | Each NPC hashes independently; no shared mutable state |
| Coherence | Optional `voice_from_spawn_biased` adds smooth regional variation via bilinear-interpolated noise |
| Hierarchy | `WORLD_SEED → WORLD_SEED ^ SALT_x → position_hash → hash_to_float → property` |
| Determinism | xxHash64 is platform-independent; WORLD_SEED is a compile-time const; no time/random state |

---

## Extended Features (Optional)

- **Batch export**: synthesise all 4 preset slot voices with a shared text string and export as a ZIP of four WAV files labelled by coordinate.
- **Voice map canvas**: extend the 2D region bias map to colour-code which voice pair (A index mod 5) occupies each pixel, giving a geographic view of voice distribution.
- **Seed explorer**: a separate view that plots hash distribution across a random sample of 10,000 coordinates to verify statistical uniformity.
- **misaki-rs phonemizer**: replace espeak-ng subprocess with an in-process Rust G2P library, eliminating the subprocess dependency and reducing MSI size by ~17 MB.
- **Multi-language**: Kokoro supports Japanese, Korean, French, and Mandarin alongside English. Expose a language selector in the Synthesis panel; pass the language code to espeak-ng `-v` flag.
