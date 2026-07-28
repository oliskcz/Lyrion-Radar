# FMCW Radar Front-End — 5.5–6 GHz

## Goal

Design and build a complete FMCW radar front-end at 5.5–6 GHz for **drone detection (counter-UAS)**. Open-loop DAC-ramped VCO, diode-ring mixer RX, external 14-bit ADC, STM32H723-based signal processing. Target: **1–3 km detection of drones** (σ = 0.01–0.1 m²) with ~30 cm range resolution using **24 dBi antennas**.

### Drone RCS Reference

| Drone class | Example | RCS (σ) | dBsm |
|-------------|---------|---------|------|
| Micro (< 250 g) | DJI Mini 4 | 0.001–0.005 m² | −30 to −23 dBsm |
| Small (250 g – 2 kg) | DJI Phantom 4 | 0.01–0.05 m² | −20 to −13 dBsm |
| Medium (2–25 kg) | DJI Matrice 300 | 0.05–0.1 m² | −13 to −10 dBsm |
| Large (> 25 kg) | Fixed-wing UAV | 0.1–1 m² | −10 to 0 dBsm |

**Prototype 1**: Single IF channel (one mixer, one IF chain, ADC channel A). Channel B reserved for future I/Q or second RX antenna. Triangular ramp extracts range + velocity (critical for distinguishing drones from birds/clutter).
**Future rev**: Dual-channel I/Q (90° hybrid + second mixer) for better clutter rejection and MTI, more powerful PA (MMG3H21NT1 or HMC637), optionally Xilinx Artix-7 35T FPGA for hardware DDC/FFT.

## Confirmed BOM

| # | Part Number | Role | Qty | Key Specs |
|---|-------------|------|-----|-----------|
| 1 | **YSGM556006** (Innotion) | VCO | 1 | 5320–6060 MHz, +6 dBm, 0–5 V tune, 14 mA |
| 2 | **DAC8830IDR** (TI) | 16-bit SPI DAC | 1 | 1 µs settling, unbuffered R-2R, SOIC-8 |
| 3 | **REF5050AIDGKR** (TI) | DAC voltage reference | 1 | 5.0 V, 3 ppm/°C, needs >5.3 V input (12 V rail available) |
| 4 | **TLV9062IDR** (TI) | DAC output buffer | 1 | Dual op-amp, rail-to-rail, 10 MHz GBW, SOIC-8. Ch A = DAC buffer, Ch B = spare |
| 5 | **TMP102AIDRLR** (TI) | Temperature sensor | 1 | I²C, ±0.5 °C, SOT-563 |
| 6 | **YG802020W** (Innotion) | TX driver amplifier | 1 | 50 MHz–8 GHz, +15 dB gain @ 5.5 GHz (93 mA), **P1dB ~+16 dBm @ 5.5 GHz**, DFN 2×2 |
| 7 | **GP2X+** (Mini-Circuits) | 2-way power divider | 1 | 2.9–6.2 GHz, 3.6 dB total loss, 19–21 dB isolation, 3×3 mm QFN |
| 8 | **QPL9547TR7** (Qorvo) | RX LNA | 1 | 0.05–6 GHz, 0.6 dB NF, 17 dB gain, +19 dBm P1dB, DFN 2×2 |
| 9 | **YX18** (Innotion) | Mixer diode quad | 1 | GaAs Schottky cross-over quad, 1.4 V turn-on, DFN 2×2 |
| 10 | **NCS4-63+** (Mini-Circuits) | Mixer baluns | 2 | 4.5–6 GHz, 1:4 ratio, 0.5 dB amplitude unbalance, 0805 |
| 11 | **OPA838IDBVR** (TI) | IF preamplifier | 1 | 0.9 nV/√Hz, **300 MHz GBW**, 1 mA supply, SOT-23-5 |
| 12 | **MCP6S91T-E/MS** (Microchip) | IF PGA (AGC) | 1 | 1×–32× (0–30 dB), 18 MHz GBW, SPI, MSOP-8 |
| 13 | **ADC3642IRSBR** (TI) | External ADC | 1 | Dual 14-bit, 25 MSPS, ultra-low power, QFN-40. Ch A = IF, Ch B = future I/Q |
| 14 | **STM32H723VGT6** (ST) | MCU / signal processor | 1 | Cortex-M7 @ 550 MHz, HW FPU, 320 KB SRAM, DCMI, LQFP-100 |
| 15 | **ADP150AUJZ-5.0** (ADI) | Ultra-low noise LDO | 2 | 5 V out, <10 µVrms noise. One for VCO VCC, one for analog/RF |
| 16 | **LD1117S33** or similar | 3.3 V LDO for digital | 1 | STM32H723, ADC3642 digital supply |
| 17 | Resistors: 68 Ω, 150 Ω ×2 | 6 dB resistive pad | 3 | 0603 thin-film, after VCO output |
| 18 | Resistors: 1 kΩ, 9.09 kΩ | OPA838 gain setting (×10) | 2 | 0603, E96 values |
| 19 | Caps: 100 nF ×6, 10 µF ×2, 33 pF ×1, 10 nF ×2, 1000 pF ×2, 1 pF ×2 | Bypass, DC block, LPF, matching | ~15 | 0603/0805 |
| 20 | Inductors: 68 nH ×2, 2.2 nH ×1 | YG802020W bias feed, matching | 3 | 0603 |
| 21 | I²C pull-ups: 4.7 kΩ ×2 | TMP102 bus | 2 | 0603 |
| 22 | Antennas: **24 dBi** ×2 | TX and RX (separate) | 2 | 5.5–6 GHz, ~5° beamwidth, patch array or dish |

**Total active component cost: ~$40** (excluding antennas, PCB, VCOs/diodes/baluns already owned)

## Signal Chain

### TX Path

```
YSGM556006 VCO (+6 dBm)
    │
  100 nF DC block
    │
  6 dB resistive pad (68Ω series + 2×150Ω shunt) → 0 dBm
    │
  YG802020W (G=+15 dB @ 93 mA, Rex=open) → +15 dBm
    │
  GP2X+ divider (−3.6 dB) → +11.4 dBm per port
    │
    ├── Port 1 → TX antenna (24 dBi) → +35.4 dBm EIRP
    │
    └── Port 2 → NCS4-63+ balun → YX18 mixer LO port
```

**Note:** YG802020W P1dB is ~+16 dBm at 5.5 GHz. Output of +15 dBm is 1 dB below compression — acceptable. Future: add MMG3H21NT1 PA (+30 dBm P1dB) for +27 dBm TX.

### RX Path

```
RX antenna (24 dBi)
    │
  QPL9547TR7 LNA (G=+17 dB, NF=0.6 dB)
    │
  NCS4-63+ balun (RF port, −0.5 dB)
    │
  YX18 diode mixer (conversion loss ~7 dB, LO=+11.4 dBm)
    │
  RC LPF (R=1 kΩ, C=33 pF, fc ≈ 4.8 MHz)
    │
  OPA838IDBVR (G=×10 = +20 dB, fixed, non-inverting, BW=30 MHz)
    │
  MCP6S91T-E/MS (G=1×–32× = 0–30 dB, SPI AGC)
    │
  ADC3642IRSBR ch A (14-bit, 25 MSPS, parallel CMOS → DCMI)
    │
  STM32H723 (DMA capture → decimate → FFT → detection)
```

### IF Gain/Bandwidth at AGC Settings

| MCP6S91 gain | MCP6S91 BW | Combined BW (with OPA838 ×10) | Total gain | Use case |
|-------------|-----------|-------------------------------|------------|----------|
| ×1 (0 dB) | 18 MHz | 7.2 MHz | 20 dB | Close targets, strong signal |
| ×4 (12 dB) | 4.5 MHz | 3.6 MHz | 32 dB | Medium range, short ramps |
| ×8 (18 dB) | 2.25 MHz | 2.2 MHz | 38 dB | 1 km, 5 ms ramp |
| ×32 (30 dB) | 562 kHz | 558 kHz | 50 dB | 3 km, 10 ms ramp (f_IF=100 kHz) |

### Control / Support

```
STM32H723VGT6
  ├── SPI1 ──→ DAC8830 (ramp generation, 16-bit @ 1 µs/step)
  ├── SPI2 ──→ MCP6S91 (AGC gain control)
  ├── SPI2 ──→ ADC3642 (configuration: sample rate, format)
  ├── I2C1 ──→ TMP102 (temperature compensation)
  ├── DCMI ──→ ADC3642 (14-bit parallel data @ 25 MSPS, DMA double-buffer)
  ├── TIM  ──→ ramp trigger (synchronize DAC ramp + ADC capture)
  └── USB/UART ──→ PC (range data, FFT spectrum visualization)

REF5050A (12V in → 5.0 V out) ──→ DAC8830 VREF + VDD
ADP150-5.0 #1 ──→ YSGM556006 VCC (ultra-low noise)
ADP150-5.0 #2 ──→ TLV9062, OPA838, QPL9547, YG802020W VCC
LD1117-3.3 ──→ STM32H723 VDD, ADC3642 digital supply
```

## Power Budgets

### TX Power Budget

| Stage | Input | Gain/Loss | Output |
|-------|-------|-----------|--------|
| VCO | — | — | +6 dBm |
| 6 dB pad | +6 dBm | −6 dB | 0 dBm |
| YG802020W | 0 dBm | +15 dB | +15 dBm |
| GP2X+ | +15 dBm | −3.6 dB | +11.4 dBm (each port) |

TX EIRP with 24 dBi antenna: **+35.4 dBm** (~3.5 kW ERP).

### RX Link Budget — Drone Detection (Pt = +11.4 dBm, Gt = Gr = 24 dBi, NF = 0.8 dB)

| Range | σ (m²) | Target | Pr (dBm) | SNR (single chirp) | SNR (64-chirp integration) | Detectable? |
|-------|--------|--------|----------|--------------------|-----------------------------|-------------|
| 500 m | 0.1 | Medium drone | −107.2 | 36 dB | 54 dB | ✅ |
| 500 m | 0.01 | Small drone | −117.2 | 26 dB | 44 dB | ✅ |
| 1 km | 0.1 | Medium drone | −127.2 | 16 dB | 34 dB | ✅ |
| 1 km | 0.01 | Small drone | −137.2 | 6 dB | 24 dB | ✅ |
| 2 km | 0.1 | Medium drone | −139.2 | 4 dB | 22 dB | ✅ (with integration) |
| 2 km | 0.01 | Small drone | −149.2 | −6 dB | 12 dB | ⚠️ marginal |
| 3 km | 0.1 | Medium drone | −148.7 | −5.5 dB | 12.5 dB | ⚠️ needs integration |
| 3 km | 0.01 | Small drone | −158.7 | −15.5 dB | 2.5 dB | ❌ needs PA upgrade |

**Key insight:** Coherent integration (64 chirps, +18 dB) is **essential** for drone detection. Without it, small drones (σ=0.01 m²) are only detectable to ~1 km. With integration, 2 km is achievable.

**For 3 km small drone detection:** Requires PA upgrade (+20 dBm TX → +8.6 dB improvement) + 64-chirp integration.

Noise floor per 1 kHz bin: −174 + 30 + 0.8 = −143.2 dBm.

### FMCW Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Center frequency | 5.75 GHz | |
| Sweep bandwidth | 500 MHz (5.5–6.0 GHz) | VCO tuning: ~1.2 V → 4.6 V |
| Range resolution | **30 cm** | ΔR = c / (2B) |

### Chirp Configurations

| Config | Ramp T | LPF fc | Max range | f_IF @ 3 km | Samples @ 25 MSPS | FFT | Update rate |
|--------|--------|--------|-----------|-------------|-------------------|-----|-------------|
| **Search (default)** | 10 ms | 194 kHz | 3 km | 100 kHz | 250,000 | 2048-pt | 100 Hz |
| **Track** | 5 ms | 500 kHz | 3 km | 200 kHz | 125,000 | 1024-pt | 200 Hz |
| **Fast** | 1 ms | 4.8 MHz | 600 m | 1 MHz | 25,000 | 256-pt | 1000 Hz |

**Search mode** (10 ms ramp): IF for 3 km target is only 100 kHz — within MCP6S91 ×32 bandwidth (562 kHz). Maximum sensitivity.

**Memory management:** 250K samples at 14 bits = 500 KB — exceeds H723's 320 KB SRAM. Solution: **decimate on-the-fly** in DMA ISR (accumulate + dump every 125th sample) → only 2,000 samples stored per ramp.

### Velocity (Doppler) with Triangular Ramp

Drone velocities: 5–20 m/s (hover to fast flight). Birds: 5–15 m/s. This overlap makes Doppler alone insufficient for drone/bird discrimination — micro-Doppler (rotor blade modulation) is the key differentiator.

| Ramp T | Max unambiguous velocity | Velocity resolution (64 chirps) |
|--------|-------------------------|-------------------------------|
| 10 ms | 1.25 m/s | 0.02 m/s |
| 5 ms | 2.5 m/s | 0.04 m/s |
| 1 ms | 12.5 m/s | 0.2 m/s |

**For drone detection:** Use 5 ms ramp (v_max = 2.5 m/s per chirp pair, covers hovering and slow flight). For fast drones (20 m/s), use 1 ms ramp or unwrap across multiple chirp pairs.

### Drone-Specific Signal Processing

| Technique | Purpose | Implementation |
|-----------|---------|----------------|
| **Coherent integration** (64–256 chirps) | +18 to +24 dB SNR for small RCS | Stack N chirps, 2D FFT (range-Doppler map) |
| **MTI (Moving Target Indication)** | Reject stationary clutter (buildings, ground) | Subtract consecutive range profiles or high-pass filter in Doppler |
| **Micro-Doppler detection** | Distinguish drones from birds (rotor blade modulation at 100–500 Hz) | Longer CPI (coherent processing interval), spectrogram analysis |
| **CFAR detection** | Adaptive threshold for varying clutter environments | Cell-averaging CFAR on range-Doppler map |
| **Track-before-detect** | Detect sub-threshold targets by accumulating energy across scans | Multi-frame integration with motion compensation |

## Key Design Decisions

1. **Open-loop DAC ramp with one-time SA calibration** — VCO frequency vs. tuning voltage measured on spectrum analyzer, stored as LUT in firmware. No PLL.
2. **Temperature compensation via TMP102** — LUT baseline shifted by measured ΔT × drift coefficient (~0.6 MHz/°C ≈ 262 DAC codes/°C at 16-bit).
3. **6 dB resistive pad after VCO** — prevents 9 MHz pp pulling from load mismatch.
4. **Single YG802020W before GP2X+ divider** — one amp drives both TX and LO paths. P1dB ~+16 dBm at 5.5 GHz, output +15 dBm (1 dB below compression). Future: add MMG3H21NT1 PA for +27 dBm.
5. **GP2X+ divider** — guaranteed specs, 3×3 mm, no PCB RF design needed for the split.
6. **QPL9547 as LNA** — 0.6 dB NF gives ~5 dB better sensitivity than YG802020W.
7. **YX18 + 2× NCS4-63+ as mixer** — 1:4 balun provides voltage step-up to overcome 1.4 V diode turn-on at +11.4 dBm LO.
8. **OPA838 ×10 fixed gain (300 MHz GBW)** — 20 dB gain, 30 MHz bandwidth. Not the bandwidth bottleneck. Resistors: R1=1 kΩ, R2=9.09 kΩ.
9. **MCP6S91 SPI AGC (0–30 dB)** — adjusts gain between ramps. At ×32, BW=562 kHz — sufficient for 100 kHz IF at 3 km with 10 ms ramp. Future upgrade: AD603 VGA (90 MHz BW at all gains) if wide BW at high gain is needed.
10. **REF5050A from 12 V rail** — 3 ppm/°C reference for DAC accuracy.
11. **Separate TX/RX antennas (24 dBi)** — avoids TX-RX coupling issues of a shared antenna + circulator. 24 dBi needed for 3 km link budget.
12. **ADC3642IRSBR (dual 14-bit, 25 MSPS)** — +22.7 dB oversampling gain (~17.8 effective bits). Dual channel reserved for future I/Q. Chosen over single-channel ADCs for expansion headroom, over 250 MSPS parts to save power.
13. **STM32H723VGT6 as signal processor** — Cortex-M7 @ 550 MHz with hardware FPU runs 1024-point FFT in ~50 µs. DCMI captures ADC3642 parallel output natively. 320 KB SRAM with on-the-fly decimation.
14. **Single IF channel for prototype 1** — one mixer, one IF chain, ADC channel A only. Triangular ramp (up + down chirp) extracts range and velocity without I/Q. Channel B and second mixer deferred to rev 2.
15. **Configurable ramp time in firmware** — 10 ms for search (3 km, 100 Hz update), 1 ms for tracking (600 m, 1 kHz update). LPF can be swapped (33 pF for 4.8 MHz, 820 pF for 194 kHz) depending on mode.
16. **IF LPF: 1 kΩ + 33 pF (fc ≈ 4.8 MHz)** — passes full IF range for all ramp configs. For search mode only, can swap to 820 pF (194 kHz) for better noise rejection.

## Implementation Tasks

### Phase 0: Documentation

- [ ] Update Lyrion-Radar README.md with: ADC3642IRSBR, STM32H723VGT6, single-IF architecture, 1-3 km range target, 24 dBi antennas, corrected OPA838 GBW (300 MHz), revised IF chain (×10 + MCP6S91 AGC), chirp configurations, link budget at 1-3 km
- [ ] Push to GitHub

### Phase 1: Signal Source (VCO + DAC + Calibration)

- [ ] Solder VCO, DAC8830, TLV9062 buffer, REF5050A, ADP150 LDOs on breakout or PCB
- [ ] Wire SPI from H723 to DAC8830
- [ ] Measure VCO frequency vs. DAC code on spectrum analyzer (100 mV steps, 0–5 V)
- [ ] Build inverse LUT: `dac_lut[1000]` mapping 5500–6000 MHz → DAC codes
- [ ] Generate ramp in firmware, verify linearity on SA
- [ ] Add TMP102, implement temperature compensation offset

### Phase 2: TX Chain

- [ ] Solder 6 dB resistive pad (68Ω + 2×150Ω, 0603 thin-film)
- [ ] Solder YG802020W with EVB-03 matching (68 nH bias, 1000 pF bypass, 1 pF DC block)
- [ ] Solder GP2X+ divider
- [ ] Verify +11.4 dBm on both output ports with SA or power meter
- [ ] Connect TX antenna (24 dBi), verify radiation pattern

### Phase 3: RX Chain + Mixer

- [ ] Solder QPL9547 LNA (follow Qorvo EVB layout)
- [ ] Build YX18 mixer with 2× NCS4-63+ baluns on PCB
  - RF balun: unbalanced port → LNA output, balanced ports → YX18 pins 1,6
  - LO balun: unbalanced port → GP2X+ port 2, balanced ports → YX18 pins 2,5
  - IF output: YX18 pins 3,4 → RC LPF
- [ ] Solder RC LPF (1 kΩ + 33 pF, fc ≈ 4.8 MHz)
- [ ] Solder OPA838 (non-inverting, R1=1 kΩ, R2=9.09 kΩ, G=×10)
- [ ] Solder MCP6S91 (SPI from H723)
- [ ] Verify mixer conversion loss with two signal generators (LO + RF → IF)

### Phase 4: Firmware (STM32H723)

- [ ] CubeMX project for STM32H723VGT6: SPI1, SPI2, I2C1, DCMI, TIM, USB/UART
- [ ] DAC8830 SPI driver + ramp generator (LUT-driven, configurable 1–10 ms period, TIM-triggered)
- [ ] TMP102 I²C driver + temperature compensation (LUT baseline shift)
- [ ] MCP6S91 SPI driver + AGC loop (measure IF peak after FFT → set gain for next ramp)
- [ ] ADC3642 SPI config driver (sample rate, output format, channel select)
- [ ] DCMI + DMA double-buffered capture (25 MSPS, 14-bit, synchronized to ramp via TIM)
- [ ] On-the-fly decimation in DMA ISR (25 MSPS → ~200 kSPS, boxcar average)
- [ ] Range FFT (256–2048 point, CMSIS-DSP `arm_cfft_f32`, magnitude, peak detection)
- [ ] Range calculation: R = f_IF × c × T / (2B)
- [ ] Triangular ramp: up/down chirp pairing for range + velocity extraction
- [ ] Coherent integration: stack 64–256 chirps, 2D FFT (range-Doppler map)
- [ ] MTI: subtract consecutive range profiles to reject stationary clutter
- [ ] CFAR detection on range-Doppler map
- [ ] USB/UART output: range bins, FFT spectrum, AGC state, detected targets
- [ ] Optional: micro-Doppler spectrogram for drone/bird discrimination
- [ ] Optional: second ADC channel for I/Q (future rev, better clutter rejection)

### Phase 5: Integration + Validation

- [ ] End-to-end test: place target at known distance (100 m, 500 m, 1 km), verify range FFT peak
- [ ] Measure range accuracy and resolution
- [ ] Test AGC convergence at various ranges
- [ ] Thermal test: heat/cool board, verify TMP102 compensation
- [ ] Measure TX leakage into RX (antenna isolation)
- [ ] Test coherent integration (64 chirps) — verify +18 dB SNR improvement
- [ ] Drone detection test: fly a small drone (DJI Mini class, σ≈0.01 m²) at 500 m, 1 km, 2 km
- [ ] Verify Doppler extraction: distinguish hovering drone from stationary clutter
- [ ] Bird vs. drone discrimination: compare micro-Doppler signatures

## PCB Layout Guidelines

- **4-layer stackup recommended**: Signal / GND / Power / Signal
- **RF traces**: 50 Ω microstrip, ~3 mm wide on 1.6 mm FR4 (εr ≈ 4.4)
- **VCO placement**: Keep 6 dB pad resistors within 2 mm of VCO RF output pin
- **Ground**: Solid ground plane under all RF components, multiple vias around GND pads
- **Mixer**: NCS4-63+ baluns as close as possible to YX18, symmetric trace lengths
- **Power**: Separate ADP150 for VCO (ultra-low noise), star-ground analog and digital
- **DAC/REF**: Keep REF5050A close to DAC8830 VREF pin, short traces
- **TMP102**: Place within 3 mm of VCO for thermal coupling
- **IF section**: Keep OPA838 and MCP6S91 away from RF section, ground guard between
- **ADC3642**: Place close to STM32H723 DCMI pins, matched-length parallel data traces (< 5 mm skew), separate analog/digital ground connection at ADC pad
- **STM32H723**: Keep DCMI data bus traces short and equal-length; place 100 nF bypass on every VDD pin; separate 3.3 V digital supply from 5 V analog

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| VCO tuning non-linearity after calibration | Range bin spreading | Pre-distorted LUT; re-calibrate if temperature range is wide |
| YX18 mixer conversion loss higher than 7 dB at 5.5 GHz | Reduced sensitivity | Measure on EVB first; increase LO drive if needed (reduce pad to 3 dB) |
| TX-RX antenna coupling saturates LNA | Blind zone at close range | Maximize antenna separation; QPL9547 P1dB = +19 dBm handles leakage |
| YG802020W P1dB only +16 dBm at 5.5 GHz | TX power limited to +15 dBm | Acceptable with 24 dBi antennas; add MMG3H21NT1 PA in rev 2 for +27 dBm |
| GP2X+ VSWR 1.45:1 at 5.5 GHz | Slight power loss | Acceptable; 0.2 dB mismatch loss |
| REF5050A dropout (needs >5.3 V) | No reference | 12 V rail confirmed available |
| ADC3642 parallel interface timing | Data corruption | Use DCMI peripheral; verify with logic analyzer |
| 25 MSPS data rate exceeds H723 SRAM | Dropped samples | On-the-fly decimation in DMA ISR; only store decimated samples |
| 3 km person detection marginal (SNR −5.5 dB) | Miss small targets | 64-chirp coherent integration (+18 dB); PA upgrade in rev 2 |
| MCP6S91 BW at ×32 (562 kHz) limits short-ramp long-range | Can't do 1 ms ramp at 3 km | Use 10 ms ramp for 3 km (f_IF=100 kHz); upgrade to AD603 if needed |
| Drone RCS very small (σ=0.01 m²) | Low SNR at range | Coherent integration (64-256 chirps) is mandatory; PA upgrade for >2 km |
| Bird clutter mimics drone returns | False alarms | Micro-Doppler analysis (rotor modulation 100-500 Hz); I/Q upgrade for better MTI |
| VCO phase noise masks small drone returns | Reduced sensitivity | Free-running VCO PN ~-80 dBc/Hz @ 100 kHz; acceptable for 1-2 km; PLL upgrade if needed |
| H723 SRAM insufficient for 256-chirp integration | Can't stack enough chirps | Process in blocks; stream to external SDRAM via FMC; or reduce to 64 chirps |

## Open Questions (resolve during implementation)

- OPA838 gain: ×10 is the plan; verify with actual mixer IF levels and adjust R2 if needed
- IF LPF: start with 33 pF (4.8 MHz); swap to 820 pF (194 kHz) for search mode if noise is an issue
- Antenna type: 8×8 patch array (~30×30 cm) or parabolic dish (~40 cm) for 24 dBi at 5.75 GHz
- ADC3642 interface: parallel CMOS via DCMI — measure actual throughput on H723
- Decimation filter: start with boxcar (125-tap moving average); upgrade to FIR if sidelobes are a problem
- ADC3642 analog input range: 2 Vpp or 3.5 Vpp — match to MCP6S91 output swing
- FPGA (Artix-7 35T): deferred to rev 2 if dual-channel I/Q or MIMO is needed
- PA upgrade: MMG3H21NT1 (+30 dBm P1dB, $6) or HMC637 (+30 dBm, $15) for rev 2
