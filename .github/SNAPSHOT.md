# USK 2.80 build snapshot

The workflow reconstructs dependencies available when rehius/usk 2.80 was
published: **2024-09-04 04:12:04 UTC**. The exact original tool revisions are not
known, but this reconstruction reproduced both published binaries byte-for-byte.

| Input | Frozen selection |
| --- | --- |
| USK | `supersnug/usk@eaceeb9538020e3e9ecd375878d25b5f2ba0fe46` |
| busk | `supersnug/busk@3f715c6585140ec3ab9128a20b719f5035191374` |
| Pico SDK | `dfb573818f0f293177a89d99424c13944ef76656` (release-time `master`) |
| picotool | `971ee85176dd17a3bf43713cc96cd52e24b449f6` (release-time `develop`) |
| CMake | 3.30.2, official Linux x86-64 archive, SHA-256 checked in the workflow |
| Ubuntu userspace | `jammy-20240808`, pinned amd64 image manifest digest |
| APT packages | `20240904T041204Z` archive, including updates, security and transitive dependencies |
| Checkout action | `f43a0e5ff2bd294095638e18286ca9a3d1956744` (v3.6.0) |
| Release action | `de2c0eb89ae2a093876385947365aca7b0e5f844` (release-time v1) |

USK's firmware files match upstream tag `PicoFly_2.80_2024.09.04_04-12`
(`d3dca674aa41cce1fda0cc432dc1ff6b8cb4b848`). busk's files match upstream
`a38fc718ce3027a3f6ca4eeac687fa4ce2caae5d`. Only workflow metadata differs for USK.
Changing the fork's default branch does not change the source built by this workflow.
SDK submodules are fixed by its commit; pioasm is built from that SDK's source.
Picotool is built by the SDK with its normal options, using the pinned local source.

Historical evidence:

- [Release](https://github.com/rehius/usk/releases/tag/PicoFly_2.80_2024.09.04_04-12)
- [Runner manifest listing CMake 3.30.2](https://github.com/actions/runner-images/blob/0d632219e5a750d8d1b9974d1e8713a2fa111ce9/images/ubuntu/Ubuntu2204-Readme.md)
- [Ubuntu image history](https://github.com/docker-library/official-images/blob/8730cefe04ffde78c1b85746ddefc2793b7be120/library/ubuntu)
- [Ubuntu package snapshot](https://snapshot.ubuntu.com/ubuntu/20240904T041204Z/)

## Verification

A full local build using the pinned Ubuntu image filesystem, package snapshot,
CMake and source revisions produced these SHA-256 hashes. Both files were
compared directly with the upstream 2.80 release downloads and were identical.
GitHub Actions orchestration and release publishing were not exercised locally.

| Output | SHA-256 |
| --- | --- |
| `firmware.uf2` | `e2fb2d4bce9d69db294a71b6304c33c443bd13a75d0cdc1424e55ff58feed831` |
| `update.bin` | `d95f8e2cd89c4618d5ddeb33a28886281cf0dd103ad7fa9215697940dd3ebb02` |

## Limits

The host uses `ubuntu-22.04`, but GitHub still manages its kernel, container engine
and JavaScript action runtimes. Host CA certificates are mounted read-only solely
to bootstrap authenticated HTTPS package installation; package signatures remain
checked. The build tools and libraries come from the frozen container/archive,
not the host. Release timestamps and GitHub services also remain live inputs.
Historical packages/actions intentionally do not receive automatic security fixes;
use this workflow only for trusted snapshot sources.
