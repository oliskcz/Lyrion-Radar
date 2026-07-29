# Lyrion Radar — Front-End Specifications

> **System**: FMCW radar at 5.5–6 GHz for counter-UAS (drone detection)
> **Architecture**: ADF41510 PLL chirp → YSGM556006 VCO → TX/RX analog → AD9643 ADC → XC7A100T FPGA DSP

---

## 1. System Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Frequency band | 5.5 – 6.0 GHz | 500 MHz sweep bandwidth |
| Range resolution | 30 cm (theoretical) | ΔR = c / (2B); in practice limited by FFT size |
| Max range (small drone, σ=0.01 m²) | **~1 km** | 64-chirp coherent integration, prototype 1 |
| Max range (medium drone, σ=0.1 m²) | **~1.5 km** | 64-chirp coherent integration, prototype 1 |
| Path to 2–3 km | PA upgrade + higher-gain antennas | Out of reach with prototype 1 TX power |
| Scan rate | 100 Hz to 10 kHz | Configurable per mode |
| EIRP | +35.4 dBm (3.5 W) | Not ISM — needs experimental license |

---

## 2. Chirp Source (PLL)

| Parameter | Value | Notes |
|-----------|-------|-------|
| PLL | **ADF41510** | Fractional-N, 1–10 GHz |
| VCO | **YSGM556006** | 5320–6060 MHz, +6 dBm, 0–5 V tune |
| Reference | **TCXO 100 MHz** | ±1 ppm |
| PFD frequency | 100 MHz | 1× reference |
| Phase noise (fractional-N) | −231 dBc/Hz @ 100 kHz | Datasheet spec |
| Phase noise (integer-N) | −235 dBc/Hz @ 100 kHz | Datasheet spec |
| Charge pump current | 16 programmable settings | 0.5–7.5 mA |
| Loop filter BW | 100 kHz (search/track) | Active op-amp, 50° phase margin |
| Loop filter BW | 500 kHz (micro-Doppler) | Programmable (switched capacitors) |
| Ramp modes | Sawtooth, triangular | Built-in ramp generator |
| Ramp step resolution | 25-bit fixed / 49-bit variable | < 1 Hz steps |

---

## 3. TX Chain

| Parameter | Value | Notes |
|-----------|-------|-------|
| VCO output | +6 dBm | Direct from YSGM556006 |
| 6 dB π-pad | 0 dBm | 150 Ω / 37.4 Ω / 150 Ω |
| TX driver (YG802020W) | +15 dBm | 93 mA, 1 dB below P1dB |
| Divider (GP2X+) | +11.4 dBm per port | −3.6 dB total loss, 20 dB isolation |
| **TX antenna port** | **+11.4 dBm** | |
| TX antenna gain | 24 dBi | ~30×30 cm patch or 40 cm dish |
| **EIRP** | **+35.4 dBm** (3.5 W ERP) | |
| VCO pulling (at 3:1 VSWR) | < 1 MHz | Mitigated by the π-pad (was 9 MHz without) |

### TX Path Power Levels

```
Stage               Level     Cumulative
──────────────────────────────────────────
VCO output          +6 dBm    +6 dBm
6 dB π-pad          −6 dB     0 dBm
YG802020W PA       +15 dB    +15 dBm
GP2X+ divider      −3.6 dB   +11.4 dBm
TX antenna (24 dBi)  —       +35.4 dBm EIRP
```

---

## 4. RX Chain

| Parameter | Value | Notes |
|-----------|-------|-------|
| RX antenna gain | 24 dBi | Same as TX, separate |
| LNA (QPL9547) | ~10 dB gain, ~1 dB NF | At 5.75 GHz — **verify on NF meter** |
| Balun (NCS4-63+) | 1:4 ratio, −0.5 dB | Single-ended to differential |
| Mixer (YX18) | −7 dB conversion loss | GaAs Schottky diode quad |
| LO drive to mixer | +11.4 dBm | Via 1:4 balun → 2× voltage step-up |
| RC LPF | 4.8 MHz (1 kΩ + 33 pF) | Anti-aliasing |
| IF preamp (OPA838) | ×10 = +20 dB, 300 MHz GBW | 0.9 nV/√Hz |
| AGC (MCP6S91) | ×1 to ×32 = 0 to +30 dB | SPI-controlled, 18 MHz GBW |
| **System NF** | **~1 dB** (estimated) | Verify with QPL9547 band-edge measurement |

### AGC Gain vs Bandwidth

| AGC setting | Gain | Bandwidth | Use case |
|-------------|------|-----------|----------|
| ×1 | +0 dB | 18 MHz | Close targets |
| ×4 | +12 dB | 4.5 MHz | Medium range |
| ×8 | +18 dB | 2.25 MHz | 1 km+ |
| ×32 | +30 dB | 562 kHz | Long range (f_IF must be < 560 kHz) |

### RX Path Gains & Levels (Reference, 1 km, σ=0.01 m² — corrected)

| Stage | Gain/Loss | Level | Voltage (50Ω) |
|-------|-----------|-------|----------------|
| RX antenna (24 dBi) | — | **−151.2 dBm** | 0.65 µV rms |
| QPL9547 LNA | +10 dB | −141.2 dBm | 2.05 µV rms |
| NCS4-63+ balun | −0.5 dB | −141.7 dBm | 1.94 µV rms |
| YX18 mixer | −7 dB | −148.7 dBm | 0.86 µV rms |
| OPA838 (×10) | +20 dB | −128.7 dBm | 8.6 µV rms |
| MCP6S91 (×8) | +18 dB | **−110.7 dBm** | **1.85 µVpp (0.06 LSB of 14-bit ADC)** |
| AD9643 ADC | — | Digitised | |

> **Note**: At 1 km, σ=0.01 m², the signal at the ADC is **0.06 LSBs** of the 14-bit ADC. Single-chirp detection is impossible. With 64-chirp coherent integration (+18 dB), the signal rises to 1.3 LSBs — marginal. This is why the link budget shows ~1 km as the practical limit for small drones.

---

## 5. ADC (AD9643BCPZ-250)

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Model** | **AD9643BCPZ-250** | Analog Devices |
| Resolution | 14 bits | |
| Max sample rate | 250 MSPS | |
| **Channels** | **2** (dual) | Ch A = IF, Ch B = future I/Q |
| Input type | Differential | LVDS output |
| Full-scale input | 2 Vpp or 3.5 Vpp | Configurable via SPI |
| SNR (typical) | 71.1 dBFS @ 96 MHz input | Datasheet |
| SFDR (typical) | 88 dBc @ 96 MHz input | Datasheet |
| Power | 750 mW (both channels) | 1.8 V supply |
| Package | QFN-48 (7×7 mm) | |
| Interface | LVDS DDR | |

### ADC Data Rate to FPGA

| Parameter | Per channel | Both channels |
|-----------|-------------|---------------|
| Sample rate | 250 MSPS | 250 MSPS |
| Resolution | 14 bits | 14 bits |
| **Raw data rate** | **3.5 Gbps** | **7 Gbps** |
| LVDS pairs | 7 (2-wire DDR mode) | 14 total |
| Data clock (DCO) | 250 MHz DDR | 250 MHz DDR |
| Per-pair bit rate | 500 Mbps | 500 Mbps |
| FPGA ISERDES | 8:1 deserialization | 8:1 deserialization |
| Fabric clock after deser | 62.5 MHz | 62.5 MHz |
| Parallel bus width | 56 bits (7 × 8) | 112 bits |
| Usable IF bandwidth | 125 MHz (Nyquist) | 125 MHz per channel |

### LVDS Interface Details

```
ADC Clock:      250 MHz
DCO (DDR):      250 MHz clock, data on both edges → 500 Mtaps/s/pair
LVDS pairs:     7 per channel (14-bit, 2-wire mode)
Total:          14 LVDS pairs → 7 Gbps to FPGA
ISERDES:        8:1  →  8 serial bits → 1 parallel word @ 62.5 MHz
Fabric bus:     7 pairs × 8 bits = 56 bits per channel @ 62.5 MHz
```

---

## 6. Digital Processing (FPGA + MCU)

| Parameter | Value | Notes |
|-----------|-------|-------|
| **FPGA** | **Xilinx XC7A100T** | Artix-7, 324-pin BGA |
| Logic cells | 101,440 | |
| DSP slices | 240 | |
| BRAM | 4,860 Kb | 607.5 KB |
| DDR3 SDRAM | 512 MB (MT41K256M16) | For FFT buffers, data logging |
| **MCU** | **STM32H503** | Cortex-M33 @ 250 MHz |
| MCU SRAM | 128 KB | |
| MCU flash | 256 KB | |
| MCU peripherals | SPI (3×), I²C (2×), UART, USB | PLL, AGC, temp, debug |

### FPGA DSP Pipeline (Estimated Resources)

| Block | LUTs | DSP48 | BRAM (KB) | Latency |
|-------|------|-------|-----------|---------|
| ISERDES + clocking | 2,000 | 0 | 0 | < 5 ns |
| DDC (CORDIC + NCO) | 1,500 | 16 | 4 | 10 clock cycles |
| Decimation CIC (4×) + FIR (4×) | 1,000 | 24 | 32 | 50 clock cycles |
| Window function (Hann) | 500 | 4 | 0 | 1 clock/sample |
| **FFT 16384-pt (1 ch, streaming)** | 10,000 | 48 | **3,600 (DDR)** | 16,384 cycles |
| 2D FFT (64-pt) + DDR ctrl | 1,500 | 0 | 256 (DDR) | 64 clock cycles |
| MTI + CFAR | 2,000 | 8 | 64 | 50 clock cycles |
| Kalman tracking | 1,000 | 4 | 8 | 20 clock cycles |
| **Total used** | **~19,500 (31%)** | **~104 (43%)** | **~3,960 (82% of 4,860 Kb)** | |
| **Available** | 63,400 | 240 | 4,860 Kb | |

> **Note on FFT size**: Large FFTs (>8K points) exceed on-chip BRAM and must use external DDR3 SDRAM for intermediate twiddle factors and data storage. The XC7A100T is sized for 2-channel I/Q at FFT sizes up to ~8192 per channel.

---

## 7. Chirp Modes (Corrected)

| Mode | Ramp T | ADC rate (post-decimate) | Samples/ramp | **FFT size** | Resolution per bin | Max range | Scan rate | v_max (sawtooth, I/Q) |
|------|--------|--------------------------|--------------|--------------|---------------------|-----------|-----------|----------------------|
| **Search** | 10 ms | 3.9 MSPS | 39,000 | **16384** | **0.71 m** | 1.5 km | 100 Hz | 2.6 m/s |
| **Track** | 5 ms | 7.8 MSPS | 39,000 | **16384** | **0.71 m** | 1.5 km | 200 Hz | 5.2 m/s |
| **Fast** | 1 ms | 31.25 MSPS | 31,250 | **8192** | **1.14 m** | 500 m | 1 kHz | 26 m/s |
| **Micro-Doppler** | 0.1 ms | 250 MSPS | 25,000 | **4096** | **1.83 m** | 50 m | 10 kHz | 261 m/s |

**v_max formula** (for sawtooth with I/Q sampling): `v_max = λ × f_PRF / 2`
- For real sampling only (prototype 1, single AD9643 channel): halve these values
- For triangular modulation: halve these values (full period = 2× ramp time)

**Key relationship**: `N_FFT ≤ N_samples = T_ramp × f_sample`. A smaller FFT gives coarser range resolution. The theoretical 0.3 m resolution is achieved only when N_FFT = N_samples.

---

## 8. Link Budget (Corrected — 12 dB error fixed)

Conditions: Pt = +11.4 dBm, Gt = Gr = 24 dBi, NF = 1 dB, noise floor = −143 dBm per 1 kHz bin.

| Range | RCS | Target type | **Pr (dBm)** | SNR (1 chirp) | **SNR (64-chirp coherent)** |
|-------|-----|-------------|--------------|----------------|------------------------------|
| 500 m | 0.1 m² | Medium drone | −129.2 | 14 dB | **32 dB** ✅ |
| **500 m** | **0.01 m²** | **Small drone** | **−139.2** | **4 dB** | **22 dB** ✅ |
| 1 km | 0.1 m² | Medium drone | −141.2 | 2 dB | **20 dB** ✅ |
| **1 km** | **0.01 m²** | **Small drone** | **−151.2** | **−8 dB** | **10 dB** ⚠️ marginal |
| 1.5 km | 0.1 m² | Medium drone | −148.3 | −5 dB | **13 dB** ✅ |
| 1.5 km | 0.01 m² | Small drone | −158.3 | −15 dB | **3 dB** ⚠️ |
| 2 km | 0.1 m² | Medium drone | −153.3 | −10 dB | **8 dB** ⚠️ |
| 2 km | 0.01 m² | Small drone | −163.3 | −20 dB | **−2 dB** ❌ |
| 3 km | 0.1 m² | Medium drone | −160.3 | −17 dB | **1 dB** ⚠️ |

### Honest range targets for prototype 1

| Target | Max range | Condition |
|--------|-----------|-----------|
| Small drone (σ=0.01 m²) | **~1 km** | With 64-chirp coherent integration, +18 dB |
| Medium drone (σ=0.1 m²) | **~1.5 km** | With 64-chirp coherent integration |
| Large drone (σ=1 m²) | **~3 km** | With 64-chirp coherent integration |

> **Conclusion**: The earlier "2-3 km for small drones" claim is **not supported** by the math. The 12 dB link budget error (from using `10·log10(λ)` instead of `20·log10(λ²)` in the radar equation) inflated every range by ~40%. To reach 2-3 km for small drones, you need a real PA upgrade (MMG3H21NT1, +27 dBm TX → +12 dB link improvement) and/or 30 dBi antennas (+6 dB) and/or 256-chirp integration (+6 dB).

---

## 9. Power Supply

| Rail | Voltage | Current (est.) | Used by |
|------|---------|---------------|---------|
| VCO supply (ADP150) | 5.0 V | 100 mA | YSGM556006 VCO |
| Analog supply | 5.0 V | 200 mA | Op-amps, LNA, PA |
| FPGA core (LTM4644) | 1.0 V | 1.5 A | XC7A100T VCCINT |
| FPGA aux | 1.8 V | 500 mA | XC7A100T VCCAUX |
| FPGA I/O | 3.3 V | 300 mA | XC7A100T VCCO |
| ADC analog | 1.8 V | 400 mA | AD9643 AVDD |
| ADC digital | 3.3 V | 100 mA | AD9643 DRVDD |
| MCU | 3.3 V | 100 mA | STM32H503 |
| DDR3 VDD | 1.5 V | 500 mA | MT41K256M16 |
| DDR3 VTT | 0.75 V | 200 mA | DDR termination |
| PLL | 3.3 V | 100 mA | ADF41510 |
| **Total** | — | **~4 A** | — |

---

## 10. Regulatory & Licensing

| Issue | Status | Required action |
|-------|--------|----------------|
| +35.4 dBm EIRP at 5.5–6 GHz | **Not ISM** | Experimental or STA license needed |
| UNII/DFS band incursion | Possible | Check local frequency allocation |
| RF exposure at 24 dBi | Significant | Safe stand-off distance required |

---

## 11. Verification Notes

This document was independently reviewed and several errors were found and corrected:

| Error | Original | Corrected | Impact |
|-------|----------|-----------|--------|
| **FFT sizes too small** | 4096/2048/512/256 | 16384/16384/8192/4096 | Range resolution was 2.85 m instead of 0.71 m |
| **Link budget off by +12 dB** | 1 km small drone = 22 dB SNR | 1 km small drone = 10 dB SNR | Max range overestimated by ~40% |
| **v_max too low by 2×** | 0.63 m/s at 10 ms | 2.6 m/s at 10 ms (sawtooth, I/Q) | Velocity coverage was half of actual |
| **RX path voltage at ADC** | 4.6 mVpp at 1 km | 1.85 µVpp at 1 km | Off by 2500× — the signal is 0.06 LSBs, not 50% full scale |

**Root cause of the link budget error**: I used `10·log10(λ) = −12.8 dB` instead of `20·log10(λ) = −25.65 dB` (which equals `10·log10(λ²)`) in the radar equation. This `+12.8 dB` error carried through every Pr and SNR value.

**Root cause of the FFT size error**: I forgot that `N_FFT ≤ N_samples = T_ramp × f_sample`. With 39,000 samples per ramp, a 4096-pt FFT only covers ~10% of the available frequency bins, throwing away resolution.

**Root cause of the v_max error**: I used `λ/(8T)` instead of `λ/(4T)` for sawtooth real sampling. The correct formula is `λ × f_PRF / 4` (real) or `λ × f_PRF / 2` (I/Q).

**The corrected values show that prototype 1 is a 1 km-class radar for small drones, not 2-3 km**. To reach 2-3 km for small drones, the design needs: a real PA (MMG3H21NT1, +27 dBm TX), higher-gain antennas (30 dBi), and/or 256-chirp integration.

---

*Specification version 1.1 · Lyrion Radar · MIT License*
