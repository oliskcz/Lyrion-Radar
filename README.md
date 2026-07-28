# Lyrion Radar 🛰️

**FMCW radar at 5.5–6 GHz for counter-UAS (drone detection) and general-purpose ranging.**

[![Status](https://img.shields.io/badge/Status-Planning-yellow)]()
[![License](https://img.shields.io/badge/License-TBD-lightgrey)]()

---

## Table of Contents

- [What is Lyrion Radar?](#what-is-lyrion-radar)
- [FMCW Principle in 30 Seconds](#fmcw-principle-in-30-seconds)
- [Drone Detection — Why This Works](#drone-detection--why-this-works)
- [System Architecture](#system-architecture)
- [Complete Component List](#complete-component-list)
- [Signal Chain Walkthrough](#signal-chain-walkthrough)
- [Power Budgets & Link Budget](#power-budgets--link-budget)
- [Chirp Configurations](#chirp-configurations)
- [Signal Processing Pipeline](#signal-processing-pipeline)
- [Design Decisions Explained](#design-decisions-explained)
- [Project Status](#project-status)
- [Open Questions](#open-questions)

---

## What is Lyrion Radar?

Lyrion Radar is a **Frequency-Modulated Continuous-Wave (FMCW) radar** operating in the 5.5–6.0 GHz band. It is designed primarily for **drone detection (counter-UAS)** at ranges of 1–3 km, with secondary capability for general short-to-medium range detection and ranging.

The radar uses an **open-loop VCO ramp** (no PLL chip) driven by a 16-bit DAC, a **diode-ring mixer** downconverter, a **14-bit ADC** with massive oversampling, and an **STM32H723** Cortex-M7 processor for real-time signal processing.

### Quick Facts

| Parameter | Value |
|-----------|-------|
| Frequency band | 5.5–6.0 GHz (500 MHz sweep) |
| Range resolution | **30 cm** |
| Max range (small drone) | ~2 km (with 64-chirp integration) |
| Max range (medium drone) | ~3 km (with integration) |
| TX power | +11.4 dBm (prototype 1), +27 dBm (future PA upgrade) |
| Antenna gain | 24 dBi each (TX and RX, separate) |
| ADC | 14-bit, 25 MSPS (187× oversampled) |
| Processor | STM32H723 (Cortex-M7 @ 550 MHz) |
| Update rate | 100–1000 Hz (configurable) |
| Estimated BOM cost | ~$40 (excluding antennas) |

---

## FMCW Principle in 30 Seconds

FMCW radar transmits a signal whose frequency sweeps linearly over time (a "chirp"). The signal reflects off a target and returns after a time delay `τ = 2R/c`. The received (delayed) signal is mixed with the currently transmitted signal.

This produces a **beat frequency** directly proportional to target range:

```
f_IF = (2 · B · R) / (c · T)
```

| Symbol | Meaning | Typical value |
|--------|---------|---------------|
| B | Sweep bandwidth | 500 MHz |
| R | Target range | 1–3000 m |
| c | Speed of light | 3·10⁸ m/s |
| T | Ramp time | 1–10 ms |

An FFT of this beat signal reveals targets as distinct frequency peaks. With a **triangular ramp** (up-chirp + down-chirp), we can extract both range and velocity simultaneously.

---

## Drone Detection — Why This Works

Drones present a unique detection challenge:

| Drone class | Example | RCS (radar cross-section) | Detection range (with integration) |
|-------------|---------|--------------------------|-----------------------------------|
| Micro (< 250 g) | DJI Mini 4 | 0.001–0.005 m² (−30 to −23 dBsm) | ~500 m (marginal) |
| Small (250 g – 2 kg) | DJI Phantom 4 | 0.01–0.05 m² (−20 to −13 dBsm) | **~2 km** ✅ |
| Medium (2–25 kg) | DJI Matrice 300 | 0.05–0.1 m² (−13 to −10 dBsm) | **~3 km** ✅ |
| Large (> 25 kg) | Fixed-wing UAV | 0.1–1 m² (−10 to 0 dBsm) | > 3 km ✅ |

**Why 5.5–6 GHz for drone detection?**
- Good balance between resolution (500 MHz → 30 cm) and atmospheric attenuation
- Smaller antennas than S-band, less rain fade than X-band
- Components are widely available and affordable
- Legal to operate in ISM-adjacent bands

### Key Technical Challenges Solved

| Challenge | Our solution |
|-----------|-------------|
| Very small RCS (0.01 m²) | Coherent integration across 64–256 chirps (+18 to +24 dB SNR) |
| Drone vs. bird discrimination | Micro-Doppler analysis of rotor blade modulation (100–500 Hz) |
| Stationary clutter rejection | MTI (Moving Target Indication) via range-Doppler processing |
| Free-running VCO drift | TMP102 temperature compensation + calibration LUT |
| High dynamic range (close vs. far) | 14-bit ADC + 25 MSPS oversampling (~17.8 effective bits) + AGC |

---

## System Architecture

### High-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SIGNAL SOURCE                                      │
│                                                                          │
│  STM32H723 ──SPI──► DAC8830 ──► TLV9062 ──► YSGM556006 VCO              │
│                     (16-bit)     (buffer)     (5320-6060 MHz, +6 dBm)    │
│                                        │                                 │
│  REF5050A (5.0V ref, 3 ppm/°C) ──► DAC VREF                             │
│  TMP102 (I²C temperature sensor) ──► compensation                        │
└─────────────────────────────────────────────────────────────────────────┘
                                         │
                                   100 nF DC block
                                         │
┌─────────────────────────────────────────────────────────────────────────┐
│                            TX CHAIN                                      │
│                                                                          │
│                                     ┌──► TX antenna (24 dBi)            │
│  VCO ──► 6 dB pad ──► YG802020W ──┤   +11.4 dBm → +35.4 dBm EIRP       │
│  +6 dBm   (resistive)  (+15 dB)    │                                     │
│             0 dBm       +15 dBm    └──► LO to mixer                     │
│                                       +11.4 dBm                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                            RX CHAIN                                      │
│                                                                          │
│  RX antenna ──► QPL9547 ──► NCS4-63+ ──► YX18 ──► RC LPF ──► OPA838   │
│  (24 dBi)       (LNA)       (balun)     (mixer)  (4.8 MHz)   (×10)      │
│                 NF=0.6dB              ─────┤                             │
│                 G=17dB                LO from GP2X+                     │
│                                                                          │
│  OPA838 ──► MCP6S91 ──► ADC3642 ──DCMI──► STM32H723                     │
│  (×10)      (PGA 1-32×)  (14-bit        (DMA → decimate → FFT →        │
│             SPI AGC       25 MSPS)        detection → output)           │
└─────────────────────────────────────────────────────────────────────────┘
```

### MCU Peripherals Map

```
                         ┌──────────────────────┐
                         │    STM32H723VGT6      │
                         │                      │
SPI1 ──► DAC8830        │  Cortex-M7 @ 550 MHz  │
       (ramp generation) │  HW FPU (double prec) │
                         │  320 KB SRAM          │
SPI2 ──► MCP6S91         │  DCMI, SPI, I2C, TIM  │
       (AGC gain)        │  USB OTG FS, UART     │
       ► ADC3642          │                      │
       (config only)     └──────────────────────┘
                                │
I2C1 ──► TMP102                  │
       (temperature)           DCMI
                                │
DCMI ◄── ADC3642 (14-bit parallel data @ 25 MSPS)
       (DMA double-buffered capture)

TIM ◄── ramp trigger (synchronizes DAC + ADC)

USB/UART ──► PC (range data, FFT spectrum, detections)
```

---

## Complete Component List

### Signal Source

| Part | Manufacturer | Role | Key Specs | Why this one |
|------|-------------|------|-----------|--------------|
| **YSGM556006** | Innotion | VCO | 5320–6060 MHz, +6 dBm, 0–5 V tune, 14 mA @ 5 V | You have 10 on hand. Covers 5.5–6.0 GHz with margin. |
| **DAC8830IDR** | Texas Instruments | 16-bit SPI DAC | 1 µs settling, unbuffered R-2R, SOIC-8 | 16-bit precision for fine frequency steps. Fast settling supports 1 µs step interval. |
| **REF5050AIDGKR** | Texas Instruments | Voltage reference | 5.0 V, 3 ppm/°C, 3 µVpp noise | Ultra-low drift reference — VCO frequency accuracy depends on this. Needs >5.3 V input. |
| **TLV9062IDR** | Texas Instruments | DAC output buffer | Dual op-amp, rail-to-rail, 10 MHz GBW | Buffers the DAC's R-2R output to drive the VCO's tuning pin. Second channel is spare. |
| **TMP102AIDRLR** | Texas Instruments | Temperature sensor | I²C, ±0.5 °C, SOT-563 | Compensates VCO drift with temperature. Placed near VCO for thermal coupling. |

### TX Chain

| Part | Manufacturer | Role | Key Specs | Why this one |
|------|-------------|------|-----------|--------------|
| **YG802020W** | Innotion | TX driver | 50 MHz–8 GHz, +15 dB gain @ 5.5 GHz, P1dB ~+16 dBm | You have 6 on hand. Drives the divider from 0 dBm to +15 dBm. Good linearity 1 dB below P1dB. |
| **GP2X+** | Mini-Circuits | 2-way power divider | 2.9–6.2 GHz, 3.6 dB total loss, 20 dB isolation | Guaranteed specs, tiny 3×3 mm package. Saves PCB design effort vs. a Wilkinson divider. |
| 68 Ω + 2× 150 Ω resistors | — | 6 dB resistive pad | 0603 thin-film | Essential: prevents VCO frequency pulling (9 MHz pp at 3:1 VSWR) by presenting a solid 50 Ω load. |

### RX Chain

| Part | Manufacturer | Role | Key Specs | Why this one |
|------|-------------|------|-----------|--------------|
| **QPL9547TR7** | Qorvo | LNA | 0.05–6 GHz, **0.6 dB NF**, 17 dB gain, +19 dBm P1dB | Industry-leading NF. Adds only 0.6 dB noise to the system. Swamps downstream noise with 17 dB gain. |
| **YX18** | Innotion | Diode quad mixer | GaAs Schottky, 1.4 V turn-on, cross-over quad | You have 6 on hand. GaAs gives higher breakdown than Si, allowing +12 dBm LO drive. |
| **NCS4-63+** (×2) | Mini-Circuits | Baluns (RF + LO) | 4.5–6 GHz, 1:4 impedance ratio, 0805 | You have 12 on hand. 1:4 balun provides 2× voltage step-up to cleanly switch the 1.4 V diodes. |
| **OPA838IDBVR** | Texas Instruments | IF preamplifier | **0.9 nV/√Hz**, **300 MHz GBW**, 1 mA supply | Extremely low noise for weak IF signals. 300 MHz GBW means bandwidth is never the bottleneck. |
| **MCP6S91T-E/MS** | Microchip | AGC (PGA) | 1×–32× (0–30 dB), **18 MHz GBW**, SPI | SPI-controlled gain. 18 MHz GBW gives 562 kHz BW at ×32 — sufficient for 100 kHz IF at 3 km. |

### Digital

| Part | Manufacturer | Role | Key Specs | Why this one |
|------|-------------|------|-----------|--------------|
| **ADC3642IRSBR** | Texas Instruments | ADC | **Dual 14-bit**, **25 MSPS**, ultra-low power, QFN-40 | 2× oversampling needed for full sensitivity. Dual channel: Ch A for IF now, Ch B for I/Q later. |
| **STM32H723VGT6** | ST | Processor | Cortex-M7 @ **550 MHz**, HW FPU, **320 KB SRAM**, DCMI | You have 3 on hand. Runs 1024-point FFT in ~50 µs. DCMI captures ADC3642 parallel output natively. |

### Power

| Part | Manufacturer | Role | Key Specs |
|------|-------------|------|-----------|
| **ADP150AUJZ-5.0** (×2) | Analog Devices | Ultra-low noise LDO | 5 V, <10 µVrms output noise, TSOT-23-5 |
| **LD1117S33** or similar | ST / TI | 3.3 V LDO | For STM32H723 and ADC3642 digital supply |

### Antennas

| Item | Spec | Size estimate | Notes |
|------|------|---------------|-------|
| TX antenna | **24 dBi**, 5.5–6 GHz | ~30×30 cm (8×8 patch) or 40 cm dish | Separate from RX (no circulator) |
| RX antenna | **24 dBi**, 5.5–6 GHz | Same | Pointed same direction as TX |

**Total component cost (estimate): ~$40** (excluding antennas, and parts you already own: VCO, diodes, baluns, STM32H723).

---

## Signal Chain Walkthrough

### TX Path — From VCO to Antenna

```
VCO (+6 dBm) → 6 dB pad (0 dBm) → YG802020W (+15 dBm) → GP2X+ divider (+11.4 dBm each)
                                                              │
                                                              ├── TX antenna: +11.4 dBm + 24 dBi = +35.4 dBm EIRP
                                                              │
                                                              └── LO to mixer: +11.4 dBm → NCS4-63+ (1:4 balun) → YX18
```

The 6 dB pad after the VCO is **critical**: without it, antenna impedance variations would shift the VCO frequency by up to 9 MHz (pulling spec). The YG802020W then restores the signal level. The GP2X+ splits it equally to TX and LO.

### RX Path — From Antenna to ADC

```
RX antenna (+18 to +6 dBm signal range depending on target)
    │
  QPL9547 LNA (+17 dB, NF=0.6 dB) — amplifies while adding almost no noise
    │
  NCS4-63+ balun (1:4, −0.5 dB) — converts single-ended to differential for mixer
    │
  YX18 diode mixer (−7 dB conversion loss) — downconverts RF to IF
    │
  RC LPF (1 kΩ + 33 pF, fc ≈ 4.8 MHz) — anti-aliasing filter
    │
  OPA838 fixed gain (×10 = +20 dB, 300 MHz GBW) — lifts signal above noise floor
    │
  MCP6S91 PGA (×1 to ×32 = 0 to +30 dB, SPI AGC) — adjusts gain per target range
    │
  ADC3642 ch A (14-bit, 25 MSPS) — digitizes with massive oversampling
    │
  STM32H723 (DCMI + DMA) — captures, decimates, FFT, detects
```

### IF Gain vs. Bandwidth Trade-off

The MCP6S91's GBW (18 MHz) means gain and bandwidth trade off:

| AGC setting | Gain | Bandwidth | Use when |
|-------------|------|-----------|----------|
| ×1 | +0 dB | 18 MHz | Close targets, strong IF signal |
| ×4 | +12 dB | 4.5 MHz | Medium range, 100–500 m |
| ×8 | +18 dB | 2.25 MHz | Long range, 1 km+ |
| ×32 | +30 dB | **562 kHz** | 3 km with 10 ms ramp (f_IF = 100 kHz) |

The OPA838 (×10, 30 MHz BW) is never the bottleneck. All bandwidth limits come from the MCP6S91.

---

## Power Budgets & Link Budget

### TX Power Budget

| Stage | Level | Notes |
|-------|-------|-------|
| VCO output | +6 dBm | YSGM556006 nominal |
| 6 dB resistive pad | −6 dB | Protects VCO from load pulling |
| YG802020W amplifier | +15 dB | 93 mA current, 1 dB below P1dB |
| GP2X+ divider | −3.6 dB | Typical total loss at 5.5 GHz |
| **Each port output** | **+11.4 dBm** | TX to antenna, LO to mixer |
| With 24 dBi antenna (TX) | **+35.4 dBm EIRP** | ~3.5 kW effective radiated power |

### RX Link Budget — Drone Detection

Conditions: Pt = +11.4 dBm, Gt = Gr = 24 dBi, NF = 0.8 dB, noise floor = −143.2 dBm (1 kHz bin).

| Range | RCS | Target type | Received power | SNR (single chirp) | **SNR (64-chirp integration)** |
|-------|-----|-------------|---------------|-------------------|-------------------------------|
| 500 m | 0.1 m² | Medium drone | −107.2 dBm | 36 dB | **54 dB** ✅ |
| 500 m | 0.01 m² | Small drone | −117.2 dBm | 26 dB | **44 dB** ✅ |
| 1 km | 0.1 m² | Medium drone | −127.2 dBm | 16 dB | **34 dB** ✅ |
| 1 km | 0.01 m² | Small drone | −137.2 dBm | 6 dB | **24 dB** ✅ |
| 2 km | 0.1 m² | Medium drone | −139.2 dBm | 4 dB | **22 dB** ✅ |
| 2 km | 0.01 m² | Small drone | −149.2 dBm | −6 dB | **12 dB** ⚠️ marginal |
| 3 km | 0.1 m² | Medium drone | −148.7 dBm | −5.5 dB | **12.5 dB** ⚠️ |
| 3 km | 0.01 m² | Small drone | −158.7 dBm | −15.5 dB | **2.5 dB** ❌ needs PA |

**The key takeaway:** Coherent integration across 64 chirps gives +18 dB of SNR improvement. This is **essential** for drone detection. A future PA upgrade (MMG3H21NT1, +27 dBm TX) would add another +8.6 dB, enabling 3 km small drone detection.

---

## Chirp Configurations

The radar supports **three operating modes** configurable in firmware:

| Mode | Ramp time | LPF cutoff | Max range | f_IF at 3 km | Samples (25 MSPS) | FFT size | Update rate |
|------|-----------|-----------|-----------|-------------|-------------------|----------|-------------|
| **Search** 🎯 | 10 ms | 194 kHz | 3 km | 100 kHz | 250,000 | 2048-pt | 100 Hz |
| **Track** 📍 | 5 ms | 500 kHz | 3 km | 200 kHz | 125,000 | 1024-pt | 200 Hz |
| **Fast** ⚡ | 1 ms | 4.8 MHz | 600 m | 1 MHz | 25,000 | 256-pt | 1000 Hz |

- **Search mode**: Maximum sensitivity. 100 Hz update rate is fine for scanning a sector.
- **Track mode**: Balanced sensitivity and update rate. Good for following a detected drone.
- **Fast mode**: High update rate for tracking fast-moving drones (> 10 m/s) at close range.

### Velocity (Doppler) with Triangular Ramp

A triangular ramp (up-chirp + down-chirp) extracts both range and velocity:

```
Up-chirp:   f_IF = f_range − f_doppler
Down-chirp: f_IF = f_range + f_doppler

→ f_range   = (f_up + f_down) / 2
→ f_doppler = (f_down − f_up) / 2
→ velocity  = λ · f_doppler / 2
```

| Ramp time | Max unambiguous velocity | Resolution (64 chirps) | Drones covered |
|-----------|------------------------|-----------------------|----------------|
| 10 ms | 1.25 m/s | 0.02 m/s | Hovering, slow flight |
| 5 ms | 2.5 m/s | 0.04 m/s | Hovering to 9 km/h |
| 1 ms | 12.5 m/s | 0.2 m/s | Hovering to 45 km/h |

---

## Signal Processing Pipeline

The firmware on the STM32H723 runs this pipeline in real-time:

```
                ┌─────────────────────────────────────────────────────────────┐
                │                   ONE MEASUREMENT CYCLE                      │
                │                                                              │
  DAC ramp ────► Generate pre-distorted ramp from LUT (SPI @ 1 µs/step)       │
                    │                                                           │
  ADC capture ──► DCMI + DMA double-buffered capture (25 MSPS, 14-bit)         │
                    │                                                           │
  Decimation ───► On-the-fly DMA ISR: average 125 samples → 1 output          │
                  (25 MSPS → 200 kSPS, reduces data from 500 KB → 4 KB)       │
                    │                                                           │
  Range FFT ────► CMSIS-DSP real FFT (256 to 2048 points, ~50–400 µs)         │
                    │                                                           │
  Up/down pair ─► Pair consecutive up-chirp and down-chirp for velocity        │
                    │                                                           │
  Integration ──► Stack 64–256 chirps coherently (2D range-Doppler map)        │
                    │                                                           │
  Detection ────► CFAR threshold → peak detection → range & velocity output    │
                    │                                                           │
  AGC update ───► Measure peak IF amplitude → set MCP6S91 gain for next ramp   │
                    │                                                           │
  Output ──────► USB/UART send detected targets, spectrum, status              │
                    │                                                           │
  ←──── repeat ────►                                                          │
```

### Drone-Specific Processing

| Technique | Purpose | How it works |
|-----------|---------|-------------|
| **Coherent integration** | +18 to +24 dB SNR for tiny RCS | Stack N chirp FFTs while preserving phase → 2D FFT → range-Doppler map |
| **MTI** | Reject buildings, ground, trees | Subtract consecutive range profiles or high-pass filter the Doppler axis |
| **Micro-Doppler** | Distinguish drones from birds | Drone rotor blades modulate at 100–500 Hz. Spectrogram across longer CPI reveals this signature |
| **CFAR** | Adaptive target detection | Cell-averaging CFAR on range-Doppler map handles varying clutter |
| **Track-before-detect** | Find sub-threshold targets | Accumulate energy across multiple scans — a stationary noise peak won't persist |

---

## Design Decisions Explained

### 1. Open-Loop VCO Ramp (No PLL)

The VCO is driven directly by the DAC — no PLL chip. This saves ~$15 and the PCB area of a loop filter. The trade-off is you need:

1. **One-time calibration**: Measure the VCO frequency vs. voltage curve on a spectrum analyzer.
2. **Pre-distorted LUT**: The DAC outputs non-linear voltage steps that produce a *linear* frequency sweep.
3. **Temperature compensation**: The TMP102 corrects the ~0.6 MHz/°C drift by shifting the LUT baseline.

*Why this works:* For a 500 MHz sweep, a 16 MHz drift (20 °C change) is only 3.2% error — invisible for presence detection and coarse ranging.

### 2. Single IF Channel (Not I/Q)

Prototype 1 uses one mixer, one IF chain, and ADC channel A. Range + velocity are extracted from a triangular ramp. This halves the analog hardware vs. I/Q.

*Future upgrade:* Channel B of the ADC3642 + a 90° hybrid + a second mixer chain unlocks I/Q operation for unambiguous Doppler and better MTI.

### 3. 6 dB Resistive Pad After VCO

The YSGM556006 datasheet specifies **9 MHz peak-to-peak pulling at 3:1 VSWR**. Without isolation, antenna impedance variations would directly modulate the transmitted frequency.

A resistive π-pad (68 Ω + 2× 150 Ω) presents a near-perfect 50 Ω load regardless of what's downstream. The 6 dB loss is immediately recovered by the YG802020W amplifier.

### 4. GP2X+ Divider (Not Wilkinson or Resistive)

| Type | Loss per port | Isolation | Implementation |
|------|--------------|-----------|----------------|
| Resistive Y-divider | −6 dB | 6 dB | 3 resistors |
| **GP2X+ (chosen)** | **−3.6 dB** | **20 dB** | **1 IC, 3×3 mm** |
| Wilkinson on PCB | −3.2 dB | 20 dB | Requires λ/4 traces (~13 mm) |

The GP2X+ is the simplest path to a good split with guaranteed isolation.

### 5. OPA838 at ×10 (Not ×47)

The OPA838IDBVR has **300 MHz GBW** — I initially calculated with 29 MHz, which was wrong. At ×10, it has 30 MHz bandwidth (far more than we need). At ×47, it would still have 6.4 MHz.

The real bottleneck is the MCP6S91 PGA (18 MHz GBW, 562 kHz at ×32). The lower OPA838 gain (×10 vs. ×47) is a deliberate choice: it prevents the fixed stage from saturating on close-range targets, keeping more dynamic range available for the AGC to manage.

### 6. ADC3642IRSBR (Dual 14-bit, 25 MSPS)

| Alternative | Why not chosen |
|-------------|---------------|
| STM32 internal ADC (12-bit, 2.5 MSPS) | Not enough resolution or speed |
| AD9643 (14-bit, 250 MSPS, dual) | Massively overkill. 750 mW power. $40. |
| ADS4142 (14-bit, 65 MSPS, single) | Single channel — no I/Q upgrade path |
| AD9248 (14-bit, 20 MSPS, dual) | Older, higher power (150 mW), larger package |

The ADC3642 hits the sweet spot: dual channel (future I/Q), 25 MSPS (187× oversampling → +22.7 dB processing gain), ultra-low power, and a small QFN-40 package.

### 7. STM32H723VGT6 (Cortex-M7 @ 550 MHz)

The H723 has exactly the peripherals we need:

- **DCMI** (Digital Camera Interface): Designed for parallel CMOS sensor data — maps perfectly to the ADC3642's 14-bit parallel output
- **Dual SPI**: One for DAC (ramp), one for MCP6S91 + ADC config
- **Hardware FPU**: CMSIS-DSP `arm_cfft_f32` with double precision — 1024-point FFT in ~50 µs
- **320 KB SRAM**: Enough for 2,000 decimated samples + FFT buffers + detection state

### 8. 24 dBi Antennas (Separate TX/RX)

Two antennas (TX, RX) instead of a single antenna + circulator:

| Approach | Isolation | Complexity | Cost |
|----------|-----------|------------|------|
| **Separate (chosen)** | **30–40 dB** (with physical separation) | **Simple** | **2× antenna** |
| Single + circulator | 18–23 dB typical | Requires circulator at 5.75 GHz (~$15) | 1× antenna + circulator |

At 24 dBi (~5° beamwidth), the antennas need accurate pointing but give us the link budget for 3 km detection. Physical separation of 1–2 m provides adequate TX-RX isolation.

---

## Project Status

| Phase | Status |
|-------|--------|
| Architecture definition | ✅ Done |
| Component selection | ✅ Done |
| Link budget validation | ✅ Done |
| VCO calibration (SA) | ⬜ Pending |
| PCB schematic (KiCad) | ⬜ Pending |
| PCB layout | ⬜ Pending |
| Firmware: DAC ramp + drivers | ⬜ Pending |
| Firmware: FFT + detection | ⬜ Pending |
| Integration + validation | ⬜ Pending |

---

## Open Questions

- **OPA838 gain**: ×10 is the current plan. Verify with actual mixer IF levels during testing and adjust R2 (9.09 kΩ) if needed.
- **IF LPF**: Start with 33 pF (4.8 MHz). Swap to 820 pF (194 kHz) for search mode if wideband noise is a problem.
- **Antenna type**: 8×8 patch array (~30×30 cm) or parabolic dish (~40 cm). Patch is easier to fabricate, dish gives better sidelobe performance.
- **ADC3642 input range**: 2 Vpp or 3.5 Vpp — depends on MCP6S91 output swing. Configure via SPI register.
- **Decimation filter**: Start with boxcar averaging. Upgrade to FIR if frequency-domain sidelobes are a problem.
- **Future**: PA upgrade (MMG3H21NT1 for +27 dBm TX), I/Q upgrade (90° hybrid + second mixer chain), FPGA offload (Artix-7 35T for hardware DDC/FFT).

---

*Lyrion Radar — open hardware drone detection radar*
