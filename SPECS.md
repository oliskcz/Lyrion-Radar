# Lyrion Radar, Front-End Specifications

> **System**: FMCW radar at 5.5 to 6 GHz for counter-UAS (drone detection)
> **Architecture**: ADF41510 PLL chirp → YSGM556006 VCO → TX/RX analog → AD9643 ADC → XC7A100T-2FTG256I FPGA DSP

---

## 1. System Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Frequency band | 5.5 to 6.0 GHz | 500 MHz sweep bandwidth |
| Range resolution | 30 cm (theoretical) | ΔR = c / (2B); in practice limited by FFT size |
| Max range (small drone, σ=0.01 m²) | **~1.5 km** | 64-chirp coherent integration, I/Q, NF_sys=5.5 dB |
| Max range (medium drone, σ=0.1 m²) | **~2.3 km** | 64-chirp coherent integration |
| Max range (large drone, σ=1 m²) | **~3.5 km** | 64-chirp coherent integration |
| Path to 3+ km (small drones) | PA upgrade + 27 dBi antennas | +18 dB combined link improvement |
| Scan rate | 100 Hz to 10 kHz | Configurable per mode |
| EIRP | +35.4 dBm (3.5 W) | Not ISM, needs experimental license |
| Demodulation | **I/Q (complex)** | LTC5586, Doppler direction resolved |

---

## 2. Chirp Source (PLL)

| Parameter | Value | Notes |
|-----------|-------|-------|
| PLL | **ADF41510** | Fractional-N, 1 to 10 GHz |
| VCO | **YSGM556006** | 5320 to 6060 MHz, +6 dBm, 0 to 5 V tune |
| Reference | **TCXO 100 MHz** | ±1 ppm |
| PFD frequency | 100 MHz | 1× reference |
| Phase noise (fractional-N) | −231 dBc/Hz @ 100 kHz | Datasheet spec |
| Phase noise (integer-N) | −235 dBc/Hz @ 100 kHz | Datasheet spec |
| Charge pump current | 16 programmable settings | 0.5 to 7.5 mA |
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
TX antenna (24 dBi)  n/a     +35.4 dBm EIRP
```

---

## 4. RX Chain (I/Q Architecture)

| Parameter | Value | Notes |
|-----------|-------|-------|
| RX antenna gain | 24 dBi | StarterDish™ 24 UM, separate from TX |
| LNA (QPL9547) | **+11.2 dB gain** (S21 @ 5.5 GHz) | 0.3 dB NF @ 1.9 GHz, ~1 dB NF @ 5.5 GHz (est.) |
| LNA OIP3 | +39 dBm @ 1.9 GHz | Datasheet; expect >33 dBm @ 5.5 GHz |
| LNA P1dB | +22.7 dBm @ 1.9 GHz | Handles TX leakage without compression |
| I/Q Demodulator (LTC5586) | SE RF in (on-chip transformer), diff I/Q out | 300 MHz to 6 GHz, SPI-controlled |
| LTC5586 conversion gain | +7.7 dB @ 1.9 GHz | **UNVERIFIED @ 5.8 GHz**: expect +3 to +6 dB |
| LTC5586 RF attenuator | 0 to 31 dB, 1 dB steps (SPI) | Coarse AGC for close targets |
| LTC5586 IF amplifier | 8 gain steps (SPI) | ~0 to +15 dB range |
| LTC5586 DC offset null | SPI adjustable | Critical for zero-IF FMCW (TX leakage) |
| LTC5586 image rejection | 37 dB, adjustable to 60 dB (SPI) | |
| LTC5586 OIP3 | 40 dBm @ 1.9 GHz | UNVERIFIED @ 5.8 GHz |
| LO drive to LTC5586 | ~0 dBm required | GP2X+ outputs +11.4 dBm → **10 dB atten pad** |
| IF anti-alias filter | **LC 3rd-order Butterworth, 70 MHz** | 470 nH + 12 pF × 2 per leg (differential) |
| AGC / DVGA (LMH6521) | **Dual channel, −5.5 to +26 dB, 0.5 dB steps** | 1400 MHz BW (constant at all gains), SPI |
| LMH6521 OIP3 | 48.5 dBm @ 200 MHz | Datasheet |
| LMH6521 NF | 7.3 dB @ max gain | Datasheet |
| LMH6521 output | Differential, 10 Vppd swing | Drives AD9643 directly |
| **System NF** | **~5.5 dB** (expected @ 5.8 GHz) | Friis: dominated by LTC5586 NF / LNA gain |
| **Architecture** | **I/Q (complex)** | Doppler direction resolved, image rejection |

### Signal Path

```
RX Antenna (24 dBi, SE)
  → QPL9547 LNA (+11.2 dB, NF ~1 dB, SE in/out)
    → LTC5586 (SE RF in via on-chip transformer, diff I/Q out, SPI)
        LO ← GP2X+ divider (+11.4 dBm) → 10 dB atten pad → +1.4 dBm
        │
        ├── I+, I- (diff) → LC LPF (70 MHz) → LMH6521 CH A (AGC) → AD9643 CH A
        │
        └── Q+, Q- (diff) → LC LPF (70 MHz) → LMH6521 CH B (AGC) → AD9643 CH B
```

### AGC Architecture (all SPI, FPGA-controlled)

| Stage | Range | Resolution | Purpose |
|-------|-------|-----------|---------|
| LTC5586 RF attenuator | 0 to 31 dB | 1 dB | Coarse, close targets, jammers, TX leakage |
| LTC5586 IF amplifier | 0 to 15 dB (8 steps) | ~2 dB | Medium, coarse gain trim |
| LMH6521 | −5.5 to +26 dB | **0.5 dB** | Fine, precision AGC loop |
| **Total AGC range** | **~72 dB** | | |

### IF Anti-Alias Filter (LC 3rd-Order Butterworth, 70 MHz)

Replaces the old 4.8 MHz RC filter which limited range to 1.4 km at 1 kHz chirp rate.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Topology | 3rd-order Butterworth, differential pi-network | 1 inductor + 2 capacitors per leg |
| Cutoff (−3 dB) | 70 MHz | Passes IF up to 2 km @ 10 kHz chirp rate |
| Rolloff | −60 dB/decade | 3rd order |
| Attenuation @ 125 MHz (Nyquist) | ~−15 dB | Plus ADC analog BW rolloff |
| L (per leg) | 470 nH, 0402, SRF > 200 MHz | |
| C (per leg) | 12 pF, C0G/NP0, 0402, 50V | ×2 (input + output shunt) |
| Components total | 4 inductors + 8 capacitors | 2 legs (I and Q) × 3 elements |
| Insertion loss (passband) | < 0.5 dB | Passive, no noise added |

### RX Path Gains & Levels (1 km, σ = 0.01 m², max gain)

| Stage | Gain/Loss | Cumulative | Level |
|-------|-----------|-----------|-------|
| RX antenna (24 dBi) | n/a | n/a | **−139.2 dBm** |
| QPL9547 LNA | +11.2 dB | +11.2 dB | −128.0 dBm |
| LTC5586 (conv. + IF amp max) | +20 dB (est.) | +31.2 dB | −108.0 dBm |
| LC LPF (70 MHz) | −0.5 dB | +30.7 dB | −108.5 dBm |
| LMH6521 (max +26 dB) | +26 dB | **+56.7 dB** | **−82.5 dBm** |
| AD9643 full scale | n/a | n/a | +7 dBm |
| **Headroom below FS** | n/a | n/a | **89.5 dB** |

> **Note**: Signal at ADC is 89.5 dB below full scale. The analog noise floor (amplified thermal noise) at the ADC input is approximately −86 dBm per 1 kHz bin, which is well above the ADC quantization noise (−130 dBm/bin). The ADC is not the noise bottleneck.

### System Noise Figure (Friis Formula)

```
F_sys = F_LNA + (F_mixer − 1) / G_LNA

QPL9547 @ 5.5 GHz:  G = 11.2 dB (13.2 linear),  NF ≈ 1.0 dB (F = 1.26)
LTC5586 @ 5.8 GHz:  NF ≈ 15 dB (F = 31.6), UNVERIFIED, estimated

F_sys = 1.26 + (31.6 − 1) / 13.2 = 1.26 + 2.32 = 3.58
NF_sys = 10×log₁₀(3.58) = 5.5 dB
```

| LTC5586 NF @ 5.8 GHz | System NF | Impact |
|----------------------|-----------|--------|
| 10 dB (optimistic) | 3.0 dB | Best case |
| 15 dB (expected) | 5.5 dB | Design point |
| 20 dB (pessimistic) | 8.0 dB | Still functional |

> **Key**: The QPL9547's 11.2 dB gain at 5.5 GHz (from S-parameter data) shields the system from the LTC5586's degraded NF at band edge. A higher-gain LNA would improve NF further, but the QPL9547's 39 dBm OIP3 and 0.3 dB NF make it the best choice for linearity.

### Components Removed (vs. previous design)

| Removed | Reason |
|---------|--------|
| YX18 mixer | Replaced by LTC5586 (integrated I/Q) |
| NCS4-63+ balun (×2) | Eliminated, LTC5586 has on-chip RF transformer |
| OPA838 IF preamp | Replaced by LTC5586 IF amplifier + LMH6521 |
| MCP6S91 AGC | Replaced by LMH6521 (1400 MHz BW vs 18 MHz GBW) |
| RC LPF 4.8 MHz | Replaced by LC LPF 70 MHz (was limiting range) |

---

## 5. ADC (AD9643BCPZ-250)

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Model** | **AD9643BCPZ-250** | Analog Devices |
| Resolution | 14 bits | |
| Max sample rate | 250 MSPS | |
| **Channels** | **2** (dual) | Ch A = I, Ch B = Q (I/Q demodulation) |
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
| **FPGA** | **Xilinx XC7A100T-2FTG256I** | Artix-7, 256-ball FTG BGA, speed grade -2, industrial temp |
| Logic cells | 101,440 | |
| DSP slices | 240 | |
| BRAM | 4,860 Kb | 607.5 KB |
| DDR3 SDRAM | 512 MB (MT41K256M16) | **Required for large FFTs (>8K points)** |
| **MCU** | **STM32H503CBU6** | Cortex-M33 @ 250 MHz, LQFP-48 |
| MCU SRAM | 64 KB | (some variants 128 KB) |
| MCU flash | 128 to 256 KB | (variant dependent) |
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

**v_max formula** (for sawtooth with I/Q sampling): `v_max = λ × f_PRF / 2`. For real sampling only (prototype 1, single AD9643 channel), halve these values, and for triangular modulation halve them again (full period = 2× ramp time).

**Key relationship**: `N_FFT ≤ N_samples = T_ramp × f_sample`. A smaller FFT gives coarser range resolution. The theoretical 0.3 m resolution is achieved only when N_FFT = N_samples.

---

## 8. Link Budget (Corrected, I/Q architecture, verified radar equation)

### Radar Equation

```
Pr = Pt × Gt × Gr × λ² × σ / ((4π)³ × R⁴)

In dB:
Pr(dBm) = Pt + Gt + Gr + 20×log₁₀(λ) + 10×log₁₀(σ) − 30×log₁₀(4π) − 40×log₁₀(R)

Constants:
  Pt = +11.4 dBm (TX antenna port, after GP2X+ divider)
  Gt = Gr = 24 dBi (StarterDish 24 UM)
  λ = c/f = 3×10⁸ / 5.75×10⁹ = 0.0522 m
  20×log₁₀(λ) = −25.65 dB
  30×log₁₀(4π) = 32.97 dB

Simplified:
  Pr(dBm) = 0.78 + 10×log₁₀(σ) − 40×log₁₀(R_m)
```

### Signal at RX Antenna

| Range | σ = 0.01 m² (small) | σ = 0.1 m² (medium) | σ = 1 m² (large) |
|-------|---------------------|---------------------|-------------------|
| 100 m | −99.2 dBm | −89.2 dBm | −79.2 dBm |
| 250 m | −111.2 dBm | −101.2 dBm | −91.2 dBm |
| 500 m | −127.2 dBm | −117.2 dBm | −107.2 dBm |
| 1 km | −139.2 dBm | −129.2 dBm | −119.2 dBm |
| 1.5 km | −146.3 dBm | −136.3 dBm | −126.3 dBm |
| 2 km | −151.2 dBm | −141.2 dBm | −131.2 dBm |
| 3 km | −158.8 dBm | −148.8 dBm | −138.8 dBm |

### Noise & SNR

Conditions: NF_sys = 5.5 dB (expected), noise BW = 1 kHz (= 1/T for T = 1 ms chirp), 64-chirp coherent integration (+18 dB).

Noise floor per 1 kHz bin = −174 + 30 + 5.5 = **−138.5 dBm**

| Range | σ = 0.01 m² | SNR (1 chirp) | **SNR (64-chirp)** | σ = 0.1 m² | **SNR (64-chirp)** | σ = 1 m² | **SNR (64-chirp)** |
|-------|-------------|---------------|---------------------|------------|---------------------|----------|---------------------|
| 100 m | −99.2 | 39.3 dB | **57.3 dB** ✅ | −89.2 | **67.3 dB** ✅ | −79.2 | **77.3 dB** ✅ |
| 500 m | −127.2 | 11.3 dB | **29.3 dB** ✅ | −117.2 | **39.3 dB** ✅ | −107.2 | **49.3 dB** ✅ |
| **1 km** | **−139.2** | **−0.7 dB** | **17.3 dB** ✅ | −129.2 | **27.3 dB** ✅ | −119.2 | **37.3 dB** ✅ |
| 1.5 km | −146.3 | −7.8 dB | **10.2 dB** ⚠️ | −136.3 | **20.2 dB** ✅ | −126.3 | **30.2 dB** ✅ |
| **2 km** | −151.2 | −12.7 dB | **5.3 dB** ⚠️ | **−141.2** | **15.3 dB** ✅ | −131.2 | **25.3 dB** ✅ |
| 3 km | −158.8 | −20.3 dB | **−2.3 dB** ❌ | −148.8 | **7.7 dB** ⚠️ | −138.8 | **17.7 dB** ✅ |

### Detection Limits (13 dB threshold, Pfa = 10⁻⁶)

| Target | Max range (24 dBi) | Condition |
|--------|-------------------|-----------|
| Small drone (σ = 0.01 m², DJI Mini) | **~1.5 km** | 64-chirp coherent, NF_sys = 5.5 dB |
| Medium drone (σ = 0.1 m², DJI Phantom) | **~2.3 km** | 64-chirp coherent |
| Large drone (σ = 1 m², Matrice 600) | **~3.5 km** | 64-chirp coherent |

### NF Sensitivity

| LTC5586 NF @ 5.8 GHz | NF_sys | SNR @ 1 km, σ=0.01 (64-chirp) | Max range (σ=0.01) |
|----------------------|--------|-------------------------------|---------------------|
| 10 dB (optimistic) | 3.0 dB | 19.8 dB | ~1.7 km |
| 15 dB (expected) | 5.5 dB | 17.3 dB | ~1.5 km |
| 20 dB (pessimistic) | 8.0 dB | 14.8 dB | ~1.3 km |

> **Conclusion**: Even in the pessimistic case (LTC5586 NF = 20 dB at 5.8 GHz), the radar detects small drones at 1.3 km with 64-chirp integration. The I/Q architecture provides Doppler direction and +37 dB image rejection from the LTC5586. To extend range beyond 2 km for small drones: add PA (+12 dB), 27 dBi antennas (+6 dB), or 256-chirp integration (+6 dB).

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
| **Total** | n/a | **~4 A** | n/a |

---

## 10. Regulatory & Licensing

| Issue | Status | Required action |
|-------|--------|----------------|
| +35.4 dBm EIRP at 5.5 to 6 GHz | **Not ISM** | Experimental or STA license needed |
| UNII/DFS band incursion | Possible | Check local frequency allocation |
| RF exposure at 24 dBi | Significant | Safe stand-off distance required |

---

## 11. Verification Notes

This document was independently reviewed and several errors were found and corrected:

| Error | Original | Corrected | Impact |
|-------|----------|-----------|--------|
| **FFT sizes too small** | 4096/2048/512/256 | 16384/16384/8192/4096 | Range resolution was 2.85 m instead of 0.71 m |
| **Link budget Pr values 12 dB too low** | 1 km small drone = −151.2 dBm | 1 km small drone = **−139.2 dBm** | Ranges were underestimated; verified via linear radar equation |
| **v_max too low by 2×** | 0.63 m/s at 10 ms | 2.6 m/s at 10 ms (sawtooth, I/Q) | Velocity coverage was half of actual |
| **RX chain non-functional above 2 MHz** | MCP6S91 (18 MHz GBW) | LMH6521 (1400 MHz BW) | Old AGC was useless at radar IF frequencies |
| **4.8 MHz RC LPF limiting range** | Max 1.4 km @ 1 kHz chirp | 70 MHz LC LPF (3rd order) | Was blocking 80% of usable IF bandwidth |
| **Single-channel (real) only** | YX18 + OPA838 + MCP6S91 | LTC5586 I/Q + LMH6521 dual | No Doppler direction, no image rejection |
| **System NF underestimated** | "~1 dB" | **5.5 dB** (Friis, linear) | Was using dB values directly instead of linear conversion |

**Root cause of the Pr error**: The previous "correction" subtracted 12 dB from the radar equation result, making values 12 dB too pessimistic. Verified by computing Pr in linear:
```
Pr = (0.0138 × 251.2 × 251.2 × 0.00272 × 0.01) / (1984.4 × 10¹²) = 1.195×10⁻¹⁷ W = −139.2 dBm
```
at R = 1 km, σ = 0.01 m², Pt = +11.4 dBm, Gt = Gr = 24 dBi, f = 5.75 GHz.

**Root cause of the NF error**: Friis formula requires linear values. Using NF(dB) directly (e.g., "1 + (15-1)/10 = 2.4 dB") is wrong. Correct: F = 1.26 + 30.6/13.2 = 3.58 → NF = 5.5 dB.

**Root cause of the RX chain failure**: The MCP6S91 PGA has a gain-bandwidth product of 18 MHz. At ×8 gain (needed for the link budget), bandwidth collapses to 2.25 MHz. The radar IF at 1 km (1 kHz chirp rate) is 3.3 MHz, already outside the MCP6S91's bandwidth. The part was never suitable for this application.

---

*Specification version 2.0, Lyrion Radar, MIT License*
