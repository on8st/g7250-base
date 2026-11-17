# CW subsystem (both units)

Status: structure HIGH CONFIDENCE unless noted. Settings offsets refer
to the SLIP `0x01` block (`docs/protocol.md`).

## Key sources

- **Paddle / straight key → Main PH3/PH4** (VERIFIED). EXTI on both
  edges; low = closed; callback `FUN_08023ab4`. Settings `0x139` swaps
  the contacts.
- **KEY jack (HIGH CONFIDENCE).** Control firmware does not read the
  Controller KEY jack. It is presumably wired through the CTRL cable to
  PH3/PH4 (HYPOTHESIS for the wiring).
- **CW memories** are keyed by the Control Unit, which pulls PE3 and PE4
  low together (VERIFIED; `control_unit/docs/voice_cw_memories.md`).
- **Serial key** (`0x15E` bit 1). Main honours it as a straight key,
  but Control V1.0.1 never sets it.
- **Outside CW/CWR** the key contacts act like PTT (MEDIUM).

## Settings (`0x01`)

| Offset | Meaning | Units |
| ---: | --- | --- |
| 0x124 | DI/DA ratio (dash = dot × ratio) | f32 |
| 0x128 | CW tone (sidetone pitch) | Hz |
| 0x12A | QSK / break-in delay | 0.1 dot on the wire; Main stores ms (× 120 / WPM) |
| 0x12E | speed | WPM |
| 0x134 | keyer on (paddles) / off (straight) | bool |
| 0x137 | straight-key flag (Main ignores it) | bool |
| 0x138 | iambic type: 0 = A, 1 = B | u8 |
| 0x139 | paddle reverse | bool |
| 0x140 / 0x141 | decoder threshold / enable; Main also reads them for a Main-side decoder (`FUN_08037728` feed, `FUN_08037054`; HIGH, not analysed) | u8 / bool |
| 0x159 | CW trainer | bool |
| 0x15A | sidetone level | 0–63 scale |

There is no separate weight parameter. Control maps its key type
(straight / paddle / paddle reversed) onto 0x134, 0x137 and 0x139.

## Keyer (Main, `FUN_080362c0`, called from the TX DSP)

**Timing.** Dot = 1200 / WPM ms, dash = dot × ratio, element space =
1 dot. Element timing uses the 1 ms tick inside 4 ms DSP blocks
(effective resolution probably 4 ms, HYPOTHESIS). The keyer outputs
zero until a TX-start delay from calibration field `0x021` has elapsed
(0 in CW modes).

**Iambic modes (mapping VERIFIED, behaviour MEDIUM).** In mode A the
keyer stops after the current element unless both paddles are still
held. Mode B uses a press-memory latch, which Main fixes on at boot.

**Envelope.** A linear ramp of about 5 ms (MEDIUM).

## Break-in

**Hang counter.** Each key-down sets a hang counter to the QSK delay
in ms + 20. The 10 ms housekeeping tick (`FUN_08037c10`) counts it
down, and the radio returns to RX at 0.

**PTT sources.** In CW/CWR the built-in, CI-V and one other PTT source
are ignored.

## CW trainer

With `0x159` set, entering TX in CW skips the PA bias and T/R
switching, while the sidetone still plays (VERIFIED).

## Sidetone (Main, `FUN_08032b44`)

A sine at the CW tone (48 kHz synthesis), shaped by the keyer envelope.
It is written to both I²S3 TX channels, so it comes out of the Main
codec DAC. Level = (fixed 85/100) × (sidetone level / 63).

## Decoder (Control, `FUN_08051bc8`; init `FUN_08051b08`)

**Signal path.**
1. Left-channel codec audio is copied from the TIM5 audio service into
   a 1536-sample ring.
2. It is decimated by 16.
3. A 192-sample window (4-term Blackman-Harris) advances 3 samples per
   call.
4. A 256-point FFT is taken, and peak and mean are computed over the
   lowest 63 bins.

**Tone detection.** A tone is present when the peak is at least 0.6
and exceeds (threshold + 4) × mean. Edges are debounced at 3 ms.

**Timing and output.**
- The dot length is tracked adaptively (EMA 0.3); dash = 3 dots, word
  gap = 7 dots.
- At most 9 symbols per character.
- The output is a 63-character scrolling text line, with a WPM
  estimate capped at 36.
- The bin range assumes about 3 kHz after decimation (MEDIUM).

## Open

- The CI-V/wfview CW level handlers (pitch, WPM, break-in) sit in code
  near `0x08052400` that Ghidra has not disassembled.
- The exact keyer call rate and the true rise time.
- The meaning of TRX+0x130, which gates CW TX.

**Why this needed the firmware:** key routing, field units, A/B
behaviour, sidetone generation and the decoder exist only in the code.
