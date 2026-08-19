<!---

This file is used to generate your project datasheet. Please fill in the information below and delete any unused
sections.

You can also include images in this folder and reference them in the markdown. Each image must be less than
512 kb in size, and the combined size of all images must be less than 1 MB.
-->

## How it works

AgilA8 is a compact 8-bit microcontroller built around A8, a custom
16-instruction CPU (see `docs/ISA.md`), with memory-mapped GPIO, a 16-bit
timer, and PWM generation.

Both instruction and data memory live off-chip on the Tiny Tapeout QSPI
Pmod. Program code is fetched from external SPI flash (CS0) using a
standard `03h` Read Data command; data memory lives on one of the
Pmod's two PSRAM chips (RAM A / CS1) using standard `02h`/`03h`
Write/Read commands. A third front-end, a general-purpose SPI master
for driving an external device (an LCD, an ADC, another MCU - anything
that isn't the flash/PSRAM already covered), shares the same physical
lines using the Pmod's previously-unused CS2 ("RAM B", never actually
populated on the real Pmod). This keeps the on-chip design small enough
to fit a 1x2 tile budget - Tiny Tapeout's own RAM32 macro is *half* the
size of this design's DMEM and needs 3x2 tiles on its own, so a plain
on-chip flip-flop array was never going to fit. Only plain, single-line
SPI commands are used for flash/PSRAM - deliberately not flash's
continuous-read mode or PSRAM's QPI mode, both of which need a
mode-byte/setup sequence that's easy to get subtly wrong without
hardware to verify against.

All three front-ends (flash, PSRAM, and the general-purpose SPI
controller) are driven by one shared SPI shift engine rather than three
separate FSMs, since they're never active at the same time (see below)
and consolidating saves real area - roughly 85 flip-flops for the
shared engine plus three thin front-ends, versus about 196 flip-flops
for three independent controllers.

Because `imem_valid` and `dmem_valid` are never asserted in the same
cycle (fetch and memory-access are separate, sequential states in the
core's FSM), and the DMEM-side peripherals are mutually exclusive by
address decode, the shared engine can grant flash/PSRAM/SPI with a
simple fixed-priority mux rather than needing real bus arbitration -
by construction, at most one of the three is ever requesting at once.

### Address map

| Address range | Device                                  |
| -------------- | --------------------------------------- |
| 0x00 - 0xEF, 0xF5 - 0xF7 | RAM (external PSRAM, RAM A)   |
| 0xF0 - 0xF2    | GPIO                                    |
| 0xF3 - 0xF4    | SPI (general-purpose)                   |
| 0xF8 - 0xFB    | Timer                                   |
| 0xFC - 0xFD    | PWM                                     |

> **Note:** `0xF3`/`0xF4` used to be plain RAM in earlier revisions of
> this design. They now belong to the general-purpose SPI controller
> (see below) - any program that stored ordinary data at those two
> addresses will now silently hit the SPI controller instead of RAM.

Instructions are fetched separately, as two consecutive bytes from
external flash (big-endian: high byte at PC, low byte at PC+1) - flash
isn't part of the 8-bit DMEM address space above.

### IO

| # | Input       | Output       | Bidirectional                    |
| - | ----------- | ------------ | --------------------------------- |
| 0 | GPIO in 0   | GPIO out 0   | Flash CS (CS0)                    |
| 1 | GPIO in 1   | GPIO out 1   | SD0 - MOSI (shared flash/PSRAM)   |
| 2 | GPIO in 2   | GPIO out 2   | SD1 - MISO (shared flash/PSRAM)   |
| 3 | GPIO in 3   | GPIO out 3   | SCK (shared flash/PSRAM)          |
| 4 | GPIO in 4   | GPIO out 4   | SD2 (held high, unused)           |
| 5 | GPIO in 5   | GPIO out 5   | SD3 (held high, unused)           |
| 6 | GPIO in 6   | GPIO out 6   | RAM A CS (CS1)                    |
| 7 | GPIO in 7   | PWM output   | SPI CS (CS2, general-purpose SPI)  |


#### GPIO

| Register | Address     | Description                                                      |
| -------- | ----------- | ------------------------------------------------------------------ |
| GPIO_OUT | 0xF0 (R/W)  | Write sets `uo_out[6:0]`; read returns the last value written    |
| GPIO_IN  | 0xF1 (R)    | Reads the current state of `ui_in[7:0]`                          |
| GPIO_DIR | 0xF2 (R/W)  | Read/write register; not wired to anything (`ui_in`/`uo_out` are fixed-direction TT pins, so there's no direction to control) |

`uo_out[7]` is dedicated to the PWM output, not GPIO - a write of
`0xAA` to GPIO_OUT reads back as `0xAA` internally, but only
`uo_out[6:0]` (`0x2A` in that example) reaches a physical pin.

#### SPI (general-purpose)

A separate SPI master for driving an external device that isn't the
flash/PSRAM already covered above (an LCD, an ADC, another MCU, etc.),
using CS2 - the Pmod's second PSRAM chip-select, never actually
populated on the real board.

| Register | Address     | Description                                                        |
| -------- | ----------- | -------------------------------------------------------------------- |
| SPI_DATA | 0xF3 (R/W)  | Write: shifts the byte out (CS auto-asserted for the transfer, **blocking** until the 8-bit transfer physically completes). Read: returns the byte simultaneously shifted in from MISO during the most recent transfer, without starting a new one - to read a byte from a slave, write a dummy `0x00` and then read DATA back (standard full-duplex SPI) |
| SPI_CTRL | 0xF4 (R/W)  | Bits[1:0] = SCK clock divider: `00` = fastest (~sys_clk/2, matches flash/PSRAM speed), `01` = ~sys_clk/8, `10` = ~sys_clk/32, `11` = ~sys_clk/128 (**reset default** - start slow, let software speed up once the attached device's timing is known to tolerate it) |

Each `SPI_DATA` write is deliberately blocking rather than
fire-and-forget: the core has no instruction cache, so the very next
instruction fetch also needs this same shared bus. Blocking keeps this
peripheral's transfers inside the same single-active-transaction
invariant the shared engine already depends on for flash/PSRAM, with no
separate arbitration hardware needed. CS is likewise auto-pulsed per
byte (asserted only during the active transfer) rather than held low
across a logical multi-byte burst - genuinely continuous bursts aren't
possible on this hardware anyway, since unrelated flash-fetch traffic
would otherwise appear on the shared lines mid-burst; auto-pulsing at
least keeps CS deasserted while that happens, so the attached device
correctly ignores it.

#### Timer

| Register    | Address     | Description                                                    |
| ----------- | ----------- | ---------------------------------------------------------------- |
| TIMER_LO    | 0xF8 (R)    | Bits 7:0 of the free-running 16-bit counter                    |
| TIMER_HI    | 0xF9 (R)    | Bits 15:8 of the counter                                       |
| TIMER_CTRL  | 0xFA (R/W)  | Bit 0 = enable (counts up once per clock while set). Writing bit 1 = 1 resets the counter to 0 |
| TIMER_FLAG  | 0xFB (R/W)  | Bit 0 = overflow (set when the counter wraps past 0xFFFF); any write clears it |

#### PWM

| Register  | Address     | Description                                                        |
| --------- | ----------- | ---------------------------------------------------------------------- |
| PWM_DUTY  | 0xFC (R/W)  | 8-bit duty cycle out of a free-running 256-cycle period. `0xFF` is a special-cased always-on |
| PWM_CTRL  | 0xFD (R/W)  | Bit 0 = enable. Output is forced low whenever disabled, regardless of PWM_DUTY |

## How to test

1. Program the test image onto the Pmod's flash chip (`test/imem.hex`,
   built by `test/build_prog.py` - exercises every opcode plus the GPIO,
   timer, and PWM registers) and leave the PSRAM chip's contents as-is;
   the program initializes any RAM it depends on.
2. Reset the design (`rst_n` low then high).
3. Run the clock. The CPU fetches from flash and reads/writes RAM over
   the shared SPI bus automatically - no host intervention needed once
   running.
4. Check final state against the golden reference model
   (`test/golden.py`) - `test/TESTING.md` and `test/TESTING_round2.md`
   document the expected register values, and the pipeline this was
   last verified against.

Before committing to a tapeout, the QSPI Pmod flash-read timing margin
(`read_delay_cfg`, now handled centrally in `qspi_shared_engine.v`) is
worth validating on real hardware first, since interconnect delay isn't
visible in behavioral simulation - see the FPGA bring-up guide for the
Tiny Tapeout FPGA Development Kit + QSPI Pmod path used for that.

## External hardware

- [Tiny Tapeout QSPI Pmod](https://store.tinytapeout.com/products/QSPI-Pmod-p716541602),
  plugged into the demoboard's bidirectional Pmod header. One flash chip
  (program memory) and one of its two PSRAM chips (data memory) are
  used; the second PSRAM chip's chip-select (CS2) is repurposed for the
  general-purpose SPI controller instead - that chip itself is still
  unpopulated on the real Pmod, but its CS line now carries traffic for
  whatever external SPI device gets connected there.
- Tiny Tapeout demoboard, or the
  [FPGA Development Kit](https://store.tinytapeout.com/products/FPGA-Development-Kit-p813805747)
  for pre-tapeout bring-up on real silicon-adjacent hardware.
