# Hardware init — both units

This page lists, per unit, every hardware init step in the order `main()`
runs it. Each row gives the init function, the key configuration and the
doc that holds the details. Register values come from the handle-init and
MSP-init functions (the CubeMX pattern: the init function fills the HAL
handle, the HAL calls the MSP function for clocks, pins, DMA and NVIC).
Constant meanings use the STM32H7 HAL encodings (HIGH CONFIDENCE); the
values themselves are VERIFIED unless noted. CSV form:
`main_unit/analysis/flow/hw_init_table.csv` and
`control_unit/analysis/flow/hw_init_table.csv`.

Common to both units:
- Priority grouping 4 (four pre-emption bits, no sub-priority); lower
  number wins (`docs/dma.md`).
- Error_Handler is called on any HAL init failure; it does not reset.
- No unit configures the IWDG: there is no start or unlock key and the
  prescaler and reload are never written. The timeout comes from the
  option bytes or the bootloader (`docs/architecture.md`).
- The hardware CRC unit is initialised on both units (CRC-32 default
  polynomial and init value, no bit inversion, byte input) but **nothing
  uses it**: its handle is referenced only by its init and the de-init
  paths. The SLIP CRC-16 and the EEPROM CRC-8 are table-driven software
  (HIGH CONFIDENCE).

## Main Unit (STM32H72x/H73x, 550 MHz)

### Before the HAL

| # | Step | Function | Key configuration | Doc |
| ---: | --- | --- | --- | --- |
| R1 | SystemInit | `FUN_08023bd0` | FPU on, RCC to reset state (HSI 64 MHz), VTOR = 0x08020000 | `startup.md` |
| R2 | `.data`/`.bss`, static constructors | reset handler, `FUN_0805407c` | one constructor enables the DWT cycle counter | `startup.md` |
| 1 | MPU | `FUN_0804a7c0` | 4 regions: FMC bank 1 guard, D2 SRAM non-cacheable, backup SRAM strongly ordered, SRAM4 non-cacheable | `memory_map.md` |
| 2–5 | stdout unbuffered, slave-info block | `FUN_08053b3c`, `FUN_08053f64/fb6/ca4` | software only | `flow.md` |
| 6 | I-cache, D-cache | inline in `main` | invalidate, then CCR.IC/DC | `flow.md` |

### HAL bring-up (call sites `0x08022474`–`0x080224dc`)

| # | Peripheral | Init (MSP) | Key configuration | IRQ (prio) | Conf | Doc |
| ---: | --- | --- | --- | --- | --- | --- |
| 7 | HAL_Init | `FUN_08027774` (`FUN_080233e0`) | grouping 4; SysTick 1 kHz at priority 15; MSP enables the SYSCFG clock only. Runs **on HSI**, before the PLL | SysTick (15) | VERIFIED | `flow.md` |
| 8 | SystemClock_Config | `FUN_080222f0` | SMPS 2.5 V + LDO, VOS0, HSE 25 MHz crystal, PLL1 550/275/275 MHz, HCLK 275, all PCLK 137.5 MHz, PLL2 100 MHz (ADC), LSI on, 3 wait states | — | VERIFIED | `flow.md`, `clocks/clock_config.csv` |
| 9 | GPIO + EXTI | `FUN_08021470` | all discrete pins; PC9 = I2S_CKIN (AF5); EXTI3/4/9_5 at 6 (enabled at the end of boot), EXTI15_10 at 2 | 9, 10, 23 (6); 40 (2) | VERIFIED | `peripherals.md`, `gpio_map.csv` |
| 10 | DMA1/DMA2 | `FUN_080213ac` | DMA1 + DMA2 clocks; DMA2 Stream5 memory-to-memory (half-word, FIFO full, 8-beat bursts), never started | 15 (10), 47 (0), 68 (7) | VERIFIED | `docs/dma.md` |
| 11 | MDMA ch0–4 | `FUN_08022dc8` | software request, polled, no IRQ | — | VERIFIED | `docs/dma.md` |
| 12 | BDMA | `FUN_080211bc` | BDMA clock; NVIC only | 129, 130 (13) | VERIFIED | `docs/dma.md` |
| 13 | I²S3 (SPI3) | `FUN_08021c78` (`FUN_08021ccc`) | master full-duplex, Philips, 32-bit, 48 kHz, MCLK off; kernel = I2S_CKIN (12.288 MHz assumed); PA15/PC10/PC11/PC12 AF6; DMA1 S5 TX, S0 RX circular | 11, 16 (1) | VERIFIED | `peripherals.md` (I²S3) |
| 14 | USART1 | `FUN_08024108` (`FUN_08024218`) | 115 200 8N1, oversampling 16, FIFO off; PCLK2; PB6/PB7 AF7; RX DMA1 S3 circular (started later) | none | VERIFIED | `peripherals.md` (USART1) |
| 15 | ADC1 | `FUN_08020de0` (`FUN_08020f44`) | 12-bit, 7-rank scan, continuous, software trigger, 32.5 cycles; PLL2P /32; DMA1 S7 circular | 18 (9) | VERIFIED | `peripherals.md` (ADC1) |
| 16 | USART2 | `FUN_08024190` (`FUN_08024218`) | 460 800 8N1, FIFO off; PCLK1; PD5/PD6 AF7; DMA1 S4 TX normal, S2 RX circular | 38 (10) | VERIFIED | `docs/protocol.md` |
| 17 | I2C3 | `FUN_0802198c` (`FUN_08021a00`) | TIMINGR 0x00D049FB (about 400 kHz); own address 0x72 (7-bit); analog filter on; PH7/PH8 AF4 open-drain; **unused** | 72, 73 (13), default vectors | VERIFIED | this page |
| 18 | CRC | `FUN_08021204` | CRC-32 defaults, byte input; **unused** | — | VERIFIED | this page |
| 19 | DAC1 | `FUN_08021288` | channels 1 and 2: no trigger, output buffer on, no sample-and-hold, external pin, factory trim; PA4/PA5 | 54 (8) | VERIFIED | `peripherals.md` (DAC1) |
| 20 | I2C1 | `FUN_080218b4` (`FUN_08021a00`) | TIMINGR 0x00D049FB (about 400 kHz), 7-bit, analog filter on; PCLK1; PB8/PB9 AF4 open-drain; polled | none | VERIFIED | `peripherals.md` (I2C devices) |
| 21 | UART4 | `FUN_08024080` | 115 200 8N1, FIFO off; PD1/PH14 AF8; **unused** | none | VERIFIED | `peripherals.md` |
| 22 | I2C2 | `FUN_08021920` (`FUN_08021a00`) | TIMINGR 0x60404E72 (about 100 kHz), analog filter on; PF0/PF1 AF4 open-drain; **unused** | none | VERIFIED | this page |
| 23–26 | TIM5, TIM6, TIM7, TIM17 | `FUN_08023d90`, `FUN_08023e00`, `FUN_08023e54`, `FUN_08023ea8` | 20 kHz, 100 Hz, 1 kHz, 20 kHz time bases (timer clock 275 MHz); started later | 50 (5), 54 (8), 55 (9), 118 (4) | VERIFIED | `peripherals.md` (timers) |
| 27 | RTC | `FUN_08022f88` (`FUN_08023008`) | LSI; 24 h; prescalers 127/255 (about 1 Hz); time and date reset to 2000-01-01 | 3 never enabled | VERIFIED | `peripherals.md` (RTC) |
| 28 | SPI2 | `FUN_080230c0` (`FUN_0802317c`) | master TX-only; PB13/PB15 AF5; kernel = I2S_CKIN mux; DAC load PB12, reset PB14 | none | VERIFIED | `peripherals.md` (SPI2) |
| 29 | SPI6 | `FUN_08023120` (`FUN_0802317c`) | master full-duplex, mode 0, /8 from PCLK4 (about 17.19 MHz); PG12–14 AF5; BDMA ch0 RX / ch1 TX | 86 (13) | VERIFIED | `peripherals.md` (SPI6) |
| 30 | SPI1 | `FUN_08023060` (`FUN_0802317c`) | master TX-only, mode 0, /2; PG11/PD7 AF5 | none | HIGH | `peripherals.md` (SPI1) |
| 31 | TIM4 | `FUN_08023c88` | PWM CH1 on PD12, 19.92 kHz, pulse 0, never started | none | VERIFIED | `peripherals.md` (timers) |
| 32 | extra GPIO | `FUN_08021830` | PH11, PA8, PC8 | — | HIGH | `gpio_map.csv` |

### Device bring-up after the HAL

These run in `main` after the table above (order and timing in
`flow.md`, "Power-on and power-off"):

1. DMA1 clock re-enabled, console USART1 RX DMA started (`FUN_08041c10`).
2. PH9 high, 1 s; AXP223 rails (`FUN_0804a8e8`), 1 s.
3. CRC-8 table; EEPROM records: settings (`FUN_0803c08c`, forced
   defaults), calibration (`FUN_0803c5e4`), NVPARAM (`FUN_0803c448`).
4. VersaClock 6 (`FUN_0804abbc`: address 0x68, then 0x6A; 112-byte table,
   then a 256-byte read-back that is only logged).
5. CH394 and LAN service; SPI1 latches; SPI2 DAC (PB14 pulse); ATU table.
6. TIM7 start; ADC calibration and ADC DMA; AIC3204 (`FUN_0804a65c`);
   TRX hardware and I²S3 DMA start (`FUN_0803e2f4`); audio processor.
7. TIM5/TIM6/TIM17 start; FPGA bus test and version (`FUN_0803a3a0`);
   PH15 low, PH13 high.
8. PA bias trim; final RX state; EXTI IRQs 9/10/23 enabled.

## Control Unit (STM32H743, 480 MHz)

### Before the HAL

| # | Step | Function | Key configuration | Doc |
| ---: | --- | --- | --- | --- |
| R1 | PWR supply / VOS wait | `FUN_08022a84` | PWR_CR3 bit 1, wait ACTVOSRDY | `startup.md` |
| R2 | SystemInit | `FUN_0802298c` | stock CMSIS + rev-Y AXI errata fix | `startup.md` |
| R3 | `.data`/`.bss`, 4 static constructors | `FUN_080874ac` | | `startup.md` |
| 1 | MPU | `FUN_08024db8` ×5 | DTCM, AXI, D2, D3, FMC SDRAM 32 MiB | `flow.md` |
| 2 | I-cache, D-cache | inline in `main` | | `flow.md` |

### HAL bring-up

| # | Peripheral | Init (MSP) | Key configuration | IRQ (prio) | Conf | Doc |
| ---: | --- | --- | --- | --- | --- | --- |
| 9 | HAL_Init | `FUN_08024bb4` | grouping 4; SysTick at priority 0; SysTick also feeds the IWDG every 256 ticks | SysTick (0) | VERIFIED | `startup.md`, `docs/dma.md` |
| 10 | SystemClock_Config | `FUN_08021d98` | LDO, VOS1 + ODEN, LSE high drive, HSE 25 MHz, PLL1 480/96/480, PLL2R 240 (FMC), PLL3R 40 (LTDC), HSI48, 4 wait states | — | VERIFIED | `flow.md`, `clocks/clock_config.csv` |
| 11 | GPIO | `FUN_080210d8` | discretes, key matrix, encoders, panel lines | — | VERIFIED | `peripherals.md`, `gpio_map.csv` |
| 12 | DMA1 | `FUN_08020dc8` | DMA1 clock; NVIC only | 11, 12 (3), 13 (1) | VERIFIED | `docs/dma.md` |
| 13 | DMA2D | `FUN_08020e20` | memory-to-memory, RGB565 output, layer 1 RGB565, polled | none | VERIFIED | `peripherals.md` (display) |
| 14 | FMC SDRAM | `FUN_08020ecc` (`FUN_08020f38`) | bank 1, 16-bit, 4 banks, CAS 2, SDCLK = HCLK/2; AF12 pins | none | VERIFIED | `peripherals.md` (display) |
| 15 | I2C1 | `FUN_08021414` (`FUN_080214f0`) | TIMINGR 0x00B03FDB (about 400 kHz), analog filter on; PCLK1; PB6/PB7 AF4 open-drain, very high speed; GT911, polled | none | VERIFIED | `peripherals.md` (GT911) |
| 16 | I2C2 | `FUN_08021480` (`FUN_080214f0`) | TIMINGR 0x307075B1 (about 100 kHz); PH4/PH5 AF4 open-drain; AIC3204 + PCA9557 | none | VERIFIED | `peripherals.md` (I2C devices) |
| 17 | RTC | `FUN_08022220` (`FUN_080222c0`) | LSE; prescalers 127/255 (1 Hz); time set only when backup register 0 lacks 0xA5A5 | none | VERIFIED | `peripherals.md` (RTC) |
| 18 | SDMMC1 handle | `FUN_08022318` | 4-bit, rising edge, no power save, no flow control, ClockDiv 1; HAL_SD_Init runs later, on card detect; after card init the clock is not capped (HAL rounding: 96/50 = 1), so SDMMC_CK = 48 MHz, 4-bit, default-speed mode (no CMD6, no 1.8 V switch); identification at 400 kHz, 1-bit | 49 (8) | VERIFIED (fields) | `storage.md` |
| 19 | SPI1 | `FUN_080225d4` | master full-duplex, mode 0, /2 from PLL1Q (48 MHz); PB3–5 AF5, CS PG13; polled | none | VERIFIED | `peripherals.md` (SPI1) |
| 20–25 | TIM2, TIM3, TIM4, TIM8, TIM13, TIM14 | `FUN_08022aa0` … `FUN_08022e78` | encoder, 100 Hz, 3.66 kHz PWM, 30 MHz input capture, 50 Hz, 1 kHz (timer clock 240 MHz) | 29 (10), 44 (1), 45 (11), 46 (10) | VERIFIED | `peripherals.md` (timers) |
| 26 | USART1 | `FUN_08023234` (`FUN_08023344`) | 115 200 8N1; PA9/PA10 AF7; RX DMA1 S4 circular | none | VERIFIED | `peripherals.md` (USART1) |
| 27 | USART2 | `FUN_080232bc` (`FUN_08023344`) | 460 800 8N1; PD5/PD6 AF7; DMA1 S2 TX, S3 RX circular | 38 (1) | VERIFIED | `docs/protocol.md` |
| 28 | CRC | `FUN_08020d48` | CRC-32 defaults, byte input; **unused** | — | VERIFIED | this page |
| 29 | I²S2 (SPI2) | `FUN_08021680` (`FUN_080216c4`) | slave full-duplex, 32-bit, 48 kHz; PB12–15 AF5; DMA1 S0 RX / S1 TX circular | 11, 12 (3) | VERIFIED | `peripherals.md` (codec) |
| 30 | LTDC | `FUN_08021878` (`FUN_08021928`) | 1024×600 RGB565, 40 MHz pixel clock | 88 (13) | VERIFIED | `peripherals.md` (display) |
| 31 | application objects | `FUN_08036ec0` | software | — | MEDIUM | `flow.md` |
| 32 | USB OTG_FS | `FUN_0803d90c` (`FUN_0803dc10`) | MSC device, HSI48; PA11/PA12 AF10 | 101 (12) | HIGH | `peripherals.md` (USB) |
| 33–34 | TIM5, TIM7 | `FUN_08022c78`, `FUN_08022ce8` | 20 kHz audio; CW element clock | 50 (4), 55 (0) | VERIFIED | `peripherals.md` (timers) |
| 35 | USART2 RX start | `FUN_080348dc` | 4 KiB D3 buffer; retried on failure | — | HIGH | `docs/protocol.md` |

Device bring-up after the HAL (SDRAM commands, codec, FatFS, GT911,
encoders, PCA9557, INI loads) is listed in `flow.md` step 3b.

**Why this needed the firmware:** the reference manual lists what each
peripheral can do. The order, the bus speeds (TIMINGR values), the
pins, the DMA streams and the priorities actually chosen exist only in
the init code.
