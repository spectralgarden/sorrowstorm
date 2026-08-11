```
# Sorrow Storm

**An ambient shoegaze reverb & fuzz plugin for VST3**

Sorrow Storm is a mood-drenched effects processor built for dream-pop, shoegaze, and ambient production. It combines a saturating fuzz stage with a modulated four-tap feedback reverb, built-in shimmer pitch-shifting, and 30 hand-tuned presets — wrapped in a custom-animated, glitch-titled interface.

---

## Table of Contents

- [For Musicians & Producers](#for-musicians--producers)
  - [Features](#features)
  - [Parameters](#parameters)
  - [Presets](#presets)
  - [Installation](#installation)
  - [System Requirements](#system-requirements)
  - [Quick Start](#quick-start)
- [For Developers](#for-developers)
  - [Architecture Overview](#architecture-overview)
  - [Signal Chain](#signal-chain)
  - [DSP Internals](#dsp-internals)
  - [UI / Editor](#ui--editor)
  - [Project Structure](#project-structure)
  - [Building from Source](#building-from-source)
  - [Extending the Plugin](#extending-the-plugin)
- [License](#license)

---

## For Musicians & Producers

### Features

- **Fuzz Chaos** — tanh-based saturation with a shaped low/high-pass blend for a gritty, tape-worn character
- **Four-tap feedback reverb** — a Hadamard-mixed delay network for dense, evolving ambience
- **Built-in Shimmer** — octave-up pitch shifting woven directly into the reverb feedback path
- **Tape Flutter** — smooth, click-free LFO modulation of the reverb delay times for a wobbly, analog feel
- **Drone Resonance** — feedback-weighted sustain for infinite, pad-like tails
- **30 curated presets**, from subtle rainy-window ambience to full-blown kawaii chaos
- **Save Memory** — export your current settings as an XML preset file
- **Reactive UI** — knobs, a circular spectrogram, and glitch typography all animate live with the input signal

### Parameters

| Parameter | Name in UI | Range | Description |
|---|---|---|---|
| `fuzz` | Fuzz Chaos | 0.0 – 1.0 | Saturation/distortion drive |
| `decay` | Liminal Decay | 0.0 – 0.98 | Reverb tail length / feedback amount |
| `shimmer` | Ethereal Glow | 0.0 – 1.0 | Amount of octave-up shimmer blended into the reverb |
| `flutter` | Nostalgia Flutter | 0.0 – 1.0 | Tape-style modulation speed/depth on the reverb |
| `drone` | Drone Resonance | 0.0 – 1.0 | Cross-feedback amount for sustained, droning tails |
| `mix` | Sorrow Mix | 0.0 – 1.0 | Dry/wet balance |
| `bypass` | Bypass | on/off | Fully bypasses processing |

### Presets

Sorrow Storm ships with 30 presets across four moods, selectable from the dropdown in the top toolbar:

- **Melancholy & Rain** — *Rainy Classroom, Pale Blue Room, Forgotten Swing, Teardrops in Static, Sinking Slowly, Cold Concrete, December Rust*
- **Angelic & Ethereal** — *Angelic Decay, Stardust Velvet, Holographic Mist, Cosmic Cradle, Nebula Sigh, Ghostly Choir, Silver Glow, Floating in Nothing*
- **Liminal Spaces** — *Empty Mall 1999, Endless Hallway, Backroom Lullaby, Yellow Wallpaper, The Waiting Pool, Uncanny Atrium, Midnight Playground, Echoes of Elsewhere*
- **Gothic Pastel** — *Gothic Pastel, Stitched Bunny, Toxic Cupcake, Spiderweb Lace, Screaming Plushie, Gooey Heart, Acid Sparkles, Kawaii Chaos*

Click **Save Memory** at any time to export your current knob positions as a standalone `.xml` file you can share or reload later.

### Installation

1. Download the plugin build for your platform (`.vst3` or `.component`).
2. Copy it into your system's plugin folder:
   - **Windows (VST3):** `C:\Program Files\Common Files\VST3\`
   - **macOS (VST3):** `/Library/Audio/Plug-Ins/VST3/`
   - **macOS (AU):** `/Library/Audio/Plug-Ins/Components/`
3. Rescan plugins in your DAW.
4. Load **Sorrow Storm** on an audio or bus track.

### System Requirements

- A VST3- or AU-compatible DAW (Ableton Live, Logic Pro, Reaper, FL Studio, Cubase, etc.)
- macOS 10.13+ or Windows 10+
- 64-bit host required

### Quick Start

1. Load Sorrow Storm on a vocal, guitar, or synth bus.
2. Pick a starting preset from the dropdown — try **"Pale Blue Room"** for a subtle wash, or **"Screaming Plushie"** for something more extreme.
3. Turn up **Shimmer** for an octave-up angelic layer.
4. Turn up **Drone** and **Decay** together for a self-sustaining ambient pad from any input.
5. Use **Mix** to blend the effect under your dry signal.

---

## For Developers

### Architecture Overview

Sorrow Storm is a [JUCE](https://juce.com/) audio plugin (VST3/AU) with a standard `AudioProcessor` / `AudioProcessorEditor` split:

- **`PluginProcessor.h/.cpp`** — DSP, parameter management (`AudioProcessorValueTreeState`), preset handling, and state serialization
- **`PluginEditor.h/.cpp`** — Custom `LookAndFeel`, animated UI, circular spectrogram visualizer, and glitch-text rendering

### Signal Chain


Input
  │
  ▼
Fuzz Saturation (tanh waveshaping, gain: 1 + fuzz * 32)
  │
  ▼
Low-pass / High-pass shaping (fuzz-dependent coefficients)
  │
  ▼
Dry/Fuzz blend (85% processed / 15% dry pre-mix)
  │
  ▼
4-Tap Feedback Delay Network (Hadamard mixing)
  │  ├─ Tape Flutter LFO modulates each delay time
  │  ├─ Shimmer pitch-shifters inserted in feedback path (taps 1 & 2)
  │  └─ Drone Resonance adds cross-feedback weighting
  ▼
Reverb output
  │
  ▼
Dry/Wet Mix (Sorrow Mix)
  │
  ▼
Output


### DSP Internals

**`SorrowReverb`** — A 4-delay-line feedback network:
- Base delay times: 89ms, 127ms, 179ms, 251ms (mutually prime-ish for diffusion)
- Delay times are modulated by a sine/cosine LFO pair scaled by `flutter`
- Outputs are mixed via a 4×4 Hadamard-style matrix (`m1..m4`) for decorrelation
- Feedback is passed through `SorrowPitchShifter` instances (taps 1 & 2 only) for shimmer
- `drone` scales a cross-feedback term added to each delay's write signal
- All feedback writes are soft-clipped with `tanh()` to prevent runaway energy

**`SorrowPitchShifter`** — A dual-tap delay-line pitch shifter:
- Two read taps 180° out of phase, each windowed with a triangular crossfade
- Fixed octave-up ratio (`rate = (2.0 - 1.0) / windowDuration`), 50ms window
- Shimmer amount crossfades between the shifted and dry signal

**`processBlock`**:
- All input samples are sanitized (`NaN`/`Inf` guarded) before and after each processing stage
- A running envelope follower (`envelopeTracker`, atomic) smooths absolute sample magnitude at a 0.82/0.18 pole for UI-reactive animation
- A lightweight 64-bin pseudo-spectrum (`spectrumOutput`) is derived from the output envelope with a sinusoidal per-bin phase offset — this drives the circular visualizer rather than a true FFT, keeping CPU cost minimal

**Presets** — Defined as a static `constexpr`-style array of `SorrowPreset` structs (`SorrowPresetLibrary`) in `PluginProcessor.h`. `setCurrentProgram()` pushes preset values into the `AudioProcessorValueTreeState` via `setValueNotifyingHost()`.

### UI / Editor

- **`SorrowStormLookAndFeel`** — custom rotary knob rendering (spinning thorn bezel, sawblade gear, animated needle) and button styling. Reacts to `currentEnvelope` and `globalPhase`, updated every timer tick.
- **`SorrowStormVisualizer`** — a `juce::Component` that paints the circular spectrum ring and ambient glow from `processor.getSpectrumData()` and `getEnvelopeLevel()`.
- **`drawGlitchTitle`** — offsets and re-draws title text in red/white with random jitter proportional to `fuzz`/`flutter`, for a glitch effect.
- **Sparkle particles** — a simple particle system (`SparkleParticle`) spawned on preset changes and high-envelope moments, animated and faded each timer tick.
- **60Hz `Timer`** drives all animation, envelope smoothing, and particle updates via `timerCallback()`.

### Project Structure


SorrowStorm/
├── Source/
│   ├── PluginProcessor.h      # DSP classes, parameter layout, processor interface
│   ├── PluginProcessor.cpp    # processBlock, preset loading, state save/load
│   ├── PluginEditor.h         # Editor, LookAndFeel, Visualizer class declarations
│   └── PluginEditor.cpp       # UI painting, layout, glitch title, sparkles
├── SorrowStorm.jucer          # (or CMakeLists.txt) JUCE project file
└── README.md


### Building from Source

**Requirements:**
- [JUCE](https://juce.com/get-juce) 7.x or later
- A C++17-compatible compiler (Xcode / MSVC / GCC)
- Projucer or CMake, depending on your JUCE project setup

**Steps:**

1. Clone or download the repository.
2. Open the `.jucer` file in Projucer (or configure via `CMakeLists.txt` if using CMake).
3. Set your exporter target (Xcode, Visual Studio, etc.) and generate the project.
4. Build the `VST3` and/or `AU` target in your IDE.
5. The built plugin binary will be output to your configured plugin destination folder, or you can manually copy it from the build directory.

### Extending the Plugin

Some natural extension points:

- **Add a real FFT** in place of the sinusoidal pseudo-spectrum in `processBlock` for a more accurate visualizer.
- **Expose flutter/drone LFO rate** as a separate parameter rather than deriving it from `flutter`.
- **Add stereo-width control** to the reverb output stage.
- **MIDI-triggered preset changes** — `processBlock` currently ignores `midiMessages` entirely.
- **User preset save/load into the preset dropdown** (currently `Save Memory` only exports raw XML state, it doesn't re-populate the preset list).

---

## License

Spectral Garden Proprietary License
© 2026 Spectral Garden

You are granted a non-exclusive license to use the Forsaken plugin for personal or commercial audio production.

You may not:
- redistribute the plugin
- modify, decompile, or reverse-engineer the plugin
- sell or repackage the plugin

All rights reserved.

DISCLAIMER — USE AT YOUR OWN RISK  
All VST plugins, software units, and applications provided through Spectral Garden are experimental digital tools. While extreme care and testing have been applied to ensure stable, high‑performance architecture, all downloads and modules are provided “AS IS” without warranty of any kind.

Use of these VST units and software applications is strictly at your own risk. The creator assumes no liability for audio hardware clipping, data corruption, DAW crashes, or unexpected sonic anomalies.
