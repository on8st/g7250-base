# Main Unit — peripherals

MCU: STM32H7 (Cortex-M7). Evidence: active interrupt vectors
(`analysis/interrupts/active_irqs.csv`), peripheral register accesses
(`analysis/peripherals/peripheral_accesses.csv`), and literal references
to peripheral base addresses. IRQ numbers use the **STM32H7** NVIC table
(differs from the STM32F4 used by the X6100).

## Active interrupts (→ H7 peripheral)

| IRQ | Handler | H7 peripheral | Inferred role | Conf |
| ---: | --- | --- | --- | --- |
| 3 | `FUN_080222dc` | RTC_WKUP | wakeup timer | HIGH |
| 9 | `FUN_08023584` | EXTI3 | external signal line | HIGH |
| 10 | `FUN_0802358c` | EXTI4 | external signal line | HIGH |
| 11 | `FUN_08023b20` | DMA1_Stream0 | DMA (audio/ADC/UART) | HIGH |
| 15 | `FUN_08023594` | DMA1_Stream4 | DMA | HIGH |
| 16 | `FUN_08023b34` | DMA1_Stream5 | DMA | HIGH |
| 18 | `FUN_080235a0` | ADC1/ADC2 | analog metering | HIGH |
| 23 | `FUN_080235b4` | EXTI9_5 | external signal lines 5–9 | HIGH |
| 38 | `FUN_080235bc` | **USART2** | **inter-MCU SLIP link** | HIGH |
| 40 | `FUN_080235c8` | EXTI15_10 | external signal lines 10–15 | HIGH |
| 47 | `FUN_080235e8` | DMA1_Stream7 | DMA | HIGH |
| 50 | `FUN_080239c8` | TIM5 | periodic service | HIGH |
| 54 | `FUN_080239e0` | TIM6_DAC | time base / DAC trigger | HIGH |
| 55 | `FUN_08023a48` | TIM7 | time base | HIGH |
| 68 | `FUN_08023a64` | DMA2_Stream5 | DMA | HIGH |
| 86 | `FUN_08023a78` | **SPI6** | **CH394 Ethernet** (see SPI6 section) | VERIFIED |
| 118 | `FUN_08023a84` | TIM17 | periodic service (started at boot) | HIGH |
| 129 | `FUN_08023a9c` | BDMA_Channel0 | SPI6 RX DMA (CH394) | VERIFIED |
| 130 | `FUN_08023aa8` | BDMA_Channel1 | SPI6 TX DMA (CH394) | VERIFIED |

IRQ→peripheral mapping is HIGH CONFIDENCE (standard H7 NVIC table);
roles are inferred.

## Buses and peripherals in use

From base-address references and register-access counts:

| Peripheral | Evidence | Role (candidate) | Conf |
| --- | --- | --- | --- |
| **I2C1** | init `FUN_080218b4`; 13 driver funcs | EEPROM + AXP223 PMIC + VersaClock 6 (see I2C section) | VERIFIED |
| I2C2, I2C3 | init only | initialized but unused by any driver (idle/reserved) | HIGH |
| **SPI6** | init `FUN_08023120`, MSP in `FUN_0802317c`, IRQ86, BDMA ch0/ch1 | **WCH CH394 SPI-Ethernet**, exclusive bus (see SPI6 section) | VERIFIED |
| **SPI1** | init `FUN_08023060`, TX-only | five serial-in latches: TX low-pass bank + T/R, RX BPF/front-end, and three built-in ATU latches (see SPI1 section) | HIGH |
| **SPI2** | init `FUN_080230c0`, TX-only | **8-channel 10-bit DAC**: PA bias, drive, fan (see SPI2 section) | VERIFIED |
| **SPI3 = I²S3** | init `FUN_08021c78`, DMA1 Str5/Str0 (IRQ16/11) | full-duplex 48 kHz audio/IQ sample stream (see I²S3 section) | VERIFIED |
| **USART2** | IRQ38; PD5/PD6 AF7; 460 800 8N1; DMA1 S4 TX / S2 RX circular | inter-MCU SLIP link (↔ Control USART2) | VERIFIED |
| USART1 | init `FUN_08024108`; PB6/PB7 AF7; 115 200 8N1; RX DMA1 S3 circular; no IRQ | **debug console (stdout) and service menu**; no CAT (see USART1 section) | VERIFIED |
| UART4 | init `FUN_08024080`; PD1 TX / PH14 RX AF8; 115 200 8N1; no DMA, no IRQ | **initialised but unused** in V1.0.1 | HIGH |
| **ADC1** | IRQ18 + DMA1 Stream7 | RF metering: Vext, forward/reflected power, Idd, PA temperature, EALC, ALC (see ADC1 metering); ADC2/ADC3 unused | VERIFIED |
| TIM5/6/7 | active IRQs | time bases / DAC & audio triggers | HIGH |
| USB OTG | none (earlier "×32 refs" were false positives) | **not used at runtime** (see USB section) | HIGH |
| FMC | none: the only FMC access is the stock SystemInit write that disables bank 1; FMCEN is never set; no AF12 pins; the single aligned 0xC0000000 word is −2.0f in a float table (the earlier ×65 was a byte-pattern count) | **no external memory** | VERIFIED |
| DMA1 (str0/4/5/7), DMA2 (str5) | active IRQs | peripheral DMA | HIGH |
| GPIOA/B/E/G/H | heavy GPIOE/G access | RF band switching, PA control, discretes | HIGH |
| RTC, IWDG1, EXTI | accesses + IRQs | wakeup, watchdog, external lines | HIGH |

Note: SPI/I2C/SAI/UART are driven through HAL handle structs, so their
register-access counts understate real usage; base-address references
and IRQs are the reliable signal.

## SPI6 — WCH CH394 Ethernet, VERIFIED

SPI6 (D3 domain, `0x58001400`) drives the **CH394** SPI-Ethernet
controller behind the LAN connector, and nothing else.
Addresses and values: `analysis/peripherals/spi6_ch394.csv`.

**Evidence chain:**

1. **The init function is `FUN_08023120`** (called from `FUN_0802239c`).
   It fills the HAL handle at **`0x24005420`** with `Instance = SPI6`
   and calls `FUN_0802ebb0` (HAL_SPI_Init).
2. **The MSP init (`FUN_0802317c`, SPI6 branch) does the following:**
   - selects the SPI6 kernel clock (`PeriphClockSelection = 0x4000`,
     source 0 = PCLK4);
   - sets `RCC_APB4ENR.SPI6EN` (bit 5) and enables the GPIOG clock;
   - puts **PG12/PG13/PG14** in AF5, push-pull. On the H7 that is
     SPI6 MISO/SCK/MOSI;
   - links two BDMA channels: **BDMA_Channel0** (`0x58025408`) with
     request **11** = SPI6_RX (periph→mem) and **BDMA_Channel1**
     (`0x5802541C`) with request **12** = SPI6_TX (mem→periph). Both
     are byte-wide, with memory increment, normal mode;
   - sets the NVIC for **IRQ86** to priority 13 and enables it.
3. **The IRQ86 vector `0x08023a78` is a two-instruction stub:** it
   loads `0x24005420` and branches to `0x0802f570`, the HAL SPI IRQ
   dispatcher.
4. **Every function that refers to the handle `0x24005420`, other than
   the init and IRQ code, lies in one contiguous driver cluster**
   (`0x080450c8`–`0x08048d6f`, 26 functions). Every log message in that
   cluster carries the CH394 tag. Between them they report the chip
   version and MAC address, PHY and link state, opening and closing UDP
   sockets, reallocating socket buffers, a hardware reset, and errors on
   SPI write verification, stuck DMA transmits and stuck receives. Three
   of the functions that print these messages (`FUN_080487d4` chip
   version, `FUN_08047abc` write verification, `FUN_080455dc` TX DMA)
   use the handle directly.
5. **No other driver uses the handle,** so the bus serves only the
   CH394 (VERIFIED for references through literal pools).

**Why this needed the firmware:** the STM32H7 datasheet only lists the
pins SPI6 *can* use, and the CH394 datasheet only describes the chip.
Neither says that the G7250 wires the CH394 to SPI6, which pins and DMA
channels it uses, what the bus settings are, or that the bus carries
nothing else. No schematic of the G7250 has been published, and no
hardware was on hand for measurements.

**Link parameters (handle init), VERIFIED:**

| Field | Value |
| --- | --- |
| Role | master, full-duplex (2 lines) |
| Frame | 8-bit, MSB first, mode 0 (CPOL 0, CPHA 0), no CRC, no TI mode |
| NSS | software (`SSM`); NSS pulse mode set in the init struct; **no NSS pin in AF** → CS is a GPIO |
| Prescaler | `/8` from PCLK4 (137.5 MHz), so **SCK ≈ 17.19 MHz** (HIGH CONFIDENCE; assumes the 25 MHz HSE the firmware is built for) |
| Transfer | DMA (BDMA ch0 RX, ch1 TX) + SPI IRQ86 |

**Control pins (VERIFIED unless noted):**

| Signal | Pin | Configuration | Evidence |
| --- | --- | --- | --- |
| CS# | **PG15** | push-pull output, very high speed, idles high | every CH394 register routine asserts it through GPIOG_BSRR bit 31 before its SPI bytes and releases it with bit 15 afterwards; no HAL write-pin calls in the cluster |
| RST# | **PB5** | push-pull output, low at boot, so the chip is held in reset until its init runs | the hardware-reset routine `FUN_08048d28` (called from the hard re-init path `FUN_08048fb0`) stops the SPI6 DMA and releases CS, holds PB5 low for 10 ms, raises it, waits 100 ms and reruns chip init `FUN_080487d4`. That init raises PB5 and then soft-resets the chip (0x80 into register 0x0039). Power-off clears PB5 again |
| INT# | **PC13** (EXTI13) | interrupt input, falling edge, pull-up; EXTI15_10 (IRQ40) at priority 2 | the EXTI callback `FUN_08023ab4` sets a "data pending" byte: `0x2401A646` when settings byte 0x182 = 1 (wfview), `0x24019729` otherwise (HDSDR). The HDSDR tick `FUN_0804af68` receives on socket 0 only when this flag is set or 5 s have passed, and clears it first. The wfview loop `FUN_0804de38` clears its flag and polls anyway, so INT# has no effect in wfview mode. No interrupt register is ever read (see "Interrupt use" below) (HIGH CONFIDENCE) |

The link-monitor "edge detect" in the logs is a software detector of
link-state changes, not a GPIO.

**Register model (HIGH CONFIDENCE).** Each access is one CS window with
three phases:
1. a 16-bit offset, high byte first;
2. a control byte: block select in bits 7..3, the write flag in bit 2,
   variable-length mode;
3. the data bytes.

Block selects: common registers; socket *n* registers `(n<<5)|0x08`;
socket *n* TX buffer `(n<<5)|0x14`; socket *n* RX buffer `(n<<5)|0x18`.
Every offset the driver uses matches the public WIZnet W5500 register
map. That covers the common registers (gateway, mask, MAC, IP,
socket-interrupt mask, PHY configuration, version) and the per-socket
registers (mode, command, status, port, destination IP/port, buffer
sizes, TX write pointer, RX received size, RX read pointer).

**Interrupt use (HIGH CONFIDENCE).** Chip init enables all eight
sockets in the socket-interrupt mask (SIMR, `0x0018` = 0xFF). Each used
socket (HDSDR 0, wfview 1–3) enables only the RECV interrupt (Sn_IMR,
`0x002C` = 0x04, `FUN_0804785c`). Each receive call starts by writing
0x04 to Sn_IR (`0x0002`), a blind write-1-to-clear of RECV, and then
reads the received size. The interrupt register (`0x0015`), its mask
(`0x0016`), SIR (`0x0017`) and Sn_IR are never read: INT# only means "a
datagram arrived", and the code does not ask which socket fired.
SEND_OK is never enabled. Correction: an earlier version said the
wfview server reads its flag; it only clears it.

**Write to 0x0039 (position VERIFIED, effect HYPOTHESIS).** After
raising RST# (PB5), chip init `FUN_080487d4` writes 0x80 to common
register `0x0039`, waits 20 ms, then reads the same register back and
logs it as the chip version.
- **What WCH documents (VERIFIED).** The CH394 manual (CH394DS1 V1.2,
  table 6-1 and §7.1.16) lists `0x0039` as CHIPV, a read-only version
  register, like VERSIONR on the W5500, with no write function. The
  documented software reset is bit 7 of MODE (`0x0000`, §7.1.1), which
  this driver never writes. The common register map is W5500-compatible;
  the CH394 adds CHIP_CONFIG at `0x003E` (IPv4/IPv6 select).
- **Timing (HIGH).** Chip init raises PB5 with its very first
  statement and sends the 0x0039 write right after, with no delay: it is
  complete a few microseconds after RST# goes high. The read-back follows
  HAL_Delay(20), so 20–21 ms later. At boot nothing else touches the
  chip before this; PB5 has been low since GPIO init (more than 2 s).
- **Reading (HIGH for timing, HYPOTHESIS for effect).** At boot the
  write falls inside WCH's 12–19 ms no-access window after RSTB goes
  high (§10.4.3), so the chip most likely ignores it. On the hard
  re-init path (`FUN_08048d28`: RST# low 10 ms, high, then 100 ms) it
  reaches a ready chip, where 0x0039 is the read-only CHIPV. Either way
  the reset comes from the RST# pulse; only the 20 ms wait before the
  version read respects the window.

**Transfers.** Register writes are byte-by-byte SPI transmits.
Single-register reads switch SPI6 to receive-only and poll it. Bulk
payload moves by DMA. The socket plan, the services and the self-heal
logic are in `network.md`.

## SPI1 — RF filter/switch shift registers, HIGH CONFIDENCE

SPI1 has no IRQ and no DMA. The init (`FUN_08023060`) sets it up as a
**transmit-only master**: 8-bit, mode 0, software NSS, prescaler /2.
The MSP init (`FUN_0802317c`, SPI1 branch) selects the SPI1/2/3 kernel
clock and puts **PG11** (SCK) and **PD7** (MOSI) in AF5. There is no
MISO pin.

A small device layer drives it (`FUN_08044010` binds a device,
`FUN_08043fe0` writes a frame). Each device has its own SPI handle and
two GPIO callbacks:

- a **latch line**, pulled low before each frame and high after it;
- an **output-enable line**, held inactive during init and made active
  after the first frame is written.

This latch-plus-output-enable pattern, with write-only frames and a
RAM shadow copy, is that of serial-in/parallel-out shift registers of
the 74HC595 kind (chip type: HYPOTHESIS). SPI1 carries two such chains:

| Chain | Shadow | Frame | Latch | Output enable | Driver functions |
| --- | --- | --- | --- | --- | --- |
| A (8 bit) | `0x24017884` | 1 byte | **PA11** | **PA10** | init `FUN_080495a0`; band `FUN_080495d8`; bit 7 `FUN_0804960c` |
| B (24 bit) | `0x24019148` | 3 bytes, MSB first | **PH12** | **PH10** | init `FUN_0804a2f0`; BPF `FUN_0804a430`; others below |

Both chains are written from the band and frequency update path
(`FUN_0803ec3c`, `FUN_0803ebb0`) and from the RX/TX switch-over paths
(`FUN_0803d03c` to RX, `FUN_0803e338` to TX).

**Chain A: TX low-pass filter bank plus T/R line**
- **Bits 0–6** select one filter at a time from the operating
  frequency. Edges: 2, 4, 8, 15, 23, 33 and 60 MHz, giving 7 bands.
  Above 60 MHz no bit is set. (HIGH CONFIDENCE.) These are the usual
  edges of an HF+6 m PA low-pass bank (role: HIGH CONFIDENCE).
- **Bit 7** is set by the switch-to-TX path and cleared by the
  switch-to-RX path, so it is a T/R relay or PA-enable line. (HIGH
  CONFIDENCE.)

**Chain B: RX band-pass bank and front-end switches**
- **Bits 0–15 and 23** select an RX band-pass/preselector segment from
  the frequency (`FUN_0804a430`). Edges are at about 1.6, 2, 3, 4.5,
  6.5, 8, 10, 15, 22, 30, 50, 54 and 70 MHz. A few segments have an
  alternate selection that depends on two caller flags (TX state and
  one other). The firmware only rewrites the chain when the value
  changes. (Mechanism HIGH CONFIDENCE; exact segment edges MEDIUM
  CONFIDENCE.)
- **Bits 16 and 17** are set by operating mode: bit 16 for mode codes
  4, 8 and 9, bit 17 for mode codes 6 and 7, and neither for the
  others. This is probably an IF/filter path per mode. (Mechanism HIGH
  CONFIDENCE; meaning HYPOTHESIS.)
- **Bits 18–20** form a 1-of-3 selector from one TRX-state setting
  (`FUN_0804a370`). **Bits 21–22** form a 2-way selector from another
  setting (`FUN_0804a608`). The switch-to-TX path sets both to the "off"
  value. Bits 18–20 are the **preamp** selector (from the VFO PRE byte)
  and bits 21–22 the **attenuator** (from the VFO ATT byte); VERIFIED.
- **Init value:** bits 17, 20 and 22 set, all others cleared.

**Built-in ATU latches, also on SPI1 (HIGH CONFIDENCE).**

Three more SPI1 devices sit in the initial `.data` image as
{SPI handle, CS port, CS pin} records. Each keeps an 8-bit shadow and
sends it as one byte, with the CS line acting as the latch.

| Record | CS | Driver functions | Callers | Role |
| --- | --- | --- | --- | --- |
| `0x2400484c` | **PD13** | `FUN_08044d38`, `FUN_08044d78`, `FUN_08044ea4` (bit pairs 0–1, 2–3, 4–5) | ATU tune/recall (`FUN_08041358`, `FUN_080412bc`) and the settings consumer | ATU control: network style and bypass/enable lines |
| `0x24004858` | **PG2** | `FUN_08044e34` (8-bit value, bit order kept) | ATU tuning engine (`FUN_08042bd8`, `FUN_08042c48`, `FUN_08042d34`) | one reactance bank (L or C) |
| `0x24004864` | **PG5** | `FUN_08044dc0` (8-bit value, bit order mirrored) | same | the other reactance bank |

The tuning engine logs coarse and fine search steps over a style, an
L value and a C value. These are the same three bytes the EEPROM ATU
table stores per 100 kHz slot (`memory_map.md`). PG2 is the L bank
(bit order kept) and PG5 the C bank (mirrored); see `docs/atu.md`.

SPI1 therefore drives five serial-in latches in total: chain A, chain
B and these three ATU latches.

## SPI2 — PA bias / drive DAC, VERIFIED

> Correction: an earlier version of this file called SPI2 idle. Its
> transmit routine is reached only through a function pointer, so the
> first handle scan missed it.

**Bus.** SPI2 is transmit-only (PB13 SCK, PB15 MOSI; init
`FUN_080230c0`). It drives an **8-channel 10-bit serial DAC**
(BU2506FV-type, see below) through a small device layer at
`0x240177cc`.

**Frame.** `FUN_08045088(ch, value)` clamps the value to 0x3FF and keeps
a shadow copy at `0x240177bc`. It sends two bytes:
1. the 4-bit address (the bit-reversed channel number, from a table at
   `0x0807e9c4`), shifted left by 2, with value bits 9..8;
2. value bits 7..0.

The transmit callback `FUN_08045018` (HAL SPI transmit on the SPI2
handle) is followed by a high pulse on **PB12** (load strobe). At init
`FUN_0804506c`, **PB14** is pulsed low and then high (reset/enable).

**Chip (HIGH CONFIDENCE for the family, MEDIUM for the part).** The word
format matches Rohm's BU2505FV/BU2506FV 10-bit DACs (public datasheet)
with REVERSE tied low: a 14-bit shift register takes a 4-bit address
LSB first and 10 data bits MSB first, so the two leading zero bits of
the 16-bit SPI frame shift out. LD high (PB12) latches the word; PB14 is
the active-low RESET that forces all outputs to VrefL. Eight channels
and the firmware's refusal of channels above 8 point to the BU2506FV.
The part has no power-down or reference commands, and the firmware
sends none. SPI2 runs at 6.144 MHz (12.288 MHz kernel clock / 2),
mode 0.

| Ch | Use |
| ---: | --- |
| 1 | drive code (`FUN_0803acb0`, clamped to 300–1000) |
| 2 | driver-stage bias |
| 3 | fan level (10·T − 50 above 35 °C, off at ≤ 30 °C) |
| 4 | unused (never written; stays at VrefL after reset) |
| 5 | 0x3FF once at boot, next to the RF-gain DAC; 0 at power-off; no other writer (role HYPOTHESIS: enable or reference) |
| 6 | power reference 75·√P (P = TX power, ×0.25 in AM/SAM) |
| 7 | final-stage 2 bias |
| 8 | final-stage 1 bias |

**Bias trim at boot** (in `main`). The static Idd is measured with all
bias at 0. Then each stage is trimmed on its own:
- **targets:** static Idd + 0.5 A for the driver, + 0.9 A for each
  final;
- **loop:** 100 ms settle per step, 30 counts per amp, converged within
  ±0.1 A, no iteration cap;
- **valid ranges:** driver code 1–750, finals 1–1200. The finals' upper
  bound is above the 10-bit clamp (a quirk). A code outside its range
  fails the trim: the stage goes to 0 and the stored value is kept.

A changed value is saved at shutdown. TX entry writes the calibrated
biases to channels 2/8/7; RX entry and power-off write 0.

**TX power → DAC** (HIGH CONFIDENCE for the power function). The channel 1
drive code is:
- below 50 MHz: 828.3 − 1295·P^(−0.6732);
- above: 2·(375.5 + 2.231·P^1.164).

Both are clamped to 300–1000. Channel 6 = 75·√P. The ATU sets channel 6
directly for its 10 W carrier and restores it afterwards.

**Why this needed the firmware:** the DAC bus, channel map, trim targets
and power curves cannot be seen on a schematic. Values are in
`analysis/peripherals/spi2_dac.csv`.

## SPI3 = I²S3 — audio codec link, VERIFIED

SPI3 runs in **I²S mode**: init `FUN_08021c78` (HAL I²S init
`FUN_0802bde8`), MSP init `FUN_08021ccc`.

| Parameter | Value |
| --- | --- |
| Mode | **master, full-duplex** |
| Standard | Philips I²S, clock polarity low, MSB first |
| Data format | 32-bit data field |
| Audio frequency | **48 kHz** |
| MCLK output | disabled |
| Kernel clock | SPI1/2/3 mux set to the external clock pin I2S_CKIN (HIGH CONFIDENCE; value 3 of the mux). SPI1 and SPI2 share this mux |
| Pins (AF6, very high speed, pull-up) | **PA15** WS, **PC10** CK, **PC11** SDI, **PC12** SDO |
| TX DMA | **DMA1 Stream5**, request 62 (SPI3_TX), mem→periph, half-word, **circular**, very high priority, IRQ16 |
| RX DMA | **DMA1 Stream0**, request 61 (SPI3_RX), periph→mem, word, **circular**, very high priority, IRQ11 |

The stream starts in `FUN_08035fac` (called from `FUN_0803e2f4`). It
clears two 3 KiB buffers and starts a full-duplex DMA transfer over
both. Half-transfer and transfer-complete callbacks
(`FUN_08035ed4`, `FUN_08035f40`) set ping-pong flags for the audio/DSP
code. This explains the active IRQs 11 and 16 and accounts for
DMA1 Streams 0 and 5. When the DSP has not consumed the previous half,
the callbacks raise the codec buffer-underrun flag (`0x2400BBC6`). The
buffers are in D2 SRAM: `0x30004360` (TX) and `0x30003760` (RX).

**Role (HIGH CONFIDENCE).** I²S3 carries **audio**, not RF I/Q. Its
peer is the AIC3204 codec, which runs as I²S slave.
- **TX direction:** the RX DSP writes its final stereo audio, the CW
  sidetone and the TX monitor into the transmit buffer, which feeds the
  codec DAC.
- **RX direction:** the TX DSP takes one channel of the receive stream
  (the mic, on the codec's left ADC) as its modulating signal.

**Why this needed the firmware:** no G7250 schematic is public. The
STM32 datasheet only lists possible pin functions. The filter edges,
the shift-register bit assignment, the latch/enable pins and the I²S
format exist only in the firmware's behaviour.

## I2C devices — VERIFIED

All application I2C traffic is on **I2C1** (handle `0x24004F38`). I2C2 and
I2C3 are initialized (`FUN_08021920` / `FUN_0802198c`) but **no driver
uses them** — idle/reserved (vestigial X6100 topology, or strapped-only).

Bus speeds (register values VERIFIED, rates HIGH): I2C1 about 400 kHz
(TIMINGR 0x00D049FB), I2C2 about 100 kHz, I2C3 about 400 kHz with its
own address 0x72. See `docs/hardware_init.md`.

I2C transfer helpers: read `FUN_0802BB3C(handle, devAddr, memAddr,
addrSize, buf, len, timeout)`, write `FUN_0802B940(...)`. `devAddr` is the
8-bit (`<<1`) address written to `I2C_CR2.SADD`.

| 7-bit | 8-bit | Device | Evidence | Conf |
| ---: | ---: | --- | --- | --- |
| `0x50` | `0xA0` | 24Cxx **EEPROM** | NVPARAM (settings 424 B + header/checksum), calibration, ATU data; backup-SRAM (`0x38800000`) cached; save and read-error log lines (see memory_map.md) | VERIFIED |
| `0x18` | `0x30` | **TLV320AIC3204** audio codec | driver struct in `.data` (`0x2400487c`) holds the address and the I2C1 handle; all writes go through HAL I2C transmit; init `FUN_0804a65c`; reset PB10 (see Audio codec section) | VERIFIED |
| `0x34` | `0x68` | **AXP223 PMIC** | strings `axp223 (0x68) write reg`, `Set ALDO1/DCDC1/2/3/5/DLDO1 Vout`, `Init PSU`; driver `FUN_0804a8e8` | VERIFIED |
| `0x68`/`0x6A`* | `0xD0`/`0xD4`* | **VersaClock 6** clock generator | driver `FUN_0804abbc` logs a successful init-table write; it tries both addresses, then writes a 112-B (`0x70`) register init table | VERIFIED |

\* The VersaClock driver tries 0x68 first, then 0x6A (see the
VersaClock 6 output plan below).

### Audio codec — TLV320AIC3204 on I2C1 0x18, VERIFIED

**Init** (`FUN_0804a65c`, from `main`):
1. Holds PB10 low for 100 ms as reset, then high.
2. Records MCLK = 12.288 MHz and fs = 48 kHz, then soft-resets the codec.
3. Programs a register table (`FUN_08044504`):
   - clock: CODEC_CLKIN = MCLK, NDAC 1, MDAC 2, AOSR 128, so
     fs = MCLK/256;
   - interface: **I²S, 32-bit words, BCLK and WCLK as inputs**, so the
     codec is a slave;
   - ADC and DAC powered;
   - DAC routed to the HPL/HPR and LOL/LOR outputs.
4. Sets HPL/HPR to +10 dB and LOL to +1 dB (LOR stays muted), unmutes
   the ADC and DAC, and enables MICBIAS.
5. Routes IN1L (+) and IN2R (−) to the left MICPGA, a differential
   input (MEDIUM). The right MICPGA has no input. The DOSR write sets
   only reserved bits of the high byte, so DOSR stays 128. Register
   values: `control_unit/analysis/peripherals/aic3204_init.csv`.

MICPGA gain starts at 15 dB. It is updated from the MIC gain field of
the SLIP `0x01` settings block (`FUN_0804a790`). MCLK is the same
12.288 MHz external clock as the MCU's I2S_CKIN (MCLK output of the MCU
is off).

## FPGA bus — bit-banged parallel GPIO, VERIFIED

The FPGA is the **RF I/Q front end**. The MCU reaches it over an 8-bit
bit-banged bus on GPIOE. No FMC bank is used.

| Signal | Pin | Use |
| --- | --- | --- |
| D0–D7 | PE0–PE7 | bidirectional data; direction switched by rewriting GPIOE_MODER |
| CMD | PE8 (with PE9) | both pulsed together latch a command byte (`FUN_08039f70`) |
| STB | PE9 | per-byte strobe (`FUN_08039fbc` / `FUN_08039fcc`) |
| DRDY | PE10 | rising-edge EXTI (EXTI15_10, IRQ40, priority 2); the callback runs the FPGA sample routine `FUN_08039078` |

| Code | Function |
| --- | --- |
| 0x00 | loopback/test register; the boot bus check writes and reads back 0–255 (`FUN_0803a3a0`) |
| 0x01 | 24-byte configuration frame (`FUN_08039238`), sent when the frequency-apply routine `FUN_0803d658` raises its request flag: status flag bytes, three big-endian 32-bit NCO words from the TRX struct (offsets 0x200–0x208), per-rate calibration bytes from the calibration block, and the rate index |
| 0x02 | status read (`FUN_08039950`): two flag bits (TRX+0x22C, +0x22B; ignored for the first 50 reads), then three big-endian 16-bit values: TRX+0x226, the ADC peak (TRX+0x224, shown in the service menu) and `0x2400E9E4` |
| 0x03 | TX I/Q write (`FUN_08039d90`): two signed 24-bit values per sample. Zeros are sent for the first 200 interrupts after keying |
| 0x04 | RX I/Q read (`FUN_08039a6c`): two signed 24-bit values per sample, scaled to float. 8/4/2/1 samples per interrupt for sample-rate settings 0–3, in 192-sample blocks. An unconsumed block sets the FPGA underrun flag |
| 0x05/0x06, 0x09/0x0A | two pulse commands: init sends the "off" codes 0x06 and 0x0A; pulse helpers exist (`FUN_08039fdc`, `FUN_0803a02c`) but nothing sets their request flags in V1.0.1 (HIGH); roles unknown |
| 0x08 | 3-byte FPGA firmware version read |

**Bitstream and version.**
- **Loading (HIGH CONFIDENCE).** The MCU never loads or reprograms the
  FPGA. The strings about a missing FPGA and FPGA-flash verification
  are unreferenced, and the command set has no flash command, so the
  FPGA boots from its own configuration flash.
- **Version.** The 3-byte version read at boot (command 0x08) is only
  logged. The version reported to the Control Unit in the slave-info
  block is the compiled-in "1.0.0" string.
- **NCO tuning words (VERIFIED).** The three words are 32-bit phase
  increments for a 122.88 MHz clock (f × 2³² / 122.88 MHz). The
  frequency is first folded into 0–61.44 MHz (Nyquist zones of
  122.88 MS/s), with a spectral-inversion flag that the RX reader uses.
  TRX+0x208 is the main VFO plus RIT (plus the transverter offset when enabled; dormant in V1.0.1, see `flow.md`),
  TRX+0x204 the other VFO plus RIT (MEDIUM for its role). 122.88 MHz is
  half of the VersaClock 6 output, which supports OUT1 = 245.76 MHz
  (MEDIUM). Branches above 245.76 MHz that divide the frequency by 3 or
  9 are unused on this radio (shared code, MEDIUM).
- **Sample rate (MEDIUM).** The interrupt delivers 8/4/2/1 samples for
  rate index 0–3, and each 192-sample block is decimated by the same
  factor into the 48 kHz chain. The wfview span thresholds are
  48/96/192 kHz for indices 3/2/1. With the 122.88 MHz NCO clock this
  gives a 48 kHz PE10 rate and 384/192/96/48 kS/s I/Q; still unmeasured.

**Error handling — log only (VERIFIED).**
- **Boot.** The bus test writes and reads back 0–255 twice. A mismatch
  is logged and clears an "FPGA OK" flag that nothing reads; there is
  no retry or reset.
- **Runtime.** The TIM6 interrupt runs a health check (`FUN_08037e68`)
  about ten times a second:
  - It logs an FPGA bus error when the first RX sample stays unchanged
    for more than five checks (outside WFM, and not during the bus
    test).
  - It logs and clears the FPGA and codec buffer-underrun flags.

None of these errors leads to an FPGA re-init or a TRX reset.

**Why this needed the firmware:** no schematic or FPGA documentation is
public. The bus, its command set, the 24-bit I/Q format and the codec
address and mode exist only in the code.

## AXP223 rail plan — VERIFIED (use: HYPOTHESIS)

`FUN_0804a8e8` is called from `main` and only ever addresses the 7-bit
address 0x34. It first forces a safe state with only DCDC1 on, then
programs the voltages, waits 10 ms and enables everything in one
write. Each code matches the voltage the driver logs just before it
writes. There is no sequencing between rails. `FUN_0804a854` restores
the safe state; it is called from the shutdown routine `FUN_08021edc`
and before the jump to the bootloader in `FUN_0802200c`.

| Rail | Register / code | Voltage | Enabled | Likely use |
| --- | --- | ---: | :---: | --- |
| DCDC1 | 0x21 / 0x11 | 3.3 V | yes (also in safe state) | MCU 3.3 V (MEDIUM) |
| DCDC2 | 0x22 / 0x1E | 1.2 V | yes | FPGA core (HYPOTHESIS) |
| DCDC3 | 0x23 / 0x3C | 1.8 V | yes | FPGA/converter 1.8 V (HYPOTHESIS) |
| DCDC5 | 0x25 / 0x1E | 2.5 V | yes | FPGA aux / LVDS bank (HYPOTHESIS) |
| ALDO1 | 0x28 / 0x1A | 3.3 V | yes | analog/clock 3.3 V (HYPOTHESIS) |
| DLDO1 | 0x15 / 0x0B | 1.8 V | yes | analog 1.8 V (HYPOTHESIS) |
| DCDC4, DC5LDO, ALDO2/3, ELDO1–3, DLDO2–4, DC1SW | — | — | off | unused |

The enables are reg 0x10 = 0x6E and reg 0x12 = 0x08, decoded with the
public AXP22x bit layout (HIGH CONFIDENCE). Reg 0x13 bit 0 is set; its
meaning is a HYPOTHESIS. Per-rail values are in
`analysis/peripherals/axp223_rails.csv`.

### VersaClock 6 output plan — HIGH (frequency MEDIUM)

`FUN_0804abbc` (called from `FUN_0804ae68`) writes registers 0x00–0x6F
in one burst from a 112-byte table in flash at `0x0807F840`. It tries
the 7-bit address **0x68 first**, then 0x6A, so the earlier "0x6A" is
only the fallback. No code reprograms the chip later.

- **Reference:** crystal input; CLKIN and the doubler are off; the
  divide-by-2 pre-divider is selected. The PLL feedback is integer 222
  with no fraction, so VCO = 111 × Fref. (HIGH CONFIDENCE.)
- **Dividers:** FOD1 is a fractional divider of 11.7431640625, without
  spread spectrum. FOD2–4 are disabled. (HIGH CONFIDENCE.)
- **Outputs:** OUT1 is LVDS at 3.3 V and enabled. OUT2–4 are set up as
  3.3 V CMOS, but their buffers are off. (OUT1 HIGH CONFIDENCE,
  OUT2–4 MEDIUM CONFIDENCE.)
- **Frequency:** a 26 MHz crystal gives VCO = 2886 MHz and
  **OUT1 = 245.76 MHz**, exactly 5120 × 48 kHz. Other common crystals
  give no round value. (MEDIUM CONFIDENCE.) If the output stage halves
  the divider output, OUT1 is 122.88 MHz instead (HYPOTHESIS).
- **Role:** the chip has a single output, and it is **not** the MCU's
  HSE: it is far above the HSE limit, and the chip is programmed after
  the MCU clocks are up. It is the sample clock for the FPGA or the
  data converters (MEDIUM CONFIDENCE). The I²S3 kernel clock on
  I2S_CKIN is then most likely derived from it by the FPGA
  (HYPOTHESIS). Decoded values are in
  `analysis/peripherals/vc6_clock_plan.csv`.

**Why this needed the firmware:** both chips are programmed over I2C
or from OTP. Their datasheets give register maps only. The voltages,
enables, divider plan and output types exist only in the init code and
table.

## USB — not used at runtime, HIGH CONFIDENCE

The Main application never brings up USB:
- No 4-byte word in the image equals an OTG register base (0x40080000
  or 0x40040000).
- The OTG interrupt vectors point at the default handler.
- `main` calls no USB init, and the bootloader-jump teardown list has no
  USB handle.

The earlier "×32 references" were Thumb instruction encodings and
floating-point constants that happen to fall in the 0x4008xxxx range.
This matches the manual (no USB connector on the Main Unit). On the
likely H725/H735 part there is in any case only an OTG_HS block at
0x40040000.

## Bootloader jump — `FUN_0802200c`, HIGH CONFIDENCE

**Trigger.** It is reached only from the RTC wake-up vector stub
(IRQ 3, `0x080222dc`), before the HAL wake-up handler.

**Sequence:**
1. Logs a notice.
2. Masks the EXTI and USART interrupts.
3. De-initialises TIM5/6/7/17, I²S3, ADC1, DAC1, USART1/2, UART4,
   I2C1/2/3, an MDMA channel and CRC.
4. Puts the PMIC in its safe state, gates the DMA clocks and disables
   the caches and SysTick.
5. Resets RCC and the peripheral blocks, and clears all NVIC
   enable/pending bits.
6. Sets VTOR = 0x08000000, loads the MSP from there and jumps through
   0x08000004, into the bootloader in flash sector 0, which is not part
   of this image.

**What arms it — nothing in V1.0.1 (HIGH CONFIDENCE).**
- The RTC write-protection unlock appears only in the HAL RTC init,
  set-time and set-date functions, so there is no wake-up-timer setup.
- NVIC IRQ 3 is never enabled.
- The RTC handle is used only by the RTC init.

The jump is therefore dead code in this release. The bootloader runs
only from reset, through the reset vector at `0x08000000`. Just
before the branch, the routine clears all NVIC enable and pending bits,
selects the MSP and re-enables interrupts.

**Bootloader self-update — not present (HIGH CONFIDENCE).**
- **Orphan strings.** Strings describe a checksum check of a new
  bootloader, an erase of flash sector 0, a 128-block write, an erase
  of sector 6 and a flash lock. No code refers to them, and the image
  has no flash unlock key.
- **Plausible design (HYPOTHESIS).** A staging copy in sector 6
  (`0x080C0000` on a 1 MiB part), copied to sector 0 in 128 × 1 KiB
  blocks.
- **No transfer path.** The Main application has no firmware-transfer
  path (no SLIP type, no LAN code), so updates are entirely the
  bootloaders' job.

**Update transport — not in the applications (HIGH).** The Control
application has no forwarding code either: no file name, no SLIP type
and no Main reset or boot line (its unexplained outputs PG9/PG10 are
set high once and never changed). The transport therefore lives in the
two sector-0 bootloaders. Most likely the image goes from the MSC disk
via the Control bootloader over USART2, or in through the Main console
USART1 (HYPOTHESIS). Settling it needs a sector-0 dump or a bus capture
during a vendor update.

**Why this needed the firmware:** the manual only says there is no USB
connector. Only the absence of init code, vectors and base constants
settles that there is no internal USB link either.

## ADC1 metering and DAC1 — VERIFIED

**Setup.** Only ADC1 is used; the ADC3 references are generic HAL
instance checks.
- `FUN_08020de0`: 12-bit, 7-rank scan, continuous, software trigger,
  sampling 32.5 cycles, async clock /32 from PLL2P.
- Circular DMA1 Stream7 into `0x30003740`.
- Offset and linearity calibration at boot; `FUN_08021148` starts the
  DMA.

The conversions run inside the DMA1 Stream7 interrupt (`FUN_080235e8`),
not in a HAL callback. Results land in the meter struct `0x24004D5C`
(`analysis/memory/meter_fields.csv`).

| Rank | Input | Pin | Quantity | Conversion |
| --- | --- | --- | --- | --- |
| 1 | INP1 | PA1_C (analog switch opened) | Vext | IIR, steady state 0.004835 V/LSB (divider HYPOTHESIS) |
| 2 | INP2 | PF11 | forward detector | x = 2·raw; P = 0.003405·x + 5.557e-6·x² W |
| 3 | INP3 | PA6 | reflected detector | same law |
| 4 | INP4 | PC4 | Idd | IIR, 0.008059 A/LSB |
| 5 | INP5 | PB1 | PA temperature | double IIR, then a 4096-entry °C table (NTC, B ≈ 3950, raw 2047 = 25 °C) |
| 6 | INP8 | PC5 | EALC % | IIR of raw·100/4095 |
| 7 | INP14 | PA2 | ALC level % | filtered from the EALC value rather than its own previous value (a quirk) |

**SWR.**
- **Calculation.** Every 15 samples, from the summed forward (F) and
  reflected (R) values: Γ = min(0.999·F, max(R·REVGAIN − F·LEAK, 0))/F
  and SWR = (1+Γ)/(1−Γ), or 999.9 if F ≤ 0.
- **Reporting.** The reported SWR is updated only after 20 consecutive
  blocks with enough signal; otherwise it is 1.0.
- **Directivity correction.** `FUN_0803ec3c` sets REVGAIN (0.68–0.725)
  and LEAK (0.001–0.09) piecewise-linearly from the frequency on every
  frequency change. These are firmware constants, not calibration
  fields (HIGH CONFIDENCE).
- **Forward power** is shown as a peak hold.

**EALC/ALC.** They are measured and displayed only; no firmware loop
uses them.

**DAC1.**
- OUT1 (PA4) is the **RF-gain control voltage**: 0x447 + 30·gain,
  maximum 0xFFF, and 0xFFF at boot.
- OUT2 (PA5) is configured but never written.
- No trigger is used: TIM6 is a 100 Hz housekeeping tick, not a DAC
  trigger.

**Why this needed the firmware:** the pin-to-quantity assignment,
scaling constants, detector law, directivity correction and gating
exist only in code.

## USART1 — debug console and service menu, VERIFIED

**Console.** stdout (all log output) goes out blocking on USART1 with a
100 ms timeout (`_write` → `FUN_080417a0`). Reception uses circular DMA
into a 1 KiB ring at `0x30004F60` (set up by `FUN_08041c10`).

**Service menu.** Pressing Enter on the console switches the superloop
into a VT100 service menu (`FUN_08041a94`):
- the cursor keys select an item and change the step size;
- `+` and `-` change the value;
- the screen refreshes once per second.

The five items are:
1. the reverse-power gain;
2. the leakage-compensation factor (the SWR directivity terms of the
   meter block);
3. the drive-DAC level (SPI2 DAC channel 1);
4. the "opt filter" toggle (settings byte 0x008 bit 0);
5. SPI2 DAC channel 6.

A telemetry block follows the items: CPU load, S-meter, ADC peak, AGC
gain, PA temperature, ALC, Vext/Idd, forward and reverse power.

There is no CAT protocol on this port. It is a factory/service console.

## Timer map — VERIFIED (timer clock 275 MHz)

Rate = timer clock / (PSC+1) / (ARR+1). The table is in
`analysis/peripherals/timers.csv`.

| Timer | Rate | IRQ (prio) | Role |
| --- | --- | --- | --- |
| TIM5 | 20 kHz | 50 (5) | DSP scheduler |
| TIM17 | 20 kHz | 118 (4) | sample-side service: wide-span decimation and wfview TX audio |
| TIM7 | 1 kHz | 55 (9) | 1 kHz service and SLIP status frames |
| TIM6 | 100 Hz | 54 (8) | housekeeping: the PTT/T-R state machine every tick, error reports at 10 Hz, CPU-load statistics about once per second |
| TIM4 | 19.92 kHz PWM on PD12, pulse 0 | — | **never started**, so PD12 stays low in V1.0.1 |

No other timer is initialised.

## GPIO pin map — register values VERIFIED, roles as noted

**Method.** Built from every HAL_GPIO_Init call (`FUN_0802abd8`, 49
call sites: the main GPIO init `FUN_08021470`, `FUN_08021830` and the
MSP functions), the write/read helpers (`FUN_0802afc8` /
`FUN_0802afbc`) and inline BSRR/ODR writes. The full table is
`analysis/peripherals/gpio_map.csv`. Alternate-function → signal
mapping follows the STM32H7 datasheet tables (HIGH CONFIDENCE).

**Highlights:**
- **FPGA bus (HIGH CONFIDENCE).** PE0–PE7 form a bit-banged 8-bit
  bidirectional data bus with PE8/PE9 as strobes. The FPGA init and bus
  test `FUN_0803a3a0` switches PE0–7 between output and input and
  pulses PE9 for every byte. PE10 (rising-edge EXTI) is an FPGA event
  line (MEDIUM CONFIDENCE).
- **EXTI inputs (HIGH CONFIDENCE).** All are dispatched by the callback
  `FUN_08023ab4`:
  - PC13 = CH394 INT#;
  - PE10 = FPGA event;
  - PH6 = PTT in (both edges);
  - PH3/PH4 = CW key or paddle contacts (both edges). A setting swaps
    them, like a paddle-reverse option.
- **T/R switching lines (VERIFIED).** In RX, PF4 is high and PF6, PC7
  and PH11 are low. In TX it is the reverse. The function of each line
  is a HYPOTHESIS.
- **SPI1 latch lines.**
  - Chain A: PA11 latch, PA10 enable.
  - Chain B: PH12 latch, PH10 enable.
  - ATU latches: PD13, PG2 and PG5.
- **Clocks.**
  - **PC9 = I2S_CKIN (AF5),** the external clock for SPI1/2/3 and I²S3.
  - PD12 = TIM4_CH1 PWM, role unknown.
- **Analog.** ADC1 inputs are PA2, PA6, PB1, PC4, PC5 and PF11, on
  circular DMA1 Stream7. DAC1 outputs are PA4/PA5.
- **Other.**
  - PB10 = audio-codec reset.
  - PH9 is driven high just before the PSU init (power enable?
    HYPOTHESIS).
  - The roles of several other outputs and inputs are open; see the CSV.

**Two absences (MEDIUM CONFIDENCE):**
- **No FMC pins are configured.** No pin uses AF12, and the FMC is
  never enabled: the Main Unit has **no external memory** (VERIFIED,
  see the FMC row above).
- **No USB pins are configured.** PA11/PA12 are plain outputs, which
  supports the vestigial-USB reading.

**Why this needed the firmware:** no schematic is public. Datasheets
only list which functions a pin *can* carry, not the mode, level, edge
or driver actually used.

### vs. X6100

- **No battery** (G7250 is mains/PSU powered): the X6100's **MAX17205
  fuel gauge (0x36) is absent**; instead an **AXP223 PMIC (0x34)**
  generates the rails. This is the main I2C topology difference.
- The **VersaClock 6** clock generator carries over from the X6100
  (which used a 5P49V60 at 0x6A) — LO / sample-clock synthesis.
- Of the X6100 codecs, the **TLV320AIC3204 (0x18) is on Main I2C1**, and
  the CS4272 is absent. RF I/Q does not go through a codec here; it comes
  from the FPGA over a parallel GPIO bus (see the FPGA bus section).
- NV storage via 24Cxx EEPROM + backup-SRAM cache (the X6100 kept
  calibration in flash).

## Differences vs. X6100

- MCU is STM32H7/M7 (vs. F427/M4): different NVIC table and peripheral
  map; adds caches. The Main Unit has no external SDRAM (the Control
  Unit does, for its display).
- **Networking:** CH394 Ethernet (**LAN connector is on the Main Unit**,
  per the manual) + wfview remote (no X6100 equivalent). The CH394 sits
  on **SPI6** with BDMA (VERIFIED, see SPI6 section). The PC CAT/audio USB is on the
  **Controller** (CH342), not here — see `docs/hardware.md`.
- **Power:** no battery (mains/PSU) → AXP223 PMIC replaces the X6100
  MAX17205 fuel gauge. Only I2C1 is actively used (I2C2/I2C3 idle).
- Retains the X6100's VersaClock-family clock generator.

## RTC — VERIFIED

The RTC is clocked from LSI, giving about 1 Hz (untrimmed). Time and
date are reset to 00:00:00, 1 January 2000 on every boot, and nothing
reads them back. No alarm, wake-up timer or backup register is used.
The RTC only serves the dead bootloader-jump vector.
