# Static interpretation of USK 2.80's console-side loader

Investigated **2026-11-13**, without executing the ARM payload. Scope: interpret
the `payload[]` blob in
[`rehius/usk@d3dca674aa41cce1fda0cc432dc1ff6b8cb4b848/payload.h`](https://github.com/rehius/usk/blob/d3dca674aa41cce1fda0cc432dc1ff6b8cb4b848/payload.h).
The 29,952-byte target has SHA-256
`e5908ab15746d0060780a6f434b751b2f455a7c651d30eddcbcb75bdf3ee5fdc`.

## Conclusion and confidence

The visible program is a small, standalone Tegra-side SD/file loader with
hardware initialization, button-selected paths, filesystem support, an error
display, and shutdown/reboot paths. Its successful path transfers execution
to a file loaded from storage. It is not RP2040 code or a complete operating
system. ARM-state startup plus Thumb routines and Tegra peripheral addresses
are consistent with a BPMP/ARM7-style early-boot environment; hardware execution
was not observed.

**Confirmed** below means supported by bytes and traced instructions. Names
such as `f_open`, `main`, and `memset` are analyst-assigned interpretations, not
recovered symbols. Exact upstream source, compiler, and complete behavior remain
unidentified; do not treat the pseudocode as a drop-in source reconstruction.

## Header and relocation

- The signed-format header declares version 9, a 29,584-byte body, and load/entry
  address `0x40010170`. The declared body starts at file offset `0x170`; its
  SHA-256 matches the digest stored in the header. This is not verification of
  a trusted cryptographic signature.
- A shared bootstrap component occupies `0x170:0x240`. Its full execution and
  handoff were not reconstructed here. Much of the following body is directly
  disassemblable code, not an opaque encrypted block.
- ARM instructions start at file offset `0x240`. They obtain their current
  address and compare it with `0x40004000`, the program's linked execution base.
- If relocation is needed, they place a short copying helper at `0x4003ff00`,
  copy initialized program contents to `0x40004000`, and resume there.
- Startup sets SP to `0x40004000`, clears the BSS range
  `0x4000b2b0:0x4000bb20`, and enters the Thumb main routine at `0x40004108`.

For the relocated main program, convert a file offset to a runtime address with
`address = file_offset + 0x40003dc0` (`0x240` becomes `0x40004000`). Do not apply
this mapping to the earlier wrapper/bootstrap or disassemble literal pools as
instructions.

## File loading: confirmed by strings and their call sites

| File offset | String address after relocation | Meaning established from use |
| --- | --- | --- |
| `0x6224` | `0x40009fe4` | `payload.bin`, normal first-choice filename |
| `0x6230` | `0x40009ff0` | `toolbox.bin`, attempted on a special button-selected path |
| `0x623c` | `0x40009ffc` | `bootloader/update.bin`, fallback when opening `payload.bin` fails |

Main calls a filesystem mount routine, then passes these string addresses to
the same open-like routine at `0x40008ba8` with read-mode argument 1.
The normal open/fallback branch is at `0x40004208:0x40004224`; the special toolbox
branch is at `0x4000463c:0x40004658` and rejoins the normal branch if it cannot
use that file. The special flag is set when the volume-only button routine
returns 4, consistent with volume-up in the public reference code.

After opening a file, main reads its size, rejects values greater than
`0x20000` (128 KiB), and reads into `0x4000f800` in chunks of at most `0x10000`
(64 KiB). It handles read failures, closes the file and unmounts the filesystem,
then branches to `0x4000f800` through a `bx r3` trampoline at `0x4000466c`.
The observed limit is exactly 128 KiB, not the public Spacecraft source's
different `0x1f000` limit.

The filesystem code includes `EXFAT   `, `FAT32   ` and `EFI PART` markers and
FatFs-style structures/control flow. This strongly identifies a FAT/exFAT
storage stack; actual compatibility with particular cards was not tested.

## Modchip success indication

The helper at `0x4000618c` constructs a command using argument `0xaa5458ba`
(literal at file offset `0x240c`) and issues it through the eMMC command code.
Main calls it after setting a buffer's first byte to `0xff` or `0x55` in
different branches. This is not evidence of a cryptographically authenticated
handshake or of all request/response data having been decoded.

The connection to the MCU firmware is direct:
[`glitch.c::do_glitch`](https://github.com/rehius/usk/blob/d3dca674aa41cce1fda0cc432dc1ff6b8cb4b848/glitch.c)
recognizes command words `0x40aa5458` followed by `0xba3b` and marks success.
Those words encode the same command argument within its captured frame.
Thus the RP2040's success verdict is based on seeing a recognizable command
from running console-side code, not on directly observing a skipped instruction.

## Buttons, error display and power handling

The volume-only helper at `0x40004098` reads two GPIO inputs and returns bits 2
and 4. The main routine explicitly compares results against 4 and 6. The error
loop also reads a PMIC power-button bit. A held combination selects a reboot
path; identifying all consequences of those reboot flags requires more work.

The error-message pointer table at runtime `0x40009fa8` selects:

1. `Failed to init / SD Card!`
2. `No SD Card`
3. `Failed to mount / SD Card!`
4. `Failed to open / payload.bin!`
5. `Failed! Payload, / is >128KB!`
6. `Failed to read / payload.bin!`

Main has call sites to display/backlight and glyph-rendering routines on these
paths. It waits on buttons or a deadline of 120,000 milliseconds. The clock
helper at `0x400049a4` combines RTC seconds and shadow milliseconds; main adds
`0x1d4c0` before the wait loop. A timeout enters the PMIC shutdown path.

At `0x40004a40`, code performs I2C operations on the PMIC, including writing
`0x80` to register `0x41`, then loops. Separately, `0x400040c0` writes PMC
scratch state and configures a Tegra watchdog reset. These are distinct
shutdown and reboot paths. Exact screen artwork was not decoded.

## Approximate high-level flow

```text
relocate program and clear BSS
initialize console hardware and eMMC
issue the recognizable modchip command
evaluate buttons / alternate reboot paths
initialize SD and mount its filesystem
    special selection: try toolbox.bin
    otherwise: try payload.bin, then bootloader/update.bin
check file size <= 128 KiB
read selected file into RAM
close file, unmount, finish storage interface
jump to loaded program

on storage/file errors:
    initialize error display and backlight
    show the corresponding message
    wait for button-selected actions or the two-minute timeout
    reboot or power down
```

This combines branch paths for readability; it is not an exact line-by-line
decompilation. For example, some storage errors branch directly to the display
path, and some button checks occur before filesystem initialization.

## Reference source and reproducibility

Public [Spacecraft-NX source](https://github.com/Spacecraft-NX/sdloader/tree/8b77b5727128147dfddee3d012998aa54b026e6c)
was used to interpret familiar components, not as proof of an exact match:
[`btn.h`](https://github.com/Spacecraft-NX/sdloader/blob/8b77b5727128147dfddee3d012998aa54b026e6c/src/btn.h),
[`btn.c`](https://github.com/Spacecraft-NX/sdloader/blob/8b77b5727128147dfddee3d012998aa54b026e6c/src/btn.c),
[`utils.c`](https://github.com/Spacecraft-NX/sdloader/blob/8b77b5727128147dfddee3d012998aa54b026e6c/src/utils.c),
[`pmc.h`](https://github.com/Spacecraft-NX/sdloader/blob/8b77b5727128147dfddee3d012998aa54b026e6c/src/pmc.h),
[`timers.h`](https://github.com/Spacecraft-NX/sdloader/blob/8b77b5727128147dfddee3d012998aa54b026e6c/src/timers.h),
and [`ff.c`](https://github.com/Spacecraft-NX/sdloader/blob/8b77b5727128147dfddee3d012998aa54b026e6c/src/lib/fatfs/ff.c).
The publicly available startup/linker code also differs from this blob.

Extract the hexadecimal byte literals from only the `payload[]` initializer in
`payload.h`, preserving their order, and save them as binary `payload.bin`.
Exclude the separate BCT arrays that follow it. Verify the size and SHA-256
listed above, then inspect with GNU ARM objdump:

```bash
# Initial ARM entry: map the package to 0x40010000.
arm-none-eabi-objdump -D -b binary -m arm --adjust-vma=0x40010000 \
  --start-address=0x40010240 --stop-address=0x400102d8 payload.bin
# Relocated Thumb main; literal pools are still data, not instructions.
arm-none-eabi-objdump -D -b binary -m arm -M force-thumb \
  --adjust-vma=0x40003dc0 --start-address=0x40004108 \
  --stop-address=0x4000466e payload.bin
strings -a -t x -n 5 payload.bin
```

These commands were run using the extracted blob and the pinned Ubuntu ARM
binutils in a filesystem sandbox. Disassembly is retained for this session in
`/tmp/opencode/sdloader-compare.vOwaFTKo/usk-thumb-disassembly.txt` and
`usk-selected-disassembly.txt`. Nothing was flashed or run on console hardware.
