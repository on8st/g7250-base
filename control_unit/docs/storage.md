# Control Unit — persistent storage

Status: VERIFIED unless marked. Evidence: `FUN_080367cc` (flash mount),
`FUN_080366b0` (SD state machine), `FUN_08037b9c` (INI reader),
`FUN_08048728` (config writer), descriptor table `0x080C33D0`.

## Volumes

| Drive | Medium | Mounted | No filesystem found |
| --- | --- | --- | --- |
| `1:` | SPI flash (SPI1) | at boot, by `FUN_080367cc` | formatted automatically (FAT) (VERIFIED) |
| `0:` | SD card (SDMMC1) | when a card is detected (superloop state machine, 5 s step timeout, 3 retries) | formatted automatically (FAT) |

The system page also offers *SD card format* (volume 0) and *Factory
reset*. Factory reset formats volume 1 and resets the MCU. The Main Unit
EEPROM is not affected.

## File catalogue

| File | Volume | Content | Read | Written |
| --- | --- | --- | --- | --- |
| `config.ini` | 1 | all front-panel settings | boot | orderly shutdown only |
| `bandstack_a.ini`, `bandstack_b.ini` | 1 | per-VFO band stack, 24 bands | boot (defaults if missing) | shutdown |
| `cwcall.ini` | 1 | 10 CW keyer messages | boot | shutdown |
| `memoChannelDire.ini` + `CHnnn.ini` | 1 | memory-channel directory and up to 100 channel files | boot | on save/erase |
| `voice.ini` + `VOICE-nn.wav` | 0 | 10 voice memories | when the SD card mounts | after each recording |

## SD bus timing (HIGH)

The card is identified at 400 kHz on one data line (kernel clock PLL1Q
96 MHz / 240). After init the HAL switches the card to 4-bit (ACMD6,
done twice: once by HAL_SD_Init and once by the card-detect code) and
writes ClockDiv 1 into CLKCR, so SDMMC_CK = 48 MHz. The HAL's 25 MHz
guard for default-speed cards does not trigger, because 96 MHz / 50 MHz
rounds down to 1. The firmware never sends the CMD6 high-speed switch or
the CMD11 1.8 V switch, so the card runs at 48 MHz in default-speed
mode at 3.3 V, above the 25 MHz the SD spec allows there. Most cards
tolerate this; it is a candidate cause if certain cards fail to mount
or write.

## config.ini

**Layout.** The file has six sections:
- `VFOA`, `VFOB`, `REALVFOA`, `REALVFOB`: one full VFO state each (name,
  frequency, mode, AGC/PRE/ATT, notch, NR, squelch levels and the per-mode
  FIL1/FIL2 edge tables);
- `WorkParam`: front-panel, CW, scope, audio, filter, EQ and UI state;
- `NetWork`: IPv4 settings, port, LAN mode and the wfview credentials.

**Format.**
- Each section is a `[section]` header followed by `key=value` lines.
- Integers are decimal, floats use the shortest form, booleans are written
  true/false and read as 1/T/Y in any case, strings are written raw.
- Comments start with `;` or `#`.

**Load and save.**
- A key unknown to the table is ignored; a missing key keeps its compiled
  default.
- The file is rewritten completely, and only when Main signals power-off
  (`0x03` flags bit 2). Settings changed since the last orderly shutdown are
  lost on a power cut.

**Descriptor table.** It has 507 entries of 20 bytes: section, key, offset
into the UI state, type, maximum string length. The type codes are 0–9 for
i8/u8/i16/u16/i32/u32/i64/u64/f32/f64, 10 for a string and 11 for a bool.

**Mapping to the wire.** The UI state is copied into the SLIP `0x01` image
by `slip01_build_settings` (`0x0803E76C`); see
`analysis/storage/ini_to_cmd01.csv`. Many keys are Control-only and never
reach the Main Unit, for example display, steps, scope colours and handle
keys (VERIFIED).

## Band stack files

There are 24 sections named after the band segments (mw, 160m, sw1, …,
6m, sw11, wfm), each with the keys `freq`, `mode`, `agc`, `pre`, `att`,
`ant` and `atu`. Compiled defaults (band edges, default frequency, mode
and a ham-band flag) live in flash at `0x080C2078`. Ham bands default to
LSB below 10 MHz and USB above; the broadcast segments default to AM and
78–108 MHz to WFM (VERIFIED).

**In RAM (VERIFIED).** The stack is two tables of 24 × 56-byte records
at `0x2400BA88` (VFO A, then VFO B), filled from the defaults at
`0x080C2078` by `FUN_0803ea94`; `FUN_0803ec18` resets one VFO table.
Record layout: band-name pointer (+0x00); lower and upper edge (+0x08,
+0x10) and default frequency (+0x18), all u64 Hz; default mode (+0x20),
antenna (+0x24) and ATU (+0x25 bit 0); then the live part: frequency
(+0x28), mode (+0x30), AGC, PRE, ATT, antenna and ATU (+0x31..+0x35) —
the seven keys of the file. Load `FUN_08048544`, save `FUN_08048520`.
The stack is persisted only; it is never sent over SLIP.

## CW messages, memory channels, voice

- **CW messages.** `[MSG-nn]` sections (01–10) with one quoted `msg` text of
  up to 59 characters.
- **Memory channels.**
  - The directory file has a `[home]` section with `homeCh` and one
    `[CHnnn]` section per used channel, holding its `filename`.
  - Each channel file has a `[wparam]` section (`chName`, `selectVfo`) and
    `[VFOA]`/`[VFOB]` sections with an 86-key subset of the VFO keys.
- **Voice memories.**
  - `voice.ini` has `[VOICE-nn]` sections with `filename` and `voiceSec`.
  - The recordings are 44-byte-header PCM WAV files: mono, 16-bit,
    48 kHz, at most 30 s each.

**Why this needed the firmware:** the file names, key table, save timing
and INI-to-wire mapping are not documented anywhere else.
