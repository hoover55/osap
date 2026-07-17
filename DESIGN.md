# OSAP v1 — Open Source Audio Player

**Design Document (skeleton — rev 0.1, 2026-07-16)**

> Status: outline / early architecture. Items marked **TBD** are undecided.
> Open questions are collected in [§9](#9-risks--open-questions).
> Less-common acronyms are expanded in [§12](#12-acronyms--terms).

---

## 1. Project Overview

- **Name:** OSAP (Open Source Audio Player), version 1
- **Vision:** A thin, long-battery-life, audiophile-grade portable music player whose
  hardware and firmware are fully open source.
- **Design priorities (in order):**
  1. Audio quality (high-end DAC + proper headphone amplification)
  2. Battery life
  3. Thinness
- **Out of scope for v1 (TBD/confirm):** Wi-Fi, streaming services, touchscreen, camera
- **Licenses (decided 2026-07-16 — 100% copyleft):**
  - Hardware: **CERN-OHL-S-2.0** (strongly reciprocal)
  - Firmware/software: **GPL-3.0-or-later**
  - Documentation: **CC-BY-SA-4.0** (ShareAlike)
  - Full texts in `LICENSES/`

## 2. Requirements

### 2.1 Functional

| # | Requirement |
|---|---|
| F1 | Local playback of lossless and lossy audio from removable storage |
| F2 | High-end Cirrus Logic DAC in the analog signal path |
| F3 | Built-in headphone amplifier |
| F4 | Modular headphone jack accommodating 1/8" (3.5 mm) and 1/4" (6.35 mm) connectors |
| F5 | Wireless audio output to Bluetooth headphones/speakers — Classic A2DP via on-board wireless module (DNP variant = radio-less build; LE Audio as a future addition) |
| F6 | 1× microSD card slot (was 2× — reduced 2026-07-18 to free USDHC1 for the radio) + **internal serial NAND storage** (§4.4) |
| F7 | USB-C: battery charging + USB 2.0 high-speed (480 Mbps) data for file transfer |
| F8 | 3- or 4-color e-ink display |
| F9 | Physical controls: navigation, power, media (play/pause/skip), volume |
| F10 | Battery powered, rechargeable |
| F11 | Analog auxiliary audio input (3.5 mm line-in) |
| F12 | All analog audio I/O connectors (headphone out, aux in) mounted on a removable daughterboard |

### 2.2 Non-functional targets (all TBD — set numeric goals before schematic capture)

- [ ] Battery life: ≥ **TBD** h screen-static local playback; ≥ **TBD** h BT streaming
- [x] Thickness: **10–20 mm** (drives battery and jack selection)
- [x] Footprint: approximately compact-cassette sized — target ≈ **100 × 64 mm**
      (cassette nominal: 100.4 × 63.8 × 12 mm)
- [ ] Audio: THD+N ≤ **TBD**, SNR/DR ≥ **TBD** dB, output power **TBD** mW into 16/32/300 Ω
- [ ] Output noise floor low enough for sensitive IEMs (≤ **TBD** µVrms)
- [ ] Supported sample rates: 44.1–**TBD** kHz, up to **TBD**-bit; DSD support TBD
- [ ] Cold boot to playback ≤ **TBD** s; resume from sleep ≤ **TBD** s

## 3. System Architecture

Two-chip design: an **NXP i.MX RT700** ultra-low-power crossover MCU (dual
Cortex-M33 + HiFi4 and HiFi1 DSPs, 7.5 MB SRAM, native SD hosts, USB 2.0 high-speed
over **eUSB2**) paired with an on-board **u-blox MAYA-W260-00B module** (NXP IW611:
Wi-Fi 6 + dual-mode Bluetooth 5.4) for wireless audio — Bluetooth **Classic A2DP**
works with virtually every BT headphone. RT700 chosen over the RT600 (2026-07-17)
for **longevity and lower power**; its eUSB2 port reaches the USB-C connector
through a small eUSB2→USB 2.0 repeater (§4.5). Firmware runs on upstream **Zephyr
RTOS** (Embassy/Rust was considered early and dropped). The device is fully
functional with the radio module unfitted. First hardware: **[EVT1.md](EVT1.md)**.

```mermaid
flowchart LR
    USBC["USB-C"]
    PMIC["PCA9422 PMIC<br/>640 mA charger + 3 bucks<br/>+ buck-boost + 4 LDOs"]
    BATT["Li-Po pouch cell<br/>(TBD)"]
    RPT["eUSB2 repeater<br/>(TUSB2E11-class)"]
    MCU["i.MX RT700<br/>2× M33 + HiFi4 + HiFi1"]
    FLASH["Octal-SPI NOR<br/>boot flash"]
    RADIO["u-blox MAYA-W260-00B<br/>(IW611: Wi-Fi 6 + BT 5.4)<br/>2× U.FL antennas"]
    DAC["CS43131<br/>DAC + headphone amp"]

    subgraph DB["Audio I/O daughterboard"]
        JACK["Headphone jack<br/>3.5 mm / 6.35 mm"]
        AUX["Aux line-in<br/>3.5 mm"]
    end
    SD["microSD (USDHC0)"]
    NAND["Serial NAND<br/>xSPI1, 1–8 Gbit"]
    EINK["3/4-color e-ink"]
    BTNS["Buttons"]

    USBC -- "VBUS" --> PMIC --> BATT
    PMIC -- "rails (mode-switched)" --> MCU
    USBC <-- "USB 2.0 HS (3.3 V)" --> RPT
    RPT <-- "eUSB2 (1.2 V)" --> MCU
    MCU <--> FLASH
    MCU -- "I2S + MCLK + I2C" --> DAC --> JACK
    AUX -. "pass-through switch (option)" .-> JACK
    AUX -. "ADC record path (option)" .-> MCU
    SD <--> MCU
    NAND <--> MCU
    MCU -- "SPI" --> EINK
    BTNS --> MCU
    MCU <-- "SDIO Wi-Fi (USDHC1) + UART HCI BT" --> RADIO
```

### 3.1 Compute allocation

| Core | Planned role |
|---|---|
| Main Cortex-M33 @ 325 MHz | Zephyr: UI, filesystem, USB, BT host (A2DP), playback engine; audio decode baseline |
| HiFi4 DSP | **Optional** offload: decode/SRC/EQ — proprietary Xtensa toolchain, optional build only (§7, R12) |
| Sense subsystem (M33 @ 250 MHz + HiFi1) | Candidate: always-on button scan, battery monitoring, wake logic at µW levels — [ ] evaluate at M1 (also Xtensa-toolchain-free? verify) |
| eIQ Neutron NPU / 2.5D GPU | NPU unused; GPU's JPEG/PNG decoder is a candidate for album-art rendering — strictly optional |
| MAYA-W260 module CPUs (IW611) | Wi-Fi / BT protocol processing on-module (NXP firmware) |

## 4. Hardware Design

### 4.1 SoC — NXP i.MX RT700 (MIMXRT798S lead candidate; RT758/RT735 siblings)

- Dual Cortex-M33 (325 MHz main + 250 MHz sense) + HiFi4 + HiFi1 DSPs + eIQ Neutron
  NPU + 2.5D GPU w/ JPEG decode; **7.5 MB SRAM**; no internal flash — 3× xSPI for
  external NOR (up to 16-bit DDR @ 250 MHz), XIP
- **Chosen over RT600 for longevity + lower power** (2026-07-17); trade-off: younger
  silicon and Zephyr support (R11)
- **uSDHC SD/eMMC/SDIO hosts** — [ ] **confirm instance count in the RM** (expected 2:
  USDHC0 + USDHC1); supports eMMC 5.0 HS400. Allocation: **card → USDHC0; radio
  SDIO → USDHC1 (dedicated — no mux)**. Two of the three xSPI ports are spare →
  internal-NVM expansion option (§4.4)
- USB 2.0 HS via **eUSB2** (1.2 V signaling, same 480 Mbps protocol) — requires an
  **eUSB2→USB 2.0 repeater** at the connector (§4.5)
- Audio: [ ] **verify SAI/I2S MCLK output pin and audio-PLL exact rates** vs the
  CS43131 direct-MCLK mask (fallback: CS43131 PLL-reference mode, R7)
- [ ] VDDIO domain map, power-mode map, package/pitch survey, errata (R6)
- In-tree Zephyr support (`mimxrt700_evk`) + MCUXpresso SDK (BSD-3-Clause) — younger
  than the RT600's (R11)

### 4.2 Audio subsystem

- **DAC — Cirrus Logic CS43131 (selected):** 32-bit/384 kHz DAC + integrated
  low-power ground-centered headphone amp. Verified constraints: **true MCLK
  required** (no SCLK-derived mode); **I2C-only** control; supplies VA/VCP/VL/VD at
  1.66–1.94 V plus **VP 3.0–5.25 V sequenced up-first/down-last**; HP_DETECT is
  VP-referenced. Design targets = the **32 Ω datasheet figures**: 125 dB(A) DR,
  −110 dB THD+N, 30.8 mW.
- **Headphone amplifier:** the CS43131's integrated stage is the baseline; fallback
  **CS43198 + discrete amp** (OPA1622-class) if measured drive into high-impedance
  1/4" cans falls short (R4).
- **Audio I/O daughterboard (F4, F11, F12) — decided:** all analog connectors
  (headphone output + 3.5 mm aux line-in) live on a removable daughterboard
  - [ ] Output jack format on the board: single 3.5 mm + threaded/snap 6.35 mm adapter
        vs dual jacks; swappable board variants later (e.g., 6.35 mm-only, balanced 4.4 mm)
  - [ ] Board-to-board interconnect: FPC vs mezzanine connector — pin-out carries
        headphone L/R + ground, aux L/R in + ground, jack-detect switches, shield
  - [ ] Grounding/shielding: keep analog runs short, away from RF and digital busses
- **Aux input (F11):** function to define at M0
  - [ ] Pass-through mode: analog switch routes aux → jack (line-level bypass, or via
        a small discrete buffer stage — the CS43131's amp has no external analog input)
  - [ ] Record/digitize mode: **discrete line-in ADC** (the CS43131 has none) —
        enables recording to SD and BT re-streaming
  - [ ] Or both — TBD (R10)
- **Clocking:** RT700 audio PLL → **MCLK output** for both 44.1/48 kHz families
  (verify §4.1, R7); CS43131 PLL-reference mode as fallback.
- **Analog power:** dedicated low-noise LDO rails, ground/layout strategy, pop/click
  muting circuit. TBD.

### 4.3 Power

- Single-cell Li-Po pouch (thin form factor drives selection) — capacity **TBD** mAh
- PMIC — **NXP PCA9422 (selected 2026-07-17 with the RT700 move):** NXP's designated
  charger+gauge PMIC for RT500/600/**700**. JEITA linear charger up to **640 mA**
  (resolves the old 315 mA charge-time concern), 3 bucks (SW1/SW3 core 300 mA,
  SW2 system 500 mA) + **buck-boost** + 4 LDOs, ship mode. The buck-boost is the
  natural **CS43131 VP rail** — holds > 3.3 V across the full battery discharge.
  Details: [EVT1.md §5.4](EVT1.md)
- USB-C power: 5 V; **discrete 5.1 kΩ CC pull-downs**, input current limit via I2C;
  USB-PD not planned
- Fuel gauging: PCA9422 measurement + NXP **FlexGauge** host-software algorithm —
  [ ] **license review** (must be GPL-compatible or we fall back to our own
  open ADC-based estimator)
- [ ] Power budget table: decode+playback, BT streaming, e-ink refresh, sleep, off
- E-ink draws zero power when static — key enabler for the battery-life target

### 4.4 Storage — 1× microSD + optional internal NAND

- **Native 4-bit SD on USDHC0** (single slot — reduced from two on 2026-07-18 so
  USDHC1 serves the radio SDIO directly, eliminating the analog mux). 3.3 V
  signaling, no level shifters; ~10+ MB/s-class expected (USB transfers become
  card-bound, not interface-bound) — benchmark at EVT-1 (E2/E3)
- [ ] UHS-I 1.8 V switching: evaluate (power/perf trade)
- [ ] Push-push vs hinged socket; card detect; slot power switch (hot-swap + sleep)
- **Internal serial NAND (decided 2026-07-18)** on **xSPI1** (one of the two spare
  ports): **QSPI/octal serial NAND, 1–8 Gbit**, running **littlefs** (built-in wear
  leveling, power-fail safe). Roles: music DB/index (survives card swaps), album-art
  cache, settings backup, small built-in library partition.
  - Host access note: littlefs is invisible to USB MSC — internal content is only
    host-reachable via **MTP** (feeds the §5.9 trade study)
  - [ ] Part + capacity selection at M1 (R14): QSPI vs octal, 1/2/4/8 Gbit,
        candidates Winbond W25N/W35N-class, Macronix MX35-class
  - Rejected alternatives: large octal NOR (poor cost/GB), eMMC (needs a uSDHC host;
    both known instances are taken)

### 4.5 USB-C

- Roles: charging sink + USB 2.0 HS device (no DRP/host planned — confirm)
- **eUSB2 interface chain:** RT700 eUSB2 pins (1.2 V) → **eUSB2→USB 2.0 repeater**
  (TI **TUSB2E11**-class: LS/FS/HS 480 Mbps, host/device DRD, strap or I2C config)
  → standard 3.3 V D+/D− → connector. Place the repeater near the connector;
  [ ] repeater supply rails + eUSB vs USB-side termination per datasheet;
  [ ] check what NXP's RT700-EVK uses and copy that reference design
- Device classes: MSC (expose cards directly) vs MTP (database-friendly) — TBD, see §5.9
- ESD/CC protection, connector mid-mount for thinness — TBD

### 4.6 Display — 3/4-color e-ink

- Panel candidates: **TBD** (survey Good Display / Waveshare SPI panels, ~2.9–4.2")
- [ ] Characterize refresh: full-color refresh is seconds-long; verify fast grayscale
      partial-refresh mode for browsing UI (color reserved for accents/album art)
- [ ] Size vs. thinness vs. resolution trade

### 4.7 Controls

- Buttons: power, 4-way/enter navigation, play/pause, next, prev, vol+, vol− (≈10)
- [ ] GPIO matrix vs direct; wake-from-off wiring via PMIC ON pin; software debounce
- [ ] Hold/lock switch? TBD

### 4.8 Wireless — u-blox MAYA-W260-00B (on-board, decided 2026-07-17)

- **u-blox MAYA-W260-00B**: NXP **IW611** (2.4/5 GHz 1×1 Wi-Fi 6 + **dual-mode
  BT 5.4**), 86-pin BFLGA 10.4 × 14.3 mm, **2× U.FL antenna connectors** (Wi-Fi + BT);
  host interfaces: SDIO 3.0 (Wi-Fi, **dedicated USDHC1**) + flow-controlled UART (BT HCI)
- Pre-certified module preserves modular RF certification — [ ] antennas from
  u-blox's approved list; keep-out and coax routing per integration manual
- **DNP build variant** = radio-less SKU (unintentional-radiator cert only)
- [ ] Lifecycle/availability confirmation with u-blox (R9); [ ] confirm IW611 vs
  IW612 variant (802.15.4 not needed)
- Wi-Fi *features* deferred past v1 (R13) — BT Classic A2DP is the v1 wireless feature

## 5. Firmware & Software Stack (Zephyr)

### 5.1 Platform & build system

- **RTOS:** upstream **Zephyr** (NXP HAL module `hal_nxp`) — version **TBD**, pin at M1
- **Build:** `west` workspace, single-image app build; optional separate HiFi4 DSP
  build (§5.2)
- **Configuration:** custom Zephyr **board definition** `osap_v1` (devicetree +
  Kconfig) — DAC on I2S + I2C, e-ink on SPI, `gpio-keys`, SDHC nodes (card + radio),
  PCA9422 regulators, MAYA-W260 on SDIO + UART
- **Toolchain/CI:** Zephyr SDK container; GitHub Actions build + Twister on every PR

### 5.2 Runtime architecture — images & processors

| Image | Target | Contents |
|---|---|---|
| `app` | Cortex-M33 | Zephyr app: UI, playback engine, decoders, SBC encode, BT host, USB, FS |
| `dsp` (optional) | HiFi4 | Decode/SRC/EQ offload — optional build, proprietary Xtensa toolchain (§7, R12) |
| radio firmware | MAYA-W260 module (IW611) | NXP-provided binary, loaded at runtime over SDIO/UART — runs on separate hardware |

- **M33 ↔ DSP / sense subsystem:** shared-SRAM mailbox/IPM ([ ] verify Zephyr RT700
  multi-core support and openamp/ipm story at M1 — relevant if the DSP or sense-core
  builds are used)
- **BT HCI:** Zephyr host on the M33 ↔ module controller over flow-controlled UART

### 5.3 Layered stack

| Layer | Components |
|---|---|
| Application | playback engine (SMF state machine), library manager, UI screens, settings/EQ |
| Middleware | LVGL (UI), FatFs, SBC codec, audio decoders (§5.5), music index |
| Zephyr subsystems | BT host incl. **Classic (A2DP/AVRCP — experimental)**, USB device (usbd), FS/disk, input, settings, PM, logging, shell |
| Vendor/HAL | MCUXpresso HAL (`hal_nxp`), PCA9422 PMIC driver ([ ] verify Zephyr driver status — `nxp,pca9420` driver as reference if absent), IW611 radio-firmware loader |
| Out-of-tree drivers | Cirrus DAC codec driver (Zephyr `audio_codec` API) — **to be written**, e-ink panel driver if not in tree (ssd16xx/uc81xx families are) |

### 5.4 Thread / task model (initial sketch — priorities TBD via profiling)

| Thread | Priority | Role |
|---|---|---|
| Audio datapath | High (cooperative) | Feed I2S DMA or SBC framing; hard real-time |
| Decoder | Medium | File → PCM into ring buffer (target **TBD** ms of buffered audio) |
| UI (LVGL) | Low | Screen updates, e-ink refresh scheduling |
| Library indexer | Lowest | Background scan/tag parse of the card + internal NAND |
| BT host / USB / input | Zephyr-managed | Stack work queues |

### 5.5 Audio pipeline & codecs

- **Local path:** SD → decoder (M33, optionally HiFi4) → PCM ring buffer → I2S (DMA) → CS43131
- **BT path:** same ring buffer → sample-rate convert (44.1/48 k per sink) → **SBC encode** → A2DP over module HCI
- **Codecs (permissively-licensed C libs, candidates):** FLAC (`dr_flac`/libFLAC), WAV,
  MP3 (`minimp3`), Opus (libopus), Vorbis (`stb_vorbis`); AAC **TBD** (licensing);
  DSD **TBD** (depends on DAC path choice)
- [ ] Gapless playback, ReplayGain, EQ/DSP policy — TBD
- [ ] Bit-perfect path definition (volume in DAC vs host) — TBD
- [ ] Aux-in handling (if digitized per §4.2): capture path, record-to-SD and/or
      BT re-stream — pending the pass-through vs record decision

### 5.6 Storage & filesystem

- Zephyr disk-access + **FatFs** for the card (`/SD0`); the internal NAND mounts
  as a **littlefs** volume (`/NAND0`) — both presented as one logical library
- FAT32 baseline; **exFAT TBD** (FatFs supports it behind a config flag, but it is
  patent-encumbered — licensing decision needed for >32 GB cards as shipped)
- Card hot-insert/removal handling and index invalidation — TBD

### 5.7 Music library / database

- On-device index of the card + internal NAND: tags (ID3/Vorbis/FLAC comments),
  album/artist trees, playlists (M3U), resume positions — the index itself lives on
  the internal NAND (survives card swaps, faster boot)
- Format: custom compact binary index vs embedded DB — **TBD** (SQLite likely too heavy;
  benchmark at M1)

### 5.8 Bluetooth — Classic A2DP (via MAYA-W260 module)

- **Zephyr Classic host (experimental, NXP-driven)** over UART HCI: **A2DP source**
  role streaming to ordinary BT headphones — in-tree `a2dp_source` sample as the
  starting point (R1)
- **Codec:** SBC (A2DP mandatory); [ ] optional codecs (AAC/aptX) later — licensing review
- **Remote control:** AVRCP target — headphone buttons drive the playback engine
- **Volume:** AVRCP absolute volume — [ ] verify Zephyr support depth
- **LE Audio:** future addition once IW61x LE Audio support matures — track, don't block v1
- Pairing/bonding UX on e-ink + buttons; bond storage via settings (§5.11); multi-device TBD
- [ ] BT/Wi-Fi coexistence is handled on-module; verify host-side arbitration needs

### 5.9 USB (device)

- Zephyr **usbd** (next-gen) stack over the RT700's high-speed UDC (eUSB2 PHY; the
  external repeater is transparent to software)
- **MSC vs MTP trade study:** MSC is in-tree and simple but requires unmounting the local
  FS while the host owns the cards; **MTP is not upstream** — custom class work if chosen
- Charge-only vs data enumeration behavior; USB audio (UAC2 "USB DAC" mode) as stretch — TBD

### 5.10 UI & input

- **LVGL** on the Zephyr display subsystem; monochrome/limited-palette theme
- E-ink strategy: fast grayscale **partial refresh** for navigation; full color refresh
  reserved for idle/album-art screens (ties to §4.6 panel characterization)
- Input: Zephyr `input` subsystem from `gpio-keys` (long-press, hold-to-power);
  wake wiring through the PMIC ON pin
- Screen map (browse / now-playing / settings / pairing) — TBD wireframes

### 5.11 Settings & persistence

- Zephyr **settings** subsystem on **NVS**, in a partition of the external xSPI NOR:
  BT bonds, EQ presets, last-played position, UI preferences

### 5.12 Power management

- Zephyr PM: device runtime PM on all peripherals; CPU idle between buffer fills;
  radio module powered down when wireless unused; PCA9422 rails track RT700 sleep
  states; [ ] evaluate the **sense subsystem** (M33 + HiFi1) for always-on button
  scan/battery watch with the main domain off
- Deep sleep with e-ink image retained (zero display draw); PCA9422 **ship mode**
  for power-off; wake on power button and VBUS
- [ ] Per-state current targets feed the §4.3 power budget table

### 5.13 DFU & updates

- **MCUboot** on the external xSPI NOR (XIP + update slots) — the standard, boring
  i.MX RT path
- Transport: **SMP over USB**; fallback: firmware file dropped on an SD card, applied
  at boot
- Recovery path and anti-rollback policy — TBD; radio-module firmware updates ride
  along as files loaded by the app

### 5.14 Observability & testing

- Logging via Zephyr `log` → RTT; `shell` enabled in dev builds only
- Unit tests: `ztest` + **Twister**; library/UI logic additionally run on `native_sim`
- HIL bench at M1+: MIMXRT700-EVK + MAYA-W2 EVK + CS43131 eval; audio analyzer
  measurements at M4 (§10)

### 5.15 Firmware repository layout (planned)

```
firmware/
  west.yml            # Zephyr-pinned manifest (hal_nxp module)
  app/                # main Zephyr application (src/, Kconfig, prj.conf)
  boards/osap_v1/     # custom board definition (DTS, defconfig)
  drivers/            # out-of-tree: Cirrus DAC codec driver, etc.
  lib/                # decoders, music index, playback engine, battery gauge
  dsp/                # optional HiFi4 offload build (Xtensa toolchain)
  tests/              # ztest suites
```

## 6. Mechanical / Industrial Design

- **Dimensional targets:** ≈ **100 × 64 mm** footprint (compact-cassette sized,
  nominal 100.4 × 63.8 mm) × **10–20 mm** thick — aim for the low end of the
  thickness range; 20 mm is the hard ceiling
- [ ] Verify fit: 6.35 mm jack barrel, aux jack, microSD socket, and e-ink module
      within the 10 mm best-case stack-up (jack body height is likely the floor-setter)
- [ ] Audio I/O daughterboard: mounting/retention, connector alignment with enclosure
      cutouts, swap/service access, interconnect strain relief
- [ ] Display size constraint: cassette footprint supports roughly a 2.9–3.5" panel
      after bezel/button allowance — feeds §4.6 panel selection
- [ ] Enclosure concept + materials (RF window needed if metal)
- [ ] Stack-up study: battery + PCB + e-ink + jack = thickness floor
- [ ] Button feel, membrane vs discrete tact switches

## 7. Compliance & Licensing

- [ ] FCC Part 15 / CE RED — the pre-certified MAYA-W260 carries the modular radio
      grant (with approved antennas); end-product testing still required (the DNP
      radio-less SKU certifies as an unintentional radiator only)
- [ ] Bluetooth SIG qualification (QDID) — budget line item
- [ ] Battery transport (UN38.3), RoHS/REACH
- [ ] Codec/FS patent review: exFAT, AAC (MP3 patents expired)
- [ ] GPLv3 compatibility review: Apache-2.0 deps (Zephyr, hal_nxp) are one-way
      compatible; the IW611 radio firmware is an NXP binary loaded onto the module's
      own CPUs — separate hardware, clean aggregation ([ ] confirm redistribution
      terms for shipping it in images/repos)
- [ ] HiFi4 DSP build uses the proprietary Cadence Xtensa toolchain — policy: DSP
      offload is **optional**, the shipped baseline is fully buildable with free
      toolchains; the DSP firmware source itself is GPL (R12)

## 8. Repository Layout

- `osap_v1/` — this document, firmware (planned), hardware (planned)
- `../osaplib/osapv1lib.kicad_sym` — shared KiCad symbol library
  - Note: contains STM32 symbols from an early architecture study (unused);
    **RT700, PCA9422, CS43131, eUSB2 repeater, and MAYA-W260 symbols need to be drawn**

## 9. Risks & Open Questions

| # | Risk / question | Impact | Next step |
|---|---|---|---|
| R1 | Zephyr Bluetooth **Classic host / A2DP source is experimental** — qualification depth unknown | v1 wireless feature | Prototype on RT700-EVK + MAYA-W2 EVK **before** EVT-1 layout; fallback: NXP EtherMind stack (license review) |
| R2 | Color e-ink refresh latency hurts browsing UX | Usability | Panel eval; grayscale partial refresh |
| R3 | Thin pouch cell sourcing at needed capacity | Battery life vs thickness | Cell vendor survey |
| R4 | CS43131 integrated amp drive into high-impedance 1/4" cans | Audio target | Measure at EVT-1 (E8); fallback CS43198 + discrete amp |
| R5 | FlexGauge (PCA9422 gauging software) license may be GPL-incompatible | Licensing | Review license; fallback: own open ADC/coulomb estimator on PCA9422 measurements |
| R6 | RT700 VDDIO domains / power-mode map unverified vs rail plan | Schematic rework | RM review before capture (EVT1 V2) |
| R7 | RT700 audio MCLK output pin + PLL exact-rate/jitter for CS43131 direct MCLK **unverified** | Clocking | Verify RM at M1; CS43131 PLL-ref fallback (EVT1 V4) |
| R8 | USB MTP class not upstream in Zephyr — custom work if chosen over MSC | FW effort | §5.9 trade study at M1 |
| R9 | MAYA-W260-00B lifecycle/availability flagged by one distributor | Radio sourcing | Confirm status with u-blox before layout; MAYA-W2 siblings as fallback (EVT1 V6) |
| R10 | Aux-input function undefined (pass-through vs record) — record needs a discrete ADC | Architecture | Decide at M0 (§4.2) |
| R11 | RT700 is young: thin errata history, newer Zephyr port, uSDHC instance count and packages unverified | Schedule/rework | RM + datasheet review at M1; RT700-EVK bring-up early |
| R12 | HiFi4 Xtensa toolchain is proprietary (sense-subsystem HiFi1 likewise) | Open-source ethos | DSP strictly optional; M33 baseline full-featured (§7) |
| R13 | Wi-Fi capability invites scope creep (streaming, sync) | Focus | Deferred past v1; revisit at DVT with product hat on |
| R14 | Internal NAND part/capacity unselected; internal content is MTP-only for hosts | Storage BOM + USB UX | Part study at M1 (§4.4); MTP implication feeds the R8/§5.9 trade |

## 10. Roadmap (draft)

- **M0** — Requirements finalized (fill every TBD in §2.2; aux decision R10)
- **M1** — Dev-kit prototyping: MIMXRT700-EVK + MAYA-W2 EVK + CS43131 eval
  board; local playback + **A2DP source proof** (retires R1); RM verifications
  (R6/R7/R11); u-blox lifecycle confirmation (R9)
- **M2** — E-ink UI, SD + internal NAND (littlefs), and USB MSC/MTP working on the
  EVK bench
- **M3** — **EVT-1**: first custom PCB, full feature set — see **[EVT1.md](EVT1.md)**
  (schematic in KiCad, `osaplib` symbols)
- **M4** — DVT: enclosure + battery integration, power/audio measurements vs targets
- **M5** — Compliance testing, v1.0 release of hardware + firmware

## 11. References

- NXP i.MX RT700: <https://www.nxp.com/products/i.MX-RT700> (EVK: MIMXRT700-EVK;
  Zephyr board docs: <https://docs.zephyrproject.org/latest/boards/nxp/mimxrt700_evk/doc/index.html>)
- u-blox MAYA-W2 series (MAYA-W260-00B): <https://www.u-blox.com/en/product/maya-w2-series>
- NXP IW61x tri-radio family reference: <https://www.nxp.com/products/IW612> (datasheet: <https://www.nxp.com/docs/en/data-sheet/IW612.pdf>)
- NXP PCA9422 PMIC (charger + gauge, RT700 companion): <https://www.nxp.com/products/PCA9422>
- TI TUSB2E11 eUSB2→USB 2.0 repeater: <https://www.ti.com/product/TUSB2E11>
- Cirrus Logic CS43131 datasheet (DS1155F2): <https://statics.cirrus.com/pubs/proDatasheet/CS43131_DS1155F2.pdf>
- Zephyr RTOS: <https://zephyrproject.org> — Classic A2DP
  source sample: <https://docs.zephyrproject.org/latest/samples/bluetooth/classic/a2dp_source/README.html>
- LVGL: <https://lvgl.io>
- E-ink panel datasheets — TBD
- EVT-1 board document: [EVT1.md](EVT1.md)

## 12. Acronyms & Terms

### Audio & signal path

| Term | Meaning |
|---|---|
| THD+N | Total Harmonic Distortion plus Noise — the primary analog-quality figure of merit for the DAC/amp path |
| SNR / DR | Signal-to-Noise Ratio / Dynamic Range (dB) |
| PCM | Pulse-Code Modulation — plain uncompressed digital audio samples; what decoders output and the DAC consumes |
| TDM | Time-Division Multiplexing — multichannel serial audio bus; superset of I2S framing |
| DSD | Direct Stream Digital — 1-bit, very-high-rate audio format (SACD heritage); some Cirrus DACs accept it natively |
| PLL | Phase-Locked Loop — clock synthesizer; lets one crystal serve both 44.1 kHz and 48 kHz sample-rate families |
| IEM | In-Ear Monitor — high-sensitivity earphones; the demanding case for amplifier noise floor |
| ID3 | Metadata tag format embedded in MP3 files (title/artist/album) |

### Bluetooth LE Audio

| Term | Meaning |
|---|---|
| A2DP | Advanced Audio Distribution Profile — Bluetooth Classic audio streaming; we are the *source* role |
| AVRCP | Audio/Video Remote Control Profile — headphone buttons (play/pause/skip) control our playback engine |
| SBC | Subband Codec — A2DP's mandatory codec; encoded in firmware before streaming |
| BR/EDR | Basic Rate / Enhanced Data Rate — "Bluetooth Classic", the mode A2DP runs on (vs LE) |
| LE Audio / LC3 | The modern BLE-based audio system and its codec — a future addition once IW61x support matures (§5.8) |
| HCI | Host Controller Interface — the standard protocol between the Zephyr BT host (RT700) and the controller (MAYA-W260 module, over UART) |
| SIG | (Bluetooth) Special Interest Group — the standards body; products must be qualified with it |
| QDID | Qualified Design ID — the Bluetooth SIG listing number a qualified product receives |

### USB & power

| Term | Meaning |
|---|---|
| MSC | Mass Storage Class — USB device class exposing the SD cards as raw drives |
| MTP | Media Transfer Protocol — file-level USB transfer class (as used by cameras/phones); FS stays owned by the device |
| UAC2 | USB Audio Class 2 — would let the player double as a USB DAC for a computer (stretch goal) |
| UDC | USB Device Controller — the SoC-side USB hardware a device stack drives |
| VBUS | The 5 V supply pin of a USB connection; also our USB-insertion wake signal |
| CC | Configuration Channel — USB-C pins used to detect cable orientation and advertised supply current |
| BC1.2 | USB Battery Charging 1.2 — legacy spec for detecting charger current capability |
| PD | (USB) Power Delivery — higher-power USB-C negotiation; likely unnecessary here |
| DRP | Dual-Role Port — USB-C port that can be host or device; not planned |
| PMIC | Power Management IC — combined charger/regulator chip (PCA9422) |
| eUSB2 | Embedded USB 2 — USB 2.0 protocol at 1.0/1.2 V signaling for advanced-node SoCs; a repeater converts to standard 3.3 V USB at the connector |
| LDO | Low-DropOut regulator — linear regulator; used for quiet analog supply rails |
| ESD | ElectroStatic Discharge — protection required on user-touchable connectors |

### Zephyr / platform

| Term | Meaning |
|---|---|
| HiFi4 / HiFi1 | Cadence Tensilica audio DSP cores in the RT700 (main + low-power sense subsystem) — optional offload only |
| XIP | eXecute In Place — running code directly from the external xSPI NOR boot flash |
| SDIO / SDMMC | Native 4-bit SD card bus interfaces — RT700 uSDHC hosts: card on USDHC0, radio on USDHC1 |
| littlefs | Fail-safe, wear-leveling embedded filesystem — used on the internal NAND (not host-readable over USB MSC; MTP only) |
| NVS | Non-Volatile Storage — Zephyr settings backend on a NOR-flash partition |
| NVM | Non-Volatile Memory — generic term for persistent storage |
| IPC | Inter-Processor Communication — M33 ↔ HiFi4 mailbox/shared-SRAM messaging (DSP build only) |
| DTS | DeviceTree Source — Zephyr's declarative hardware description format |
| SMF | State Machine Framework — Zephyr library; basis of the playback engine |
| DFU | Device Firmware Update — MCUboot + SMP-over-USB here |
| DMA | Direct Memory Access — peripheral data transfer without CPU involvement; keeps the audio path low-power |
| RTT | Real-Time Transfer — SEGGER debug-probe channel used for log output |
| DK / EVK | Development / Evaluation Kit (e.g., MIMXRT700-EVK) |
| HIL | Hardware-In-the-Loop — automated tests run against real hardware |
| U.FL | Miniature coax RF connector — the MAYA-W260-00B has two (Wi-Fi + BT antennas) |
| DNP | Do Not Populate — a build variant that omits a part; the radio-less SKU omits the MAYA-W260 |

### Project, compliance & licensing

| Term | Meaning |
|---|---|
| EVT / DVT | Engineering / Design Validation Test — successive prototype build phases (§10 M3/M4) |
| BOM | Bill Of Materials |
| RED | Radio Equipment Directive — EU (CE) regulation covering intentional radio transmitters |
| RoHS / REACH | EU regulations restricting hazardous substances / chemicals in products |
| UN38.3 | UN transport-safety test standard required to ship lithium batteries |
| CERN-OHL-S | CERN Open Hardware Licence, Strongly-reciprocal variant — the project's hardware license (copyleft) |
| GPL-3.0-or-later | GNU General Public License v3 (or later) — the project's firmware license (strong copyleft) |
| CC-BY-SA | Creative Commons Attribution-ShareAlike — the project's documentation license (copyleft via ShareAlike) |
