# FMCW Radar Front-End — 5.5–6 GHz (FPGA DSP)

## Goal

Design and build a complete FMCW radar front-end at 5.5–6 GHz for **drone detection (counter-UAS)**. Open-loop DAC-ramped VCO, diode-ring mixer RX, **dual 14-bit 250 MSPS ADC**, **Xilinx Artix-7 XC7A100T FPGA** for all real-time DSP, STM32C031 housekeeping MCU.

**Realistic prototype 1 targets:**
- ~1 km for small drones (σ = 0.01 m²) — with 64-chirp coherent integration in FPGA
- ~1.5–2 km for medium drones (σ = 0.1 m²)
- ~30 cm range resolution using 24 dBi antennas
- Scan rate 100 Hz to 10 kHz (configurable)
- Micro-Doppler drone/bird discrimination in short-range mode

**Key FPGA advantage:** Coherent integration of N chirps → +10·log₁₀(N) dB SNR is now achievable because the FPGA can track and correct phase drift between chirps. This recovers the full theoretical gain (+18 dB for 64 chirps) that the MCU approach could only partially achieve.

### Drone RCS Reference

| Drone class | Example | RCS (σ) | dBsm |
|-------------|---------|---------|------|
| Micro (< 250 g) | DJI Mini 4 | 0.001–0.005 m² | −30 to −23 dBsm |
| Small (250 g – 2 kg) | DJI Phantom 4 | 0.01–0.05 m² | −20 to −13 dBsm |
| Medium (2–25 kg) | DJI Matrice 300 | 0.05–0.1 m² | −13 to −10 dBsm |
| Large (> 25 kg) | Fixed-wing UAV | 0.1–1 m² | −10 to 0 dBsm |

---

## Confirmed BOM

| # | Part Number | Role | Qty | Key Specs |
|---|-------------|------|-----|-----------|
| 1 | **YSGM556006** (Innotion) | VCO | 1 | 5320–6060 MHz, +6 dBm, 0–5 V tune, 14 mA |
| 2 | **DAC8830IDR** (TI) | 16-bit SPI DAC | 1 | 1 µs settling, unbuffered R-2R, SOIC-8 |
| 3 | **REF5050AIDGKR** (TI) | DAC voltage reference | 1 | 5.0 V, 3 ppm/°C, needs >5.3 V input |
| 4 | **TLV9062IDR** (TI) | DAC output buffer | 1 | Dual op-amp, rail-to-rail, 10 MHz GBW |
| 5 | **TMP102AIDRLR** (TI) | Temperature sensor | 1 | I²C, ±0.5 °C, SOT-563 |
| 6 | **YG802020W** (Innotion) | TX driver | 1 | +15 dB gain, P1dB ~+16 dBm @ 5.5 GHz |
| 7 | **GP2X+** (Mini-Circuits) | 2-way power divider | 1 | 2.9–6.2 GHz, 3.6 dB total loss |
| 8 | **QPL9547TR7** (Qorvo) | RX LNA | 1 | ~1 dB NF, ~10 dB gain at 5.75 GHz (verify) |
| 9 | **YX18** (Innotion) | Mixer diode quad | 1 | GaAs Schottky, 1.4 V turn-on |
| 10 | **NCS4-63+** (Mini-Circuits) | Mixer baluns | 2 | 4.5–6 GHz, 1:4 ratio |
| 11 | **OPA838IDBVR** (TI) | IF preamplifier | 1 | 0.9 nV/√Hz, 300 MHz GBW |
| 12 | **MCP6S91T-E/MS** (Microchip) | AGC (PGA) | 1 | 1×–32× (0–30 dB), 18 MHz GBW, SPI |
| 13 | **AD9643BCPZ-250** (ADI) | ADC | 1 | **Dual 14-bit, 250 MSPS, LVDS**, QFN-48 |
| 14 | **XC7A100T-CSG324** (Xilinx) | FPGA | 1 | Artix-7, 101K logic cells, 240 DSP slices, 4,860 Kb BRAM, 324-pin BGA |
| 15 | **MT41K256M16TW-107** (Micron) | DDR3L SDRAM | 1 | 256M × 16 = 512 MB, for FFT buffers |
| 16 | **STM32C031C6T6** (ST) | Housekeeping MCU | 1 | Cortex-M0+ @ 48 MHz, 32 KB flash, 12 KB SRAM |
| 17 | **ADP150AUJZ-5.0** (ADI) | Ultra-low noise LDO | 2 | 5 V, <10 µVrms |
| 18 | **LTM4644** (ADI) | FPGA power module | 1 | Multi-rail DC/DC (1.0V, 1.8V, 3.3V) for Artix-7 |
| 19 | **LD1117S33** | 3.3 V LDO | 1 | MCU, ADC digital |
| 20 | Resistors: 150 Ω, **37.4 Ω**, 150 Ω | 6 dB π-pad (corrected) | 3 | 0603 thin-film |
| 21 | Resistors: 1 kΩ, 9.09 kΩ | OPA838 gain (×10) | 2 | 0603, E96 |
| 22 | Caps, inductors, passives | Bypass, DC block, LPF, matching | ~20 | 0603/0805 |
| 23 | Antennas: **24 dBi** ×2 | TX and RX | 2 | 5.5–6 GHz |

**Total component cost: ~$150 (custom PCB) or ~$250 (Arty A7-100T dev board + parts)**

---

## Signal Chain

### TX Path

```
YSGM556006 VCO (+6 dBm)
    │
  100 nF DC block
    │
  6 dB π-pad (150/37.4/150) → 0 dBm
    │
  YG802020W (G=+15 dB @ 93 mA) → +15 dBm
    │
  GP2X+ divider (−3.6 dB) → +11.4 dBm per port
    │
    ├── Port 1 → TX antenna (24 dBi) → +35.4 dBm EIRP (3.5 W)
    │
    └── Port 2 → NCS4-63+ balun → YX18 mixer LO port
```

### RX Path

```
RX antenna (24 dBi) — target echoes: −107 to −159 dBm
    │
  QPL9547TR7 LNA (G=~10 dB, NF=~1 dB at 5.75 GHz)
    │
  NCS4-63+ balun (RF port, −0.5 dB)
    │
  YX18 diode mixer (−7 dB conversion loss, LO=+11.4 dBm)
    │
  RC LPF (1 kΩ + 33 pF, fc ≈ 4.8 MHz)
    │
  OPA838IDBVR (G=×10 = +20 dB, 300 MHz GBW)
    │
  MCP6S91T-E/MS (G=1×–32× = 0–30 dB, SPI AGC)
    │
  AD9643BCPZ-250 ch A (14-bit, 250 MSPS, LVDS)
    │
  XC7A100T ISERDES → DDC → decimate → window → FFT → 2D FFT → MTI → CFAR
    │
  USB/UART output (detected targets, range-Doppler maps)
```

### FPGA and MCU Division

| Function | Component | Notes |
|----------|-----------|-------|
| Chirp generation (DAC ramp) | **STM32C031** SPI → DAC8830 | Slow (~1 ms updates), 16-bit LUT-driven |
| AGC control (MCP6S91) | **STM32C031** SPI → MCP6S91 | One SPI write per chirp |
| Temperature compensation | **STM32C031** I2C → TMP102 | Reads temp, adjusts LUT baseline |
| ADC interface | **XC7A100T** ISERDES (LVDS deserialization) | 14-bit @ 250 MSPS, DDR |
| DDC + decimation | **XC7A100T** CORDIC + CIC + FIR | 250 MSPS → 3.9 MSPS |
| Range FFT | **XC7A100T** Xilinx FFT IP | 1024-pt, < 1 µs latency |
| Doppler FFT (2D) | **XC7A100T** | 64-pt across chirps |
| MTI, CFAR, detection | **XC7A100T** | Hardware, deterministic |
| Micro-Doppler | **XC7A100T** | High-PRF mode (10 kHz) |
| Tracking, output | **XC7A100T** | Kalman filter, USB/UART |
| Debug / config | **STM32C031** UART | Mode switching, LUT updates |

---

## FPGA DSP Pipeline (Detailed)

```
                ┌─────────────────────────────────────────────────────────────┐
                │                   ONE CHIRP PROCESSING                        │
                │                                                              │
  AD9643 ch A ─► ISERDES: 250 MSPS × 14-bit LVDS → 8-bit DDR @ 125 MHz words │
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

The XC7A100T is overkill for one channel — easily handles two channels (I/Q) simultaneously.

---

## Power Budgets

### TX Power Budget

| Stage | Level | Notes |
|-------|-------|-------|
| VCO output | +6 dBm | |
| 6 dB π-pad (150/37.4/150) | −6 dB | |
| YG802020W | +15 dB | 1 dB below P1dB |
| GP2X+ | −3.6 dB | |
| **Each port** | **+11.4 dBm** | |
| With 24 dBi TX antenna | **+35.4 dBm EIRP** (3.5 W) | Not ISM — needs license |

### RX Link Budget — Drone Detection (with FPGA coherent integration)

Conditions: Pt = +11.4 dBm, Gt = Gr = 24 dBi, NF = 1 dB, noise floor = −143 dBm (1 kHz bin).

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

**Key advantage:** FPGA coherent integration gives +18 dB (vs +9 dB noncoherent on MCU). 2 km small drone detection is now within reach without a PA upgrade.

---

## Chirp Configurations

| Mode | Ramp T | LPF fc | Max range | f_IF @ max | ADC rate (after decimate) | FFT | Scan rate | v_max |
|------|--------|--------|-----------|------------|--------------------------|-----|-----------|-------|
| **Search** | 10 ms | 500 kHz | 1.5 km | 500 kHz | 3.9 MSPS | 4096-pt | 100 Hz | 0.625 m/s |
| **Track** | 5 ms | 1 MHz | 1.5 km | 1 MHz | 7.8 MSPS | 2048-pt | 200 Hz | 1.25 m/s |
| **Fast** | 1 ms | 5 MHz | 500 m | 1.67 MHz | 31.25 MSPS | 512-pt | 1 kHz | 6.25 m/s |
| **Micro-Doppler** | 0.1 ms | 5 MHz | 50 m | 1.67 MHz | 250 MSPS (no decimate) | 256-pt | 10 kHz | 62.5 m/s |

**Micro-Doppler mode** is new: 0.1 ms ramps at 10 kHz PRF resolves 100–500 Hz rotor blade modulation. This is the mode that distinguishes drones from birds.

---

## Key Design Decisions

1. **FPGA-based DSP (XC7A100T)** — all real-time signal processing. Replaces the MCU approach. Enables coherent integration (+18 dB), micro-Doppler, 2D range-Doppler, and deterministic latency.
2. **AD9643 at 250 MSPS** — dual 14-bit LVDS. Ch A for IF, Ch B for future I/Q. Provides 2× oversampling at 125 MHz IF bandwidth.
3. **STM32C031 housekeeping MCU** — DAC ramp, AGC, temperature, USB. Small and cheap.
4. **Open-loop DAC ramp with one-time SA calibration** — VCO frequency vs. voltage measured on spectrum analyzer, stored as LUT. FPGA can correct phase drift for coherent integration.
5. **Temperature compensation via TMP102** — LUT baseline shifted by measured ΔT.
6. **6 dB π-pad after VCO (150/37.4/150 Ω)** — prevents 9 MHz pp pulling.
7. **Single YG802020W before GP2X+ divider** — P1dB ~+16 dBm at 5.5 GHz, output +15 dBm.
8. **GP2X+ divider** — 3.6 dB loss, 20 dB isolation, 3×3 mm.
9. **QPL9547 as LNA** — ~1 dB NF, ~10 dB gain at 5.75 GHz (verify).
10. **YX18 + 2× NCS4-63+ as mixer** — 1:4 balun provides voltage step-up.
11. **OPA838 ×10 fixed gain (300 MHz GBW)** — 20 dB gain, 30 MHz bandwidth.
12. **MCP6S91 SPI AGC (0–30 dB)** — gain control. At ×32, BW=562 kHz.
13. **REF5050A from 12 V rail** — 3 ppm/°C reference.
14. **Separate TX/RX antennas (24 dBi)** — no circulator.
15. **Single IF channel for prototype 1** — Ch A only. Ch B ready for I/Q.
16. **Configurable ramp time in firmware** — 10 ms search, 1 ms track, 0.1 ms micro-Doppler.
17. **IF LPF: 1 kΩ + 33 pF (fc ≈ 4.8 MHz)** — passes full IF range for all modes.

---

## Implementation Tasks

### Phase 0: Documentation

- [x] Update Lyrion-Radar README.md with FPGA architecture (AD9643 + XC7A100T + STM32C031)
- [x] Update PLAN.md with same architecture

### Phase 1: Hardware Bring-Up

- [ ] Acquire XC7A100T dev board (Digilent Arty A7-100T) or design custom PCB
- [ ] Acquire AD9643BCPZ-250 (or use evaluation board)
- [ ] Acquire STM32C031 dev board or use existing STM32C031 from Lyrion-Link project
- [ ] Acquire DDR3L SDRAM (MT41K256M16TW-107)
- [ ] LTM4644 power module for FPGA rails
- [ ] Verify all other parts from BOM are in hand

### Phase 2: Signal Source (VCO + DAC + Calibration)

- [ ] Solder VCO, DAC8830, TLV9062 buffer, REF5050A, ADP150 LDOs
- [ ] Wire SPI from STM32C031 to DAC8830
- [ ] Measure VCO frequency vs. DAC code on spectrum analyzer (100 mV steps, 0–5 V)
- [ ] Build inverse LUT: `dac_lut[1000]` mapping 5500–6000 MHz → DAC codes
- [ ] Measure VCO phase noise and residual FM (critical for coherent integration)
- [ ] Generate ramp in firmware, verify linearity on SA
- [ ] Add TMP102, implement temperature compensation offset

### Phase 3: TX Chain

- [ ] Solder 6 dB π-pad (150/37.4/150 Ω)
- [ ] Solder YG802020W with EVB-03 matching
- [ ] Solder GP2X+ divider
- [ ] Verify +11.4 dBm on both output ports
- [ ] Connect TX antenna (24 dBi), verify radiation pattern

### Phase 4: RX Chain + Mixer

- [ ] Measure QPL9547 NF and gain at 5.75 GHz (noise figure meter + VNA)
- [ ] Solder QPL9547 LNA
- [ ] Build YX18 mixer with 2× NCS4-63+ baluns
- [ ] Solder RC LPF (1 kΩ + 33 pF, fc ≈ 4.8 MHz)
- [ ] Solder OPA838 (G=×10)
- [ ] Solder MCP6S91 (SPI from STM32C031)
- [ ] Verify mixer conversion loss with two signal generators

### Phase 5: FPGA Firmware (VHDL/Verilog in Vivado)

- [ ] Vivado project for XC7A100T-CSG324
- [ ] Clock generation: 250 MHz ADC clock, 100 MHz fabric clock, 200 MHz DDR clock
- [ ] ISERDES: 14-bit LVDS deserialization from AD9643
- [ ] AD9643 SPI config (sample rate, output format, channel select)
- [ ] DDC: CORDIC + NCO for complex downconversion
- [ ] Decimation: CIC (4×) + FIR (4×) → 3.9 MSPS
- [ ] Window function (Hann)
- [ ] FFT: Xilinx FFT IP, 1024-pt complex, pipelined streaming
- [ ] DDR3 controller (Xilinx MIG) for FFT buffer storage
- [ ] 2D FFT: 64-pt across chirps at each range bin
- [ ] MTI: subtract consecutive Doppler maps
- [ ] CFAR: cell-averaging on range-Doppler map
- [ ] Peak detection
- [ ] Micro-Doppler mode (high-PRF, 0.1 ms ramps)
- [ ] Tracking: Kalman filter across frames
- [ ] UART output: detected targets with range, velocity, amplitude

### Phase 6: MCU Firmware (STM32C031)

- [ ] CubeMX project for STM32C031C6T6
- [ ] SPI1 → DAC8830 (ramp generation, LUT-driven)
- [ ] SPI2 → MCP6S91 (AGC gain control)
- [ ] I2C1 → TMP102 (temperature)
- [ ] GPIO → FPGA mode/control
- [ ] UART → debug / config interface
- [ ] Ramp state machine (triggered by FPGA sync signal)

### Phase 7: Integration + Validation

- [ ] End-to-end test: place target at 100 m, 500 m, 1 km
- [ ] Measure range accuracy and resolution
- [ ] Test AGC convergence
- [ ] Measure coherent integration gain (1 vs 64 chirps)
- [ ] Test micro-Doppler mode with a spinning fan or drone propeller
- [ ] Thermal test: verify TMP102 compensation
- [ ] Measure TX leakage into RX
- [ ] Drone detection test: fly a small drone at 500 m, 1 km
- [ ] Bird vs. drone discrimination test

---

## PCB Layout Guidelines

### ADC (AD9643) to FPGA Interface

- **LVDS pairs**: matched-length (±0.5 mm) within each byte group, length-matched across groups
- **Differential impedance**: 100 Ω differential
- **Trace length**: keep < 50 mm total
- **AC coupling**: 100 nF caps on each LVDS pair between ADC and FPGA
- **Reference plane**: solid ground under all LVDS traces
- **ADC clock**: clean 250 MHz from FPGA MMCM, routed with controlled impedance

### FPGA (XC7A100T-CSG324) BGA Layout

- **BGA pitch**: 0.8 mm, 324 balls
- **Fanout**: dog-bone or via-in-pad
- **Layer count**: 8+ layers recommended for signal integrity
- **DDR3**: Fly-by topology, matched-length ±25 ps
- **Power**: multiple decoupling caps per rail (1.0 V, 1.8 V, 3.3 V)
- **JTAG**: standard Xilinx programming header

### RF Section (5.5–6 GHz)

- **50 Ω microstrip**: ~3 mm wide on 1.6 mm FR4
- **VCO placement**: keep 6 dB π-pad within 2 mm of VCO output
- **Mixer**: NCS4-63+ baluns as close as possible to YX18
- **Ground**: solid ground plane, multiple vias
- **Power**: separate ADP150 for VCO, star-ground analog/digital

### IF Section (DC–5 MHz)

- **OPA838**: 100 nF bypass within 2 mm, guard traces around sensitive nodes
- **MCP6S91**: similar layout care
- **RC LPF**: low-noise resistors, film caps if possible
- **Trace to ADC**: differential, 100 Ω, matched-length

### Development Board Option (Digilent Arty A7-100T)

The Arty A7-100T dev board includes:
- XC7A100T-CSG324
- 256 MB DDR3L (MT41K256M16TW-107)
- USB-UART bridge
- 4 PMOD connectors (for ADC interface board)
- Onboard clock (can be replaced with 250 MHz from ADC eval board)

This is the **recommended starting point** for prototype 1. Custom PCB is a later step.

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| VCO tuning non-linearity | Range bin spreading | Pre-distorted LUT; recalibrate if temperature range is wide |
| YX18 mixer conversion loss higher than 7 dB at 5.5 GHz | Reduced sensitivity | Measure on EVB first; increase LO drive if needed |
| TX-RX antenna coupling saturates LNA | Blind zone at close range | Maximize antenna separation; QPL9547 P1dB = +19 dBm handles leakage |
| YG802020W P1dB only +16 dBm at 5.5 GHz | TX power limited to +15 dBm | Add MMG3H21NT1 PA in rev 2 for +27 dBm |
| REF5050A dropout (needs >5.3 V) | No reference | 12 V rail confirmed available |
| AD9643 LVDS timing at 250 MSPS | Data corruption | Matched-length pairs, AC coupling, verified with logic analyzer |
| FPGA BGA PCB complexity | Expensive, hard to assemble | **Start with Arty A7-100T dev board**; custom PCB later |
| DDR3 signal integrity | Memory errors | Fly-by topology, matched-length, proper termination |
| Open-loop VCO phase incoherence | Limits coherent integration | Measure residual FM/PN; FPGA phase tracking helps but has limits |
| QPL9547 NF at 5.75 GHz unverified | System NF may be 1.5–2 dB | Measure on NF meter; add 2 dB margin |
| f_IF at 1.5 km with 5 ms ramp = 1 MHz | Exceeds MCP6S91 ×32 BW (562 kHz) | Use 1.5 km max range or lower AGC gain |
| v_max at 10 ms = 0.625 m/s | Drones alias | Use Fast or Micro-Doppler mode for moving drones |
| Micro-Doppler (100–500 Hz) vs chirp PRF | Drone/bird discrimination infeasible in Search/Track | **New: Micro-Doppler mode at 10 kHz PRF resolves this** |
| +35 dBm EIRP not ISM | Regulatory | Acquire experimental/STA license |
| FR4 at 5.5–6 GHz | Higher loss, εr tolerance | Acceptable for prototype; Rogers for production |

---

## Open Questions (resolve during implementation)

- **QPL9547 NF/gain at 5.75 GHz**: MUST measure on noise-figure meter
- **VCO residual FM/PN**: MUST measure to determine coherent integration viability
- **OPA838 gain**: ×10 is the plan. Verify with actual mixer IF levels.
- **Antenna type**: 8×8 patch array (~30×30 cm) or parabolic dish (~40 cm)
- **AD9643 input range**: 2 Vpp or 3.5 Vpp — configure via SPI
- **Decimation factor**: 4× + 4× = 16×, adjust based on IF bandwidth needs
- **FFT size**: 1024, 2048, or 4096
- **FPGA development**: start with Arty A7-100T or custom PCB?
- **DDR3 size**: 256 MB sufficient for 64-chirp × 4096-pt × 2 channels?
- **Future**: PA upgrade (MMG3H21NT1), I/Q upgrade (90° hybrid + 2nd mixer), MIMO with multiple antennas
