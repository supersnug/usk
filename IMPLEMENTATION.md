# USK 2.80 implementation

This document describes the MCU-side source in this repository and its interface
to the pinned **busk** bootloader. It describes the existing implementation, not
a redesigned boot chain or hardware installation procedure. The console-side
loader's internals are deliberately excluded; see [SDLOADER.md](SDLOADER.md).
Build revisions and binary verification are in [SNAPSHOT.md](SNAPSHOT.md).

## Components and execution boundaries

| Source | Responsibility |
| --- | --- |
| [main.c](main.c) | Startup, repair decisions, saved-offset search, randomized search, success handling |
| [board_detect.c](board_detect.c), [pins.h](pins.h) | Board selection and GPIO mapping |
| [boot_detect.c](boot_detect.c) | Initial boot-traffic observation and Erista/Mariko selection |
| [emmc.pio](emmc.pio), [pio_upload.c](pio_upload.c), [glitch.c](glitch.c) | PIO programs, traffic capture, pulse generation, and attempt classification |
| [config.c](config.c), [config.h](config.h) | eMMC identity, firmware/loader identity, and successful-offset records |
| [payload.c](payload.c) | MCU-driven eMMC access, BCT construction/copying, blob installation, update requests, descriptor |
| [fuses.c](fuses.c), [fuses.h](fuses.h) | Flash-backed slot-selection counter and busk handoff state |
| [misc.c](misc.c), [misc.h](misc.h), [ws2812.pio](ws2812.pio) | Reset, LEDs, GPIO cleanup, and low-power halt |
| [CMakeLists.txt](CMakeLists.txt), [prepare.py](prepare.py) | RAM executable and combined release images |

`payload.h` supplies an opaque console executable and Erista BCT byte arrays;
`mariko_bct.h` supplies Mariko BCT byte arrays. `payload.c` is RP2040 code that
installs these bytes, not the source of the console executable.

## 1. RP2040 bootloader and flash layout

The separate [busk source at the pinned revision](https://github.com/supersnug/busk/blob/3f715c6585140ec3ab9128a20b719f5035191374/main.c)
starts before USK. Its `startup_from_slot()` checks a firmware header's size
(nonzero and at most `0x38000`), copies the body to SRAM at `0x20000000`, verifies
CRC-32, and calls Thumb address `0x20000001`. A 100-ms watchdog covers that load
and handoff. USK is built with `pico_set_binary_type(usk no_flash)`; it is not an
ordinary execute-in-place application at its storage address.

Offsets below are relative to RP2040 external flash, whose mapped base is
`XIP_BASE` (`0x10000000`), not console eMMC addresses:

| Offset | Contents/use |
| --- | --- |
| `0x00000` | busk image |
| `0x08000` | USK configuration header and timing-record pages |
| `0x0f000` | Erase sector holding the 256-byte slot-selection counter |
| `0x10000` | Firmware slot 0: 32-bit size, 32-bit CRC-32, then executable bytes |
| `0x48000` | Firmware slot 1, same header format |

Busk normally selects `count_fuses() & 1`. On watchdog retries it uses
`(preferred_slot ^ previous_boot_try ^ 1) & 1`, increments the attempt counter,
and halts when the previous attempt counter is 3. This permits retrying the
preferred slot, trying the alternate slot, then returning to the preferred one;
it is not an unconditional guarantee of recovery from every interrupted write.
It passes the attempt counter in watchdog scratch register 0 and the selected
slot in scratch register 1; USK's `init_fuses()` reads both.

The functions named `count_fuses()` and `burn_fuse()` operate on **RP2040 flash
bits**, not Nintendo Switch hardware fuses. Each increment clears the next set
bit in a 256-byte area. At exhaustion, USK erases its 4-KiB sector and clears one
bit, preserving the required change of parity. After a successful glitch, USK
increments this counter if its parity differs from the running slot, committing
that slot as the normal selection for subsequent boots.

## 2. Startup and board selection

`main()` performs these steps before searching for a glitch:

1. Disable the inherited watchdog, detect the board, set the RP2040 regulator
   to 1.30 V, and run its system clock at 200 MHz.
2. Read busk's scratch-register handoff and upload the PIO programs.
3. Apply the tiny-board LED initialization. A watchdog-caused reboot with
   `boot_try == 0` takes the one-pulse completion/halt path.
4. Sample the force-reset connection with `detect_by_pull(1, 0, 1)` and show blue.
5. Run the ADC self-test, then `wait_for_boot(2500)`.
6. Run `fast_check()` against eMMC BOOT0, compare the configuration with the
   eMMC CID, and rewrite the installed boot data if required.
7. For Mariko, replace two trigger-program instructions with NOPs before the
   glitch search.

`detect_board()` tests WS, XIAO, ItsyBitsy, Pico, SQC, then WT layouts, in that
order, and defaults to WS. Its tests drive/sample selected GPIOs using pulls;
this is not a USB board identifier. `led_pin()`, `gli_pin()`, `sda_pin()`, and
`scl_pin()` select mappings from `pins.h`. All layouts share CLK 27, CMD 28,
DAT0 29, and reset 26. Detection of the RP2040 board is separate from detection
of the console's Erista/Mariko boot pattern.

`self_test()` waits up to 2.5 seconds for reset, CMD, and DAT0 each to sample
within 1.8 V ± 0.2 V. Although it declares `clk_ok`, it does not test CLK.
`reset_cpu()` enables the reset pad, waits 1 ms, drives reset low for 2 ms,
then releases it and disables the pad's input/output paths.

`wait_for_boot()` resets the console while observing command traffic. CMD1,
block-0 reads, and block-1 reads adjust its deadlines; a block-1 read finishes
the detection. It starts with `mariko = true` and switches to Erista after the
specific read-status sequence following block 0. Missing traffic or exhausted
deadlines enter coded halt paths rather than starting an unbounded search.

## 3. PIO hardware roles

`upload_pio()` loads programs once and preloads the command and data counters.
The C routines subsequently configure state machines for each operation.

| PIO / state machine | Role |
| --- | --- |
| PIO0 / 0 | Generate the MCU-driven eMMC clock and reset control |
| PIO0 / 1 | Receive command responses |
| PIO0 / 2 | Receive DAT0 data |
| PIO0 / 3 | Transmit commands/data; also reused separately for WS2812 LED output |
| PIO1 / 0 | Sniff console eMMC command traffic |
| PIO1 / 1 | Generate the glitch output pulse |
| PIO1 / 2 | Wait for the relevant DAT0 transfer |

During a glitch attempt the sniffer captures 48-bit frames into FIFO words.
Its preloaded comparison is `0x1351`, the tail of the selected read command.
IRQ handoffs coordinate the command match, DAT0 completion, and subsequent
command events. The trigger program consumes delay and width values, counts
down the delay with its output low, then counts down the width with it high.
Erista waits for the two additional command events; Mariko's NOP patch removes
those two waits. These parameters are **PIO loop counts**, not nanoseconds.

## 4. Attempt classification and timing search

`do_glitch()` arms the PIO machines with delay and width. It initially allows
20 ms to detect traffic, can extend to the caller's full deadline, and uses a
shorter post-trigger deadline. The search calls it with 300 ms and 6 ms:

| Observed words/state | Result |
| --- | --- |
| `0x51000000`, then `0x1351` | Mark the selected read/trigger as seen |
| `0x51000000`, then `0x142f` | `GLITCH_RESULT_FAILURE` |
| `0x40aa5458`, then `0xba3b` | `GLITCH_RESULT_SUCCESS` |
| Deadline without trigger | `GLITCH_RESULT_MISSING` |
| Deadline after trigger, without decisive result | `GLITCH_RESULT_TIMEOUT` |

Success is inferred from the installed console code's recognizable command,
not by inspecting the CPU's instruction stream. The firmware does not wait to
verify that the final SD payload or operating system has booted.

`glitch_try_offset()` resets the console for each attempt, adds a three-bit
ROSC-derived timing variation to the offset, and adjusts a shared width:

- The initial width in `main()` is 150. The step starts at 32 for the first
  offset search, otherwise 8.
- A failure increases width; a timeout decreases it, clamped to at least 1.
- Transitions between those results halve the step until it is 1, then count
  edge crossings. Exceeding the caller's edge limit abandons that offset.
- Five consecutive missing-trigger results, width 1, or width at least 1000
  take distinct halt/error paths. Success returns immediately.

For each of up to two full rounds, `main()` makes two passes through saved
offsets, highest positive record weight first, with edge limit 3. If none
succeeds, it makes up to two shuffled candidate passes with edge limit 4.
After the first failed full round it rewrites the boot data; after the second
it signals the attempt-limit error. Width is carried across offset searches.
On success it commits slot parity if necessary, records the offset, and halts.

## 5. Configuration and training records

`usk_cfg` stores the 16-byte eMMC CID, a structure-size marker, version bytes
(`2`, `80`), a lock byte, and a CRC-16 of the embedded loader. `is_configured()`
requires all identity checks to match. A forced reset or mismatch causes
`rewrite_payload()` to install the boot data and call `init_config()`, erasing
the old timing records and writing a new header.

Each offset has a 256-byte page addressed by
`CONFIG_IDX(offset) = ((offset - 6200) / 8) + 1`; page 0 contains the header.
Successful boots clear successive bits in that offset's page, up to 2,048
records per page. `get_weigth()` counts those cleared bits. `find_best_record()`
orders positive weights and tracks ties across calls. If an offset's page is
already exhausted, `add_boot_record()` locks further recording globally; the
lock does not prevent using existing records.

The exact candidate bounds have a small asymmetry: the saved-record scan uses
`6200 <= offset < 6900` in steps of 8 (through 6896), while `OFFSET_CNT` uses
integer division and produces 87 shuffled candidates (through 6888).
`CONFIG_SIZE` is `0x5800`; `erase_config()` visits 4-KiB sectors from `0x8000`
through `0xd000`, skipping blank sectors. Thus its sector-rounded erase extent
ends at `0xe000`, below the counter sector at `0xf000`.

## 6. MCU-driven eMMC access and BOOT0 installation

`start_mmc()` takes control of CLK/CMD/DAT0/reset and configures PIO0 for
single-data-line access. With a 200-MHz system clock and divider 266, the
two-phase clock is approximately 376 kHz. CMD/DAT transmission uses pin-direction
changes for open-drain output. Command frames carry CRC-7; 512-byte data reads
and writes use CRC-16 with polynomial `0x1021` and initial value zero.

`mmc_initialize()` issues idle/operating-condition requests, reads and checks
the CID, assigns relative address 2, selects the device, sets 512-byte blocks,
and switches partition access to BOOT0. `reinit_mmc()` allows ten initialization
attempts before halting. `stop_mmc()` disables PIO0's machines and releases the
interface pads.

`fast_check()` reads BOOT0 block 1 and recognizes the installed BCT by the
32-bit value at byte `0x20`: `0xa56ca203` for Mariko or `0x69696969` for Erista.
This is a **marker check**, not a full BCT/loader integrity comparison. It also
handles the command mailbox described below and emits fixed GPIO waveforms on
the board-selected regulator I2C pins; it is not a general I2C driver.

If the marker is absent or configuration does not match, `write_payload()`
prepares an Erista or Mariko `0x2800`-byte BCT from fixed fields and embedded
arrays, then performs the following block operations:

| BOOT0 block(s), each 512 bytes | Operation |
| --- | --- |
| `0x0000` → `0x07a0` | Copy the first original `0x2800`-byte BCT to backup |
| `0x0020` → `0x07c0` | Copy the second original BCT to backup |
| `0x1f80` onward | Write the 29,952-byte embedded loader (through block `0x1fba`) |
| `0x0000` and `0x0020` onward | Write the prepared BCT at both locations |
| `0x1fff` | Write firmware descriptor |

The backup copies happen only when neither an installed marker nor a mailbox
command was recognized. Writes use readback comparison, up to six attempts
per block, with reinitialization on the fourth attempt. `copy_bct()` also
retries and verifies its copies. These are blockwise operations, not an atomic
transaction. After an unrecognized, non-command installation that was not a
self-reset, the code can take a silent halt instead of continuing to glitch.

The descriptor contains signature `0x9cabe959`, major/minor version, loader
CRC-16, the running firmware slot's CRC-32, and counter value adjusted to the
running slot's parity. The source calls the CRC fields hashes; they are not
cryptographic authentication.

## 7. Firmware-update mailbox and slot handoff

BOOT0 block 1 doubles as a mailbox. `fast_check()` recognizes:

- `0x515205c5`: reset/configuration command; sets `is_command`, suppressing the
  original-BCT backup path on a subsequent rewrite. It does not directly erase
  the RP2040 configuration here.
- `0x6db92148`: update command, followed by 32-bit start-block and block-count
  fields. Update processing runs only when `boot_try == 0`.

`update_firmware()` first replaces the mailbox with the reset command. Both
fields equal to `0xffffffff` request a reboot without copying another image.
Otherwise it checks the requested block range against `0x1fff`, limits the
count to `0x1c0`, and rejects a first-block firmware CRC equal to the running
slot's header CRC. It copies BOOT0 blocks into the **other** RP2040 slot,
erasing at 4-KiB boundaries and programming 512 bytes at a time.

The copy routine does not itself validate the complete new image's size or
CRC-32, authenticate a signature, or read back RP2040 flash. After a successful
copy it stops eMMC, releases pins, and forces a watchdog reboot. Busk then
applies its slot retry policy and validates the image before execution. The
new slot becomes the preferred slot only after USK reaches glitch success and
updates the counter parity; otherwise busk can return to the earlier slot.

## 8. LEDs, fault signaling, and halt

This snapshot does not implement the extended training-color scheme described
by some later forks. `main()` uses blue for startup/search, white while rewriting
boot data, and `inc_tries()` alternates two blue intensities. Pico boards use a
single GPIO LED; other mappings use the WS2812 PIO output.

`halt_with_error(err, bits)` emits the `bits` most-significant-first bits of
`err`: 0 is a short pulse and 1 a long pulse. Multi-bit codes repeat three times;
one-bit codes do not. A successful one-bit zero uses white; other pulses use
`PIX_yel`. The nominal pulse lengths are 140 ms short and 560 ms long; the
success pulse is 280 ms. Call-site meanings include:

| Code (`*` short, `=` long) | Condition |
| --- | --- |
| `*` | Glitch success |
| `=` | Watchdog/first-attempt completion path |
| `**`, `*=`, `=*` | Reset, CMD, or DAT0 ADC self-test failure, respectively |
| `***` through `*==` | Boot observation failures; see `wait_for_boot()` branches |
| `=**` | Five consecutive missing results during search |
| `=*=` / `==*` | Width reached the lower / upper limit |
| `===` | Full search rounds exhausted |
| `=***` through `====` | eMMC initialization, write, update-read, or BCT-copy errors; see `payload.c` call sites |

A `bits == 0` call emits no pulses. Before signaling, the function disables PIO
machines, releases non-LED pads (retaining glitch-pin pulldowns), and reduces
clock/voltage. It then releases the LEDs and calls `zzz()`, which switches away
from the main clock, disables oscillators/brownout detection, lowers the
regulator setting, and spins forever. This is an MCU low-power halt, not a
console shutdown command.

## 9. Build and release packaging

`CMakeLists.txt` builds the nine C translation units listed above, generates
headers for both PIO sources, selects `MinSizeRel`, disables USB/UART stdio, and
links the Pico SDK hardware libraries. Its program metadata still says `2.7`;
the actual firmware version bytes and release naming come from `config.h`.

The snapshot workflow builds busk first using a linker-script RAM region at
`0x20038000` of length 32 KiB, restores the SDK script, then builds USK.
`prepare.py`, run from `build/usk`, reads `../busk/busk.bin` and `usk.bin`:

1. Prefix USK with little-endian 32-bit body length and `zlib.crc32(body)`;
   write that header plus body as `update.bin`.
2. Put busk at flash address `0x10000000` and an erased page at `0x1000ff00`,
   filling intervening gaps with `0xff`.
3. Put the header/body (plus three `0xff` bytes before page padding) into slot 0
   at `0x10010000` and emit `firmware.uf2` with RP2040 family ID `0xe48bff56`.

The combined UF2 initializes the gap containing configuration and counter state;
it does not populate slot 1. `update.bin` has no busk image and no UF2 framing.
Reproducing the official 2.80 outputs requires the pinned sources and toolchain
in [SNAPSHOT.md](SNAPSHOT.md), not just this packaging format.

## Reading the source literally

The checks described here should not be read as stronger guarantees than the
code provides. In particular, `mmc_initialize()` contains
`res & mask != expected` without parentheses; C parses that as
`res & (mask != expected)`, not `(res & mask) != expected`. The marker check,
CRC checks, retry counters, and slot policy also have the specific boundaries
described above. No firmware changes are made by this documentation.
