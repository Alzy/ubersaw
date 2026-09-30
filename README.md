# ÜBERSAW

**A JP-8000 supersaw recreation for Eurorack.**

ÜBERSAW runs a seven-oscillator, 24-bit phase-accumulator model on the [Electro-Smith Daisy Patch Init](https://electro-smith.com/products/patch-init). Its oscillator structure follows the TC170C140 DSP reverse engineering [presented at 39C3](https://www.youtube.com/watch?v=XM_q5T7wTpQ); its detune and mix controls follow [Adam Szabo's measurements of JP-8000 output](https://www.adamszabo.com/internet/adam_szabo_how_to_emulate_the_super_saw.pdf). Exact hardware matching still needs a direct audio comparison.

The JP-8000's Supersaw oscillator — seven detuned sawtooth waves through a pitch-tracked high-pass filter — defined the sound of trance music from Darude's *Sandstorm* to an entire generation of electronic music. This project brings that oscillator structure to Eurorack.

## Features

- **Measured detune** — 7 naive sawtooth phase accumulators with Szabo's asymmetric frequency offsets and nonlinear detune control
- **24-bit fixed-point arithmetic** — emulates the integer overflow wrapping behavior of the original hardware that gives the supersaw its unique character
- **96 kHz sample rate** — closest available match to the JP-8000's internal 88.2 kHz, preserving the aliasing characteristics that contribute brightness without harshness
- **Pitch-tracked high-pass filter** — removes sub-fundamental aliasing products, just like the original
- **Full CV control** — 1V/Oct pitch input, CV-controllable detune, mix, and filter with dedicated knobs for each
- **Random phase on trigger** — each gate trigger randomizes oscillator phases, matching the JP-8000's note-on behavior
- **10 HP Eurorack module** — runs on the open-source Daisy Patch Init platform

## Hardware

- [Electro-Smith Daisy Patch Init](https://electro-smith.com/products/patch-init) (10HP, Eurorack)
- STM32H750 ARM Cortex-M7 @ 480 MHz
- 24-bit audio codec, stereo I/O
- 4 CV inputs, 4 knobs, 2 gate inputs, 2 audio outputs

## Control mapping

| Control   | Function           | Range / Notes                        |
|-----------|--------------------|--------------------------------------|
| Knob 1    | Pitch (coarse)     | Base frequency select                |
| Knob 2    | Detune             | 0 = unison → max = full JP-8000 spread |
| Knob 3    | Mix                | Center vs. side oscillator balance   |
| Knob 4    | Tone               | HPF cutoff offset from pitch-track   |
| CV 1      | 1V/Oct pitch       | Standard Eurorack V/Oct              |
| CV 2      | Detune mod         | Adds to Knob 2                       |
| CV 3      | Mix mod            | Adds to Knob 3                       |
| CV 4      | Tone mod           | Adds to Knob 4                       |
| Gate 1    | Note trigger       | Randomizes oscillator phases         |
| Gate 2    | (reserved)         | Future use                           |
| Audio L   | Supersaw out       | Main output                          |
| Audio R   | Supersaw out       | Duplicated mono output               |
| Toggle    | Mode               | Authentic (24-bit) vs. modern (float)|
| Button    | Manual trigger     | Acts as gate when no CV connected    |
| LED       | Gate indicator     | Lights on active gate                |

## Quick start

```bash
# Clone with submodules
git clone --recurse-submodules https://github.com/yourname/uebersaw.git
cd uebersaw

# Build the libraries (first time only)
cd libDaisy && make && cd ..
cd DaisySP && make && cd ..

# Build the firmware
make

# Flash to Daisy (hold BOOT, press RESET, release both, then run:)
make program-dfu
```

### Prerequisites

- `arm-none-eabi-gcc` **v10.3-2021.10** — [download](https://developer.arm.com/downloads/-/gnu-rm/10-3-2021-10)
- `dfu-util` for USB flashing
- GNU `make`
- `git`

See [docs/build-guide.md](docs/build-guide.md) for detailed platform-specific setup instructions.

## How it works

Seven sawtooth oscillators are generated as raw 24-bit phase accumulators — no wavetables or band limiting. The phase wrap *is* the sawtooth discontinuity. Each side oscillator uses a measured asymmetric frequency offset scaled by pitch and the shaped detune control. The mix control lowers the center level and follows a separate gain curve for the six side oscillators. The wide summed output passes through a high-pass filter tracking the fundamental frequency.

```text
pitch -> seven 24-bit saw phases -> measured center/side mix gains
      -> wide sum -> pitch-tracked high-pass filter -> output
```

See [docs/algorithm.md](docs/algorithm.md) for the complete technical breakdown.

## References

- **39C3 talk** — "From Silicon to Darude Sand-storm: breaking famous synthesizer DSPs" by Giulioz ([video](https://www.youtube.com/watch?v=XM_q5T7wTpQ))
- **Gearmulator** — JE-8086 bit-accurate JP-8000 emulation ([github.com/dsp56300/gearmulator](https://github.com/dsp56300/gearmulator))
- **Adam Szabo** — "How to Emulate the Super Saw" (2010, KTH Stockholm) — black-box analysis that correctly predicted most of the algorithm's structure
- **Roland JP-8000** — [Wikipedia](https://en.wikipedia.org/wiki/Roland_JP-8000)

## License

MIT — see [LICENSE](LICENSE) for details.

Open-source hardware and firmware, following the Daisy platform's open-source tradition.
