# μsk (Picofly firmware)

μsk, or **usk**, is firmware for RP2040-based Nintendo Switch voltage-glitching
modchips, commonly called **Picofly**. It runs on the modchip, not on the Switch's
main processor. This repository preserves **Rehius's 2.80 firmware** with a
reproducible build environment.

## What it does

USK coordinates the early boot process so the console can run a separate loader
and then a payload from its SD card. At a high level, it:

- Detects the modchip board layout and observes the console's eMMC boot traffic.
- Prepares eMMC boot data and installs the embedded console-side loader when
  needed, including when its boot-data marker has been overwritten.
- Times a voltage-glitch pulse and adjusts its timing through repeated attempts.
- Remembers successful timing offsets for the attached eMMC to prioritize them
  on later boots.
- Supports firmware updates staged through eMMC, with two RP2040 firmware slots
  and a separate bootloader for validation and retry/rollback.
- Reports progress and faults through the board's LED, then releases its signals
  and enters a low-power halt after finishing.

There are three distinct pieces: **busk** starts the RP2040 and loads **usk**;
usk prepares and glitches the console's boot; the embedded **sdloader** runs on
the console and loads the next program. USK itself does not read the SD card or
implement the operating system launched afterward.

## Documentation

| Document | Contents |
| --- | --- |
| [Implementation](IMPLEMENTATION.md) | MCU-side startup, hardware control, glitch search, flash storage, eMMC writes, updates, and packaging |
| [SD loader](SDLOADER.md) | Static interpretation of the embedded console-side binary, with evidence and remaining uncertainties |
| [Build snapshot](SNAPSHOT.md) | Frozen dependencies, toolchain image, build provenance, and byte-for-byte verification against the original 2.80 release |
| [Independent copies and development](FORKING.md) | Required companion repositories, preservation instructions, and guidance for active development |

## Building this snapshot

Run **Build PicoFly binaries** manually from the repository's GitHub Actions
page. The [workflow](.github/workflows/build.yml) uses pinned USK, busk, SDK, and
picotool revisions plus a prebuilt toolchain image, and publishes `firmware.uf2`
and `update.bin` as release assets.

**The snapshot workflow builds its pinned source revision, not arbitrary edits
on the triggering branch.** For your own changes or ongoing development, follow
[FORKING.md](FORKING.md). Historical toolchain details and verified output hashes
are in [SNAPSHOT.md](SNAPSHOT.md).

## Credits and license

This copy preserves [Rehius's USK](https://github.com/rehius/usk), with credit to
abal1000x and Rehius for their work on the project. Its companion RP2040
bootloader is [busk](https://github.com/supersnug/busk). The README's organization
was inspired by [DefenderOfHyrule's USK documentation](https://github.com/DefenderOfHyrule/usk).

The repository's source license is recorded in [LICENSE](LICENSE). The embedded
console-side binary's exact source/build provenance remains unresolved; see
[SDLOADER.md](SDLOADER.md).
