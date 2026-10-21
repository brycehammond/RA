# RA

The RetroArch binaries that spruceOS ships in its `RetroArch/` folder are
built here. Each target is a GitHub Actions workflow that checks out upstream
[libretro/RetroArch](https://github.com/libretro/RetroArch) at a pinned
version, applies spruce's patches, cross-compiles, and uploads the stripped
binary to a prerelease named `beta-<branch>`. Nothing is built on anyone's
machine; running the workflow is the build.

## Targets

| Workflow | Output | spruceOS devices | RetroArch | Patches |
| --- | --- | --- | --- | --- |
| `build-universal.yml` | `ra64.universal` | Miyoo Flip, TrimUI Smart Pro / Smart Pro S / Brick / Brick Pro, MagicX XU20 / Zero28 / Zero40, dArkMoss devices | `v1.22.2` | `common` |
| `build-universal32.yml` | `ra32.universal` | 32-bit cores on the dArkMoss devices | `v1.22.2` | `common` |
| `build-h700.yml` | `ra64.h700` | Anbernic RG xx line (H700) | `v1.22.2` | `common` |
| `build-h700-32.yml` | `ra32.h700` | 32-bit cores on the Anbernic RG xx line | `v1.22.2` | `common` |
| `build-a30.yml` | `ra32.a30` | Miyoo A30 | `v1.22.2` | `common`, then `a30` |
| `build-miyoomini.yml` | `ra32.mini` | Miyoo Mini / Mini+ | `v1.22.2` | `miyoomini` |
| `build-pixel2.yml` | `ra64.pixel2` | GKD Pixel 2 | commit `e5eff6db` | `pixel2` |
| `build-rgds.yml` | `ra64.rgds` | RGDS (ROCKNIX) | commit `e5eff6db` | `rgds` |
| `build-all.yml` | all of the above | | | |

Which binary a device launches is decided in spruceOS, in
`spruce/scripts/platform/<Platform>.cfg` (`RA_BIN`, and `RA_BIN_32` /
`RA_BIN_64` where a device switches per core).

### How the targets differ

The build scripts carry the reasoning in their comments; in short:

- **universal** (`build.sh`, `Dockerfile`): Ubuntu 20.04 with the Debian
  aarch64 cross toolchain. SDL2 for video and input, GLES and Vulkan, no X11,
  Wayland or KMS. The Vulkan driver is there for the Smart Pro S.
- **universal32** (`build-universal32.sh`, `Dockerfile.universal32`): the
  same recipe for armhf, for devices that run some cores 32-bit.
- **h700** (`build-h700.sh`, `Dockerfile.h700`): the universal recipe plus
  `--enable-mali_fbdev`, so RetroArch talks to the Mali blob directly instead
  of going through SDL2's GL context. Vulkan and KMS off, since the H700 has
  neither.
- **h700-32** (`build-h700-32.sh`, `Dockerfile.h700-32`): armhf for the same
  devices, with NEON on. On aarch64 NEON is architectural; on armhf it has to
  be asked for, and it is the main reason this target exists.
- **a30** (`build-a30.sh`, `Dockerfile.a30`): built against steward-fu's A30
  buildroot toolchain (GCC 13.2, glibc 2.23 sysroot, Mali fbdev EGL and GLES2,
  SDL2 2.28.5), downloaded by the workflow at build time.
- **miyoomini** (`build-miyoomini.sh`, `Dockerfile.miyoomini`): built in the
  `aemiii91/miyoomini-toolchain` image with OnionUI's patches and spruce's own
  framebuffer sources from `miyoomini_src/`, using `Makefile.miyoomini`.
- **pixel2** and **rgds** (`build-pixel2.sh`, `build-rgds.sh`): native
  aarch64 builds on an `ubuntu-24.04-arm` runner, no Docker, with the
  Rockchip Mali driver installed from ginkage/libmali-rockchip. These pin a
  RetroArch commit rather than a tag.

## Running a build

1. Actions, pick the target's workflow, Run workflow. The `retroarch_version`
   input (or `retroarch_ref` for pixel2 and rgds) overrides the pinned
   version for that run.
2. When it finishes, the binary is on the Releases page under
   `Beta Build (<branch>)`, already renamed to its spruceOS name. Re-running
   replaces it (`--clobber`).
3. To try it, copy it over the same-named file in `RetroArch/` on a card. To
   ship it, commit it to spruceOS.

`build-all.yml` runs every target with one version input.

## Patches

`patches/common/` applies to every cross-compiled target. `patches/a30/`
goes on top of common for the A30. `patches/miyoomini/`, `patches/pixel2/`
and `patches/rgds/` are complete sets of their own.

Patches apply in filename order with strict `git apply`, except the OnionUI
patches under `patches/miyoomini/000*`, which go through `patch -p1` for fuzz
tolerance. Some upstream files have CRLF line endings; the miyoomini, pixel2
and rgds scripts normalise them before patching.

`check-patches.yml` runs on every push and pull request and applies each
target's patch set to its pinned RetroArch, so a stale hunk fails in a minute
instead of after a Docker build. It also checks that the in-game menu source
is identical between `patches/common/0001-spruce-igm.patch` and the pixel2
and rgds copies of it.

### The in-game menu

Spruce's in-game menu (IGM) is RetroArch source carried as a patch:
`patches/common/0001-spruce-igm.patch` adds `spruce_igm.c` and hooks it into
the frontend, and `common_src/` holds the platform header and theme that are
copied over the patched tree before configure. The Miyoo Mini has its own
software-rendered version in `miyoomini_src/`.

### Adding or updating a patch

1. Check out RetroArch at the target's pinned version and apply the existing
   patches in order.
2. Make the change and save it with `git diff > patches/<dir>/NNNN-name.patch`,
   numbered after the last one.
3. Push. `check-patches.yml` tells you whether the whole set still applies.

A patch that backports an upstream commit should say so in a comment, with
the upstream hash, so it can be dropped when the pinned version catches up.

## Changing the RetroArch version

The version is pinned in three places: the workflow input defaults, the
`RETROARCH_VERSION` / `RETROARCH_REF` default in each build script, and the
`check-patches.yml` matrix. Change all three together. Expect the patches to
need rebasing; the check workflow will say which hunks moved.

## Building locally

The cross-compiled targets are plain Docker and run anywhere Docker does:

```sh
docker build -t ra-builder .
mkdir -p output
docker run --rm -e RETROARCH_VERSION=v1.22.2 -v "$PWD/output:/output" ra-builder
```

Swap the Dockerfile (`-f Dockerfile.h700`) and the output name for the other
targets; the workflows show the exact pairing. On an Apple Silicon host the
image runs under amd64 emulation, which is slow and occasionally trips the
LTO link step; running the final `make` with `-j1` gets past it.

## Other files

- `retroarch-cli-arguments.txt`: a reference of RetroArch's command-line
  options, for whoever is writing the launch scripts in spruceOS.
