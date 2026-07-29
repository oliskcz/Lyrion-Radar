# FMCW Radar Front-End — 5.5–6 GHz

## Goal

Design and build a complete FMCW radar front-end at 5.5–6 GHz for **drone detection (counter-UAS)**. Open-loop DAC-ramped VCO, diode-ring mixer RX, external 14-bit ADC, STM32H723-based signal processing.

**Realistic prototype 1 targets:**
- ~500 m to 1 km for small drones (σ = 0.01 m²)
- ~1–1.5 km for medium drones (σ = 0.1 m²)
- ~30 cm range resolution using 24 dBi antennas
- Scan rate 5–100 Hz depending on mode

**Path to 2–3 km range:** Requires real PA upgrade (MMG3H21NT1, +27 dBm TX), proven coherent integration (or PLL), and chirp rate that supports micro-Doppler.

### Drone RCS Reference

| Drone class | Example | RCS (σ) | dBsm |
|-------------|---------|---------|------|
| Micro (< 250 g) | DJI Mini 4 | 0.001–0.005 m² | −30 to −23 dBsm |
| Small (250 g – 2 kg) | DJI Phantom 4 | 0.01–0.05 m² | −20 to −13 dBsm |
| Medium (2–25 kg) | DJI Matrice 300 | 0.05–0.1 m² | −13 to −10 dBsm |
| Large (> 25 kg) | Fixed-wing UAV | 0.1–1 m² | −10 to 0 dBsm |

**Prototype 1**: Single IF channel (one mixer, one IF chain, ADC channel A). Channel B reserved for future I/Q or second RX antenna. Triangular ramp extracts range + velocity.
**Future rev**: Dual-channel I/Q (90° hybrid + second mixer) for better clutter rejection and MTI, more powerful PA (MMG3H21NT1 or HMC637), optionally Xilinx Artix-7 35T FPGA for hardware DDC/FFT.

---

## Confirmed BOM (Corrected)

| # | Part Number | Role | Qty | Key Specs |
|---|-------------|------|-----|-----------|
| 1 | **YSGM556006** (Innotion) | VCO | 1 | 5320–6060 MHz, +6 dBm, 0–5 V tune, 14 mA |
| 2 | **DAC8830IDR** (TI) | 16-bit SPI DAC | 1 | 1 µs settling, unbuffered R-2R, SOIC-8 |
| 3 | **REF5050AIDGKR** (TI) | DAC voltage reference | 1 | 5.0 V, 3 ppm/°C, needs >5.3 V input (12 V rail available) |
| 4 | **TLV9062IDR** (TI) | DAC output buffer | 1 | Dual op-amp, rail-to-rail, 10 MHz GBW, SOIC-8. Ch A = DAC buffer, Ch B = spare |
| 5 | **TMP102AIDRLR** (TI) | Temperature sensor | 1 | I²C, ±0.5 °C, SOT-563 |
| 6 | **YG802020W** (Innotion) | TX driver amplifier | 1 | 50 MHz–8 GHz, +15 dB gain @ 5.5 GHz (93 mA), P1dB ~+16 dBm @ 5.5 GHz, DFN 2×2 |
| 7 | **GP2X+** (Mini-Circuits) | 2-way power divider | 1 | 2.9–6.2 GHz, 3.6 dB total loss, 19–21 dB isolation, 3×3 mm QFN |
| 8 | **QPL9547TR7** (Qorvo) | RX LNA | 1 | 0.05–6 GHz. **At 5.75 GHz: ~1 dB NF, ~10 dB gain** (datasheet headline 0.6 dB / 17 dB is mid-band ~2 GHz — **verify on VNA/NF meter**), +19 dBm P1dB, DFN 2×2 |
| 9 | **YX18** (Innotion) | Mixer diode quad | 1 | GaAs Schottky cross-over quad, 1.4 V turn-on, DFN 2×2 |
| 10 | **NCS4-63+** (Mini-Circuits) | Mixer baluns | 2 | 4.5–6 GHz, 1:4 ratio, 0.5 dB amplitude unbalance, 0805 |
| 11 | **OPA838IDBVR** (TI) | IF preamplifier | 1 | 0.9 nV/√Hz, **300 MHz GBW**, 1 mA supply, SOT-23-5 |
| 12 | **MCP6S91T-E/MS** (Microchip) | IF PGA (AGC) | 1 | 1×–32× (0–30 dB), 18 MHz GBW, SPI, MSOP-8 |
| 13 | **ADC3642IRSBR** (TI) | External ADC | 1 | Dual 14-bit, 25 MSPS, ultra-low power, QFN-40. Ch A = IF, Ch B = future I/Q |
| 14 | **STM32H723VGT6** (ST) | MCU / signal processor | 1 | Cortex-M7 @ 550 MHz, HW FPU, **564 KB SRAM**, DCMI, LQFP-100 |
| 15 | **ADP150AUJZ-5.0** (ADI) | Ultra-low noise LDO | 2 | 5 V out, <10 µVrms noise. One for VCO VCC, one for analog/RF |
| 16 | **LD1117S33** or similar | 3.3 V LDO for digital | 1 | STM32H723, ADC3642 digital supply |
| 17 | Resistors: **150 Ω, 37.4 Ω, 150 Ω** | 6 dB resistive **π-pad** (corrected) | 3 | 0603 thin-film, after VCO output |
| 18 | Resistors: 1 kΩ, 9.09 kΩ | OPA838 gain setting (×10) | 2 | 0603, E96 values |
| 19 | Caps: 100 nF ×6, 10 µF ×2, 33 pF ×1, 10 nF ×2, 1000 pF ×2, 1 pF ×2 | Bypass, DC block, LPF, matching | ~15 | 0603/0805 |
| 20 | Inductors: 68 nH ×2, 2.2 nH ×1 | YG802020W bias feed, matching | 3 | 0603 |
| 21 | I²C pull-ups: 4.7 kΩ ×2 | TMP102 bus | 2 | 0603 |
| 22 | Antennas: **24 dBi** ×2 | TX and RX (separate) | 2 | 5.5–6 GHz, ~5° beamwidth, patch array or dish |

**Total active component cost: ~$40** (excluding antennas, PCB, VCOs/diodes/baluns already owned)

---

## Signal Chain

### TX Path

```
YSGM556006 VCO (+6 dBm)
    │
  100 nF DC block
    │
  6 dB π-pad (150 Ω / 37.4 Ω / 150 Ω) → 0 dBm
    │
  YG802020W (G=+15 dB @ 93 mA, Rex=open) → +15 dBm
    │
  GP2X+ divider (−3.6 dB) → +11.4 dBm per port
    │
    ├── Port 1 → TX antenna (24 dBi) → +35.4 dBm EIRP (3.5 W ERP)
    │
    └── Port 2 → NCS4-63+ balun → YX18 mixer LO port
```

**Note:** YG802020W P1dB is ~+16 dBm at 5.5 GHz. Output of +15 dBm is 1 dB below compression — acceptable. **Regulatory:** +35.4 dBm EIRP at 5.5–6 GHz is not ISM — requires licensed or experimental authorization in most jurisdictions.

### RX Path

```
RX antenna (24 dBi) — target echoes: −107 to −159 dBm depending on range/RCS
    │
  QPL9547TR7 LNA (G=~10 dB, NF=~1 dB at 5.75 GHz — verify)
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
  STM32H723 (DMA capture → on-the-fly decimate → FFT → detection)
```

### IF Gain/Bandwidth at AGC Settings

| MCP6S91 gain | MCP6S91 BW | Combined BW (with OPA838 ×10) | Total gain | Max usable IF | Use case |
|-------------|-----------|-------------------------------|------------|---------------|----------|
| ×1 (0 dB) | 18 MHz | 7.2 MHz | 20 dB | 7.2 MHz | Close targets, strong signal |
| ×4 (12 dB) | 4.5 MHz | 3.6 MHz | 32 dB | 3.6 MHz | 100–500 m, 1 ms ramp |
| ×8 (18 dB) | 2.25 MHz | 2.2 MHz | 38 dB | 2.2 MHz | 1 km, 1 ms ramp |
| ×32 (30 dB) | 562 kHz | 558 kHz | 50 dB | **558 kHz** | **1.5 km, 5 ms ramp (f_IF=1 MHz — outside BW!)** |

**Critical:** The ×32 AGC setting cannot pass the IF for targets beyond ~1.5 km with 5 ms ramp. This is a hard bandwidth limit. Reduce AGC gain for long-range detection.

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

---

## Power Budgets

### TX Power Budget

| Stage | Input | Gain/Loss | Output |
|-------|-------|-----------|--------|
| VCO | — | — | +6 dBm |
| 6 dB π-pad (150/37.4/150) | +6 dBm | −6 dB | 0 dBm |
| YG802020W | 0 dBm | +15 dB | +15 dBm |
| GP2X+ | +15 dBm | −3.6 dB | +11.4 dBm (each port) |

TX EIRP with 24 dBi antenna: **+35.4 dBm** = **3.5 W ERP** (not "kW" — corrected).

### RX Link Budget — Drone Detection (Corrected)

Conditions: Pt = +11.4 dBm, Gt = Gr = 24 dBi, NF = 1 dB (QPL9547 at band edge), noise floor = −143 dBm (1 kHz bin).

**Radar equation uses R⁴. From 1 km to 500 m you gain 12 dB, not 20 dB.**

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

- **Noncoherent** integration: gain ≈ 5·log₁₀(N) = +9 dB for N=64. **Realistic for open-loop VCO.**
- **Coherent** integration: gain = 10·log₁₀(N) = +18 dB for N=64. **Requires chirp-to-chirp phase coherence** — unproven with this design.

**Honest target for prototype 1:** ~500 m to 1 km for small drones, ~1.5 km for medium drones — with the caveat that this assumes noncoherent integration. Achieving 2–3 km requires either (a) proven coherent integration with a stable VCO, or (b) a real PA (MMG3H21NT1, +27 dBm TX → +15.6 dB improvement).

---

## Chirp Configurations (Corrected)

### IF frequency as a function of range and ramp time

**f_IF = 2BR / (cT) — verify with this formula every time you change a parameter.**

| Range | T=1 ms | T=5 ms | T=10 ms |
|-------|--------|--------|---------|
| 100 m | 333 kHz | 66.7 kHz | 33.3 kHz |
| 500 m | 1.67 MHz | 333 kHz | 167 kHz |
| 1 km | **3.33 MHz** | 667 kHz | 333 kHz |
| 1.5 km | 5 MHz | **1 MHz** | 500 kHz |
| 2 km | 6.67 MHz | 1.33 MHz | 667 kHz |
| 3 km | 10 MHz | **2 MHz** | **1 MHz** |

### Configurations

| Mode | Ramp T | LPF fc | Max range | f_IF @ max range | Samples (25 MSPS) | FFT | Scan rate | Max velocity |
|------|--------|--------|-----------|-------------------|-------------------|-----|-----------|-------------|
| **Search** | 10 ms | 500 kHz | 1.5 km | 500 kHz | 250,000 → decimate to 5,000 | 4096-pt | **5–10 Hz** (after 64-chirp integration) | 0.625 m/s |
| **Track** | 5 ms | 1 MHz | 1.5 km | 1 MHz | 125,000 → decimate to 2,500 | 2048-pt | 10–20 Hz | 1.25 m/s |
| **Fast** | 1 ms | 5 MHz | 500 m | 1.67 MHz | 25,000 | 512-pt | 50–100 Hz | 6.25 m/s |

**Important constraints:**

- **Scan rate vs integration**: 64-chirp coherent integration at 10 ms ramp = 0.64 s per integrated frame → ~1.5 Hz. Noncoherent (no phase preservation) lets you trade integration for scan rate.
- **Micro-Doppler vs long ramp**: Rotor modulation at 100–500 Hz needs chirp PRF ≳ 1 kHz. Search mode at 10 ms only gives 50–100 Hz PRF. **Micro-Doppler drone/bird discrimination is not feasible in Search or Track modes.**
- **MCP6S91 bandwidth**: At ×32 (562 kHz BW), the IF for a 1.5 km target with 5 ms ramp is 1 MHz — **does not fit**. Reduce AGC gain or use 1.5 km max range.
- **Memory**: 250K samples × 2 bytes = 500 KB. 564 KB SRAM total. **Decimate on-the-fly** in DMA ISR (accumulate + dump every Nth sample) → only 5,000 samples stored per ramp.

### Velocity (Doppler) with Triangular Ramp

**v_max ≈ λ / (4T)** — commonly miscalculated:

| Ramp time | Max unambiguous velocity | Drones covered |
|-----------|--------------------------|----------------|
| 10 ms | **0.625 m/s** | Hovering only |
| 5 ms | 1.25 m/s | Slow drift (< 4.5 km/h) |
| 1 ms | 6.25 m/s | Hovering to 22 km/h |

**Critical:** Small drones routinely fly 5–20 m/s. At 10 ms ramp, v_max = 0.625 m/s — a drone flying at 5 m/s will alias and appear as a false stationary or slow target. **Search mode at 10 ms is not usable for moving drones.** Use Fast mode (1 ms, 6.25 m/s) for detection, then track.

### Drone-Specific Signal Processing

| Technique | Purpose | Status |
|-----------|---------|--------|
| **Coherent integration** (64 chirps) | +18 dB SNR | **Unverified** with open-loop VCO. Noncoherent (~+9 dB) is the safe assumption. |
| **MTI** | Reject stationary clutter | Implemented as range-Doppler subtraction |
| **Micro-Doppler** (rotor blade at 100–500 Hz) | Drone vs. bird discrimination | **Not feasible in Search/Track modes** (PRF too low). Would need a separate high-PRF discriminator mode. |
| **CFAR** | Adaptive detection in clutter | Cell-averaging CFAR on range-Doppler map |
| **Velocity aliasing handling** | Drones fly 5–20 m/s | Detect aliases in the up/down chirp pair and unwrap |

---

## Key Design Decisions

1. **Open-loop DAC ramp with one-time SA calibration** — VCO frequency vs. tuning voltage measured on spectrum analyzer, stored as LUT in firmware. No PLL. **Coherence unproven — integration is likely noncoherent (~+9 dB for 64 chirps).**
2. **Temperature compensation via TMP102** — LUT baseline shifted by measured ΔT × drift coefficient (~0.6 MHz/°C ≈ 262 DAC codes/°C at 16-bit).
3. **6 dB resistive π-pad after VCO (corrected values: 150/37.4/150 Ω)** — prevents 9 MHz pp pulling from load mismatch. Earlier 150/68/150 was wrong.
4. **Single YG802020W before GP2X+ divider** — one amp drives both TX and LO paths. P1dB ~+16 dBm at 5.5 GHz, output +15 dBm (1 dB below compression). Future: add MMG3H21NT1 PA for +27 dBm.
5. **GP2X+ divider** — guaranteed specs, 3×3 mm, no PCB RF design needed for the split.
6. **QPL9547 as LNA** — **At 5.75 GHz: ~1 dB NF, ~10 dB gain (verify).** Datasheet 0.6 dB / 17 dB is mid-band.
7. **YX18 + 2× NCS4-63+ as mixer** — 1:4 balun provides voltage step-up to overcome 1.4 V diode turn-on at +11.4 dBm LO.
8. **OPA838 ×10 fixed gain (300 MHz GBW)** — 20 dB gain, 30 MHz bandwidth. Not the bandwidth bottleneck. Resistors: R1=1 kΩ, R2=9.09 kΩ.
9. **MCP6S91 SPI AGC (0–30 dB)** — adjusts gain between ramps. **At ×32, BW=562 kHz — cannot pass IF > 558 kHz.** Reduces max range at high gain.
10. **REF5050A from 12 V rail** — 3 ppm/°C reference for DAC accuracy.
11. **Separate TX/RX antennas (24 dBi)** — avoids TX-RX coupling issues of a shared antenna + circulator. 24 dBi needed for ~1 km small-drone detection.
12. **ADC3642IRSBR (dual 14-bit, 25 MSPS)** — oversampled vs IF. Dual channel reserved for future I/Q. **Effective-bits claim (~17.8) is theoretical — depends on ADC linearity, jitter, and decimation filter design.**
13. **STM32H723VGT6 as signal processor** — Cortex-M7 @ 550 MHz with single-precision FPU runs 1024-point FFT in ~50 µs. DCMI captures ADC3642 parallel output natively. **564 KB SRAM** with on-the-fly decimation.
14. **Single IF channel for prototype 1** — one mixer, one IF chain, ADC channel A only. Triangular ramp (up + down chirp) extracts range and velocity without I/Q. Channel B and second mixer deferred to rev 2.
15. **Configurable ramp time in firmware** — 10 ms for search (1.5 km, 5–10 Hz scan), 1 ms for fast detection (500 m, 50–100 Hz scan). LPF: 33 pF (4.8 MHz) for fast, 820 pF (194 kHz) for search.
16. **IF LPF: 1 kΩ + 33 pF (fc ≈ 4.8 MHz)** — passes full IF range for fast mode. For search mode, swap to 820 pF (194 kHz) for better noise rejection.

---

## Implementation Tasks

### Phase 0: Documentation

- [x] Update Lyrion-Radar README.md with honest range claims, corrected f_IF table, R⁴ scaling, corrected pad values, QPL9547 band-edge caveat, regulatory note
- [x] Update PLAN.md with same corrections

### Phase 1: Signal Source (VCO + DAC + Calibration)

- [ ] Solder VCO, DAC8830, TLV9062 buffer, REF5050A, ADP150 LDOs on breakout or PCB
- [ ] Wire SPI from H723 to DAC8830
- [ ] Measure VCO frequency vs. DAC code on spectrum analyzer (100 mV steps, 0–5 V)
- [ ] Build inverse LUT: `dac_lut[1000]` mapping 5500–6000 MHz → DAC codes
- [ ] Measure VCO phase noise and residual FM (critical for coherent integration viability)
- [ ] Generate ramp in firmware, verify linearity on SA
- [ ] Add TMP102, implement temperature compensation offset

### Phase 2: TX Chain

- [ ] Solder 6 dB π-pad (**150 Ω / 37.4 Ω / 150 Ω**, corrected)
- [ ] Solder YG802020W with EVB-03 matching (68 nH bias, 1000 pF bypass, 1 pF DC block)
- [ ] Solder GP2X+ divider
- [ ] Verify +11.4 dBm on both output ports with SA or power meter
- [ ] Connect TX antenna (24 dBi), verify radiation pattern

### Phase 3: RX Chain + Mixer

- [ ] **Measure QPL9547 NF and gain at 5.75 GHz** (noise figure meter + VNA)
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
- [ ] Range FFT (1024–4096 point, CMSIS-DSP `arm_cfft_f32`, single-precision, magnitude, peak detection)
- [ ] Range calculation: R = f_IF × c × T / (2B)
- [ ] Triangular ramp: up/down chirp pairing for range + velocity extraction
- [ ] **Coherent vs noncoherent integration** — test both, measure actual SNR improvement
- [ ] MTI: subtract consecutive range profiles to reject stationary clutter
- [ ] CFAR detection on range-Doppler map
- [ ] Velocity aliasing detection and unwrapping
- [ ] USB/UART output: range bins, FFT spectrum, AGC state, detected targets
- [ ] Optional: micro-Doppler spectrogram (separate high-PRF mode)
- [ ] Optional: second ADC channel for I/Q (future rev, better clutter rejection)

### Phase 5: Integration + Validation

- [ ] End-to-end test: place target at known distance (100 m, 500 m, 1 km), verify range FFT peak
- [ ] Measure range accuracy and resolution
- [ ] Test AGC convergence at various ranges
- [ ] Thermal test: heat/cool board, verify TMP102 compensation
- [ ] Measure TX leakage into RX (antenna isolation)
- [ ] **Measure integration gain**: 1 chirp vs 64 chirps SNR — is it +9 dB (noncoherent) or +18 dB (coherent)?
- [ ] Drone detection test: fly a small drone (DJI Mini class, σ≈0.01 m²) at 500 m, 1 km
- [ ] Verify Doppler extraction: distinguish moving drone from stationary clutter
- [ ] Bird vs. drone discrimination: if feasible with current chirp rate

---

## PCB Layout Guidelines

- **4-layer stackup recommended**: Signal / GND / Power / Signal
- **RF traces**: 50 Ω microstrip, ~3 mm wide on 1.6 mm FR4 (εr ≈ 4.4). **Note: FR4 at 5.5–6 GHz has higher loss and εr tolerance than at lower frequencies. Rogers/RF-35 preferred for RF path in production.**
- **VCO placement**: Keep 6 dB π-pad resistors within 2 mm of VCO RF output pin
- **Ground**: Solid ground plane under all RF components, multiple vias around GND pads
- **Mixer**: NCS4-63+ baluns as close as possible to YX18, symmetric trace lengths
- **Power**: Separate ADP150 for VCO (ultra-low noise), star-ground analog and digital
- **DAC/REF**: Keep REF5050A close to DAC8830 VREF pin, short traces
- **TMP102**: Place within 3 mm of VCO for thermal coupling
- **IF section**: Keep OPA838 and MCP6S91 away from RF section, ground guard between
- **ADC3642**: Place close to STM32H723 DCMI pins, matched-length parallel data traces (< 5 mm skew), separate analog/digital ground connection at ADC pad
- **STM32H723**: Keep DCMI data bus traces short and equal-length; place 100 nF bypass on every VDD pin; separate 3.3 V digital supply from 5 V analog. **DCMI framing (HSYNC/VSYNC/embedded sync) needs careful handling — not plug-and-play.**

---

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
| **Open-loop VCO phase incoherence** | Integration gain only +9 dB (noncoherent) not +18 dB (coherent) | Measure residual FM/PN; if too poor, add PLL (ADF4158) in rev 2 |
| **QPL9547 NF at 5.75 GHz unverified** | System NF may be 1.5–2 dB, not 1 dB | Measure on NF meter; add 2 dB margin to all range claims |
| **f_IF at 1.5 km with 5 ms ramp = 1 MHz** | Exceeds MCP6S91 ×32 BW (562 kHz) | Use 1.5 km max range or lower AGC gain |
| **v_max at 10 ms = 0.625 m/s** | Drones alias | Use Fast mode (1 ms) for detection; triangular pairing has same limit |
| **Micro-Doppler infeasible at long ramp** | Can't discriminate drone vs. bird at 1+ km | Add separate high-PRF discriminator mode (future) |
| **+35 dBm EIRP not ISM** | Regulatory issue | Acquire experimental/STA license; or reduce EIRP |
| **FR4 at 5.5–6 GHz** | Higher insertion loss, εr tolerance | Acceptable for prototype; Rogers for production |
| **Decimation aliasing** | IF above 100 kHz aliases to 200 kSPS Nyquist | 4.8 MHz LPF + AGC bandwidth limits handle this — verify with spectrum |
| H723 SRAM (564 KB) borderline for 4096-pt × 64-chirp 2D FFT | Memory pressure | Process in blocks; or use 64-pt × 1024-range instead of 4096-pt × 256-range |

---

## Open Questions (resolve during implementation)

- **QPL9547 NF/gain at 5.75 GHz**: **MUST measure** on noise-figure meter before relying on range claims
- **VCO residual FM/PN**: **MUST measure** to determine if coherent integration is viable
- **OPA838 gain**: ×10 is the plan. Verify with actual mixer IF levels.
- **IF LPF**: start with 33 pF (4.8 MHz). May need to adjust for decimation anti-aliasing.
- **Antenna type**: 8×8 patch array (~30×30 cm) or parabolic dish (~40 cm)
- **ADC3642 input range**: 2 Vpp or 3.5 Vpp — configure via SPI
- **Decimation filter**: boxcar vs. FIR
- **Coherent vs noncoherent integration**: measure on actual hardware
- **Future**: PA upgrade (MMG3H21NT1, +27 dBm TX), I/Q upgrade, FPGA offload, PLL for coherent integration, high-PRF micro-Doppler mode
