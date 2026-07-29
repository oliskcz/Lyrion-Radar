# Lyrion Radar 🛰️

**FMCW radar at 5.5–6 GHz for counter-UAS (drone detection) — PLL-based chirp + FPGA-based DSP.**

[![Status](https://img.shields.io/badge/Status-Planning-yellow)]()
[![License](https://img.shields.io/badge/License-TBD-lightgrey)]()

---

## Table of Contents

- [What is Lyrion Radar?](#what-is-lyrion-radar)
- [Why PLL + FPGA?](#why-pll--fpga)
- [Honest Assessment](#honest-assessment)
- [FMCW Principle in 30 Seconds](#fmcw-principle-in-30-seconds)
- [Drone Detection — Realistic Targets](#drone-detection--realistic-targets)
- [System Architecture](#system-architecture)
- [FPGA DSP Pipeline](#fpga-dsp-pipeline)
- [Complete Component List](#complete-component-list)
- [Signal Chain Walkthrough](#signal-chain-walkthrough)
- [Power Budgets & Link Budget](#power-budgets--link-budget)
- [Chirp Configurations](#chirp-configurations)
- [Design Decisions Explained](#design-decisions-explained)
- [Known Issues & Open Questions](#known-issues--open-questions)
- [Project Status](#project-status)

---

## What is Lyrion Radar?

Lyrion Radar is a **Frequency-Modulated Continuous-Wave (FMCW) radar** operating in the 5.5–6.0 GHz band, designed primarily for **drone detection (counter-UAS)** with secondary capability for general ranging.

The signal chain uses an **ADF41510 fractional-N PLL** with built-in ramp generator (the YSGM556006 is the VCO in the PLL loop), a **diode-ring mixer** for downconversion, a **dual 14-bit 250 MSPS ADC** for IF digitization, and a **Xilinx Artix-7 XC7A100T FPGA** for all real-time signal processing. A small **STM32C031 MCU** handles housekeeping (PLL config, AGC, temperature, USB).

### Quick Facts

| Parameter | Value |
|-----------|-------|
| Frequency band | 5.5–6.0 GHz (500 MHz sweep) |
| Range resolution | **30 cm** |
| Realistic max range (small drone) | **~2 km** (coherent integration, no PA) |
| Realistic max range (medium drone) | **~3 km** (coherent integration, no PA) |
| Path to 5+ km | PA upgrade (MMG3H21NT1) |
| TX power | +11.4 dBm (prototype 1) |
| Antenna gain | 24 dBi each (TX and RX, separate) |
| Chirp source | **ADF41510 PLL** (built-in ramp gen) |
| ADC | **AD9643 dual 14-bit, 250 MSPS** (LVDS to FPGA) |
| DSP | **Xilinx XC7A100T** Artix-7 (101K logic cells, 240 DSP slices) |
| Control MCU | STM32C031 (PLL config, AGC, temp, USB) |
| Scan rate | 100 Hz to 10 kHz (configurable) |
| Estimated BOM cost | ~$250 (Arty A7-100T + parts) |

---

## Why PLL + FPGA?

For serious drone detection, you need three things: **phase coherence**, **micro-Doppler processing**, and **low phase noise**. Only a PLL + FPGA architecture delivers all three.

| Requirement | Open-loop DAC + MCU | **ADF41510 PLL + FPGA** |
|-------------|--------------------|-----------------------|
| Phase noise @ 100 kHz | ~−80 dBc/Hz (free VCO) | **−231 dBc/Hz** (fractional-N) |
| Chirp-to-chirp coherence | Unproven (likely noncoherent) | **Proven** (PLL locks to TCXO) |
| Coherent integration gain | +9 dB (noncoherent) | **+18 dB** (coherent) |
| Frequency linearity | Needs LUT pre-distortion | **Built-in ramp generator** |
| Temperature stability | Needs TMP102 + compensation | **Eliminated** (PLL locks) |
| Micro-Doppler (100–500 Hz) | Infeasible (PRF too low) | **Trivial** (10 kHz PRF mode) |
| 2D range-Doppler | Slow (software) | **< 100 µs** (hardware FFT) |
| I/Q channels | Future (ch B) | **Available now** (ch B) |
| Range impact (small drone) | 1 km | **2–3 km** (without PA) |

The ADF41510's built-in ramp generator is specifically designed for FMCW radar — it produces highly linear sawtooth or triangular chirps with configurable rate, step size, and dwell. The YSGM556006 (which you have 10 of) becomes the VCO in the PLL loop, so you're not wasting any parts.

---

## Honest Assessment

| Claimed | Status |
|---------|--------|
| +18 dB coherent integration gain (64 chirps) | **Achievable.** PLL + FPGA phase tracking enables this. |
| Phase noise −231 dBc/Hz | **Datasheet spec.** Real-world will be slightly worse (PLL loop filter, reference noise) but still > 50 dB better than free-running VCO. |
| Micro-Doppler drone/bird discrimination | **Feasible.** 0.1 ms ramps at 10 kHz PRF in FPGA. |
| 2–3 km small drone detection | **Plausible.** With coherent integration (+18 dB) and 24 dBi antennas, the link budget supports this. |
| +35 dBm EIRP at 5.5–6 GHz is legal/ISM | **Wrong.** UNII/DFS bands. Needs licensed or experimental authorization. |

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

| Drone class | Example | RCS (σ) | Range with prototype 1 |
|-------------|---------|---------|------------------------|
| Micro (< 250 g) | DJI Mini 4 | 0.001–0.005 m² (−30 to −23 dBsm) | **~500 m** (marginal) |
| Small (250 g – 2 kg) | DJI Phantom 4 | 0.01–0.05 m² (−20 to −13 dBsm) | **~2 km** |
| Medium (2–25 kg) | DJI Matrice 300 | 0.05–0.1 m² (−13 to −10 dBsm) | **~3 km** |
| Large (> 25 kg) | Fixed-wing UAV | 0.1–1 m² (−10 to 0 dBsm) | > 3 km |

These ranges assume:
- 64-chirp coherent integration (PLL + FPGA enables this)
- 24 dBi antennas
- 5–10 ms ramp
- Drone flying roughly head-on or head-away

### Why 5.5–6 GHz?

- Good balance between resolution (500 MHz → 30 cm) and atmospheric attenuation
- Smaller antennas than S-band, less rain fade than X-band
- Components are widely available and affordable
- **Regulatory caveat**: UNII/DFS segments. +35 dBm EIRP requires licensed or experimental authorization in most jurisdictions.

---

## System Architecture

### High-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CHIRP SOURCE (PLL)                                │
│                                                                          │
│  TCXO (100 MHz) ──► ADF41510 ──► Loop Filter ──► YSGM556006 VCO       │
│                     (fractional-N,  (active,         (VCO in PLL)        │
│                      built-in ramp   100 kHz BW)                        │
│                      generator)              │                           │
│  STM32C031 ──SPI──► ADF41510 config          │                           │
│  (RAMP_START, freq, step, time)              │                           │
│  TMP102 (I²C) ──► temp monitoring             │                           │
└─────────────────────────────────────────────────┼────────────────────────┘
                                                 │
                                            5.5–6 GHz OUT
                                           +6 dBm (locked)
                                                 │
┌─────────────────────────────────────────────────────────────────────────┐
│                            TX CHAIN                                      │
│                                                                          │
│                                     ┌──► TX antenna (24 dBi)            │
│  VCO ──► 6 dB pad ──► YG802020W ──┤   +11.4 dBm → +35.4 dBm EIRP       │
│  +6 dBm   (resistive)  (+15 dB)    │    (3.5 W ERP)                     │
│             0 dBm       +15 dBm    └──► LO to mixer                     │
│                                       +11.4 dBm                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                            RX CHAIN                                      │
│                                                                          │
│  RX antenna ──► QPL9547 ──► NCS4-63+ ──► YX18 ──► RC LPF ──► OPA838   │
│  (24 dBi)       (LNA)       (balun)     (mixer)  (4.8 MHz)   (×10)      │
│                 NF~1dB              ─────┤                             │
│                 G~10dB               LO from GP2X+                     │
│                                                                          │
│  OPA838 ──► MCP6S91 ──► AD9643 ch A ─────► XC7A100T FPGA               │
│  (×10)      (PGA 1-32×)  (14-bit          (DDC → decimate → 2D FFT →   │
│             SPI AGC       250 MSPS, LVDS)  MTI → CFAR → detection)      │
│                                                                          │
│  AD9643 ch B ◄──── future I/Q or 2nd RX antenna (channel ready)        │
│                                                                          │
│  XC7A100T ──► USB/UART ──► PC (range-Doppler, detected targets)         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Chirp Source — ADF41510 PLL Detail

```
                          ┌─────────────────────────┐
                          │      ADF41510            │
                          │   (fractional-N PLL)    │
  TCXO 100 MHz ──────────►│  REF_IN                  │
                          │                         │
  STM32C031 SPI ────────►│  LE, CLK, DATA          │
  (ramp parameters)       │  (ramp start, stop,     │
                          │   step, time, mode)     │
                          │                         │
                          │  Phase frequency        │
                          │  detector (PFD) ────────┼──► Loop filter
                          │                         │     (active op-amp)
                          │  Charge pump (CP) ──────┘            │
                          │                                     ▼
                          │                            YSGM556006 VCO
                          │                            (tuning pin VT)
                          │                                    │
                          │  RF input ◄────────────────────────┘
                          │  (prescaler ÷M, 1-10 GHz)
                          └─────────────────────────┘
                                      │
                                      ▼
                              5.5–6 GHz OUT
                          (phase-locked, low noise)
```

The ADF41510's built-in ramp generator produces the chirp directly. The YSGM556006 (which you have 10 of) is the VCO in the PLL loop. The STM32C031 configures the ramp parameters via SPI.

### FPGA and MCU Division of Labor

```
                    ┌────────────────────────────────────┐
                    │  Xilinx XC7A100T (XC7A100TCSG324)  │
                    │                                    │
  AD9643 ch A (LVDS)─►│ ISERDES (8:1 DDR deserialization) │
                    │                                    │
  AD9643 ch B (LVDS)─►│ ISERDES (future I/Q or 2nd ant)   │
                    │                                    │
                    │  DDC (CORDIC / mixer)               │
                    │  Decimation CIC/FIR                 │
                    │  Window function (Hann/Blackman)   │
                    │  FFT (Xilinx FFT IP, 1024-4096 pt) │
                    │  2D FFT (range-Doppler)            │
                    │  MTI (clutter rejection)           │
                    │  CFAR detection                    │
                    │  Micro-Doppler (high-PRF mode)     │
                    │  Target tracking (Kalman)          │
                    │                                    │
                    │  UART/USB ──► PC (data + config)   │
                    └────────────────┬───────────────────┘
                                     │
                    ┌────────────────┴───────────────────┐
                    │  STM32C031 (housekeeping MCU)      │
                    │                                    │
                    │  SPI1 ──► ADF41510 (PLL config)   │
                    │  SPI2 ──► MCP6S91 (AGC)            │
                    │  I2C1 ──► TMP102 (temp)            │
                    │  GPIO ──► RAMP_START, mode select │
                    │  UART ──► debug / config           │
                    └────────────────────────────────────┘
```

The MCU handles low-speed configuration. The FPGA handles all real-time DSP at 250 MSPS.

---

## FPGA DSP Pipeline

The FPGA processes each chirp through this pipeline (one chirp = 1 ms, parallelized):

```
                ┌─────────────────────────────────────────────────────────────┐
                │                   ONE CHIRP PROCESSING                        │
                │                                                              │
  AD9643 ch A ─► ISERDES: 250 MSPS × 14-bit LVDS → 8-bit @ 31.25 MHz words │
                       │                                                        │
                       ▼                                                        │
  DDC: complex mixer ─► CORDIC (mix with NCO) ─► decimate 4× → 62.5 MSPS  │
                       │                                                        │
                       ▼                                                        │
  Decimate: CIC (4×) + FIR (4×) ─► 3.9 MSPS (filter, anti-alias)          │
                       │                                                        │
                       ▼                                                        │
  Window: Hann/Blackman (multiply sample-by-sample)                        │
                       │                                                        │
                       ▼                                                        │
  FFT: 1024-pt complex, streaming pipelined ─► |X[k]|² magnitudes         │
                       │                                                        │
                       ▼                                                        │
  Buffer: store 64 range profiles in DDR (64 × 1024 × 4 bytes = 256 KB)    │
                       │                                                        │
                       ▼                                                        │
  2nd FFT: across 64 chirps at each range bin ─► Doppler spectrum          │
                       │                                                        │
                       ▼                                                        │
  MTI: subtract consecutive Doppler maps ─► reject stationary clutter     │
                       │                                                        │
                       ▼                                                        │
  CFAR: cell-averaging CA-CFAR on range-Doppler map                        │
                       │                                                        │
                       ▼                                                        │
  Detection: peak find → range, velocity, amplitude, micro-Doppler feat.   │
                       │                                                        │
                       ▼                                                        │
  Tracking: Kalman filter across frames → track ID, trajectory              │
                       │                                                        │
                       ▼                                                        │
  Output: UART/USB → list of detected targets with metadata               │
                └─────────────────────────────────────────────────────────────┘
```

### FPGA Resource Estimate (XC7A100T)

| Block | LUTs | DSP48 | BRAM (KB) |
|-------|------|-------|-----------|
| ISERDES + clocking | 2,000 | 0 | 0 |
| DDC (CORDIC + NCO) | 1,500 | 16 | 4 |
| Decimation CIC+FIR | 1,000 | 24 | 32 |
| Window function | 500 | 4 | 0 |
| FFT 1024-pt (1 channel) | 3,000 | 20 | 32 |
| 2D FFT buffer (DDR ctrl) | 1,500 | 0 | 256 (DDR) |
| MTI / CFAR | 2,000 | 8 | 64 |
| Tracking | 1,000 | 4 | 8 |
| **Total** | **~12,500 (20%)** | **~76 (32%)** | **~132 (3% of 4,860 KB)** |

The XC7A100T easily handles two channels (I/Q) simultaneously with headroom for future expansion.

---

## Complete Component List

### Chirp Source (PLL)

| Part | Manufacturer | Role | Key Specs | Notes |
|------|-------------|------|-----------|-------|
| **YSGM556006** | Innotion | VCO (in PLL loop) | 5320–6060 MHz, +6 dBm, 0–5 V tune | You have 10 on hand — use one as the PLL VCO |
| **ADF41510** | Analog Devices | Fractional-N PLL | 1–10 GHz, −231 dBc/Hz PN, 250 MHz PFD, built-in ramp gen | **Key new component** — replaces DAC8830 + REF5050A |
| **ECS-TXO-5032-100MHz** (or similar) | ECS Inc | TCXO reference | 100 MHz, ±1 ppm, low phase noise | PLL reference input |
| **Loop filter components** | — | Active op-amp + RC | 100 kHz BW, 50° phase margin | ADF41510 reference design |

### TX Chain

| Part | Manufacturer | Role | Key Specs |
|------|-------------|------|-----------|
| **YG802020W** | Innotion | TX driver | 50 MHz–8 GHz, +15 dB gain, P1dB ~+16 dBm @ 5.5 GHz |
| **GP2X+** | Mini-Circuits | 2-way power divider | 2.9–6.2 GHz, 3.6 dB total loss, 20 dB isolation |
| 150 Ω + **37.4 Ω** + 150 Ω | — | 6 dB π-pad (corrected) | 0603 thin-film |

### RX Chain

| Part | Manufacturer | Role | Key Specs |
|------|-------------|------|-----------|
| **QPL9547TR7** | Qorvo | LNA | ~1 dB NF, ~10 dB gain at 5.75 GHz (verify) |
| **YX18** | Innotion | Diode quad mixer | GaAs Schottky, 1.4 V turn-on |
| **NCS4-63+** (×2) | Mini-Circuits | Baluns | 4.5–6 GHz, 1:4 ratio |
| **OPA838IDBVR** | TI | IF preamplifier | 0.9 nV/√Hz, 300 MHz GBW |
| **MCP6S91T-E/MS** | Microchip | AGC (PGA) | 1×–32× (0–30 dB), 18 MHz GBW, SPI |

### Digital / DSP

| Part | Manufacturer | Role | Key Specs |
|------|-------------|------|-----------|
| **AD9643BCPZ-250** | ADI | ADC | Dual 14-bit, 250 MSPS, LVDS, QFN-48 |
| **XC7A100T-CSG324** | Xilinx | FPGA | Artix-7, 101K logic cells, 240 DSP slices, 4,860 Kb BRAM |
| **MT41K256M16TW-107** | Micron | DDR3L SDRAM | 256M × 16 = 512 MB |
| **STM32C031C6T6** | ST | Housekeeping MCU | Cortex-M0+ @ 48 MHz, 32 KB flash, 12 KB SRAM |

### Power

| Part | Manufacturer | Role | Key Specs |
|------|-------------|------|-----------|
| **ADP150AUJZ-5.0** | ADI | Ultra-low noise LDO (5 V) | VCO supply |
| **LTM4644** | ADI | FPGA power | Multi-rail DC/DC (1.0V, 1.8V, 3.3V) |
| **LD1117S33** | ST | 3.3 V LDO | MCU, ADC digital |

### Removed components (replaced by PLL)

| Removed | Reason |
|---------|--------|
| ~~DAC8830IDR~~ | PLL generates the ramp |
| ~~REF5050AIDGKR~~ | PLL uses TCXO reference |
| ~~TLV9062IDR~~ | No DAC output to buffer |
| ~~TMP102AIDRLR~~ (optional) | PLL eliminates temperature drift (but still useful for monitoring) |

### Development Options

| Option | Cost | Notes |
|--------|------|-------|
| **Digilent Arty A7-100T** | ~$200 | Ready-to-use XC7A100T board with DDR3, USB |
| Custom PCB with XC7A100T | ~$50 (chip) + $100 (PCB) | Production target |
| ADF41510 eval board | ~$100 | ADI EVAL-ADF41510 |

---

## Signal Chain Walkthrough

### TX Path

```
YSGM556006 VCO (PLL-locked, +6 dBm)
    │
  6 dB π-pad (150/37.4/150) → 0 dBm
    │
  YG802020W (+15 dB) → +15 dBm
    │
  GP2X+ divider (−3.6 dB) → +11.4 dBm each
    │
    ├── TX antenna (24 dBi) → +35.4 dBm EIRP (3.5 W)
    │
    └── LO to mixer (+11.4 dBm)
```

### RX Path

```
RX antenna (echoes: −107 to −159 dBm)
    │
  QPL9547 LNA (+10 dB, NF~1 dB)
    │
  NCS4-63+ balun (1:4, −0.5 dB)
    │
  YX18 mixer (−7 dB) → IF beat note
    │
  RC LPF (1 kΩ + 33 pF, 4.8 MHz)
    │
  OPA838 (×10 = +20 dB, 300 MHz GBW)
    │
  MCP6S91 AGC (×1 to ×32, SPI)
    │
  AD9643 ch A (14-bit, 250 MSPS, LVDS)
    │
  XC7A100T ISERDES → DDC → decimate → window → FFT → 2D FFT → MTI → CFAR
```

### IF Gain vs. Bandwidth (MCP6S91 AGC)

| AGC setting | Gain | BW | Total gain | Use case |
|-------------|------|-----|-----------|----------|
| ×1 | +0 dB | 18 MHz | 20 dB | Close targets |
| ×4 | +12 dB | 4.5 MHz | 32 dB | Medium range |
| ×8 | +18 dB | 2.25 MHz | 38 dB | 1 km+ |
| ×32 | +30 dB | 562 kHz | 50 dB | Long range |

---

## Power Budgets & Link Budget

### TX Power Budget

| Stage | Level | Notes |
|-------|-------|-------|
| PLL-locked VCO | +6 dBm | YSGM556006 in PLL loop |
| 6 dB π-pad | −6 dB | |
| YG802020W | +15 dB | 1 dB below P1dB |
| GP2X+ | −3.6 dB | |
| **Each port** | **+11.4 dBm** | |
| With 24 dBi TX | **+35.4 dBm EIRP** (3.5 W) | Not ISM |

### RX Link Budget — Drone Detection (with coherent integration)

Conditions: Pt = +11.4 dBm, Gt = Gr = 24 dBi, NF = 1 dB, noise floor = −143 dBm (1 kHz bin).

**Key advantage:** PLL + FPGA enables true coherent integration → +18 dB for 64 chirps (vs +9 dB noncoherent on MCU approach).

| Range | σ (m²) | Target | Pr (dBm) | SNR (1 chirp) | SNR (64-chirp coherent) |
|-------|--------|--------|----------|----------------|------------------------|
| 500 m | 0.1 | Medium drone | −117.2 | 26 dB | **44 dB** ✅ |
| 500 m | 0.01 | Small drone | −127.2 | 16 dB | **34 dB** ✅ |
| 1 km | 0.1 | Medium drone | −129.2 | 14 dB | **32 dB** ✅ |
| 1 km | 0.01 | Small drone | −139.2 | 4 dB | **22 dB** ✅ |
| 1.5 km | 0.1 | Medium drone | −136.2 | 7 dB | **25 dB** ✅ |
| 1.5 km | 0.01 | Small drone | −146.2 | −3 dB | **15 dB** ✅ |
| 2 km | 0.1 | Medium drone | −141.2 | 2 dB | **20 dB** ✅ |
| 2 km | 0.01 | Small drone | −151.2 | −8 dB | **10 dB** ⚠️ |
| 3 km | 0.1 | Medium drone | −148.7 | −6 dB | **12 dB** ⚠️ |

**With coherent integration (PLL + FPGA):** 2 km small drone detection is achievable. 3 km medium drone is marginal.

**Phase noise bonus:** ADF41510's −231 dBc/Hz at 100 kHz is much better than the free-running VCO. This reduces phase noise masking of small targets, potentially adding another 3–5 dB of effective sensitivity (not captured in the table above).

---

## Chirp Configurations

| Mode | Ramp T | LPF fc | Max range | f_IF @ max | ADC rate (after decimate) | FFT | Scan rate | v_max |
|------|--------|--------|-----------|------------|--------------------------|-----|-----------|-------|
| **Search** | 10 ms | 500 kHz | 1.5 km | 500 kHz | 3.9 MSPS | 4096-pt | 100 Hz | 0.625 m/s |
| **Track** | 5 ms | 1 MHz | 1.5 km | 1 MHz | 7.8 MSPS | 2048-pt | 200 Hz | 1.25 m/s |
| **Fast** | 1 ms | 5 MHz | 500 m | 1.67 MHz | 31.25 MSPS | 512-pt | 1 kHz | 6.25 m/s |
| **Micro-Doppler** | 0.1 ms | 5 MHz | 50 m | 1.67 MHz | 250 MSPS | 256-pt | 10 kHz | 62.5 m/s |

**Micro-Doppler mode** is enabled by the FPGA: 0.1 ms ramps at 10 kHz PRF resolves 100–500 Hz rotor blade modulation. This is the mode that distinguishes drones from birds.

**Note on PLL loop bandwidth vs chirp rate:** The PLL loop BW must be > 10× the chirp rate for accurate tracking. At 10 ms ramp (100 Hz chirp), 1 kHz loop BW is fine. At 0.1 ms micro-Doppler (10 kHz), need 100 kHz loop BW. Use a programmable loop filter (switched capacitor array) or two separate filters for different modes.

---

## Design Decisions Explained

### 1. ADF41510 PLL (Not Open-Loop DAC)

The open-loop DAC approach was simpler but had critical limitations:
- VCO phase noise ~−80 dBc/Hz (masks small targets)
- Chirp-to-chirp coherence unproven (limits integration gain to noncoherent)
- VCO non-linearity requires LUT pre-distortion
- Temperature drift needs active compensation

The ADF41510 PLL solves all of these:
- **Phase noise**: −231 dBc/Hz fractional-N — 150 dB better than free-running VCO
- **Coherence**: PLL locks to TCXO reference → chirp-to-chirp phase is deterministic
- **Linearity**: Built-in ramp generator produces highly linear sweeps
- **Temperature stability**: Locked to TCXO, no drift

**Cost trade-off:** ~$10 more BOM (PLL vs DAC+REF+buffer+temp sensor), but the sensitivity improvement is worth it for drone detection.

### 2. XC7A100T FPGA (Not MCU)

The MCU approach (STM32H723) was too slow for real-time coherent integration, micro-Doppler, and 2D range-Doppler processing. The XC7A100T Artix-7 FPGA delivers all of this in hardware.

### 3. AD9643 at 250 MSPS (Not 25 MSPS)

250 MSPS provides 2× oversampling at 125 MHz IF bandwidth, ample for the FPGA's DDC + decimation pipeline. Dual channel for future I/Q.

### 4. YSGM556006 as PLL VCO (Reused)

You already have 10 of these. The ADF41510 doesn't have an integrated VCO, so the YSGM556006 becomes the VCO in the PLL loop. No waste.

### 5. Single IF Channel for Prototype 1

AD9643 ch A only. Ch B ready for future I/Q upgrade (90° hybrid + second mixer chain).

### 6. 6 dB π-Pad After VCO (Corrected Values)

150 Ω shunt / 37.4 Ω series / 150 Ω shunt. Prevents VCO frequency pulling (9 MHz pp at 3:1 VSWR).

### 7. OPA838 at ×10

20 dB fixed gain, 30 MHz bandwidth (300 MHz GBW). Not the bottleneck.

### 8. MCP6S91 AGC

0–30 dB SPI-controlled gain. At ×32, BW=562 kHz — sufficient for 500 kHz IF at 1.5 km with 10 ms ramp.

### 9. 24 dBi Antennas

Separate TX and RX, 1–2 m apart. 5° beamwidth. Adequate link budget for 2 km small drone detection.

---

## Known Issues & Open Questions

### Known issues (must be addressed)

| Issue | Impact | Mitigation |
|-------|--------|-----------|
| **PLL loop BW vs chirp rate** | PLL must track fast ramps | Use 100 kHz loop BW for search (10 ms ramp), switch to 500 kHz for micro-Doppler (0.1 ms ramp) |
| **PLL loop filter design** | Critical for stability and phase noise | Use ADI reference design (ADIsimPLL tool), 50° phase margin |
| **Fractional-N spurs** | Spurs at PFD/2, etc. | Use dithering mode or integer-N for cleanest spectrum |
| **QPL9547 NF at 5.75 GHz unverified** | System NF may be 1.5–2 dB | Measure on NF meter; add 2 dB margin |
| **v_max at 10 ms = 0.625 m/s** | Drones alias | Use Fast or Micro-Doppler mode for moving drones |
| **+35 dBm EIRP not ISM** | Regulatory | Acquire experimental/STA license |
| **FPGA PCB complexity** | BGA fanout, DDR3 routing | **Start with Arty A7-100T dev board** |
| **FR4 at 5.5–6 GHz** | Higher insertion loss | Acceptable for prototype; Rogers for production |

### Open questions (resolve during implementation)

- **PLL loop filter topology**: Active vs passive, component values
- **PLL reference frequency**: 100 MHz TCXO vs other (affects PFD frequency and spur locations)
- **Loop BW for each mode**: Search (1 kHz?), Track (10 kHz?), Micro-Doppler (100 kHz?)
- **QPL9547 NF/gain at 5.75 GHz**: measure on VNA + NF meter
- **Antenna type**: 8×8 patch array (~30×30 cm) or parabolic dish (~40 cm)
- **AD9643 input range**: 2 Vpp or 3.5 Vpp
- **Future**: PA upgrade (MMG3H21NT1, +27 dBm TX), I/Q upgrade, MIMO with multiple antennas

---

## Project Status

| Phase | Status |
|-------|--------|
| Architecture definition | ✅ Done (PLL + FPGA) |
| Component selection | ✅ Done |
| Link budget validation | ✅ Done — **2–3 km for drones with coherent integration** |
| VCO/PLL calibration | ⬜ Pending |
| FPGA firmware (VHDL/Verilog) | ⬜ Pending |
| MCU firmware (STM32C031) | ⬜ Pending |
| ADC interface (LVDS deserialization) | ⬜ Pending |
| DSP pipeline (DDC → FFT → CFAR) | ⬜ Pending |
| Integration + validation | ⬜ Pending |

---

*Lyrion Radar — open hardware drone detection radar with PLL chirp source and FPGA-based DSP.*
