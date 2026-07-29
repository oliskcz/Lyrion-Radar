# Lyrion Radar — Front-End Specifications

> **System**: FMCW radar at 5.5–6 GHz for counter-UAS (drone detection)
> **Architecture**: ADF41510 PLL chirp → YSGM556006 VCO → TX/RX analog → AD9643 ADC → XC7A100T FPGA DSP

---

## 1. System Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Frequency band | 5.5 – 6.0 GHz | 500 MHz sweep bandwidth |
| Range resolution | 30 cm | ΔR = c / (2B) |
| Max range (small drone, σ=0.01 m²) | ~2 km | With 64-chirp coherent integration |
| Max range (medium drone, σ=0.1 m²) | ~3 km | With 64-chirp coherent integration |
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

### RX Path Gains & Levels (Reference)

| Stage | Gain/Loss | Level @ 1 km, σ=0.01 m² |
|-------|-----------|--------------------------|
| RX antenna (24 dBi) | — | −139.2 dBm |
| QPL9547 LNA | +10 dB | −129.2 dBm |
| NCS4-63+ balun | −0.5 dB | −129.7 dBm |
| YX18 mixer | −7 dB | −136.7 dBm |
| OPA838 (×10) | +20 dB | −116.7 dBm |
| MCP6S91 (×8) | +18 dB | −98.7 dBm (4.6 mVpp) |
| AD9643 ADC | — | Digitised |

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
| FFT 1024-pt (pipelined) | 3,000 | 20 | 32 | 1024 clock cycles |
| 2D FFT (64-pt) + DDR ctrl | 1,500 | 0 | 256 (DDR) | 64 clock cycles |
| MTI + CFAR | 2,000 | 8 | 64 | 50 clock cycles |
| Kalman tracking | 1,000 | 4 | 8 | 20 clock cycles |
| **Total used** | **~12,500 (20%)** | **~76 (32%)** | **~132 (3%)** | |
| **Available** | 63,400 | 240 | 4,860 Kb | |

---

## 7. Chirp Modes

| Mode | Ramp time | ADC decimated rate | Max range | f_IF @ max | FFT size | Scan rate | PRF | v_max |
|------|-----------|-------------------|-----------|------------|----------|-----------|-----|-------|
| **Search** | 10 ms | 3.9 MSPS | 1.5 km | 500 kHz | 4096-pt | 100 Hz | 100 Hz | 0.63 m/s |
| **Track** | 5 ms | 7.8 MSPS | 1.5 km | 1 MHz | 2048-pt | 200 Hz | 200 Hz | 1.25 m/s |
| **Fast** | 1 ms | 31.25 MSPS | 500 m | 1.67 MHz | 512-pt | 1 kHz | 1 kHz | 6.25 m/s |
| **Micro-Doppler** | 0.1 ms | 250 MSPS | 50 m | 1.67 MHz | 256-pt | 10 kHz | 10 kHz | 62.5 m/s |

---

## 8. Link Budget Summary

| Range | RCS | Target type | Pr (dBm) | SNR (1 chirp) | **SNR (64-chirp coherent)** |
|-------|-----|-------------|----------|----------------|------------------------------|
| **500 m** | 0.1 m² | Medium drone | −117.2 | 26 dB | **44 dB** ✅ |
| **500 m** | **0.01 m²** | **Small drone** | **−127.2** | **16 dB** | **34 dB** ✅ |
| **1 km** | 0.1 m² | Medium drone | −129.2 | 14 dB | **32 dB** ✅ |
| **1 km** | **0.01 m²** | **Small drone** | **−139.2** | **4 dB** | **22 dB** ✅ |
| **1.5 km** | 0.1 m² | Medium drone | −136.2 | 7 dB | **25 dB** ✅ |
| 1.5 km | 0.01 m² | Small drone | −146.2 | −3 dB | **15 dB** ✅ |
| **2 km** | 0.1 m² | Medium drone | −141.2 | 2 dB | **20 dB** ✅ |
| 2 km | 0.01 m² | Small drone | −151.2 | −8 dB | **10 dB** ⚠️ |
| 3 km | 0.1 m² | Medium drone | −148.7 | −6 dB | **12 dB** ⚠️ |

Conditions: Pt = +11.4 dBm, Gt = Gr = 24 dBi, NF = 1 dB, noise floor = −143 dBm per 1 kHz bin.

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

*Specification version 1.0 · Lyrion Radar · MIT License*
