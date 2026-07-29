# Lyrion Radar 🛰️

**FMCW radar at 5.5–6 GHz for counter-UAS (drone detection) — FPGA-based DSP.**

[![Status](https://img.shields.io/badge/Status-Planning-yellow)]()
[![License](https://img.shields.io/badge/License-TBD-lightgrey)]()

---

## Table of Contents

- [What is Lyrion Radar?](#what-is-lyrion-radar)
- [Why FPGA?](#why-fpga)
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

The signal chain uses an **open-loop DAC-ramped VCO** (no PLL) for the chirp generator, a **diode-ring mixer** for downconversion, a **dual 14-bit 250 MSPS ADC** for IF digitization, and a **Xilinx Artix-7 XC7A100T FPGA** for all real-time signal processing (DDC, decimation, FFT, MTI, micro-Doppler, CFAR). A small **STM32C031 MCU** handles housekeeping (DAC ramp, AGC, temperature, USB).

### Quick Facts

| Parameter | Value |
|-----------|-------|
| Frequency band | 5.5–6.0 GHz (500 MHz sweep) |
| Range resolution | **30 cm** |
| Realistic max range (small drone) | **~1 km** (noncoherent integration) |
| Realistic max range (medium drone) | **~1.5–2 km** |
| Path to 2–3 km | PA upgrade + I/Q channels (available now via dual ADC) |
| TX power | +11.4 dBm (prototype 1) |
| Antenna gain | 24 dBi each (TX and RX, separate) |
| ADC | **AD9643 dual 14-bit, 250 MSPS** (LVDS to FPGA) |
| DSP | **Xilinx XC7A100T** Artix-7 (101K logic cells, 240 DSP slices) |
| Control MCU | STM32C031 (DAC ramp, AGC, temp, USB) |
| Scan rate | 100 Hz to 10 kHz (configurable) |
| Estimated BOM cost | ~$150–200 (FPGA board + parts) |

---

## Why FPGA?

A small MCU (STM32C031) can do basic range FFT in real-time, but for serious drone detection you need:

| Requirement | MCU (STM32C031/H723) | FPGA (XC7A100T) |
|-------------|----------------------|-----------------|
| 1024-pt complex FFT | ~50 µs (H723) | **< 1 µs** (parallel hardware) |
| 64-chirp 2D FFT (range-Doppler) | ~10 ms | **< 100 µs** |
| Coherent integration (64 chirps) | Sequential, slow | **Parallel, real-time** |
| Micro-Doppler (high-PRF processing) | Infeasible | **Trivial** (reconfigurable logic) |
| Custom DDC / decimation | Slow (software) | **Hardware, deterministic** |
| Multi-target tracking | Difficult | **Native** (parallel peak detection) |
| Latency | Variable (RTOS) | **Deterministic** |
| I/Q simultaneous processing | Sequential | **Parallel** |

The FPGA enables **micro-Doppler analysis** (rotor blade modulation at 100–500 Hz) which is the key discriminator between drones and birds. It also enables true **coherent integration** (since the FPGA can track phase across chirps) — recovering the +18 dB theoretical gain that the open-loop VCO otherwise gives only noncoherently (~+9 dB).

---

## Honest Assessment

Several claims in this README are **assumptions**, not verified facts:

| Claimed | Status |
|---------|--------|
| Coherent integration of N chirps → +10·log₁₀(N) dB SNR | **Achievable now.** The FPGA can track phase across chirps and apply phase corrections, recovering most of the theoretical coherent gain. This is a major advantage over the MCU approach. |
| QPL9547 NF 0.6 dB at 5.5–6 GHz | **Unverified.** Datasheet 0.6 dB is mid-band (~2 GHz). At 6 GHz expect **~1 dB NF, ~10 dB gain**. |
| Micro-Doppler at 100–500 Hz for drone/bird discrimination | **Feasible now.** The FPGA can process high-PRF chirp streams in real-time, unlike the MCU approach which was bandwidth-limited. |
| +35 dBm EIRP at 5.5–6 GHz is legal/ISM | **Wrong.** This band includes UNII/DFS segments and requires licensed or experimental authorization in most jurisdictions. |

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

| Drone class | Example | RCS (radar cross-section) | Range with prototype 1 |
|-------------|---------|--------------------------|------------------------|
| Micro (< 250 g) | DJI Mini 4 | 0.001–0.005 m² (−30 to −23 dBsm) | **200–500 m** (marginal) |
| Small (250 g – 2 kg) | DJI Phantom 4 | 0.01–0.05 m² (−20 to −13 dBsm) | **~1 km** |
| Medium (2–25 kg) | DJI Matrice 300 | 0.05–0.1 m² (−13 to −10 dBsm) | **~1.5–2 km** |
| Large (> 25 kg) | Fixed-wing UAV | 0.1–1 m² (−10 to 0 dBsm) | > 2 km |

These ranges assume:
- 64-chirp coherent integration (achievable with FPGA phase tracking)
- 24 dBi antennas
- 5 ms or 10 ms ramp
- Drone flying roughly head-on or head-away

### Why 5.5–6 GHz?

- Good balance between resolution (500 MHz → 30 cm) and atmospheric attenuation
- Smaller antennas than S-band, less rain fade than X-band
- Components are widely available and affordable
- **Regulatory caveat**: this band includes UNII/DFS segments. Operation at +35 dBm EIRP requires licensed or experimental authorization in most jurisdictions.

---

## System Architecture

### High-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SIGNAL SOURCE                                      │
│                                                                          │
│  STM32C031 ──SPI──► DAC8830 ──► TLV9062 ──► YSGM556006 VCO             │
│  (housekeeping)     (16-bit)    (buffer)      (5320-6060 MHz, +6 dBm)   │
│                                         │                                │
│  REF5050A (5.0V ref, 3 ppm/°C) ──► DAC VREF                            │
│  TMP102 (I²C temperature sensor) ──► compensation                       │
└─────────────────────────────────────────────────────────────────────────┘
                                         │
                                   100 nF DC block
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
│                 NF=0.6dB              ─────┤                             │
│                 G=17dB                LO from GP2X+                     │
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

### FPGA and MCU Division of Labor

```
                    ┌────────────────────────────────────┐
                    │  Xilinx XC7A100T (XC7A100TCSG324)  │
                    │                                    │
  AD9643 ch A (LVDS)─►│ ISERDES (8:1 DDR deserialization) │
                    │                                    │
  AD9643 ch B (LVDS)─►│ ISERDES (future I/Q or 2nd ant)   │
                    │                                    │
  DAC8830 ──► SPI ──►│ Ramp sequencer (state machine)     │
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
                    │  SPI1 ──► DAC8830 (ramp)           │
                    │  SPI2 ──► MCP6S91 (AGC)            │
                    │  I2C1 ──► TMP102 (temp)            │
                    │  GPIO ──► FPGA control (mode, etc) │
                    │  UART ──► debug / config           │
                    └────────────────────────────────────┘
```

The MCU handles low-speed configuration and housekeeping. The FPGA handles all real-time DSP at 250 MSPS.

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

The XC7A100T is overkill for one channel — easily handles two channels (I/Q) simultaneously. This leaves headroom for future expansion (more channels, higher FFT size, micro-Doppler spectrogram).

---

## Complete Component List

### Signal Source

| Part | Manufacturer | Role | Key Specs | Notes |
|------|-------------|------|-----------|-------|
| **YSGM556006** | Innotion | VCO | 5320–6060 MHz, +6 dBm, 0–5 V tune, 14 mA | You have 10 on hand |
| **DAC8830IDR** | TI | 16-bit SPI DAC | 1 µs settling, unbuffered R-2R, SOIC-8 | Driven by MCU SPI |
| **REF5050AIDGKR** | TI | Voltage reference | 5.0 V, 3 ppm/°C, 3 µVpp | Needs >5.3 V input |
| **TLV9062IDR** | TI | DAC output buffer | Dual op-amp, rail-to-rail, 10 MHz GBW | Ch A = DAC buffer |
| **TMP102AIDRLR** | TI | Temperature sensor | I²C, ±0.5 °C, SOT-563 | VCO thermal compensation |

### TX Chain

| Part | Manufacturer | Role | Key Specs | Notes |
|------|-------------|------|-----------|-------|
| **YG802020W** | Innotion | TX driver | 50 MHz–8 GHz, +15 dB gain, P1dB ~+16 dBm @ 5.5 GHz | You have 6 on hand |
| **GP2X+** | Mini-Circuits | 2-way power divider | 2.9–6.2 GHz, 3.6 dB loss, 20 dB isolation | |
| 150 Ω + **37.4 Ω** + 150 Ω | — | 6 dB π-pad (corrected) | 0603 thin-film | Prevents VCO pulling |

### RX Chain

| Part | Manufacturer | Role | Key Specs | Notes |
|------|-------------|------|-----------|-------|
| **QPL9547TR7** | Qorvo | LNA | ~1 dB NF, ~10 dB gain at 5.75 GHz (verify) | Lowest NF available |
| **YX18** | Innotion | Diode quad mixer | GaAs Schottky, 1.4 V turn-on | You have 6 on hand |
| **NCS4-63+** (×2) | Mini-Circuits | Baluns (RF + LO) | 4.5–6 GHz, 1:4 ratio | You have 12 on hand |
| **OPA838IDBVR** | TI | IF preamplifier | 0.9 nV/√Hz, 300 MHz GBW | 1 mA supply |
| **MCP6S91T-E/MS** | Microchip | AGC (PGA) | 1×–32× (0–30 dB), 18 MHz GBW, SPI | |

### Digital / DSP

| Part | Manufacturer | Role | Key Specs | Notes |
|------|-------------|------|-----------|-------|
| **AD9643BCPZ-250** | Analog Devices | ADC | **Dual 14-bit, 250 MSPS**, LVDS, QFN-48 | Ch A for IF, Ch B for future I/Q |
| **XC7A100T-CSG324** | Xilinx | FPGA | Artix-7, **101K logic cells, 240 DSP slices, 4,860 Kb BRAM** | 324-pin BGA |
| **MT41K256M16TW-107** | Micron | DDR3L SDRAM | 256M × 16 = 512 MB | For FFT buffers and data logging |
| **STM32C031C6T6** | ST | Housekeeping MCU | Cortex-M0+ @ 48 MHz, 32 KB flash, 12 KB SRAM | DAC, AGC, temp, USB |

### FPGA Development Options

| Option | Cost | Notes |
|--------|------|-------|
| **Digilent Arty A7-100T** | ~$200 | Ready-to-use dev board with XC7A100T, DDR3, USB, PMOD. Best for prototyping. |
| Custom PCB with XC7A100T | ~$50 (chip) + $100 (PCB) | Production target. Requires BGA fanout, DDR3 routing, power rails. |
| **Arty Z7-20** (Zynq-7020) | ~$200 | ARM Cortex-A9 + FPGA. Overkill if you're not using the ARM. |

### Power

| Part | Manufacturer | Role | Key Specs |
|------|-------------|------|-----------|
| **ADP150AUJZ-5.0** (×2) | ADI | Ultra-low noise LDO (5 V) | VCO + analog |
| **LTM4644** or similar | ADI | FPGA power (1.0 V, 1.8 V, 3.3 V) | Multi-rail DC/DC for Artix-7 |
| **LD1117S33** | ST | 3.3 V LDO | MCU, ADC digital |

### Antennas

| Item | Spec | Notes |
|------|------|-------|
| TX antenna | 24 dBi, 5.5–6 GHz | ~30×30 cm patch array or 40 cm dish |
| RX antenna | 24 dBi, 5.5–6 GHz | Separate, same direction as TX |

**Total component cost (estimate): ~$250** (Arty A7-100T + parts) or ~$150 (custom PCB + chip).

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

### RX Path — From Antenna to FPGA

```
RX antenna (target echoes: −107 to −159 dBm depending on range/RCS)
    │
  QPL9547 LNA (+10 dB, NF~1 dB at 5.75 GHz)
    │
  NCS4-63+ balun (1:4, −0.5 dB)
    │
  YX18 diode mixer (−7 dB conversion loss) → IF beat note
    │
  RC LPF (1 kΩ + 33 pF, fc ≈ 4.8 MHz)
    │
  OPA838 fixed gain (×10 = +20 dB, 300 MHz GBW)
    │
  MCP6S91 PGA (×1 to ×32, SPI AGC)
    │
  AD9643 ch A (14-bit, 250 MSPS, LVDS)
    │
  XC7A100T ISERDES (deserialization)
    │
  DDC → decimate → window → FFT → 2D FFT → MTI → CFAR → output
```

### IF Gain vs. Bandwidth (MCP6S91 AGC)

| AGC setting | Gain | BW | Total gain | Use case |
|-------------|------|-----|-----------|----------|
| ×1 | +0 dB | 18 MHz | 20 dB | Close targets |
| ×4 | +12 dB | 4.5 MHz | 32 dB | Medium range |
| ×8 | +18 dB | 2.25 MHz | 38 dB | 1 km+ |
| ×32 | +30 dB | 562 kHz | 50 dB | Long range (limited to 1.5 km at 5 ms ramp) |

---

## Power Budgets & Link Budget

### TX Power Budget

| Stage | Level | Notes |
|-------|-------|-------|
| VCO output | +6 dBm | |
| 6 dB π-pad | −6 dB | Corrected values: 150/37.4/150 |
| YG802020W | +15 dB | 1 dB below P1dB |
| GP2X+ | −3.6 dB | |
| **Each port** | **+11.4 dBm** | |
| With 24 dBi TX antenna | **+35.4 dBm EIRP** (3.5 W) | Not ISM — needs license |

### RX Link Budget — Drone Detection

Conditions: Pt = +11.4 dBm, Gt = Gr = 24 dBi, NF = 1 dB, noise floor = −143 dBm (1 kHz bin).

| Range | σ (m²) | Target | Pr (dBm) | SNR (1 chirp) | SNR (64-chirp coherent) |
|-------|--------|--------|----------|----------------|------------------------|
| 500 m | 0.1 | Medium drone | −117.2 | 26 dB | **44 dB** ✅ |
| 500 m | 0.01 | Small drone | **−127.2** | **16 dB** | **34 dB** ✅ |
| 1 km | 0.1 | Medium drone | −129.2 | 14 dB | **32 dB** ✅ |
| 1 km | 0.01 | Small drone | **−139.2** | **4 dB** | **22 dB** ✅ |
| 1.5 km | 0.1 | Medium drone | −136.2 | 7 dB | **25 dB** ✅ |
| 1.5 km | 0.01 | Small drone | −146.2 | −3 dB | **15 dB** ✅ |
| 2 km | 0.1 | Medium drone | −141.2 | 2 dB | **20 dB** ✅ |
| 2 km | 0.01 | Small drone | −151.2 | −8 dB | **10 dB** ⚠️ |

**Key advantage of FPGA:** Coherent integration of 64 chirps → +18 dB SNR (theoretically) is now achievable because the FPGA can track and correct phase drift between chirps. With the MCU approach, integration was limited to noncoherent (~+9 dB) due to lack of phase tracking.

**For 2 km small drone detection:** +18 dB coherent integration (FPGA) is sufficient. No PA upgrade needed.

---

## Chirp Configurations

| Mode | Ramp T | LPF fc | Max range | f_IF @ max range | ADC effective rate | FFT | Scan rate | v_max |
|------|--------|--------|-----------|-------------------|-------------------|-----|-----------|-------|
| **Search** | 10 ms | 500 kHz | 1.5 km | 500 kHz | 3.9 MSPS (after FPGA decimate) | 4096-pt | 100 Hz | 0.625 m/s |
| **Track** | 5 ms | 1 MHz | 1.5 km | 1 MHz | 7.8 MSPS | 2048-pt | 200 Hz | 1.25 m/s |
| **Fast** | 1 ms | 5 MHz | 500 m | 1.67 MHz | 31.25 MSPS | 512-pt | 1 kHz | 6.25 m/s |
| **Micro-Doppler** | 0.1 ms | 5 MHz | 50 m | 1.67 MHz | 250 MSPS (no decimate) | 256-pt | 10 kHz | 62.5 m/s |

**Micro-Doppler mode** is new with the FPGA: 0.1 ms ramps at 10 kHz PRF easily resolves 100–500 Hz rotor blade modulation. This is the mode that distinguishes drones from birds. Switch to this mode for short-range detailed analysis.

---

## Design Decisions Explained

### 1. FPGA-Based DSP (Not MCU)

The MCU-based approach (STM32H723) can do basic range FFT but cannot do real-time coherent integration, micro-Doppler, or 2D range-Doppler processing. The XC7A100T Artix-7 FPGA is the right tool:

- **Hardware FFT**: Xilinx FFT IP runs 1024-pt complex FFT in < 1 µs (vs ~50 µs on H723)
- **Parallel channels**: I/Q simultaneously with no throughput penalty
- **Deterministic latency**: critical for tracking
- **Phase tracking**: enables coherent integration (the +18 dB theoretical gain, not the +9 dB noncoherent)
- **Micro-Doppler**: high-PRF processing at 10 kHz is trivial in FPGA, impossible in MCU

The trade-off is complexity: VHDL/Verilog, Vivado toolchain, BGA PCB design, high-speed signal integrity. For a drone detection radar, this is the right choice.

### 2. AD9643 at 250 MSPS (Not 25 MSPS)

The 250 MSPS ADC3642's older cousin (AD9643) is used because:

- **2× oversampling** at 125 MHz IF bandwidth gives +14 dB processing gain
- **Dual channel** for future I/Q (with 90° hybrid + second mixer chain) or second RX antenna
- **LVDS output** maps directly to Xilinx ISERDES — clean, high-speed interface
- **Enough for 4096-pt FFT at 31.25 MHz word rate** after decimation

The 25 MSPS ADC3642 was fine for a MCU-based design but underutilizes the FPGA. 250 MSPS gives headroom for future high-PRF modes.

### 3. Open-Loop VCO Ramp (No PLL)

Same as before, but now the FPGA can correct the phase drift between chirps in software, recovering most of the theoretical coherent integration gain. The TMP102 provides temperature compensation for slow drift; the FPGA provides real-time phase correction for fast residual FM.

### 4. Single IF Channel (Channel A) for Prototype 1

The AD9643 has two channels, but prototype 1 uses only one. Channel B is ready for future I/Q expansion:

- Add 90° hybrid coupler at mixer output (Mini-Circuits QCN-4-143+ or microstrip branch-line)
- Add second OPA838 + MCP6S91 chain
- FPGA processes both channels in parallel

### 5. 6 dB π-Pad After VCO (Corrected Values)

150 Ω shunt / **37.4 Ω** series / 150 Ω shunt. Prevents VCO frequency pulling (9 MHz pp at 3:1 VSWR).

### 6. OPA838 at ×10

20 dB fixed gain, 30 MHz bandwidth (300 MHz GBW). Not the bottleneck.

### 7. MCP6S91 AGC

0–30 dB SPI-controlled gain. At ×32, BW=562 kHz — sufficient for 500 kHz IF at 1.5 km with 10 ms ramp. Lower AGC gain for longer ranges.

### 8. 24 dBi Antennas

Separate TX and RX, 1–2 m apart. 5° beamwidth. Adequate link budget for ~1 km small drone detection.

---

## Known Issues & Open Questions

### Known issues (must be addressed)

| Issue | Impact | Mitigation |
|-------|--------|-----------|
| **f_IF at 1.5 km with 5 ms ramp = 1 MHz** | Exceeds MCP6S91 ×32 BW (562 kHz) | Use 1.5 km max range or lower AGC gain |
| **v_max at 10 ms = 0.625 m/s** | Drones alias | Use Fast or Micro-Doppler mode for moving drones |
| **QPL9547 NF/gain at 5.75 GHz unverified** | System NF may be 1.5–2 dB, not 1 dB | Measure on NF meter; add 2 dB margin |
| **+35 dBm EIRP at 5.5–6 GHz is not ISM** | Regulatory issue | Acquire experimental/STA license |
| **FPGA PCB complexity** | BGA fanout, DDR3 routing, signal integrity | Start with Digilent Arty A7-100T dev board |
| **Open-loop VCO residual FM** | Limits coherent integration gain | FPGA can correct slow drift; measure to characterize |
| **Decimation aliasing** | IF above 100 kHz aliases to 200 kSPS Nyquist | 4.8 MHz LPF + AGC bandwidth limits handle this — verify |
| **FR4 at 5.5–6 GHz** | Higher insertion loss, εr tolerance | Acceptable for prototype; Rogers for production |

### Open questions (resolve during implementation)

- **QPL9547 NF/gain at 5.75 GHz**: measure on VNA + NF meter
- **VCO residual FM/PN**: measure to determine coherent integration viability
- **OPA838 gain**: ×10 is the plan. Verify with actual mixer IF levels.
- **Antenna type**: 8×8 patch array (~30×30 cm) or parabolic dish (~40 cm)
- **AD9643 input range**: 2 Vpp or 3.5 Vpp — configure via SPI
- **Decimation factor**: start with 4× + 4× = 16×, adjust based on IF bandwidth needs
- **FFT size**: 1024, 2048, or 4096 — trade-off between range bins and compute
- **Future**: PA upgrade (MMG3H21NT1, +27 dBm TX), I/Q upgrade (90° hybrid + 2nd mixer), MIMO with multiple antennas

---

## Project Status

| Phase | Status |
|-------|--------|
| Architecture definition | ✅ Done (FPGA-based) |
| Component selection | ✅ Done |
| Link budget validation | ✅ Done — **realistic 1–1.5 km for small drones** |
| VCO calibration (SA) | ⬜ Pending |
| FPGA firmware (VHDL/Verilog) | ⬜ Pending |
| MCU firmware (STM32C031) | ⬜ Pending |
| ADC interface (LVDS deserialization) | ⬜ Pending |
| DSP pipeline (DDC → FFT → CFAR) | ⬜ Pending |
| Integration + validation | ⬜ Pending |

---

*Lyrion Radar — open hardware drone detection radar with FPGA-based DSP.*
