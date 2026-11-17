# Main Unit — DSP pipeline (PARTIAL)

Status: the structure, data paths, formats and scheduling are VERIFIED
unless noted. Several individual stages are only MEDIUM CONFIDENCE or
HYPOTHESIS. Function list: `analysis/dsp/dsp_functions.csv`.

## Data paths

- **RF I/Q from the FPGA.** The FPGA bit-banged bus on PE0–PE9 (see
  `peripherals.md`, FPGA bus) carries 24-bit signed I and Q.
  - The FPGA sample interrupt (PE10 / EXTI10, `FUN_08039078`) reads
    8, 4, 2 or 1 complex samples for span settings 0, 1, 2 and 3
    (settings `0x088`, or `0x08C` in NFM/WFM; `0x08C` is never updated
    from `0x01` and stays 0, so NFM/WFM always use index 0) (VERIFIED).
  - The samples go into a 192-sample ping-pong for the audio DSP and a
    256-sample ping-pong for the FFT/scope.
  - TX samples go back to the FPGA from a 2×192-sample ring
    (`0x2400E9F0` / `0x2400EFF0`) (VERIFIED).
  - The interrupt rate is not measured. 48 kHz, giving 384/192/96/48
    kS/s, is MEDIUM: it follows from the decimate-by-8/4/2/1 path
    `FUN_08033390`, the wfview span thresholds and the 122.88 MHz NCO
    clock (`peripherals.md`).
- **Audio through the AIC3204 on I²S3.** 48 kHz, 32-bit stereo, in
  4 ms DMA halves of 192 frames.
- **Block size and scheduling.** All baseband DSP runs on
  **192-sample 32-bit float blocks**. The TIM5 interrupt (`FUN_080239c8`)
  runs the TX routine `FUN_08034b88` while PTT, VOX or tune is active,
  and the RX routine `FUN_08033610` otherwise. The TIM6 interrupt
  (`FUN_080239e0`) runs housekeeping, including underrun checks and PTT
  handling. The RX routine is timed with the DWT cycle counter (VERIFIED).
- **Mode codes** (VFO record byte 0x18): 0 LSB, 1 L-D, 2 USB, 3 U-D,
  4 CW, 5 CWR, 6 AM, 7 SAM, 8 NFM, 9 WFM, 10 IQ, 11 LBK (loopback,
  skips the FPGA bus), 12 RTTY.

## RX chain (`FUN_08033610`)

1. **Input.** The FPGA I/Q block is copied into working buffers; the
   codec input is kept for a line-monitor mix.
2. **IQ phase-skew estimate (HIGH; not applied).** Every block the code
   averages Q·sign(I) and |Q| (one-pole, α = 0.003 per block, about
   1.3 s) and stores asin of their ratio, the I/Q phase skew in radians,
   at TRX+0x210 (`0x24013E1C`). Nothing reads it and no I/Q correction
   is applied; the stage is a dormant meter.
3. **LAN IQ tap.** The block is converted to int16 for the HDSDR
   service (`network.md`).
4. **IF gain and S-meter (MEDIUM).** IF gain with overload detection,
   then the S-meter.
5. **Demodulation per mode:**
   - **SSB/data/RTTY:** the phasing method, using a **201-tap Hilbert
     FIR** on one branch and a 201-tap matched delay FIR on the other,
     subtracted for LSB and added for USB (VERIFIED).
   - **CW (HIGH):** a complex low-pass on I/Q at half the CW bandwidth,
     then a mix by the CW pitch (settings `0x128`; NCO increment
     pitch/48 000 per sample, CMSIS sin/cos `FUN_08024e4c`/`FUN_08024dac`),
     keeping one real component, so a carrier at 0 Hz baseband is heard
     at the pitch. Then the optional CW peak filter (`0x136`), notch, NB,
     S-meter, squelch, a 768-sample tap for the Main-side CW decoder, NR
     and AGC.
   - **CWR, LBK (HIGH):** no RX case. They fall through to the S-meter
     only and output raw I/Q on L/R, like IQ mode.
   - **AM (HIGH):** complex low-pass on I/Q, envelope = ½·√(I²+Q²), NB,
     then a one-pole DC blocker (pole 0.999512, about 3.7 Hz) and an
     audio low-pass.
   - **SAM (HIGH):** a second-order PLL on the de-rotated signal (atan2
     phase detector, ωn = 300 rad/s, ζ = 0.70 at 48 kHz, pull-in
     ±300 Hz; the gains fit exactly only at 48 kHz, which independently
     supports the 48 kHz audio rate).
     - Four 7-section allpass chains (a polyphase IIR Hilbert pair) on
       cos·I, sin·I, cos·Q and sin·Q give both sidebands, each with a
       fade leveller (20 ms average removed, 1.4 s carrier average
       added).
     - After AGC the first sideband is copied to both channels, so SAM
       outputs a single sideband (MEDIUM; which one is a HYPOTHESIS for
       an on-air check).
     - A smoothed carrier offset in Hz is kept at `0x2400A2F0` (no
       reader found).
   - **FM (VERIFIED):** a conjugate-product discriminator (`FUN_080330c8`):
     atan2 of the current sample times the conjugate of the previous
     one (CMSIS atan2 `FUN_08024aa8`), scaled by 1/(1000π). Squelch uses
     the VFO +0x1D threshold with 20 ms hysteresis and a soft-mute ramp.
     In WFM, with settings +0x11D set, an optional stereo stage (pilot
     phase via atan2, doubled) splits L/R (MEDIUM). The stereo flag
     +0x11D is never written by code; it comes only from the stored
     settings image, so stereo cannot be switched in V1.0.1 (HIGH).
6. **Optional stages:**
   - **Noise blanker** (`FUN_0803b100`, HIGH): switch `0x11B`, level
     `0x0F7` (1–20 on Control), on demodulated audio. A sample is an
     impulse when it exceeds level × the running mean |x| (about 4 ms).
     Each impulse reloads a 119-sample (about 2.5 ms) hold, during which
     the last good sample is output with a gain falling ×0.9 per sample;
     afterwards the gain recovers ×1.1 per sample up to 1. The read
     pointer of its 240-sample ring stops during blanking, so the audio
     delay shifts after each event (MEDIUM). Not used in data modes or
     SAM.
   - **Auto-notch** (`FUN_08035c50`, HIGH): VFO +0x1F, unless the manual
     notch +0x1E is on. A normalised LMS (CMSIS shape, `FUN_08025e38`)
     with 32 taps and μ = 0.0005 on 64-sample sub-blocks; the reference
     is the input delayed by 64 samples (1.33 ms) and the error (the
     tone-free audio) is the output. If any coefficient exceeds 1.0, the
     filter is reset and bypassed for 500 sub-blocks (about 0.67 s).
   - squelch;
   - AGC (`FUN_080318d0`, VERIFIED; see below).
7. **Noise reduction (HIGH).** The SpeexDSP preprocessor (float build),
   frame 256 at 16 kHz, with denoise on and the Speex AGC and VAD off
   (init `FUN_0803b228`, run `FUN_08040238`, control `FUN_08040cd8`).
   - **Vendor controls (HIGH).** `0x30` sets the band count through an
     index table {3, 8, 16, 24, 32, 64, 128} (the G7250 uses index 3 =
     24 bands, the stock value). `0x32` sets the a-priori-SNR smoothing
     (stock 0.7; G7250 0.75). `0x34` sets the floor of the noise-estimate
     update rate (stock 0.03; G7250 0.024). `0x36` sets the minimum
     decision-directed weight (stock 0.1; unchanged). Odd codes read
     back.
   - The wrapper `FUN_0803b4c4` collects 768 samples at 48 kHz,
     decimates by 3 to 256, runs Speex, then interpolates back by 3
     (about 16 ms latency) (HIGH).
   - **NR selection (HIGH).** VFO +0x1C is only an on/off switch
     (Control toggles 0/1); any non-zero value runs Speex, except in WFM.
     The DNR1 level 1–100 (`0x0F2`) maps to −1…−25 dB. DNR2 (`0x0F3`) is
     copied by Main but never used: it has no effect in V1.0.1.
8. **RX equaliser.** Five-band biquad EQ, designed at runtime.
9. **Output.**
   - the wfview RX audio tap;
   - AF volume;
   - the line-monitor mix and beeps;
   - an MDMA copy into the I²S3 TX buffer.

**Dead flags (HIGH).** main() clears +0x118 (RX output scaled by 0, a
mute) and +0x11E (AGC bypass), and no code sets them.

### AGC (`FUN_080318d0`, VERIFIED)

A feed-forward peak AGC with look-ahead.
- **Detector.** Instant attack, 0.25 step toward lower values; level is
  20·log10(peak + 1e-6).
- **Gain update.** Error = target − gain − level. A negative error
  lowers the gain by error × attack coefficient and reloads the hang
  counter; a positive error raises it by error × decay coefficient once
  the hang counter has run out.
- **Parameters** (init `FUN_08031c60`, struct `0x24031DE0`): target
  −20 dB re float full scale, gain range −100…+30 dB, attack about
  0.36 ms, initial gain −10 dB, a 480-sample (10 ms) look-ahead delay on
  both channels.
- **AGC off** (VFO byte +0x23 = 0) gives a fixed +30 dB. The stage is
  skipped when settings +0x11E is set.
- **Outputs.** Gain at `0x24005E50` (the service-menu AGC value), hang
  count at `0x24005E54`.

Decay time constants by VFO byte +0x23 and mode:

| +0x23 | SSB (mode < 4) | CW (4–5) | AM/SAM etc. | FM (8–9) |
| --- | --- | --- | --- | --- |
| 1 (and other) | 0.5 s | 0.12 s | 1.2 s | 30 ms |
| 2 | 1.4 s | 0.28 s | 1.6 s | 30 ms |
| 3 | 80 ms + 40 ms hang | 30 ms | 0.7 s | 30 ms |

**Labels (VERIFIED).** Control shows +0x23 as 0 OFF, 1 AUTO, 2 SLOW,
3 FAST, which matches the decay table. The wfview CI-V `1A 04` handler
(`FUN_0804fdd2`) remaps the wire codes: 1 FAST → 3, 3 SLOW → 2, 2 MID
→ 1, and sets +0x20 for any non-zero code. So "AUTO" on the panel is
the medium setting, and no label is wrong. Correction: an earlier
version compared CI-V wire codes directly with VFO codes.

## TX chain (`FUN_08034b88`)

Status: audio/baseband stages VERIFIED unless noted. The power/drive
stages after modulation (per-band scaling, soft-start ramp, TX-start
mute, ALC) are out of scope here.

**Block and rate (HIGH).** One run per new I²S half (half index
`0x24000775` vs last processed `0x240094EC`), skipped when the codec TX
DMA is too far into the half. 192 float samples per run, so the TX DSP
runs at the codec rate of 48 kHz in 4 ms blocks. Two branches:
I `0x2400A2FC` and Q `0x2400A5FC`.

Stage order:
1. **Snapshot.** `FUN_0803a5ac` copies the newest 192 stereo frames from
   the circular I²S RX buffer (`0x30003760`) at the RX DMA position.
2. **Source.**
   - Tune carrier (TRX+0x22E): I = 1, Q = 0, all audio stages skipped.
   - TX-audio mute flag `0x24013E4A` (T/R sequencing): both branches 0.
   - LAN TX audio: hook `0x240096DC` (registered by the WFV-G init) →
     `FUN_0804ec5c` pulls int16 samples (×2⁻¹⁵) from the wfview ring
     while a session is streaming; zeros on underrun. The mic is then
     ignored (no mixing).
   - Otherwise the codec left ADC (the MICPGA mic input) ×2⁻³¹.
   - The sample goes to both I and Q. Mic gain is analogue only
     (codec MICPGA, `peripherals.md`); there is no USB or line source.
3. **Voice gate.** `FUN_0803ab00` returns settings `0x094` for data
   modes (1, 3, 10, 12) and `0x090` otherwise (Main-only; init 1 and 2).
   EQ and comb run only for code 0 or 1, i.e. voice modes.
4. **TX EQ (I only).** Five peaking biquads, skipped per band when the
   gain byte is 0. Bands 300/700/1200/1800/2300 Hz with Q
   0.5/0.7/1.0/1.5/2.0, gain = signed byte in dB (`FUN_08031cdc`).
   Settings `0x10C`–`0x110` for SSB/CW/LBK and `0x111`–`0x115` for
   AM/SAM/NFM/WFM. Control V1.0.1 never writes them, so the EQ is flat.
5. **Comb (dormant).** Settings `0x0F1` = N gives ½·(x + x delayed by
   N+1 blocks) from a 10-block ring (`0x24031E20`); nothing sets `0x0F1`.
6. **DC blockers** on I and Q (`FUN_08031f7c`, pole 0.99951, ≈3.7 Hz).
7. **TX HPF/LPF (I only).** Butterworth biquad cascades (`FUN_080317b8`
   / `FUN_080316b0`) from VFO +0x12 / +0x16, each skipped when 0. The
   order byte (`0x0EC`/`0x0ED`/`0x0EE`) is the section count (HPF capped
   at 5), so the default SSB LPF is 30th order.
8. **Compressor (I only).** All modes except AM/SAM and data. Peak
   detector, instant attack, 20 ms release; gain law
   ln(1 + μe) / (ln(1 + μ)·e) with μ = 2^(ratio + 1) (settings `0x171`;
   ratio ≥ 4 turns it off); held below −60 dBFS; smoothed per block
   (≈100 ms) and applied once per block. Switch `0x170`.
9. **Modulation** (mode switch):
   - **LSB, USB, U-D, RTTY:** soft clipper `FUN_08032cc4` first (pre-gain
     10^(v/10), v = settings `0x0C0`, default 1.0; samples above −3 dBFS
     divided by 2√2|s| − 1; TX LPF-shaped post-filter; then I is copied
     to Q). Switch `0x120` defaults to 1 and Main never takes it from
     Control, so the clipper is always on.
   - **SSB phasing:** 201-tap FIRs (copies of the RX Hilbert/delay pair
     at `0x08079998` / `0x08079674`). LSB/L-D: I = Hilbert, Q = delay;
     USB/U-D/RTTY: swapped. Computed opposite-sideband rejection at
     48 kHz: −46 dBc at 300 Hz, −53…−61 dBc from 1 to 20 kHz, −21 dBc
     at 200 Hz.
   - **L-D (MEDIUM, possible defect):** no clipper, so Q never receives
     the DIGI LPF output; out-of-band audio is only about 6 dB down.
   - **AM/SAM:** I = 0.5·(1 + m·x), Q = 0, m = calibration `0x06A`/100
     (default 25 %).
   - **NFM/WFM:** optional CTCSS 0.25·sin at settings `0x0BC` (dormant),
     one-pole high-pass (0.95 pole, ≈390 Hz), phase step x·k with
     k = calibration `0x068` × TX width / 48000 (default 4), then
     I = sin, Q = cos. Peak deviation ≈ peak × `0x068` × width / 2π.
   - **CW:** keyer envelope (see the CW docs). **LBK:** processed audio
     goes to the monitor only. **IQ:** no modulation stage (MEDIUM).
10. **TX scope (MEDIUM).** While transmitting and settings `0x0B7` = 0,
    the modulated I/Q feed the 256-sample FFT ping-pong.
11. **Monitor.** (I + Q) × taper(`0x0FF`/100) (0 to −50 dB, mute below
    1 %) to both codec channels; the CW sidetone replaces it in CW.
    The tap is after modulation, so in FM it is not plain audio.
12. Hand-off to the drive/output path and the MDMA copy into the FPGA
    TX ring.

**VOX (`FUN_08035b50`, every 10 ms from the TIM6 arbiter) (HIGH).**
RMS of the first I²S RX half (both channels, q31, ×2⁻³²) in dB against
the signed threshold `0x117`. Immediate attack; hang = `0x0EA` ms, which
also blocks VOX for that long after boot. Gated by `0x11F` and the
radio-ready flag. No anti-VOX: no RX or speaker reference is read.

**Why this needed the firmware:** the stage order, the I-only filtering,
the compressor law and time constants, the clipper curve, the sideband
I/Q mapping and which settings are live or dormant exist only in the
code.

## Filter implementation

- **Biquads.** Cascades in transposed direct form II (5 coefficients
  and 2 state values per stage). They follow CMSIS-DSP-shaped
  primitives; see the CSV for FIR, FIR decimator/interpolator and
  vector helpers.
- **Runtime design (HIGH CONFIDENCE).** The coefficients are designed at
  runtime in `FUN_08032020` from the filter-edge settings, with the EQ
  peaking design in `FUN_08031cdc`, and are recomputed when the mode or
  filter changes.
- **Fixed tables.** Only the 201-tap SSB Hilbert and delay tables are
  fixed in flash, at `0x08079FE0` and `0x08079CBC` (VERIFIED). The TX
  path uses byte-identical copies at `0x08079998` and `0x08079674`.

**Fixed SSB FIR pair (VERIFIED numerically).**
- `0x08079FE0` holds a 201-tap antisymmetric FIR with every other tap
  zero: a Hilbert transformer (VERIFIED).
- `0x08079CBC` holds a 201-tap symmetric FIR whose centre tap is close
  to 1: a delay FIR (VERIFIED).
- Both have a linear-phase delay of 100 samples, and their phase
  difference is +90° at all frequencies.
- Assuming 48 kHz (HYPOTHESIS), the Hilbert magnitude is about −0.45 dB
  at 300 Hz and flat within 0.04 dB from about 1 kHz to 20 kHz.

Reproduce with `tools/filter_response.py` (FIR mode on the image,
address, 201 taps). The X6100 biquad checker was removed, because the
G7250 biquads are designed at runtime.

**Why this needed the firmware:** no public document describes the
I/Q-vs-audio split, the sample formats, the block scheduling, the LAN
taps or the filter structures. They exist only in the code.
