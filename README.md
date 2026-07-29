# Lyrion Radar 🛰️

**FMCW radar at 5.5–6 GHz for counter-UAS (drone detection) and general-purpose ranging.**

[![Status](https://img.shields.io/badge/Status-Planning-yellow)]()
[![License](https://img.shields.io/badge/License-TBD-lightgrey)]()

---

## Table of Contents

- [What is Lyrion Radar?](#what-is-lyrion-radar)
- [Honest Assessment First](#honest-assessment-first)
- [FMCW Principle in 30 Seconds](#fmcw-principle-in-30-seconds)
- [Drone Detection — Realistic Targets](#drone-detection--realistic-targets)
- [System Architecture](#system-architecture)
- [Complete Component List](#complete-component-list)
- [Signal Chain Walkthrough](#signal-chain-walkthrough)
- [Power Budgets & Link Budget](#power-budgets--link-budget)
- [Chirp Configurations](#chirp-configurations)
- [Signal Processing Pipeline](#signal-processing-pipeline)
- [Design Decisions Explained](#design-decisions-explained)
- [Known Issues & Open Questions](#known-issues--open-questions)
- [Project Status](#project-status)

---

## What is Lyrion Radar?

Lyrion Radar is a **Frequency-Modulated Continuous-Wave (FMCW) radar** operating in the 5.5–6.0 GHz band. It is designed primarily for **drone detection (counter-UAS)** with secondary capability for general ranging.

The radar uses an **open-loop VCO ramp** (no PLL chip) driven by a 16-bit DAC, a **diode-ring mixer** downconverter, a **14-bit ADC** with oversampling, and an **STM32H723** Cortex-M7 processor for real-time signal processing.

### Quick Facts

| Parameter | Value |
|-----------|-------|
| Frequency band | 5.5–6.0 GHz (500 MHz sweep) |
| Range resolution | **30 cm** |
| Realistic max range (small drone) | **~500 m to 1 km** (prototype 1, noncoherent integration) |
| Realistic max range (medium drone) | **~1–1.5 km** (prototype 1) |
| Path to 2–3 km | Requires real PA + proven coherent integration + chirp rate that supports micro-Doppler |
| TX power | +11.4 dBm (prototype 1), +27 dBm (future PA upgrade) |
| Antenna gain | 24 dBi each (TX and RX, separate) |
| ADC | 14-bit, 25 MSPS (oversampled vs IF) |
| Processor | STM32H723 (Cortex-M7 @ 550 MHz) |
| Scan rate | **5–100 Hz** (not 100–1000 Hz — limited by integration window) |
| Estimated BOM cost | ~$40 (excluding antennas) |

---

## Honest Assessment First

This is a planning-stage project. Several things you read in this README are **assumptions**, not verified facts:

| Claimed | Status |
|---------|--------|
| Coherent integration of N chirps → +10·log₁₀(N) dB SNR | **Unverified.** Open-loop VCO has no chirp-to-chirp phase coherence guarantee. Real gain is likely closer to +5·log₁₀(N) (≈ +9 dB for N=64) unless residual FM/PN is measured and bounded. |
| QPL9547 NF 0.6 dB at 5.5–6 GHz | **Unverified.** Datasheet 0.6 dB is mid-band (~2 GHz). At 6 GHz the NF is higher and gain is lower. Expect **~1 dB NF, ~10 dB gain** at band edge. |
| Micro-Doppler at 100–500 Hz for drone/bird discrimination | **Wishful.** Needs chirp PRF ≳ 1 kHz to see 500 Hz Doppler. Search mode at 10 ms ramp only gives 50–100 Hz PRF. Conflict with the long-ramp / long-range mode. |
| +35 dBm EIRP at 5.5–6 GHz is legal/ISM | **Wrong.** This band includes UNII/DFS segments, and EIRP limits are country-dependent. Requires licensed or experimental authorization. |

These caveats drive most of the design constraints below. Treat all "realistic" numbers as **upper bounds** pending measurement.

---

## FMCW Principle in 30 Seconds

FMCW radar transmits a signal whose frequency sweeps linearly over time (a "chirp"). The signal reflects off a target and returns after a time delay `τ = 2R/c`. Mixing the received (delayed) signal with the current transmit signal produces a **beat frequency** directly proportional to target range:

```
f_IF = (2 · B · R) / (c · T)
```

| Symbol | Meaning | Typical value |
|--------|---------|---------------|
| B | Sweep bandwidth | 500 MHz |
| R | Target range | 1–3000 m |
| c | Speed of light | 3·10⁸ m/s |
| T | Ramp time | 1–10 ms |

An FFT of this beat signal reveals targets as distinct frequency peaks. With a **triangular ramp** (up-chirp + down-chirp), range and velocity can be extracted simultaneously.

---

## Drone Detection — Realistic Targets

Drones present a unique detection challenge — very small RCS:

| Drone class | Example | RCS (radar cross-section) | Range with prototype 1 (best case) |
|-------------|---------|--------------------------|-----------------------------------|
| Micro (< 250 g) | DJI Mini 4 | 0.001–0.005 m² (−30 to −23 dBsm) | **200–500 m** (marginal) |
| Small (250 g – 2 kg) | DJI Phantom 4 | 0.01–0.05 m² (−20 to −13 dBsm) | **~1 km** |
| Medium (2–25 kg) | DJI Matrice 300 | 0.05–0.1 m² (−13 to −10 dBsm) | **~1.5 km** |
| Large (> 25 kg) | Fixed-wing UAV | 0.1–1 m² (−10 to 0 dBsm) | > 2 km |

These ranges assume:
- Coherent integration across 64 chirps (likely achievable as noncoherent, ~+9 dB)
- 24 dBi antennas
- 5 ms or 10 ms ramp (which conflicts with micro-Doppler requirements — see below)
- Drone flying roughly head-on or head-away (Doppler is a small perturbation)

**The 2–3 km claim from earlier drafts is not supported** with the prototype 1 hardware.

### Why 5.5–6 GHz for drone detection?

- Good balance between resolution (500 MHz → 30 cm) and atmospheric attenuation
- Smaller antennas than S-band, less rain fade than X-band
- Components are widely available and affordable
- **Regulatory caveat**: this band includes UNII/DFS segments. Operation at +35 dBm EIRP requires licensed or experimental authorization in most jurisdictions.

### Key Technical Challenges

| Challenge | Status / solution |
|-----------|---------|
| Very small RCS (0.01 m²) | Integration (coherent or noncoherent), but open-loop VCO limits coherent gain |
| Drone vs. bird discrimination | **Unresolved.** Micro-Doppler (rotor blade modulation at 100–500 Hz) needs ≥1 kHz chirp PRF, which conflicts with the long ramps needed for 1+ km range |
| Stationary clutter rejection | MTI (Moving Target Indication) via range-Doppler processing |
| Free-running VCO drift | TMP102 temperature compensation + calibration LUT |
| High dynamic range | 14-bit ADC + 25 MSPS oversampling + MCP6S91 AGC |

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
       (ramp generation) │  HW FPU (single+DP)   │
                         │  564 KB SRAM          │
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
| 150 Ω + 37.4 Ω + 150 Ω resistors | — | 6 dB resistive π-pad | 0603 thin-film | **Correct values** for a matched 6 dB π-pad (50 Ω in/out). Prevents VCO frequency pulling. |

### RX Chain

| Part | Manufacturer | Role | Key Specs | Why this one |
|------|-------------|------|-----------|--------------|
| **QPL9547TR7** | Qorvo | LNA | 0.05–6 GHz, **~1 dB NF, ~10 dB gain at 5.5–6 GHz** (datasheet headline 0.6 dB / 17 dB is mid-band ~2 GHz) | Lowest NF we can get at 5.5–6 GHz. **Verify NF and gain at 5.75 GHz on VNA/noise-figure meter.** |
| **YX18** | Innotion | Diode quad mixer | GaAs Schottky, 1.4 V turn-on, cross-over quad | You have 6 on hand. GaAs gives higher breakdown than Si, allowing +12 dBm LO drive. |
| **NCS4-63+** (×2) | Mini-Circuits | Baluns (RF + LO) | 4.5–6 GHz, 1:4 impedance ratio, 0805 | You have 12 on hand. 1:4 balun provides 2× voltage step-up to cleanly switch the 1.4 V diodes. |
| **OPA838IDBVR** | Texas Instruments | IF preamplifier | **0.9 nV/√Hz**, **300 MHz GBW**, 1 mA supply | Extremely low noise for weak IF signals. 300 MHz GBW means bandwidth is never the bottleneck. |
| **MCP6S91T-E/MS** | Microchip | AGC (PGA) | 1×–32× (0–30 dB), **18 MHz GBW**, SPI | SPI-controlled gain. Bandwidth at each setting: 18 MHz / gain. At ×32: 562 kHz. |

### Digital

| Part | Manufacturer | Role | Key Specs | Why this one |
|------|-------------|------|-----------|--------------|
| **ADC3642IRSBR** | Texas Instruments | ADC | **Dual 14-bit**, **25 MSPS**, ultra-low power, QFN-40 | Oversampled vs IF. Dual channel: Ch A for IF now, Ch B for I/Q later. |
| **STM32H723VGT6** | ST | Processor | Cortex-M7 @ **550 MHz**, HW FPU, **564 KB SRAM**, DCMI | You have 3 on hand. Runs 1024-point FFT in ~50 µs. DCMI captures ADC3642 parallel output natively. |

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
VCO (+6 dBm) → 6 dB π-pad (0 dBm) → YG802020W (+15 dBm) → GP2X+ divider (+11.4 dBm each)
                                                              │
                                                              ├── TX antenna: +11.4 dBm + 24 dBi = +35.4 dBm EIRP (3.5 W)
                                                              │
                                                              └── LO to mixer: +11.4 dBm → NCS4-63+ (1:4 balun) → YX18
```

The 6 dB pad after the VCO is **critical**: without it, antenna impedance variations would shift the VCO frequency by up to 9 MHz (pulling spec). The YG802020W then restores the signal level. The GP2X+ splits it equally to TX and LO.

### RX Path — From Antenna to ADC

```
RX antenna (target echoes: −107 to −159 dBm depending on range/RCS)
    │
  QPL9547 LNA (+10 dB, NF~1 dB at 5.75 GHz) — amplifies while adding minimal noise
    │
  NCS4-63+ balun (1:4, −0.5 dB) — converts single-ended to differential for mixer
    │
  YX18 diode mixer (−7 dB conversion loss) — downconverts RF to IF beat note
    │
  RC LPF (1 kΩ + 33 pF, fc ≈ 4.8 MHz) — anti-aliasing filter
    │
  OPA838 fixed gain (×10 = +20 dB, 300 MHz GBW) — lifts signal above noise floor
    │
  MCP6S91 PGA (×1 to ×32 = 0 to +30 dB, SPI AGC) — adjusts gain per target range
    │
  ADC3642 ch A (14-bit, 25 MSPS) — digitizes with oversampling
    │
  STM32H723 (DCMI + DMA) — captures, decimates, FFT, detects
```

**Target echo levels for context** (1 km, σ=0.01 m², 24 dBi antennas, 11.4 dBm TX):
- At antenna: ~−137 dBm
- After LNA (+10 dB): ~−127 dBm
- After mixer (−7 dB): ~−134 dBm
- After OPA838 (×10): ~−114 dBm → about 56 µVpp into the ADC

### IF Gain vs. Bandwidth Trade-off

The MCP6S91's GBW (18 MHz) means gain and bandwidth trade off:

| AGC setting | Gain | Bandwidth | Use when |
|-------------|------|-----------|----------|
| ×1 | +0 dB | 18 MHz | Close targets, strong IF signal |
| ×4 | +12 dB | 4.5 MHz | Medium range, 100–500 m |
| ×8 | +18 dB | 2.25 MHz | Long range, 1 km+ |
| ×32 | +30 dB | **562 kHz** | 3 km with 10 ms ramp (f_IF = 1 MHz — **outside this BW!**) |

**The ×32 setting cannot pass the 1 MHz IF from a 3 km target with a 10 ms ramp.** This is a hard conflict — either use a longer ramp, use lower AGC gain (wider BW but less sensitivity), or accept that 3 km is out of reach.

---

## Power Budgets & Link Budget

### TX Power Budget

| Stage | Level | Notes |
|-------|-------|-------|
| VCO output | +6 dBm | YSGM556006 nominal |
| 6 dB π-pad (150/37.4/150) | −6 dB | Protects VCO from load pulling |
| YG802020W amplifier | +15 dB | 93 mA current, 1 dB below P1dB |
| GP2X+ divider | −3.6 dB | Typical total loss at 5.5 GHz |
| **Each port output** | **+11.4 dBm** | TX to antenna, LO to mixer |
| With 24 dBi antenna (TX) | **+35.4 dBm EIRP** (3.5 W ERP) | **Not ISM-legal without authorization** |

### RX Link Budget — Drone Detection

Conditions: Pt = +11.4 dBm, Gt = Gr = 24 dBi, NF = 1 dB (QPL9547 at band edge), noise floor = −143 dBm (1 kHz bin).

**Critical: radar equation uses R⁴ (not R²). From 1 km to 500 m you gain 12 dB, not 20 dB.**

| Range | σ (m²) | Target | Pr (dBm) | SNR (single chirp) | SNR (64-chirp noncoherent) | SNR (64-chirp coherent, if achievable) |
|-------|--------|--------|----------|--------------------|----------------------------|--------------------------------------|
| 500 m | 0.1 | Medium drone | −117.2 | 26 dB | 35 dB ✅ | 44 dB ✅ |
| 500 m | 0.01 | Small drone | **−127.2** | **16 dB** | **25 dB** ✅ | **34 dB** ✅ |
| 1 km | 0.1 | Medium drone | −129.2 | 14 dB | 23 dB ✅ | 32 dB ✅ |
| 1 km | 0.01 | Small drone | **−139.2** | **4 dB** | **13 dB** ⚠️ | **22 dB** ✅ |
| 1.5 km | 0.1 | Medium drone | −136.2 | 7 dB | 16 dB ✅ | 25 dB ✅ |
| 1.5 km | 0.01 | Small drone | −146.2 | −3 dB | 6 dB ⚠️ | 15 dB ✅ |
| 2 km | 0.1 | Medium drone | −141.2 | 2 dB | 11 dB ⚠️ | 20 dB ✅ |
| 2 km | 0.01 | Small drone | −151.2 | −8 dB | 1 dB ❌ | 10 dB ⚠️ |
| 3 km | 0.1 | Medium drone | −148.7 | −6 dB | 3 dB ❌ | 12 dB ⚠️ |

**Key insight: coherent vs noncoherent matters enormously.**

- **Noncoherent** integration: gain ≈ 5·log₁₀(N) = +9 dB for N=64. Realistic for open-loop VCO.
- **Coherent** integration: gain = 10·log₁₀(N) = +18 dB for N=64. **Requires chirp-to-chirp phase coherence** — unproven with this design.

**Honest target for prototype 1:** ~500 m to 1 km for small drones, ~1.5 km for medium drones — with the caveat that this assumes noncoherent integration. Achieving 2–3 km requires either (a) proven coherent integration with a stable VCO, or (b) a real PA (MMG3H21NT1, +27 dBm TX → +15.6 dB improvement).

---

## Chirp Configurations

The radar supports **three operating modes** configurable in firmware.

### IF frequency as a function of range and ramp time

**f_IF = 2BR / (cT) — verify with this formula every time you change a parameter.**

| Range | T=1 ms | T=5 ms | T=10 ms |
|-------|--------|--------|---------|
| 100 m | 333 kHz | 66.7 kHz | 33.3 kHz |
| 500 m | 1.67 MHz | 333 kHz | 167 kHz |
| 1 km | **3.33 MHz** | 667 kHz | 333 kHz |
| 1.5 km | 5 MHz | 1 MHz | 500 kHz |
| 2 km | 6.67 MHz | 1.33 MHz | 667 kHz |
| 3 km | 10 MHz | **2 MHz** | **1 MHz** |

### Configurations

| Mode | Ramp T | LPF fc | Max range | f_IF @ 3 km | Samples (25 MSPS) | FFT | Scan rate | Doppler range |
|------|--------|--------|-----------|-------------|-------------------|-----|-----------|---------------|
| **Search** | 10 ms | 500 kHz | 1.5 km | 500 kHz | 250,000 → decimate to 5,000 | 4096-pt | **5–10 Hz** (after 64-chirp integration) | ±0.6 m/s (ambiguous above) |
| **Track** | 5 ms | 1 MHz | 1.5 km | 1 MHz | 125,000 → decimate to 2,500 | 2048-pt | 10–20 Hz | ±1.25 m/s |
| **Fast** | 1 ms | 5 MHz | 500 m | (out of range) | 25,000 | 512-pt | 50–100 Hz | ±6.25 m/s |

**Important constraints:**

- **Scan rate vs integration**: 64-chirp coherent integration at 10 ms ramp = 0.64 s per integrated frame → ~1.5 Hz. Noncoherent (no phase preservation) lets you trade integration for scan rate.
- **Micro-Doppler vs long ramp**: Rotor modulation at 100–500 Hz needs chirp PRF ≳ 1 kHz. Search mode at 10 ms only gives 50–100 Hz PRF. **Micro-Doppler drone/bird discrimination is not feasible in Search or Track modes.** It would require a separate high-PRF "discriminator" mode at short range.
- **MCP6S91 bandwidth**: At ×32 (562 kHz BW), the IF for a 1.5 km target with 10 ms ramp is 500 kHz — fits. At 3 km with 10 ms ramp, IF is 1 MHz — **does not fit** at ×32. Reduce AGC gain or accept 3 km is out of reach.
- **Memory**: 250K samples × 2 bytes = 500 KB — exceeds 564 KB SRAM if you try to store full ramp. **Decimate on-the-fly** in DMA ISR (accumulate + dump every Nth sample) → only 5,000 samples stored per ramp.

### Velocity (Doppler) with Triangular Ramp

```
Up-chirp:   f_IF = f_range − f_doppler
Down-chirp: f_IF = f_range + f_doppler

→ f_range   = (f_up + f_down) / 2
→ f_doppler = (f_down − f_up) / 2
→ velocity  = λ · f_doppler / 2
```

**v_max ≈ λ / (4T)** — one of the most commonly miscalculated numbers in FMCW:

| Ramp time | Max unambiguous velocity | Drones covered |
|-----------|--------------------------|----------------|
| 10 ms | **0.625 m/s** | Hovering only |
| 5 ms | 1.25 m/s | Slow drift (< 4.5 km/h) |
| 1 ms | 6.25 m/s | Hovering to 22 km/h |

**Critical:** Small drones routinely fly 5–20 m/s. At 10 ms ramp, the maximum unambiguous velocity is 0.625 m/s — a drone flying at 5 m/s will alias and appear as a false stationary or slow target. **Search mode at 10 ms is not usable for moving drones.** Use Fast mode (1 ms, 6.25 m/s) for detection, then track.

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
  Decimation ───► On-the-fly DMA ISR: boxcar-average to 200 kSPS               │
                  (25 MSPS → 200 kSPS, reduces data from 500 KB → 4 KB)       │
                    │                                                           │
  Range FFT ────► CMSIS-DSP real FFT (1024–4096 points, single-precision)    │
                    │                                                           │
  Up/down pair ─► Pair consecutive up-chirp and down-chirp for velocity        │
                    │                                                           │
  Integration ──► Stack 64 chirps (coherent if VCO phase allows,              │
                  noncoherent otherwise)                                       │
                    │                                                           │
  MTI ──────────► Subtract consecutive range profiles to reject clutter         │
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

| Technique | Purpose | Status |
|-----------|---------|--------|
| **Coherent integration** (64 chirps) | +18 dB SNR | **Unverified** with open-loop VCO. Noncoherent (~+9 dB) is the safe assumption. |
| **MTI** | Reject stationary clutter | Implemented as range-Doppler subtraction |
| **Micro-Doppler** (rotor blade at 100–500 Hz) | Drone vs. bird discrimination | **Not feasible in Search/Track modes** (PRF too low). Would need a separate high-PRF discriminator mode. |
| **CFAR** | Adaptive detection in clutter | Cell-averaging CFAR on range-Doppler map |
| **Velocity aliasing handling** | Drones fly 5–20 m/s | Detect aliases in the up/down chirp pair and unwrap |

---

## Design Decisions Explained

### 1. Open-Loop VCO Ramp (No PLL)

The VCO is driven directly by the DAC — no PLL chip. This saves ~$15 and the PCB area of a loop filter. The trade-offs:

- **Calibration required**: Measure the VCO frequency vs. voltage curve on a spectrum analyzer.
- **Pre-distorted LUT**: The DAC outputs non-linear voltage steps that produce a *linear* frequency sweep.
- **Temperature compensation**: The TMP102 corrects the ~0.6 MHz/°C drift by shifting the LUT baseline.
- **No phase coherence**: Chirp-to-chirp phase is random. Coherent integration is reduced to noncoherent (~+9 dB for 64 chirps vs. the theoretical +18 dB).
- **Phase noise**: Free-running VCO is worse than PLL-locked. May limit detection of small targets at long range.

*Why this works for prototype 1:* Validation of the RF chain, IF chain, and ADC pipeline. If phase coherence becomes a problem, the upgrade path is a real PLL (ADF4158, LMX2492, or HMC703).

### 2. Single IF Channel (Not I/Q)

Prototype 1 uses one mixer, one IF chain, and ADC channel A. Range + velocity are extracted from a triangular ramp. This halves the analog hardware vs. I/Q.

*Future upgrade:* Channel B of the ADC3642 + a 90° hybrid + a second mixer chain unlocks I/Q operation for unambiguous Doppler and better MTI.

### 3. 6 dB Resistive π-Pad After VCO — Corrected Values

The YSGM556006 datasheet specifies **9 MHz peak-to-peak pulling at 3:1 VSWR**. Without isolation, antenna impedance variations would directly modulate the transmitted frequency.

A resistive **π-pad** (150 Ω shunt / **37.4 Ω series** / 150 Ω shunt) presents a near-perfect 50 Ω load regardless of what's downstream. The 6 dB loss is immediately recovered by the YG802020W amplifier.

> **Earlier draft had 68 Ω series — that gives the wrong attenuation and impedance match.** Corrected here.

### 4. GP2X+ Divider (Not Wilkinson or Resistive)

| Type | Loss per port | Isolation | Implementation |
|------|--------------|-----------|----------------|
| Resistive Y-divider | −6 dB | 6 dB | 3 resistors |
| **GP2X+ (chosen)** | **−3.6 dB** | **20 dB** | **1 IC, 3×3 mm** |
| Wilkinson on PCB | −3.2 dB | 20 dB | Requires λ/4 traces (~13 mm) |

The GP2X+ is the simplest path to a good split with guaranteed isolation.

### 5. OPA838 at ×10 (Not ×47)

The OPA838IDBVR has **300 MHz GBW**. At ×10, it has 30 MHz bandwidth (far more than we need). At ×47, it would still have 6.4 MHz.

The real bottleneck is the MCP6S91 PGA (18 MHz GBW, 562 kHz at ×32). The lower OPA838 gain (×10 vs. ×47) is a deliberate choice: it prevents the fixed stage from saturating on close-range targets, keeping more dynamic range available for the AGC to manage.

### 6. ADC3642IRSBR (Dual 14-bit, 25 MSPS)

| Alternative | Why not chosen |
|-------------|---------------|
| STM32 internal ADC (12-bit, 2.5 MSPS) | Not enough resolution or speed |
| AD9643 (14-bit, 250 MSPS, dual) | Massively overkill. 750 mW power. $40. |
| ADS4142 (14-bit, 65 MSPS, single) | Single channel — no I/Q upgrade path |
| AD9248 (14-bit, 20 MSPS, dual) | Older, higher power (150 mW), larger package |

The ADC3642 hits the sweet spot: dual channel (future I/Q), 25 MSPS, ultra-low power, and a small QFN-40 package.

> **Note on oversampling gain:** I earlier claimed "187× oversampling → +22.7 dB processing gain → ~17.8 effective bits." This is **theoretically correct for thermal noise**, but the effective number of bits you actually get depends on ADC linearity, clock jitter, and how much of that gain survives decimation. Treat the effective-bits claim as unverified.

### 7. STM32H723VGT6 (Cortex-M7 @ 550 MHz)

The H723 has exactly the peripherals we need:

- **DCMI** (Digital Camera Interface): Designed for parallel CMOS sensor data — maps to the ADC3642's 14-bit parallel output. **25 MHz / 14-bit is in spec, but framing (HSYNC/VSYNC/embedded sync) needs careful handling** — not plug-and-play.
- **Dual SPI**: One for DAC (ramp), one for MCP6S91 + ADC config.
- **Single-precision FPU**: CMSIS-DSP `arm_cfft_f32` runs 1024-point FFT in ~50 µs.
- **564 KB SRAM**: Enough for 5,000 decimated samples + FFT buffers + detection state.

> **Earlier draft said 320 KB SRAM. Corrected to 564 KB.**

### 8. 24 dBi Antennas (Separate TX/RX)

Two antennas (TX, RX) instead of a single antenna + circulator:

| Approach | Isolation | Complexity | Cost |
|----------|-----------|------------|------|
| **Separate (chosen)** | **30–40 dB** (with physical separation) | **Simple** | **2× antenna** |
| Single + circulator | 18–23 dB typical | Requires circulator at 5.75 GHz (~$15) | 1× antenna + circulator |

At 24 dBi (~5° beamwidth), the antennas need accurate pointing but give us the link budget for ~1 km small-drone detection. Physical separation of 1–2 m provides adequate TX-RX isolation.

---

## Known Issues & Open Questions

### Known issues (must be addressed)

| Issue | Impact | Mitigation |
|-------|--------|-----------|
| **f_IF at 3 km = 1 MHz** with 10 ms ramp | Exceeds MCP6S91 ×32 BW (562 kHz) | Use 1.5 km max range in Search mode, or accept lower AGC gain |
| **v_max = 0.625 m/s** at 10 ms ramp | Aliases any moving drone | Use Fast mode (1 ms) for detection; triangular pairing has the same limit |
| **Micro-Doppler at 100–500 Hz** vs **chirp PRF 50–100 Hz** | Drone/bird discriminator infeasible in Search/Track | Add a separate high-PRF mode (future); accept that you can't classify drone vs. bird at 1+ km |
| **Open-loop VCO phase coherence unproven** | Coherent integration may be noncoherent (~+9 dB instead of +18 dB) | Measure residual FM/PN before relying on long-range detection; consider ADF4158 PLL for rev 2 |
| **QPL9547 NF/gain at 5.75 GHz unverified** | System NF may be 1.5–2 dB, not 1 dB | Measure on noise-figure meter; add 2 dB margin to all range claims |
| **+35 dBm EIRP at 5.5–6 GHz is not ISM** | Regulatory issue | Acquire experimental/STA license; or reduce EIRP and accept shorter range |
| **FR4 at 5.5–6 GHz** | Higher insertion loss, εr tolerance, dispersion | Acceptable for prototype; use Rogers/RF-35 for production if last dB matters |
| **Decimation aliasing** | 25 MSPS → 200 kSPS must have anti-alias filter before decimation | The 4.8 MHz RC LPF + AGC bandwidth limits handle this — verify with spectrum |

### Open questions (resolve during implementation)

- **QPL9547 NF/gain at 5.75 GHz**: measure on VNA + NF meter
- **OPA838 gain**: ×10 is the plan. Verify with actual mixer IF levels.
- **IF LPF**: start with 33 pF (4.8 MHz). May need adjustment for decimation anti-aliasing.
- **Antenna type**: 8×8 patch array (~30×30 cm) or parabolic dish (~40 cm)
- **ADC3642 input range**: 2 Vpp or 3.5 Vpp — configure via SPI
- **Decimation filter**: boxcar vs. FIR
- **Future**: PA upgrade (MMG3H21NT1, +27 dBm TX), I/Q upgrade, FPGA offload, PLL for coherent integration

---

## Project Status

| Phase | Status |
|-------|--------|
| Architecture definition | ✅ Done (with caveats noted above) |
| Component selection | ✅ Done |
| Link budget validation | ✅ Done — **realistic targets 0.5–1.5 km** |
| VCO calibration (SA) | ⬜ Pending |
| PCB schematic (KiCad) | ⬜ Pending |
| PCB layout | ⬜ Pending |
| Firmware: DAC ramp + drivers | ⬜ Pending |
| Firmware: FFT + detection | ⬜ Pending |
| Integration + validation | ⬜ Pending |

---

*Lyrion Radar — open hardware drone detection radar. Range claims are honest.*
