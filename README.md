# Lyrion Radar

FMCW (Frequency-Modulated Continuous-Wave) radar at **5.5–6.0 GHz** for short-to-medium range detection and ranging.

## Concept

FMCW radar transmits a continuous signal whose frequency sweeps linearly over time (a "chirp"). The transmitted signal reflects off targets and returns to the receiver with a time delay proportional to the target's distance. By mixing the received (delayed) signal with the currently transmitted signal, a low-frequency **beat signal** (IF) is produced whose frequency is directly proportional to the target range:

```
f_IF = (2 · B · R) / (c · T)
```

Where:
- `B` = sweep bandwidth (500 MHz)
- `R` = target range
- `c` = speed of light
- `T` = ramp time

A 500 MHz sweep gives **30 cm range resolution**. An FFT of the IF signal reveals all targets in the field of view as distinct frequency peaks.

### Why 5.5–6 GHz?

- ISM-adjacent band with manageable PCB design constraints
- Wavelength ~5.2 cm allows compact high-gain antennas
- 500 MHz of sweep bandwidth fits comfortably in the band
- Components (VCOs, mixers, amplifiers) are readily available and affordable

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          SIGNAL SOURCE                                   │
│                                                                         │
│  STM32 ──SPI──► DAC8830 ──► TLV9062 ──► YSGM556006 VCO                 │
│                 (16-bit)     (buffer)     (5320-6060 MHz, +6 dBm)       │
│                                    │                                    │
│  REF5050A (5.0V ref) ──► DAC VREF │                                    │
│  TMP102 (I²C) ──► temp comp ──────┘                                    │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                               100 nF DC block
                                     │
┌─────────────────────────────────────────────────────────────────────────┐
│                            TX CHAIN                                      │
│                                                                         │
│  VCO ──► 6 dB pad ──► YG802020W ──► GP2X+ ──┬──► TX Antenna (18 dBi)  │
│  +6 dBm   (resistive)  (+15 dB)    (divider) │    +11.4 dBm            │
│             0 dBm       +15 dBm     -3.6 dB   │                         │
│                                               └──► LO to mixer          │
│                                                    +11.4 dBm            │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                            RX CHAIN                                      │
│                                                                         │
│  RX Antenna ──► QPL9547 ──► NCS4-63+ ──► YX18 ──► RC LPF ──► OPA838  │
│  (18 dBi)       (LNA)       (balun)     (mixer)  (194 kHz)   (×47)     │
│                 NF=0.6dB                            │                    │
│                 G=17dB                         NCS4-63+                  │
│                                                (balun)                   │
│                                                   │                      │
│                                               LO from GP2X+             │
│                                                                         │
│  OPA838 ──► MCP6S91 ──► STM32 ADC                                      │
│  (×47)      (PGA 1-32×)  (12-bit)                                      │
│             SPI AGC                                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

## Component List

### Signal Source

| Part | Manufacturer | Role | Key Specs |
|------|-------------|------|-----------|
| **YSGM556006** | Innotion | VCO | 5320–6060 MHz, +6 dBm, 0–5 V tuning, 14 mA @ 5 V |
| **DAC8830IDR** | Texas Instruments | 16-bit SPI DAC | 1 µs settling, unbuffered R-2R, SOIC-8 |
| **REF5050AIDGKR** | Texas Instruments | Voltage reference | 5.0 V, 3 ppm/°C, 3 µVpp noise |
| **TLV9062IDR** | Texas Instruments | DAC output buffer | Dual op-amp, rail-to-rail I/O, 10 MHz GBW, SOIC-8 |
| **TMP102AIDRLR** | Texas Instruments | Temperature sensor | I²C, ±0.5 °C accuracy, SOT-563 |

### TX Chain

| Part | Manufacturer | Role | Key Specs |
|------|-------------|------|-----------|
| **YG802020W** | Innotion | TX driver amplifier | 50 MHz–8 GHz, 18 dB gain, +21 dBm P1dB, DFN 2×2 |
| **GP2X+** | Mini-Circuits | 2-way power divider | 2.9–6.2 GHz, 3.6 dB total loss, 20 dB isolation, 3×3 mm |
| Resistors (68Ω, 150Ω×2) | — | 6 dB resistive pad | 0603 thin-film, protects VCO from load pulling |

### RX Chain

| Part | Manufacturer | Role | Key Specs |
|------|-------------|------|-----------|
| **QPL9547TR7** | Qorvo | RX LNA | 0.05–6 GHz, **0.6 dB NF**, 17 dB gain, +19 dBm P1dB |
| **YX18** | Innotion | Mixer diode quad | GaAs Schottky cross-over quad, 1.4 V turn-on, DFN 2×2 |
| **NCS4-63+** (×2) | Mini-Circuits | Mixer baluns | 4.5–6 GHz, 1:4 impedance ratio, 0805 ceramic |
| **OPA838IDBVR** | Texas Instruments | IF preamplifier | 0.9 nV/√Hz, 29 MHz GBW, SOT-23-5 |
| **MCP6S91T-E/MS** | Microchip | IF PGA (AGC) | 1×–32× gain (0–30 dB), 18 MHz GBW, SPI, MSOP-8 |

### Power & Support

| Part | Manufacturer | Role | Key Specs |
|------|-------------|------|-----------|
| **ADP150AUJZ-5.0** (×2) | Analog Devices | Ultra-low noise LDO | 5 V, <10 µVrms, TSOT-23-5 |
| Antennas (×2) | — | TX and RX | 18 dBi, 5.5–6 GHz, separate |

## Design Decisions

### Open-Loop VCO Ramp (No PLL)

The VCO is driven directly by a DAC-generated voltage ramp. No PLL chip is used. Instead:

1. **One-time calibration**: The VCO's frequency-vs-voltage curve is measured on a spectrum analyzer and stored as a lookup table (LUT) in firmware.
2. **Pre-distorted ramp**: The DAC outputs non-linear voltage steps that produce a linear frequency sweep.
3. **Temperature compensation**: The TMP102 measures die temperature near the VCO. The LUT baseline is shifted by `ΔT × 0.6 MHz/°C` to correct for thermal drift.

This approach saves cost and complexity (~$15 less BOM) at the expense of requiring calibration. For short-range detection, the residual drift (< 16 MHz over 20 °C on a 500 MHz sweep) is negligible.

### 6 dB Resistive Pad After VCO

The YSGM556006 has 9 MHz peak-to-peak frequency pulling at 3:1 VSWR. A resistive π-pad (68Ω series + 2×150Ω shunt) presents a near-perfect 50Ω load to the VCO regardless of downstream impedance variations, at the cost of 6 dB attenuation. The YG802020W amplifier after the pad restores the signal level.

### GP2X+ Power Divider

A Mini-Circuits GP2X+ MMIC divider splits the amplified signal equally to the TX antenna and the mixer LO port. Compared to a PCB Wilkinson divider, it adds ~0.4 dB loss but guarantees isolation (20 dB) and saves RF layout effort. Compared to a resistive divider, it saves 3 dB of power.

### YX18 + NCS4-63+ Mixer

The YX18 GaAs Schottky diode quad forms a double-balanced mixer with two NCS4-63+ baluns (one for RF, one for LO). The 1:4 impedance ratio of the baluns provides a 2× voltage step-up at the diodes, ensuring clean switching even at +11.4 dBm LO drive (well above the 1.4 V turn-on threshold).

### Two-Stage IF Amplification with AGC

- **OPA838** (fixed gain ×47): Ultra-low noise (0.9 nV/√Hz) preamplifier that lifts the weak mixer output above the noise floor of subsequent stages.
- **MCP6S91** (variable gain 1×–32×): SPI-controlled PGA provides 30 dB of AGC range. Firmware adjusts gain between ramps to keep the IF signal at ~50% ADC full-scale.

Total IF gain: 33.4 dB + 0–30 dB = 33–63 dB, covering targets from 1 m to 300 m.

## Power Budgets

### TX Chain

| Stage | Input | Gain/Loss | Output |
|-------|-------|-----------|--------|
| YSGM556006 VCO | — | — | **+6 dBm** |
| 6 dB resistive pad | +6 dBm | −6 dB | **0 dBm** |
| YG802020W (93 mA) | 0 dBm | +15 dB | **+15 dBm** |
| GP2X+ divider | +15 dBm | −3.6 dB | **+11.4 dBm** per port |

TX EIRP with 18 dBi antenna: **+29.4 dBm** (~870 mW ERP).

### RX Link Budget

Conditions: Pt = +11.4 dBm, Gt = Gr = 18 dBi, σ = 1 m² (pedestrian), system NF ≈ 0.8 dB.

| Range | Pr (dBm) | After LNA | After mixer | After OPA838 | MCP6S91 setting | ADC level | SNR |
|-------|----------|-----------|-------------|--------------|-----------------|-----------|-----|
| 10 m | −43.2 | −26.2 | −33.2 | +0.2 dBm | 1× | 0.32 Vpp | ~95 dB |
| 50 m | −71.2 | −54.2 | −61.2 | −27.8 dBm | 8× | 104 mVpp | ~67 dB |
| 100 m | −83.2 | −66.2 | −73.2 | −39.8 dBm | 32× | 102 mVpp | ~55 dB |
| 200 m | −95.2 | −78.2 | −85.2 | −51.8 dBm | 32× | 26 mVpp | ~43 dB |

Noise floor per 1 kHz FFT bin: **−143 dBm**.

## FMCW Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Center frequency | 5.75 GHz | |
| Sweep bandwidth | 500 MHz (5.5–6.0 GHz) | VCO tuning: ~1.2 V → 4.6 V |
| Ramp time | 1–2 ms | 1000–2000 DAC steps |
| Range resolution | **30 cm** | ΔR = c / (2B) |
| IF frequency per meter | 333 Hz/m (T=1 ms) | f_IF = 2BR / (cT) |
| Max IF frequency (200 m) | 66.7 kHz | Within 194 kHz LPF |
| ADC sample rate | ≥ 200 kSPS | Nyquist for max IF |
| FFT size | 256–512 points | Range bins |
| Processing gain | ~23 dB (200 samples) | 10·log₁₀(N/2) |

## Signal Processing (Firmware)

1. **Ramp generation**: MCU writes pre-distorted DAC values from calibration LUT via SPI at 1 µs intervals.
2. **ADC acquisition**: DMA-driven ADC samples the IF signal during the ramp (200+ samples per chirp).
3. **Range FFT**: 256-point FFT on the IF samples. Magnitude spectrum reveals target peaks.
4. **AGC**: After each ramp, measure peak IF amplitude. Adjust MCP6S91 gain to maintain ~50% ADC full-scale.
5. **Temperature compensation**: Read TMP102 periodically. Shift DAC LUT baseline by calibrated drift coefficient.
6. **Detection**: Peak detection on FFT magnitude with CFAR or simple threshold. Range = f_IF × c × T / (2B).

## Repository Structure (planned)

```
Lyrion-Radar/
├── README.md              ← this file
├── Hardware/
│   ├── Schematic/         ← KiCad schematic
│   ├── PCB/               ← KiCad PCB layout
│   └── BOM/               ← bill of materials
├── Firmware/
│   ├── Core/
│   │   ├── Inc/
│   │   │   ├── main.h
│   │   │   ├── radar_config.h
│   │   │   ├── dac8830.h
│   │   │   ├── mcp6s91.h
│   │   │   ├── tmp102.h
│   │   │   ├── fmcw_ramp.h
│   │   │   ├── range_fft.h
│   │   │   └── agc.h
│   │   └── Src/
│   │       ├── main.c
│   │       ├── dac8830.c
│   │       ├── mcp6s91.c
│   │       ├── tmp102.c
│   │       ├── fmcw_ramp.c
│   │       ├── range_fft.c
│   │       └── agc.c
│   └── Calibration/
│       └── vco_cal.py     ← SA measurement → LUT generation script
└── Docs/
    └── calibration.md     ← VCO calibration procedure
```

## Status

- [x] Architecture defined
- [x] Components selected and sourced
- [x] Link budget validated
- [ ] VCO calibration on spectrum analyzer
- [ ] PCB schematic (KiCad)
- [ ] PCB layout
- [ ] Firmware: DAC ramp + drivers
- [ ] Firmware: FFT + detection
- [ ] End-to-end validation

## License

TBD
