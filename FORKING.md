# Independent copies and active development

This repository preserves usk **2.80**. Its historical dependency pins and the
[`usk-toolchain`](https://github.com/supersnug/usk-toolchain) image are for
reproducing that snapshot, not a recommended starting environment for ongoing
firmware development.

## Prefer an independent repository to a GitHub fork

For independent maintenance or preservation, prefer cloning the source,
removing the clone's `.git` directory, initializing a new repository, and pushing
it to a newly created, empty GitHub repository instead of using **Fork**.
This avoids placing your copy in GitHub's upstream fork network.

Keep the license, applicable notices, and clear attribution to upstream authors.
Reinitializing Git does not change ownership or licensing of the source.

### Required repositories

For a self-contained copy, independently clone and reupload **each** required
repository, not just `usk`:

| Repository | Required for |
| --- | --- |
| [`usk`](https://github.com/supersnug/usk) | Every copy: the main firmware |
| [`busk`](https://github.com/supersnug/busk) | Every copy: the bootloader included in the firmware image |
| [`usk-toolchain`](https://github.com/supersnug/usk-toolchain) | A preserved 2.80 snapshot: the frozen build-image recipe and publisher |

For active development, copy `usk` and `busk`, then modernize the build as
described below instead of using the historical `usk-toolchain` environment.

### Create a fresh independent copy

Create an **empty repository**, not a GitHub fork, under your own account or
organization. Do not initialize it with a README or license before this push.
Replace the destination URL below with your new repository's URL.

Run these commands as a script from outside any existing working copy:

```bash
set -e
git clone https://github.com/supersnug/usk.git usk-independent
cd usk-independent

# Record provenance before discarding this fresh clone's Git history.
git remote get-url origin > UPSTREAM_COMMIT
git rev-parse HEAD >> UPSTREAM_COMMIT

# Only do this in the new, disposable clone, never your existing working copy.
rm -rf -- .git
git init -b main
git add .
git commit -m "Import USK sources; record upstream in UPSTREAM_COMMIT"
git remote add origin https://github.com/YOUR_OWNER/YOUR_REPOSITORY.git
git push -u origin main
```

Deleting `.git` discards the clone's commits, branches, tags, remotes, and local
Git settings. It leaves the source and license files intact. Do not use this
procedure on a working copy containing history or settings you need to retain.
Repeat the procedure for `busk` and, for a 2.80 snapshot, `usk-toolchain`.

### Connect your copies

- Update the `usk` and `busk` checkout repositories in
  `.github/workflows/build.yml` to your own repositories.
- Reinitializing Git creates new commit IDs. For a 2.80 snapshot, pin your own
  imported `usk` and `busk` commits; the old source SHAs do not exist in your new
  histories. Preserve the SDK and picotool revision pins.
- In your snapshot's `usk-toolchain` copy, update the GHCR image name in
  `.github/workflows/publish.yml` and the source-repository label in `Dockerfile`.
  Run its publisher, enable public access to your container package, and use
  the resulting image digest in your `usk` workflow. Copying the toolchain
  repository alone does not copy the published container image.

## For active development, modernize the original workflow

Do **not** base an actively developed project on this snapshot's old toolchain
image or copy its historical dependency/source pins unchanged. Those choices
intentionally freeze 2024 behavior and receive no automatic security updates.

Use [`build.yml.original`](build.yml.original) as a reference when reworking
`.github/workflows/build.yml` for supported, modern versions. It is an old
workflow, **not** a ready-to-run modern replacement. In particular:

- Change the USK and busk checkout repositories to your own copies. Build the
  triggering USK commit rather than this snapshot's fixed source SHA; otherwise
  your firmware edits will not be built.
- Select current supported Ubuntu, compiler, CMake, Pico SDK, and picotool
  versions. Update the GitHub Actions and use an explicitly compatible
  SDK/picotool pair rather than accidentally fetching an unrelated newer tool.
- Rework the linker configuration for the selected SDK. The original workflow's
  hard-coded linker-script path and text replacement do not work with newer
  SDK layouts. Prefer the SDK's supported, target-specific linker overrides
  instead of modifying its installed files.
- Adapt firmware code to any changed SDK APIs, and verify binary generation,
  memory placement, packaging, and behavior on the intended hardware.

Pinning tested **modern** dependencies is still useful for reproducibility;
the recommendation is to replace this snapshot's obsolete baseline, not to
make every dependency float. Maintain and update your own toolchain separately.
