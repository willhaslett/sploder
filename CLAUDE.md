# Sploder - MIDI Echo Plugin

## Project Overview
Building a MIDI echo effect plugin using Rust and nih-plug framework. Target is Logic Pro (VST3 format).

## Tech Stack
- **Language**: Rust
- **Framework**: nih-plug (production-ready audio plugin framework)
- **Format**: VST3 (Logic Pro compatible)
- **Build Tool**: cargo-xtask

## Current Goal
Get a basic MIDI echo effect working in Logic Pro with:
- Delay time parameter (10-2000ms)
- Feedback parameter (0-95%)
- Mix parameter (0-100%)

## Quick Build Commands

```bash
# Build plugin
cargo xtask bundle midi_echo --release

# Install to Logic Pro
cp -r target/bundled/*.vst3 ~/Library/Audio/Plug-Ins/VST3/

# Quick rebuild and install
cargo xtask bundle midi_echo --release && cp -r target/bundled/*.vst3 ~/Library/Audio/Plug-Ins/VST3/
```

## Project Structure
```
sploder/
├── Cargo.toml          # Dependencies and project config
├── src/
│   └── lib.rs          # Plugin implementation
└── xtask/              # Build tasks
```

## Testing in Logic Pro
1. Create Software Instrument track
2. Load any synth/instrument
3. Add "MIDI Echo" in MIDI FX slot (before instrument)
4. Play MIDI to hear delayed echoes

## Debug Logging
```rust
nih_log!("Debug message here");
```
View logs: `tail -f /tmp/nih-plug.log`

## Resources
- [nih-plug GitHub](https://github.com/robbert-vdh/nih-plug)
- [Example Plugins](https://github.com/robbert-vdh/nih-plug/tree/master/plugins)
