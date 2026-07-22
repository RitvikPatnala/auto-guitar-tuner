# Auto Guitar Tuner

A handheld device that detects a guitar string's pitch and physically rotates the tuning peg until the string is in tune. Built around a custom ESP32-S3 PCB designed from schematic capture to routed layout, with real-time pitch detection firmware and closed-loop servo control. Tunes to within ±12 cents across all six strings and multiple tuning standards (Standard, Eb, Drop D, Open G).

<img width="749" height="491" alt="Screenshot 2026-07-20 at 3 30 26 PM" src="https://github.com/user-attachments/assets/685666df-b003-4b2e-a9de-a5b04610d845" />

## How it works?

1. A piezo vibration sensor mounted on the guitar picks up string vibrations (immune to ambient room noise, unlike a microphone).
2. An LM386 amplifier stage (gain ≈ 50) with a 1.6 kHz RC anti-aliasing filter conditions the signal for the ESP32-S3's ADC (8192 Hz, 12-bit).
3. Firmware detects the fundamental frequency using time-domain autocorrelation and computes the error from the target pitch in cents.
4. A PID controller drives an HS-318 servo attached to the tuning peg, making small angular corrections until the string is in tune — with safety interlocks (rotation limits, stall           detection, voicing threshold) to protect the guitar.
5. A TFT LCD (SPI) and two-button interface let the user select strings and tuning modes via an FSM-driven UI.

## Why Autocorrelation instaead of FFT?

The most interesting design decision in the project. Guitar strings produce strong harmonics — the 2nd harmonic often carries more energy than the fundamental, so naive FFT peak-picking frequently reports the octave above the true pitch. Time-domain autocorrelation measures the signal's self-similarity across lag values, which naturally locks onto the fundamental period. It was also faster on the ESP32-S3:

| Approach | Latency |
|---|---|
| Autocorrelation | ~22 ms |
| FFT + peak detection | ~48 ms |

The pipeline: Hann window → autocorrelation over the guitar's lag range (73–330 Hz) → normalization → zero-crossing peak detection → parabolic sub-sample interpolation for resolution beyond the lag grid. The ~22 ms loop lets the tuner track pitch continuously while the motor turns, so the user doesn't need to re-pluck after every adjustment.

## Hardware Design

- Power: dual AP62150 buck converters (9V battery → 5V and 3.3V rails). Chosen over LDOs after a dissipation analysis — a stalled servo draws 800 mA, which would burn 3.2 W in an LDO vs ~90% buck efficiency.
- Custom PCB: designed in KiCAD (schematic → layout), simulated in LTSpice; noise mitigated with ground partitioning, supply decoupling, and the RC input filter.

<img width="1167" height="794" alt="Screenshot 2026-07-21 at 8 55 34 PM" src="https://github.com/user-attachments/assets/7e43bce2-07ff-4429-8112-eaeafcd312ae" />

<img width="793" height="683" alt="Screenshot 2026-07-21 at 8 56 06 PM" src="https://github.com/user-attachments/assets/79170432-f1c7-4735-829e-0fb514777db7" />

- Motor: HS-318 analog servo, selected over a stepper for closed-loop position feedback and full torque at low speed. Torque analysis: 0.25 kg·cm required at the peg vs 3.7 kg·cm available.

## Verification Highlights

| Subsystem | Result |
|---|---|
| Power rails | 3.353 V / 4.995 V measured (±1% of nominal) |
| Audio chain | Gain of ~52 measured (target 50); piezo signal 84 mV |
| Pitch detection | ±12 cents final tuning accuracy, ~22 ms latency |
| Control | Autonomous tuning verified on all 6 strings + alternate tunings |






