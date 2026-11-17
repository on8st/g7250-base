# Control Unit — Startup

Image: `G7250_ControlUnit_Fw_V1.0.1.bin` · load base `0x08020000` (VERIFIED)
Evidence: `control_unit/analysis/startup/`

## Vector table (core entries)

| Entry | Value | Meaning | Confidence |
| --- | --- | --- | --- |
| `vector[0]` | `0x24080000` | Initial MSP (top of 512 KiB AXI SRAM) | VERIFIED |
| `vector[1]` | `0x080235AD` | Reset_Handler (Thumb → `0x080235AC`) | VERIFIED |

`0x24080000` is exactly the top of the H743/H750/H753-class 512 KiB AXI
SRAM (VERIFIED) — the strongest single pointer to the STM32H7 part class.

## Reset handler — `FUN_080235ac`

VERIFIED from disassembly (`reset_disassembly.txt`):

```
ldr sp, =0x24080000      ; initial stack pointer (top of AXI SRAM)  VERIFIED
bl  0x08022a84           ; PWR supply config + wait VOSRDY           VERIFIED
bl  0x0802298c           ; SystemInit                                VERIFIED
                          ; copy .data  0x24000000..0x24001698 <- 0x080DD568
                          ; clear .bss  0x240016A0..0x240614A4
bl  0x080874ac           ; __libc_init_array                         VERIFIED
bl  0x08021e74           ; main (candidate)                          HIGH CONFIDENCE
bx  lr
```

Chain:

    Reset_Handler
        -> PWR config            FUN_08022a84   VERIFIED
        -> SystemInit            FUN_0802298c   VERIFIED
        -> copy .data / clear .bss              VERIFIED (see memory_map.md)
        -> __libc_init_array     FUN_080874ac   VERIFIED
        -> main                  FUN_08021e74   HIGH CONFIDENCE (candidate)

The Control Unit makes **two** calls before `.data` init where the Main
Unit makes one: an explicit PWR/voltage-scaling step precedes SystemInit.

## PWR config — `FUN_08022a84` (VERIFIED)

Base literal `0x08022A98` = `0x58024800` (STM32H7 **PWR**). Sets
`PWR->CR3` (@+0x0C) bit 1, then spins until `PWR->CSR1` (@+0x04) bit 13
(`ACTVOSRDY`, voltage-scaling ready) is set. Standard H7 power bring-up
before raising clocks.

## SystemInit — `FUN_0802298c` (VERIFIED STM32H7 CMSIS)

Same stock STM32H7 `SystemInit()` as the Main Unit (FPU enable, FLASH
latency 7, RCC/PLL reset, `SCB->VTOR = 0x08020000`), plus two H7-specific
extras. Resolved literals (`0x08022A54`+), all VERIFIED:

| Literal | Value | Register / meaning |
| --- | --- | --- |
| `0x08022A54` | `0xE000ED00` | SCB |
| `0x08022A58` | `0x52002000` | FLASH ACR |
| `0x08022A5C` | `0x58024400` | RCC |
| `0x08022A70` | `0x5C001000` | DBGMCU (IDCODE) |
| `0x08022A74` | `0xFFFF0000` | revision mask |
| `0x08022A78` | `0x51008000` | AXI interconnect (GPV) |
| `0x08022A80` | `0x08020000` | value written to `SCB->VTOR` |

Extras vs. Main:
- **Rev-Y AXI SRAM errata workaround:** `if ((DBGMCU->IDCODE & 0xFFFF0000)
  < 0x20000000) *(uint32_t*)0x51008108 = 1;` — sets the AXI switch-matrix
  read-issuing capability to 1 for the AXI SRAM target. This is an
  STM32H7-specific CMSIS workaround and is strong independent
  confirmation of the H7 part.
- A conditional RCC register tweak (offset `0xD4`, value `0x30D2`).

`__libc_init_array` (`FUN_080874ac`) VERIFIED (same pattern as Main;
`_init` = `FUN_0808a0fc`).

## Static initialisers (`.init_array`) — VERIFIED

These were located from the literal pool of `__libc_init_array` by the
corrected `ExportStartupEvidence.java`. `.preinit_array` is empty.
`.init_array` = `0x080DD54C`–`0x080DD55C` and has 4 entries:
frame_dummy (`0x080202c4`) and static initialisers at `0x0804aa50`,
`0x08071948` and `0x080799d4`. The first fills UI/value tables (MEDIUM
CONFIDENCE). The other two touch no hardware (HIGH: no peripheral
address and no HAL, emWin or FatFS call within four call levels).
`0x08071948` constructs about 70 C++ GUI objects of the application's
own widget and page classes in AXI SRAM; each gets a vtable, a short
class tag, colours and zero geometry. Before that it fills
label-descriptor tables (position as a fraction, text, colour, font).
`0x080799d4` only fills more label tables. The page objects are rebuilt
with real geometry at runtime by `FUN_0806d3c8`.

## Unknowns

- (Resolved) The running clock config is `FUN_08021d98`; see `flow.md`.
- The conditional RCC `0xD4`/`0x30D2` tweak's exact purpose.
- Whether an RTOS is started inside `main`.

## Exception handlers — VERIFIED

**Handlers:**
- The default handler (`0x08023600`), NMI, MemManage, BusFault and
  UsageFault are bare infinite loops.
- SVC, DebugMon and PendSV return at once.
- HardFault logs a one-line error through the console logger
  (`FUN_08021840`) and halts, without a register dump.

**Watchdog in SysTick.** The SysTick handler advances the HAL tick and
also writes the IWDG refresh key every 256 ticks. The watchdog is
therefore fed from interrupt context, so a stuck superloop does not by
itself cause a watchdog reset.
