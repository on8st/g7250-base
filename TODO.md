# TODO

Roadmap mirrors `REVERSE_ENGINEERING_GUIDE.md`, tracked per unit.

## Done

- [x] Integrate the OEM **Operation Manual** as external ground-truth →
      `docs/hardware.md` (connectors per unit, accessories, specs, full
      menu/parameter catalog, manual↔firmware reconciliation). Corrections
      applied: shipped mic is the **H-22** (not HM-133); **PC USB
      (CAT+audio+FW-update, CH342) is on the Controller**; **LAN is on the
      Main Unit**; Main Unit has no external USB.
- [x] Scaffold repo; port tooling from X6100 BASE.
- [x] Replace F427 memory map with STM32H7 map (`MapSTM32H7Memory.java`).
- [x] Import both images into shared Ghidra project (base `0x08020000`).
- [x] Identify MCU family: STM32H7 / Cortex-M7 (HIGH CONFIDENCE). (README)
- [x] Export startup chain + vectors + function ranges for both units.
- [x] Seed `main_unit/docs/startup.md`, `control_unit/docs/startup.md`.
- [x] MCU family VERIFIED (H7 peripheral bases + AXI-SRAM errata).
- [x] Resolve `.data`/`.bss` ranges → `*/docs/memory_map.md` (both units).
- [x] Verify SystemInit for each unit (stock H7 CMSIS; FPU/FLASH/RCC/VTOR).
- [x] Control Unit: both pre-`.data` calls identified (PWR+VOSRDY, SystemInit).
- [x] Verify `__libc_init_array` for each unit.

- [x] Follow `main()` on both units → both are bare-metal superloops,
      NO RTOS. Documented in `*/docs/flow.md` and `docs/architecture.md`.
- [x] Establish system architecture: two M7s + SLIP/CRC16 serial link;
      Main = RF/DSP/network, Control = touchscreen front panel.
- [x] Dump strings for both units (`*/analysis/strings/strings.txt`).

## Phase 1 — Startup / memory (remaining)

- [x] STM32H7 part number inferred: Main = H725xG/H735xG (MEDIUM),
      Control = H743xI rev V (HIGH). See docs/mcu_identification.md.
- [ ] Verify both part numbers against chip markings.
- [x] Main Unit Ghidra memory map: H72x layout selected per program.
- [x] Running clock config: Main `FUN_080222f0` (550/275/137.5 MHz, HSE
      25 MHz, PLL2 = ADC), Control `FUN_08021d98` (480/240/120 MHz, PLL2 =
      FMC, PLL3 = LTDC). See */docs/flow.md.
- [x] Control Unit 8-byte `.data`/`.bss` gap = linker alignment padding
      before a ≥16/32-byte aligned `.bss` (control_unit/docs/memory_map.md).
- [x] VERIFIED renames as CSV (*/analysis/renames/verified_renames.csv: Main
      59, Control 14), applied with tools/run_headless_verified_renames.sh.
      Note: on a fresh import `0x080487d4` (CH394 init) may sit inside a larger
      Ghidra function; it is left out of the CSV.

## Phase 2/3 — Peripherals & interrupts

- [x] Map active IRQs to H7 peripherals for both units (per-unit
      `docs/peripherals.md`). TIM services cross-check the H7 numbering.
- [x] Bus/peripheral inventory from base refs + access counts.
- [x] Inter-MCU SLIP link identified as **USART2** on both MCUs.
- [x] Control I2C1: GT911 touch, address 0x14 with fallback 0x5D (probe order VERIFIED).
- [x] Main Unit SPI6 = **WCH CH394 Ethernet**, exclusive bus: master,
      mode 0, /8, PG12/13/14 AF5, BDMA ch0 RX (req 11) / ch1 TX (req 12),
      IRQ86; handle 0x24005420. See main_unit/docs/peripherals.md and
      analysis/peripherals/spi6_ch394.csv.
- [x] Main Unit SPI1/2/3: SPI1 = two RF shift-register chains (TX LPF bank +
      T/R; RX BPF + front-end switches), SPI2 idle, SPI3 = I2S3 48 kHz
      full-duplex sample stream (DMA1 Str5/Str0). See peripherals.md.
- [x] Main Unit I2C devices (all on I2C1): 24Cxx EEPROM @0x50 (NVPARAM/
      calib/ATU), AXP223 PMIC @0x34, VersaClock 6 clock gen @~0x6A.
      I2C2/I2C3 idle. Read/write helpers FUN_0802BB3C / FUN_0802B940.
      (No battery → AXP223 replaces X6100 MAX17205.) See i2c_devices.csv.
- [x] Audio / I-Q path: AIC3204 on Main I2C1 0x18 (I²S3 slave) and on
      Control I2C2 0x18 (I²S2 master); RF I/Q via the FPGA GPIOE parallel
      bus (24-bit I/Q, command set). See */docs/peripherals.md.
- [~] FPGA: NCO words (122.88 MHz phase increments), status read and
      toggle state documented (peripherals.md FPGA bus). Open: roles of
      0x05/06 and 0x09/0A, the status values TRX+0x226 and 0x2400E9E4;
      measure PE10 (expected 48 kHz).
- [ ] Analog audio routing across the CTRL cable; the PC USB sound-card chip.
- [x] CH394 control pins: CS# PG15, RST# PB5, INT# PC13/EXTI13 (falling);
      SPI6 SCK ≈17.19 MHz. See peripherals.md SPI6 section.
- [x] CH394 INT#: no interrupt register is read; HDSDR uses the flag plus a
      5 s poll, wfview ignores it; only RECV is enabled (peripherals.md SPI6).
- [x] CH394 0x0039: WCH lists it as CHIPV (read-only); the documented soft reset is MODE bit 7,
      never written. The 0x80 write has no documented effect (peripherals.md SPI6).
- [x] EEPROM layout: settings / calibration / NVPARAM marker / 2× ATU
      tables, CRC-8 0x31, 64-B pages (main_unit/docs/memory_map.md).
- [x] Who writes the settings area (EEPROM 0x0000 / backup SRAM)? Nothing in either V1.0.1
      image; both copies are read-only and ignored, because defaults are forced (memory_map.md).
- [ ] Contents of EEPROM 0x0000 and backup SRAM on a shipped unit (factory tool, bootloader or
      older firmware?): needs an EEPROM or SWD read.
- [x] AXP223 rail map + VersaClock output plan (peripherals.md). Rail
      *use* is HYPOTHESIS.
- [ ] Confirm the VC6 crystal (26 MHz?), whether OUT1 is 245.76 or
      122.88 MHz, and where OUT1 goes (FPGA/ADC): needs a board photo or
      a probe.
- [x] GPIO pin maps for both units (*/analysis/peripherals/gpio_map.csv,
      */docs/peripherals.md). X6100 comparison skipped: its pin map is not
      in this repo.
- [x] Main GPIO roles resolved where code allows (PH5 power button, PB11 heartbeat,
      PB12/PB14 DAC, PF13 unused).
- [x] GPIO roles from firmware: exhausted. The remaining pins (Main PH9/PH13/PH15,
      PE14/PE15, PA8/PC8, PA12; Control PH13/PG9/PG10/PI4) are only set at init or
      power-off and have no other code consumer, so firmware cannot name them.
- [ ] Hardware follow-up: board trace for those pins and the CTRL-cable routing of PE3/PE4/PI6.
- [x] Main Unit has no external SDRAM/FMC (VERIFIED).
- [x] Control pull-up inputs = button matrix, 8 encoders, SD detect (peripherals.md).
- [x] Init functions from `main()` identified on both units (*/docs/flow.md,
      */analysis/flow/init_sequence.csv).
- [x] TIM4 PWM: Main PD12 configured but never started; Control PD12/PD13
      backlight enable + dimming (MEDIUM). See */docs/peripherals.md timer maps.
- [~] TRX-state struct `0x24013C0C`: first 0x1A8 B = 0x01 layout; runtime
      tail partial (main_unit/analysis/memory/trx_runtime_fields.csv).
- [x] Main Unit USB: not used at runtime (false-positive references); see
      main_unit/docs/peripherals.md. Bootloader jump via RTC wake-up documented.
- [x] Main RTC wake-up timer: armed by nobody in V1.0.1; the bootloader jump is dead code.
- [x] Bluetooth XG-BT-G7250 on Control USART1 (AT commands, PCA9557 power
      bit); see control_unit/docs/peripherals.md.
- [~] Bluetooth: +CW/+BD/+B4/+CZ power-on sequence and meanings done (MEDIUM,
      BT401-family dialect). Open: module chip and the analog BT audio entry
      point (firmware changes no routing; needs a board trace).

## Phase 5 — Inter-MCU SLIP protocol (docs/protocol.md)

- [x] SLIP framing (RFC1055, END 0xC0) VERIFIED; encode/decode primitives.
- [x] CRC-16/ARC (poly 0x8005, init 0) VERIFIED, identical on both units.
- [x] RX handlers + dispatch; payload sizes for types 0x01/02/03/06/08.
- [x] Identify message semantics: 0x01=remote settings/state (wfview),
      0x02=calibration data, 0x06=slave info. Structs located.
- [x] 0x01 settings block: partial field map (18 fields) in
      docs/protocol.md + cmd01_fields.csv; consumer FUN_08041EF4.
- [x] Header bytes after `type`: reserved, always zero (docs/protocol.md).
- [x] 0x01 settings layout mostly mapped (cmd01_fields.csv, cmd01_vfo_record.csv).
- [x] 0x01: offsets 0x0C8/0x100/0x136/0x137/0x159/0x15A and the RX filter
      half-words 0xD4–0xE8 named (docs/protocol.md, cmd01_fields.csv).
- [x] 0x02 calibration, 0x03 spectrum/meters, 0x06 slave info and 0x08
      network state decoded (*_fields.csv); 0x02 is separate from the TRX struct.
- [x] Complete message-type map: 0x01/0x02/0x03/0x06/0x08 only, with timing
      and the 0x08 sequence/ack scheme.
- [ ] Who sends SLIP 0x02 in the field (factory tool?).
- [x] Physical layer: USART2 PD5/PD6 AF7, 460 800 8N1, DMA (docs/protocol.md).
- [x] UART roles: Main USART1 = console + service menu, Main UART4 unused, Control USART1 = Bluetooth (AT + SPP CI-V) + console.
- [~] Main Unit networking: CH394 driver, socket plan, HDSDR mode, wfview
      session and self-heal documented (main_unit/docs/network.md).
- [x] CI-V command map for both parsers (docs/civ.md).
- [x] wfview audio and scope packet layouts (network.md, wfview_packets.csv).
- [~] wfview scope span: wfview sends and reads ± half-widths, the G7250 takes full width and reports
      half, so the axis is 2x too narrow (network.md). Confirm with a UDP capture.
- [ ] wfview: value the radio returns on a 27 15 read; whether Xiegu ships a rig file for CI-V 0xA0.
- [x] Control USART1 CI-V: endpoint is Bluetooth SPP (MEDIUM); both parsers see every byte (civ.md).

## Tooling

- [x] `ExportStartupEvidence.java`: `.init_array` now located from the
      `__libc_init_array` shape; vector-table length measured instead of the
      STM32F427 cap. `ExportInterruptEvidence.java`: same cap removed.
      `*/analysis/startup/vectors.csv` and `*/analysis/interrupts/active_irqs.csv`
      regenerated (adds Main IRQ118/129/130, Control IRQ101).
- [x] STM32H7 peripheral blocks: opt-in via `periph` / `MAP_PERIPHERALS=1`;
      the default stays RAM-only (false pointers and non-reproducible globals).

## Phase 2 — Interrupts

- [x] Map active IRQ numbers → H7 peripherals (main: 16, control: 13
      active) — done, see Phase 2/3 above and per-unit `docs/peripherals.md`.
- [x] Default/fault handlers and IRQ callback paths (*/docs/startup.md, */docs/flow.md).
- [ ] Is the IWDG started by the bootloader or by option bytes?

## Later phases (per RE guide)

- [x] Phase 3 Hardware init (RCC/GPIO/DMA/SPI/I2C/USART/TIM/WDG): docs/hardware_init.md.
- [x] Phase 4 Scheduler: bare-metal superloops, no RTOS, on both units
      (see Done; `*/docs/flow.md`).
- Phase 5 Inter-MCU protocol: tracked in its own section above.
- [~] Phase 6 RX DSP pipeline: structure mapped (main_unit/docs/dsp.md).
- [~] Phase 7 TX DSP pipeline: structure mapped (main_unit/docs/dsp.md).
- [~] DSP: AGC, NR engine (SpeexDSP) and FM discriminator documented
      (dsp.md); interrupt rate MEDIUM (48 kHz). Open: AGC code-to-label
      mapping (on-air check), DNR2 and DNR-type path, Speex vendor
      controls 0x30–0x36, FM stereo flag +0x11D, TX ALC loop.
- [x] X6100 tooling leftovers: verify_tx_biquads.py and generate_analysis_markdown.py
      removed; evidence manifest and lib matcher ported; tools/filter_response.py added.
- [x] run_headless_*.sh wrappers accept absolute output paths (decompile wrapper pending).
- [x] RenameVerifiedFromCsv.java skips and logs bad rows (0x0804995c on a fresh import); wrapper fails on fatal errors.
- [x] Full export runner (tools/run_full_export.sh) + doc checker (tools/check_docs.py) + CreateVectorFunctions.java.
- [x] Confidence tags added (check_docs CONF 90 -> 3, the rest false positives); function ranges regenerated.
- [x] check_docs.py: accepts wrapped status lines, "unless marked", and tags on bold lead-in lines (tools/tests).
- [x] Compare with X6100 and G90 findings where behavior overlaps (docs/related_radios.md).

## Round 2 — follow-up items (N01–N44)

- [x] N01 Main option bytes · [x] N06 Main image tail · [x] N09 IWDG · [x] N17 RTC/backup regs
- [x] N33 Main bootloader self-update · [x] N34 FPGA flash update · [x] N36 Control FW update over USB
- [x] N02 full vector/IRQ CSVs (exporters fixed)
- [x] N03 DMA/NVIC allocation (docs/dma.md)
- [x] N04 protection thresholds · [x] N05 BandMap/TX band enforcement · [x] N20 ADC/DAC map
- [x] N21 PA bias / power control (SPI2 is the bias DAC, not idle) · [x] N22 PTT arbitration + T/R sequencing · [x] N31 power on/off
- [x] N07 stale doc claims · [x] N08 exception handlers · [x] N16 H72x Ghidra memory map
- [x] N18 MPU/cache/RAM placement · [x] N19 timer map
- [x] N10 UART roles · [x] N11 Main SDRAM? · [x] N12 Control I2C · [x] N13 Control SPI5/SPI flash
- [x] N14 GT911 · [x] N28 Control inputs · [x] N29 Control USB descriptors · [x] N30 display bring-up
- [x] N32 unknown GPIOs
- [x] N23 CW · [x] N24 ATU algorithm · [x] N35 H-22 mic protocol · [x] N37 voice recorder · [x] N41 SWR scanner
- [x] N25 SLIP 0x03/0x08 completeness · [x] N26 SLIP link state machine · [x] N27 Control storage
- [x] N42 CI-V command set · [x] N43 GUI layer
- [x] N15 X6100 tooling leftovers · [x] N38 library identification · [x] N39 RAM object map
- [x] N40 reproducible export run + doc checker
- [x] N44 X6100 comparison (docs/x6100_comparison.md: architecture, host-baseband interface,
      mode codes, feature set, register-by-register mapping) and G90 comparison from its published CI-V
      table.
- [~] G90 internals: the head-base protocol is publicly described (zeroping/xiegu-g90-headprotocol);
      MCUs still open.
- [~] SPI2 DAC = BU2506FV-type (peripherals.md); circuit roles of channels 5/6 still need a schematic.
- [ ] Confirm in hardware whether an analog SWR/ALC fold-back exists outside the MCU.
- [x] 1920-byte BandMap: an unused shared type in V1.0.1 (no object on either
      unit); layout HYPOTHESIS 2 × 24 × 40-byte VFO records (flow.md).
- [ ] What drives PH5 (front-panel key over the CTRL cable?); is PH9 a power latch; Control PCA9557 bit 5 role.
- [ ] Read the option bytes (RDP, BOOT_ADD, CPUFREQ_BOOST, IWDG_SW) from a unit over SWD, if RDP allows.
- [ ] Main firmware update transport: not in either application (peripherals.md); needs a
      sector-0 dump of both units or a USART1/USART2 capture during a vendor update.
- [x] Per-band frequency offsets: transverter feature (70cm to 3cm, QO-100), dormant in V1.0.1 (flow.md).
- [ ] Control bootloader: update file name and checks (needs a sector-0 dump).
- [ ] IWDG timeout and the IWDG_SW option byte (needs an SWD read).
- [x] Exact toolchain matching against ST's GNU Tools 12.3 build (tools/match_static_libs.py, docs/libraries.md).
- [ ] STemWin 5.44a exact match (needs a licensed download).
- [~] RAM object sizes: 19 Main and 10 Control objects pinned, Control UI state copies identified
      (ram_objects.csv, memory_map.md). Open: library structs, wfview/WFV-G buffers, RX working buffers.
- [ ] CI-V CW handlers (pitch/WPM/break-in) in undisassembled code near 0x08052400; confirm KEY jack wiring to Main PH3/PH4.
- [x] ATU merit producer: raw block SWR from the ADC ISR FUN_080235e8 (docs/atu.md).
- [ ] ATU: TUNER TRG/RET connector hardware-only?
- [ ] H-22: firmware bounds the high time below about 437 µs (h22_timing.csv); measure
      the absolute bit period and confirm which mic line feeds PI5.
- [x] Control codec slot routing (peripherals.md, aic3204_init.csv).
- [ ] What PI6, PE3 and PE4 connect to.
- [ ] Control codec: destination of CLKOUT (MFP4) and which of HPR/LOR feeds the Main mic
      input; confirm IN2R/IN3R = USB sound-card line-in.
- [x] GUI screen-id mapping VERIFIED (ids 1-3 unused, 12 = quick setting).
- [x] GUI: appUi 1 unused; system sub-pages 2/3/4/7 and windows 1/3 identified (gui.md).
- [ ] PCA9557 bits 0/1/3/5 and 6/7: electrical function (board trace; firmware write sites
      exhausted, bit 5 is high exactly during the Control shutdown/save).
- [ ] Which W25Qxx part is fitted (board photo).
- [x] Control button-code and encoder-to-function tables (gui.md, button_map.csv, encoder_map.csv).
- [ ] Panel legends for the 22 matrix buttons and the knob/push pairing (needs a photo).
- [ ] Board trace: USB hub, CH342 and USB sound-card chips on the Controller and where their UART lines go.
- [ ] Display: panel model and SDRAM part (board photo).

## Round 3 — follow-up items

- [x] HPSDR gap analysis: what it would take for the G7250 to appear as an
      OpenHPSDR P1/P2 radio, or be integrated into an app (docs/hpsdr_gap_analysis.md).
- [x] Teardown video (KM9G) reviewed: FPGA = Intel Cyclone 10 (LP likely), CTRL = 15-pin
      HD D-sub, four named boards (docs/hardware.md §7). Controller not shown.
- [x] Hardware verification plan in tiers 0–3 (docs/hardware_test_plan.md).
- [ ] HPSDR open questions Q1–Q9 (sample rates, FPGA TX rate, CH394 SPI limit,
      install path): need hardware.
