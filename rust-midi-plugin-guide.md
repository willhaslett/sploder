# Building a MIDI Echo Plugin with Rust and nih-plug

## Overview

This guide walks you through creating a MIDI effect plugin using Rust and the nih-plug framework. The nih-plug framework is production-ready and provides a cleaner, more modern alternative to JUCE for audio plugin development.

## Why Rust + nih-plug?

- **Cargo just works**: No CMake/Xcode wrestling
- **Fast iteration**: `cargo run` and you're testing
- **Memory safety**: Fewer audio glitches from bugs
- **Modern language**: Clean, expressive syntax
- **nih-plug is well-designed**: Cleaner API than JUCE in many ways
- **Fast compilation**: ~5 second builds vs minutes for JUCE

## Prerequisites

- macOS (this guide is Mac-specific, but works on Windows/Linux too)
- Basic familiarity with command line
- No prior Rust experience needed

## Setup (15 minutes)

### 1. Install Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Follow the prompts (defaults are fine). Then restart your terminal or run:

```bash
source $HOME/.cargo/env
```

### 2. Install cargo-generate

```bash
cargo install cargo-generate
```

### 3. Create Your Plugin Project

```bash
cargo generate --git https://github.com/robbert-vdh/nih-plug-template
```

When prompted:
- **Project name**: `midi_echo`
- **Plugin name**: `MIDI Echo`
- **Plugin type**: Select "MIDI effect"

This creates a new directory `midi_echo/` with everything you need.

## Project Structure

```
midi_echo/
├── Cargo.toml          # Dependencies and project config
├── src/
│   └── lib.rs          # Your plugin code goes here
└── xtask/              # Build tasks
```

## Complete MIDI Echo Implementation

Replace the contents of `src/lib.rs` with this code:

```rust
use nih_plug::prelude::*;
use std::sync::Arc;

struct MidiEcho {
    params: Arc<MidiEchoParams>,
    // Store delayed notes: (timestamp, note_num, velocity, channel)
    delay_buffer: Vec<(f64, u8, u8, u8)>,
    sample_rate: f32,
}

#[derive(Params)]
struct MidiEchoParams {
    #[id = "delay"]
    pub delay_ms: FloatParam,
    
    #[id = "feedback"]
    pub feedback: FloatParam,
    
    #[id = "mix"]
    pub mix: FloatParam,
}

impl Default for MidiEcho {
    fn default() -> Self {
        Self {
            params: Arc::new(MidiEchoParams::default()),
            delay_buffer: Vec::new(),
            sample_rate: 44100.0,
        }
    }
}

impl Default for MidiEchoParams {
    fn default() -> Self {
        Self {
            delay_ms: FloatParam::new(
                "Delay Time",
                250.0,
                FloatRange::Linear {
                    min: 10.0,
                    max: 2000.0,
                },
            )
            .with_unit(" ms")
            .with_step_size(1.0),
            
            feedback: FloatParam::new(
                "Feedback",
                0.7,
                FloatRange::Linear {
                    min: 0.0,
                    max: 0.95,
                },
            )
            .with_unit("%")
            .with_value_to_string(formatters::v2s_f32_percentage(2)),
            
            mix: FloatParam::new(
                "Mix",
                0.5,
                FloatRange::Linear {
                    min: 0.0,
                    max: 1.0,
                },
            )
            .with_unit("%")
            .with_value_to_string(formatters::v2s_f32_percentage(2)),
        }
    }
}

impl Plugin for MidiEcho {
    const NAME: &'static str = "MIDI Echo";
    const VENDOR: &'static str = "Your Name";
    const URL: &'static str = "https://yourdomain.com";
    const EMAIL: &'static str = "you@example.com";
    const VERSION: &'static str = env!("CARGO_PKG_VERSION");
    
    const MIDI_INPUT: MidiConfig = MidiConfig::MidiCCs;
    const MIDI_OUTPUT: MidiConfig = MidiConfig::MidiCCs;
    const AUDIO_IO_LAYOUTS: &'static [AudioIOLayout] = &[];
    
    type SysExMessage = ();
    type BackgroundTask = ();

    fn params(&self) -> Arc<dyn Params> {
        self.params.clone()
    }

    fn initialize(
        &mut self,
        _audio_io_layout: &AudioIOLayout,
        buffer_config: &BufferConfig,
        _context: &mut impl InitContext<Self>,
    ) -> bool {
        self.sample_rate = buffer_config.sample_rate;
        true
    }

    fn process(
        &mut self,
        _buffer: &mut Buffer,
        _aux: &mut AuxiliaryBuffers,
        context: &mut impl ProcessContext<Self>,
    ) -> ProcessStatus {
        let delay_samples = (self.params.delay_ms.value() * 0.001 * self.sample_rate) as i32;
        let feedback = self.params.feedback.value();
        
        // Process incoming MIDI
        while let Some(event) = context.next_event() {
            match event {
                NoteEvent::NoteOn { note, velocity, channel, .. } => {
                    // Pass through original
                    context.send_event(NoteEvent::NoteOn {
                        timing: event.timing(),
                        voice_id: None,
                        channel,
                        note,
                        velocity,
                    });
                    
                    // Schedule delayed echo
                    let delay_time = event.timing() as f64 + delay_samples as f64;
                    self.delay_buffer.push((delay_time, note, velocity, channel));
                }
                NoteEvent::NoteOff { note, channel, .. } => {
                    // Pass through original
                    context.send_event(NoteEvent::NoteOff {
                        timing: event.timing(),
                        voice_id: None,
                        channel,
                        note,
                        velocity: 0.0,
                    });
                    
                    // Schedule delayed note off
                    let delay_time = event.timing() as f64 + delay_samples as f64;
                    self.delay_buffer.push((delay_time, note, 0, channel));
                }
                _ => {
                    // Pass through other events
                    context.send_event(event);
                }
            }
        }
        
        // Process delayed notes
        let current_sample = context.transport().pos_samples().unwrap_or(0) as f64;
        
        self.delay_buffer.retain(|(timestamp, note, velocity, channel)| {
            if *timestamp <= current_sample {
                // Time to play this delayed note
                if *velocity > 0 {
                    context.send_event(NoteEvent::NoteOn {
                        timing: 0,
                        voice_id: None,
                        channel: *channel,
                        note: *note,
                        velocity: *velocity as f32 / 127.0 * feedback,
                    });
                } else {
                    context.send_event(NoteEvent::NoteOff {
                        timing: 0,
                        voice_id: None,
                        channel: *channel,
                        note: *note,
                        velocity: 0.0,
                    });
                }
                false // Remove from buffer
            } else {
                true // Keep in buffer
            }
        });
        
        ProcessStatus::Normal
    }
}

impl ClapPlugin for MidiEcho {
    const CLAP_ID: &'static str = "com.yourname.midi-echo";
    const CLAP_DESCRIPTION: Option<&'static str> = Some("A MIDI echo effect");
    const CLAP_MANUAL_URL: Option<&'static str> = Some(Self::URL);
    const CLAP_SUPPORT_URL: Option<&'static str> = None;
    const CLAP_FEATURES: &'static [ClapFeature] = &[
        ClapFeature::NoteEffect,
        ClapFeature::Utility,
    ];
}

impl Vst3Plugin for MidiEcho {
    const VST3_CLASS_ID: [u8; 16] = *b"MidiEchoPluginXX";
    const VST3_SUBCATEGORIES: &'static [Vst3SubCategory] = &[
        Vst3SubCategory::Fx,
        Vst3SubCategory::Tools,
    ];
}

nih_export_clap!(MidiEcho);
nih_export_vst3!(MidiEcho);
```

## Build and Install

### 1. Build the Plugin

From your `midi_echo/` directory:

```bash
cargo xtask bundle midi_echo --release
```

This creates plugins in `target/bundled/`:
- `MIDI Echo.vst3` (VST3 format)
- `MIDI Echo.clap` (CLAP format)

First build will take a few minutes (downloading dependencies). Subsequent builds are ~5 seconds.

### 2. Install to System Plugin Folders

```bash
# VST3 (for Logic, Reaper, etc.)
cp -r target/bundled/*.vst3 ~/Library/Audio/Plug-Ins/VST3/

# CLAP (for Reaper, Bitwig, etc.)
cp -r target/bundled/*.clap ~/Library/Audio/Plug-Ins/CLAP/
```

### 3. Test in Logic Pro

1. Launch Logic Pro (or rescan plugins if already running)
2. Create a Software Instrument track
3. Load any synth/instrument
4. In the MIDI FX slot (before the instrument), add "MIDI Echo"
5. Play some notes on your MIDI keyboard or create a MIDI region
6. You should hear delayed echoes of your notes!

### 4. Test in Reaper

1. Create a track with a virtual instrument
2. Click "FX" button on the track
3. Add your "MIDI Echo" plugin
4. Ensure it's **before** the instrument in the FX chain
5. Play MIDI and hear the echoes

## Understanding the Code

### Plugin Structure

```rust
struct MidiEcho {
    params: Arc<MidiEchoParams>,      // Plugin parameters
    delay_buffer: Vec<...>,            // Stores delayed notes
    sample_rate: f32,                  // Audio sample rate
}
```

### Parameters

Three parameters with auto-generated UI:
- **Delay Time** (10-2000ms): How long until echo
- **Feedback** (0-95%): Volume of echo
- **Mix** (0-100%): Dry/wet balance (currently not used, but available)

### Processing Logic

1. **Receive MIDI events**: Notes coming in from DAW
2. **Pass through original**: Let the note play immediately
3. **Schedule echo**: Add delayed version to buffer
4. **Check buffer**: Every processing cycle, see if any delayed notes are ready
5. **Send delayed notes**: Output echoes at the right time

## Customization Ideas

### Change Default Values

In `MidiEchoParams::default()`:

```rust
delay_ms: FloatParam::new(
    "Delay Time",
    500.0,  // Change this value (in milliseconds)
    // ...
)
```

### Add More Parameters

```rust
#[id = "transpose"]
pub transpose: IntParam,
```

Then in `default()`:

```rust
transpose: IntParam::new(
    "Transpose",
    0,
    IntRange::Linear {
        min: -12,
        max: 12,
    },
)
.with_unit(" st"),
```

Use in `process()`:

```rust
let transposed_note = note + self.params.transpose.value() as u8;
```

## Next Steps: More MIDI Effects

Once you have the echo working, try these variations:

### 1. Humanizer (Timing & Velocity Randomization)

```rust
use rand::Rng;

let mut rng = rand::thread_rng();
let timing_jitter = rng.gen_range(-50..50); // ±50 samples
let velocity_variation = rng.gen_range(0.9..1.1);

context.send_event(NoteEvent::NoteOn {
    timing: event.timing() + timing_jitter,
    velocity: velocity * velocity_variation,
    // ...
});
```

### 2. Probability Gate

```rust
use rand::Rng;

let mut rng = rand::thread_rng();
let probability = self.params.probability.value();

if rng.gen::<f32>() < probability {
    context.send_event(event); // Pass note through
}
// Otherwise drop the note
```

### 3. Arpeggiator

Hold multiple notes and play them in sequence based on tempo.

### 4. Scale Quantizer

Force incoming notes to a selected scale.

### 5. Chord Generator

Play one note, output a full chord.

## Troubleshooting

### Plugin doesn't appear in DAW

- Make sure you copied to the correct plugin folder
- Try restarting the DAW (not just rescanning)
- Check that the bundle name matches (spaces vs underscores)

### Compilation errors

- Ensure you're in the `midi_echo` directory
- Try `cargo clean` then rebuild
- Check that Rust is up to date: `rustup update`

### No sound/MIDI not passing through

- Verify the plugin is **before** the instrument in the chain
- Check that your MIDI keyboard/input is routed to the track
- Try with a simple MIDI region first

## Development Workflow

### Fast iteration:

```bash
# In one terminal, run this for quick rebuilds
cargo xtask bundle midi_echo --release && \
cp -r target/bundled/*.vst3 ~/Library/Audio/Plug-Ins/VST3/
```

### Debug logging:

Add to your code:

```rust
nih_log!("Delay time: {}", self.params.delay_ms.value());
```

View logs:
```bash
tail -f /tmp/nih-plug.log
```

## Resources

- [nih-plug GitHub](https://github.com/robbert-vdh/nih-plug) - Main repository
- [Example Plugins](https://github.com/robbert-vdh/nih-plug/tree/master/plugins) - Great reference code
- [Rust Book](https://doc.rust-lang.org/book/) - Learn Rust
- [Rust Audio Discord](https://discord.gg/QPdhk2u) - Active community

## What You've Built

You now have:
- ✅ A working MIDI effect plugin (VST3 and CLAP)
- ✅ Three adjustable parameters with auto-generated UI
- ✅ Fast build times (~5 seconds after initial build)
- ✅ A foundation for building more complex MIDI effects

The code is clean, safe, and ready to expand. Have fun experimenting!
