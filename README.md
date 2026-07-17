# OSAP — Open Source Audio Player

A thin, long-battery-life, audiophile-grade portable music player whose hardware and
firmware are fully open source.

- **SoC:** NXP i.MX RT685 (Cortex-M33 + HiFi4 audio DSP, dual native SD hosts,
  USB 2.0 high-speed)
- **Audio:** Cirrus Logic CS43131 DAC with built-in headphone amplifier; modular
  3.5 mm / 6.35 mm jack + aux line-in on a daughterboard
- **Wireless:** socketed NXP IW6xx tri-radio module — Bluetooth Classic A2DP to any
  BT headphone (Wi-Fi 6 capable, deferred)
- **Firmware:** Zephyr RTOS
- **Form factor:** ~cassette-tape footprint, 10–20 mm thick, 3/4-color e-ink display

📄 **Start here: [DESIGN.md](DESIGN.md)** — the v1 design document (open questions
tracked in its §9).

🧪 **[EVT1.md](EVT1.md)** — the first custom board: RT685 + CS43131 + PCA9420, with
an M.2 Key E socket for the IW6xx wireless module.

## Status

Pre-hardware. Requirements and architecture are being defined; no PCB or firmware yet.

## License

100% copyleft:

- **Hardware:** [CERN-OHL-S-2.0](LICENSES/CERN-OHL-S-2.0.txt)
- **Firmware/software:** [GPL-3.0-or-later](LICENSES/GPL-3.0-or-later.txt)
- **Documentation:** [CC-BY-SA-4.0](LICENSES/CC-BY-SA-4.0.txt)
