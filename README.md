# USB-C / Bluetooth DAC + Headphone Amp

A from-scratch portable headphone DAC and amplifier with USB-C audio input, Bluetooth audio, and USB-C power delivery. Designed as a personal project to learn PCB layout for mixed signal applications, and USB C power.  

> **Status:** [In progress — devkit firmware development]

---

![Block diagram](docs/block-diagram.png)
*System block diagram — USB C and BT to MCU, then out to DAC, Amp, and finally headphones*

---

## Why this exists

I wanted a desktop DAC/amp that I actually understood end-to-end: the power architecture, the analog signal chain, the grounding strategy. There are plenty of cheap dongles and off-the-shelf solutions, but they don't teach you anything about why a particular output stage topology works, or how to keep switching noise out of a 192 kHz signal path.

So I built one.

---

## Specs (targets)

| Parameter | Target | Notes |
|---|---|---|
| Audio inputs | USB-C (UAC 2.0), Bluetooth 5.x | A2DP / aptX via BT module |
| Power input | USB-C, 5V/3A (15W) via CC resistors | No PD controller — resistor-configured only |
| Sample rate | Up to 192 kHz / 24-bit via USB | |
| Output | 3.5mm stereo headphone jack | |
| Output power | 200 mW into 250Ω, 50 mW into 32Ω | T90 rated max is 200 mW |
| Output impedance | < 1Ω | Covers T90 and low-Z IEMs without FR interaction |
| SNR | > 110 dB (target) |
| THD+N | < 0.005% (target) |
| Layers | 4-layer PCB |

---

## System architecture

The signal chain has two parallel audio input paths that converge at the MCU:

**USB path:** USB-C receptacle → USB audio bridge MCU → I2S → DAC

**Bluetooth path:** BT module (I2S out) → MCU

The DAC drives a discrete output stage. Power comes in via USB-C PD, regulated to clean analog and digital supply rails with LDOs to keep switching noise away from the audio circuitry.

A more detailed writeup of every design decision — IC selection rationale, grounding strategy, power rail partitioning — lives in [`docs/design-notes.md`](docs/design-notes.md).

---

## Repository layout

```
usb-audio-dac-amp/
├── hardware/
│   ├── schematic/       # KiCad schematic source
│   ├── pcb/             # KiCad PCB layout source
│   ├── fab/             # Gerbers, drill files, BOM for production
│   ├── lib/             # Custom symbols and footprints
│   └── datasheets/      # Key IC datasheets
├── firmware/
│   ├── src/             # MCU source (DAC/PD config over I2C)
│   └── README.md        # Toolchain and flash instructions
├── docs/
│   ├── block-diagram.png
│   ├── design-notes.md  # Running log of decisions and trade-offs
│   └── images/          # Board photos, scope captures, renders
├── testing/
│   ├── test-plan.md
│   └── results/         # Measurements, audio analyzer output
└── README.md
```

---

## Design decisions (short version)

Longer rationale for each of these is in `docs/design-notes.md`.

**DAC IC:** [TBD — e.g. PCM5102A / ES9038Q2M] — chosen for ...

**MCU** [TBD] — USB Audio Class 2 (so I didn't need to write a custom OS driver) was critical; I2S output.   

**Bluetooth Module** [TBD] - APTX HD, Bluetooth Classic, and BLE were the ideal targets; this ended up causing incredible headaches.  

**Output stage:** [TBD — discrete class AB / opamp-based] — chosen because ...

**Power architecture:** USB-C with CC resistors for 15W of power. Separate LDO rails for analog and digital to isolate switching noise. Ground plane split at the DAC.

---

## Build status / revision history

| Rev | Status | Notes |
|---|---|---|
| v0.1 | In progress | Initial schematic, component selection, devkit firmware development |

---

## Measurements

*To be populated after first board bring-up.*

Planned measurements: output noise floor, THD+N vs. frequency, crosstalk, frequency response, power supply rejection.

---

## Full writeup

A detailed post covering the whole design process — what worked, what didn't, what I'd do differently — will be published here when the project is complete: [link TBD]

---

## Tools used

- KiCad [version] for schematic and PCB
- [Audio analyzer tool, e.g. REW or ARTA] for measurements
- [Any simulation tools]

---

## License

Hardware: [CERN OHL v2 Permissive](LICENSE)
Firmware: MIT
