# AGENTS.md — Coding Agent Guide

## What This Repo Is

A fork of [crosstool-NG](https://github.com/crosstool-ng/crosstool-ng) that builds pre-packaged ARM GCC cross-compilation toolchains matching each Ubuntu LTS's default GCC version. Built via GitHub Actions, published as `.tar.xz` archives on GitHub Releases.

**18 toolchain variants**: 6 Ubuntu LTS versions (14.04–24.04) x 3 ARM targets.

## Repository Layout

This repo uses **git worktrees** — each branch is checked out in its own directory:

```
master/                        ← default branch (this file lives here)
├── .github/workflows/         ← GitHub Actions release workflow
├── prompts/                   ← original project prompt + versions.yml
│   ├── 000_implement          ← project requirements and defconfig templates
│   └── versions.yml           ← Ubuntu LTS version matrix (GCC, glibc, binutils, etc.)
├── AGENTS.md                  ← this file
└── README.md

1.28.0/                        ← ct-ng 1.28.0 (Ubuntu 16.04–24.04 toolchains)
├── packages/                  ← component versions + patches
│   └── gdb/{9.2,12.1}/       ← includes our C23 readline patches
├── samples/                   ← defconfigs (our custom ones have -uXXYY suffix)
│   ├── arm-unknown-linux-gnueabi-u{1604..2404}/
│   ├── arm-unknown-linux-gnueabihf-u{1604..2404}/
│   └── arm-none-eabi-u{1604..2404}/
└── scripts/                   ← build scripts (DO NOT modify upstream ones unless necessary)

1.22.0/                        ← ct-ng 1.22.0 (Ubuntu 14.04 toolchains)
├── samples/
│   ├── arm-unknown-linux-gnueabi-u1404/
│   ├── arm-unknown-linux-gnueabihf-u1404/
│   └── arm-none-eabi-u1404/
└── scripts/
    └── build/companion_libs/  ← patched download URLs (ISL, libelf, expat)
```

**Important**: The worktree root is the *parent* directory, not `master/`. When working on CI, you're in `master/`. When patching ct-ng source or adding defconfigs, `cd` to `1.28.0/` or `1.22.0/`.

## Branches and What Goes Where

| Branch | Purpose | What to commit here |
|--------|---------|---------------------|
| `master` | CI/CD, docs, prompts | Workflow YAML, README, AGENTS.md, prompts |
| `1.28.0` | ct-ng source + patches for Ubuntu 16.04–24.04 | Defconfigs, source patches, build script fixes |
| `1.22.0` | ct-ng source + patches for Ubuntu 14.04 | Defconfigs, source patches, build script fixes |

## Targets

| Target tuple | Description | C library |
|-------------|-------------|-----------|
| `arm-unknown-linux-gnueabi` | ARM Linux, soft-float | glibc |
| `arm-unknown-linux-gnueabihf` | ARM Linux, hard-float | glibc |
| `arm-none-eabi` | ARM bare-metal, multilib | newlib |

## Version Matrix

See `prompts/versions.yml` for the full matrix. The rule is:
- **GCC, glibc, binutils**: major version MUST match the Ubuntu LTS. Minor/patch can be equal or newer.
- **Companion libs** (GMP, MPFR, MPC, ISL): can be equal or newer.

| Ubuntu | ct-ng | GCC | glibc | binutils | GDB |
|--------|-------|-----|-------|----------|-----|
| 24.04 | 1.28.0 | 13 | 2.39 | 2.42 | 15 |
| 22.04 | 1.28.0 | 11 | 2.35 | 2.38 | 12.1 |
| 20.04 | 1.28.0 | 9 | 2.31 | 2.34 | 9.2 |
| 18.04 | 1.28.0 | 7 | 2.27 | 2.30 | 8.3.1 |
| 16.04 | 1.28.0 | 5 | 2.23 | 2.26.1 | 8.3.1 |
| 14.04 | 1.22.0 | 4.8 | 2.19 | 2.24 | 7.7.1 |

## Sample Naming Convention

Our custom defconfigs use a `-uXXYY` suffix (Ubuntu version):
- `arm-unknown-linux-gnueabi-u2404` → Ubuntu 24.04, soft-float Linux
- `arm-unknown-linux-gnueabihf-u2004` → Ubuntu 20.04, hard-float Linux
- `arm-none-eabi-u1604` → Ubuntu 16.04, bare-metal multilib

Defconfigs live in `samples/<name>/crosstool.config`.

## Building ct-ng Locally

```bash
cd 1.28.0   # or 1.22.0
./bootstrap
./configure --enable-local
make -j$(nproc)
./ct-ng <sample-name>    # loads the defconfig into .config
./ct-ng build            # builds the toolchain
```

Redirect build stdout to `/dev/null` — check `$?` for success, read `build.log` on failure.

To resume a failed build from a specific step: `RESTART=<step> ./ct-ng build`

## CI Workflow (`master/.github/workflows/release.yml`)

**Jobs:**
1. `build-ctng` — builds ct-ng itself for each branch (1.28.0, 1.22.0), uploads as artifact
2. `download-sources` — downloads all source tarballs, caches them as artifact
3. `build-toolchains` — 18-job matrix, builds each toolchain, packages as `.tar.xz`
4. `release` — only on tag push (`v*`), creates GitHub Release with all 18 archives

**Triggers**: `workflow_dispatch` (manual) and `push tags v*`.

**Runner**: `ubuntu-22.04`. Host GCC is 14 (defaults to C23 — this matters for patches).

## Known Issues and Patches

### GDB readline vs GCC 14 / C23

GCC 14 on ubuntu-22.04 runners defaults to `-std=gnu23`. This breaks old readline code bundled with GDB:

1. `typedef RETSIGTYPE SigHandler ()` — K&R empty `()` means "zero params" in C23
2. `SIGHANDLER_RETURN` macro expands to `return (0)` in a void function

**Fix**: Source patches in `packages/gdb/{9.2,12.1}/` that change the typedef and the macro. These patches are critical — without them, u2004 and u2204 builds fail.

**NEVER** work around build errors by disabling features (e.g., `CT_GDB_NATIVE=n`). Always fix the root cause with a proper source patch.

### Dead download mirrors (1.22.0 only)

Several upstream download URLs are dead. Fixes in `scripts/build/companion_libs/`:
- ISL: `isl.gforge.inria.fr` → `libisl.sourceforge.io`
- libelf: `www.mr511.de` → `fossies.org/linux/misc/old/`
- expat: added `github.com/libexpat/libexpat/releases/` as primary mirror

### GCC 5 / u1604 special CFLAGS

The u1604 configs need extra CFLAGS for old glibc/GMP to build with a modern host compiler:
- `CT_GLIBC_EXTRA_CFLAGS` with `-Wno-*` flags
- `CT_GMP_EXTRA_CFLAGS="-std=gnu17"`
- `CT_NCURSES_EXTRA_CFLAGS="-std=gnu11"`

## How to Add a New Toolchain Variant

1. Determine which ct-ng branch covers the Ubuntu version (check version availability in `packages/` or `config/`)
2. `cd` to that branch's worktree
3. Create `samples/<target>-u<XXYY>/crosstool.config` based on the templates in `prompts/000_implement`
4. Build locally: `./ct-ng <sample> && ./ct-ng build`
5. On success: `./ct-ng savedefconfig DEFCONFIG=samples/<name>/crosstool.config`
6. Commit to the ct-ng branch
7. Add the sample to the matrix in `master/.github/workflows/release.yml` (all three sections: `download-sources`, `build-toolchains`, and the appropriate `build-ctng` branch)

## How to Fix a Build Failure

1. Read the tail of `build.log` to identify the failing component and error
2. Download the actual source code of the failing component (e.g., from ftp.gnu.org)
3. Understand the root cause in the source
4. Write a patch in `packages/<component>/<version>/NNNN-description.patch` (unified diff, `diff -urN` format with `a/` and `b/` prefixes)
5. Test the patch applies cleanly: `patch -p1 --dry-run -d /tmp/source < your.patch`
6. Commit to the ct-ng branch, push, and re-run the workflow

## Iterating on CI Failures

To speed up the build-and-error loop when debugging CI:
- Trim the workflow matrix to only the failing jobs (edit `release.yml` temporarily)
- Bare-metal multilib builds take 3–5 hours; Linux builds take ~45 minutes
- Always restore the full 18-job matrix before creating a release tag
- Use `gh run view <id>` and `gh run view <id> --log` to inspect CI results

## Release Process

1. Ensure all 18 jobs pass on a `workflow_dispatch` run
2. Create and push a tag: `git tag v<X.Y.Z> && git push origin v<X.Y.Z>`
3. The tag push triggers the release workflow which builds + uploads all 18 archives
4. If a release for the tag already exists, delete it first: `gh release delete <tag> -y`
