# The JP-8000 supersaw algorithm

## How Roland kept a secret for 28 years

The Roland JP-8000, released in 1997, introduced the "Supersaw" oscillator: a sound so distinctive it defined an entire genre of electronic music. For nearly three decades, nobody knew exactly how it worked. Roland's implementation ran on four custom Toshiba TC170C140 "ESP2" DSP chips with a completely undocumented instruction set. No datasheet was ever published.

In December 2025, a researcher known as Giulioz presented **"From Silicon to Darude Sand-storm: breaking famous synthesizer DSPs"** at the 39th Chaos Communication Congress (39C3). Through silicon-level reverse engineering: decapping the chip, microscopy, automated standard cell classification, opcode fuzzing, and JIT compilation, the team extracted the actual firmware and decoded the supersaw algorithm.

The revelation: **the supersaw is far simpler than anyone imagined.**

## Oscillator structure

This sketch shows the oscillator and mixer structure calibrated to the frequency offsets and gain curves measured by Adam Szabo. It is a behavioral model, not a transcription of the DSP program. Filter response and output gain still require comparison with original JP-8000 output.

```c
int24_t saw[7] = {0};  // Phase accumulators (randomized on note-on)

const float ratio[7] = { 0, 0.01991221, -0.01952356,
                         0.06216538, -0.06288439,
                         0.10745242, -0.11002313 };

int32_t next(int24_t pitch, float shaped_detune, float mix) {
    float center_gain = 0.99785 - 0.55366 * mix;
    float side_gain = 0.044372 + 1.2841 * mix - 0.73764 * mix * mix;
    int32_t sum = 0;  // Wide enough for all seven oscillator outputs.
    for (int i = 0; i < 7; i++) {
        int24_t voice_detune = pitch * shaped_detune * ratio[i];
        saw[i] = wrap24(saw[i] + pitch + voice_detune);
        sum += saw[i] * (i == 0 ? center_gain : side_gain);
    }
    return high_pass(sum);
}
```

The JP-8000's internal rate was approximately **88,200 Hz** (67.7376 MHz / (16 x 3 x 16)); this implementation runs the oscillator at **96,000 Hz**.

## Why it works: component breakdown

### Phase accumulator sawtooths

Each oscillator is a 24-bit integer that increments by `pitch + voice_detune` every sample. There is no wavetable. There is no waveform shaping. The raw integer value *is* the sawtooth waveform. When the 24-bit integer overflows (exceeds +/-8,388,607), it wraps around: this natural wrap-around creates the sawtooth discontinuity.

This is the simplest possible sawtooth generator: a counter that overflows.

### Measured detune offsets

The seven full-detune frequency offsets measured from JP-8000 audio are asymmetric:

| Oscillator | Frequency offset | Role |
|-----------|------------------|------|
| 0 | 0 | Center |
| 1 / 2 | +1.991% / -1.952% | Inner pair |
| 3 / 4 | +6.217% / -6.288% | Middle pair |
| 5 / 6 | +10.745% / -11.002% | Outer pair |

The asymmetry means the detuned oscillators are not perfectly mirrored around the fundamental. This subtle imperfection prevents the sterile sound of perfectly symmetric detuning: it adds organic width and movement.

### Pitch-proportional detuning

The formula `pitch * (1 + shaped_detune * ratio[i])` makes detuning proportional to pitch. A note one octave higher gets twice the absolute detuning, maintaining consistent musical-interval spread across the keyboard. The detune knob uses Szabo's nonlinear fitted curve, evaluated at control update time in double precision. Its zero endpoint is forced to exact unison.

### The mixing formula

Szabo measured a center gain of `0.99785 - 0.55366 * mix` and a gain for each side oscillator of `0.044372 + 1.2841 * mix - 0.73764 * mix^2`. The center falls as Mix rises, and the side voices retain a small level even at Mix = 0. The individual phase accumulators wrap at 24 bits; the combined signal must retain enough bits for all seven contributions. Wrapping the mixer sum to 24 bits at full mix instead produces a single ramp near seven times the requested frequency.

### The high-pass filter

After summing all seven oscillators, the result passes through a pitch-tracked high-pass filter. This is critical: the naive (non-bandlimited) sawtooth waves produce aliased harmonics that fold back below the fundamental. These sub-fundamental artifacts sound ugly and muddy. The HPF removes them while preserving the aliased harmonics *above* the fundamental, which contribute desirable brightness and "air."

The 88.2 kHz sample rate works in concert with the HPF: at this rate, the Nyquist frequency (44.1 kHz) pushes most aliasing well above audibility for musical pitches.

### Random phase initialization

On every note-on event, all seven oscillators receive random starting phases. This means every note press sounds slightly different: the phase relationships between oscillators vary, creating subtle timbral variation that gives the sound life.

## Prior art: Adam Szabo's analysis

In 2010, Adam Szabo (KTH Stockholm) published "How to Emulate the Super Saw," a black-box analysis based on FFT and oscilloscope measurements of JP-8000 output. His findings were remarkably close:

- **Correctly identified:** 7 sawtooths, pitch-tracked HPF, asymmetric detuning, random phases, aliasing contribution
- **Approximated with an 11th-degree polynomial:** The nonlinear detune knob response measured from hardware output.
- **Measured mix equations:** The center volume decreases while the side volume follows a parabola. Our implementation uses these curves until direct comparison with JP-8000 audio can settle the exact control mapping.

Szabo's paper was the best available reference for 15 years. The reverse engineering confirms his fundamental insights while revealing just how minimal the actual code is.

## Implementation notes for ÜBERSAW

### Sample rate

The Daisy runs at **96 kHz** (closest available to 88.2 kHz). This produces very similar aliasing characteristics. For a perfectionist match, internal 2x oversampling at 48 kHz could better approximate 88.2 kHz, but 96 kHz is simpler and close enough.

### 24-bit emulation on 32-bit ARM

The STM32H750 uses 32-bit integers. We emulate 24-bit overflow with:

```cpp
int32_t Wrap24(int32_t val) {
    return (val << 8) >> 8;  // Sign-extend from bit 23
}
```

This is applied to each saw's phase advance. The mixer and high-pass filter retain a wider signal range; the oscillator output uses one shared gain in both the fixed-point and floating-point modes. Whether their detailed spectra match the original JP-8000 still requires comparison with a reference recording.

### Filter implementation

The exact JP-8000 HPF type is not fully documented in the public reverse engineering data. Community analysis suggests a multi-pole SVF high-pass. The initial implementation uses a one-pole HPF for simplicity, with the roadmap prioritizing filter refinement as Phase 2. The one-pole captures the essential behavior (removing sub-fundamental content) while being stable and computationally trivial.
