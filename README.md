# strace github-cross-testing

A reusable GitHub Actions workflow for continuous cross-architecture testing of
[strace](https://strace.io/) on rarely-tested CPU architectures.  It builds a
cross toolchain and minimal root filesystem with
[Buildroot](https://buildroot.org/) (version pinned in
[`ci/buildroot-version`](ci/buildroot-version)), cross-compiles strace and its
test suite, boots the target under QEMU system emulation, and runs `make check`
in the guest.

## Usage from strace/strace

```yaml
# .github/workflows/cross.yml
name: cross
on: [push, pull_request]
jobs:
  cross:
    uses: gabifalk/strace-github-cross-testing/.github/workflows/cross.yml@v1
```

This tests the pushed commit.  To restrict arches or shards:

```yaml
    with:
      arches: 'm68k s390x'   # subset; omit for all built-in arches
      shards: 10             # default 20
```

## Inputs (`cross.yml`)

| Input               | Type   | Default | Meaning                                                               |
|---------------------|--------|---------|-----------------------------------------------------------------------|
| `arches`            | string | `''`    | Space-separated.  Empty -> all arches in [`ci/arches`](ci/arches).    |
| `shards`            | number | `20`    | Test shards per arch.                                                 |
| `strace-repository` | string | `''`    | Empty -> the calling repo (`github.repository`).                      |
| `strace-ref`        | string | `''`    | Empty -> the calling commit (`github.sha`).                           |

## Adding an architecture

1. Add the arch name to [`ci/arches`](ci/arches).
2. Add `ci/buildroot-config-<arch>` (with a `# CONFIG_VERSION:` marker).
3. Add the per-arch QEMU/guest settings to the `case "$ARCH"` block in
   [`ci/lib.sh`](ci/lib.sh).

## How it works

`cross.yml` (the engine, `workflow_call` only) resolves the arch list and strace
source, then fans out a matrix calling `arch.yml` per arch.  `arch.yml` builds
the toolchain + strace (caching both), then runs the test suite sharded across
parallel QEMU jobs.

## Patches

Patches are applied to Buildroot packages during the build (via
`BR2_GLOBAL_PATCH_DIR` in the arch configs that need them), fixing
build-environment and kernel bugs that would otherwise cause false strace
failures.  Buildroot applies each `ci/patches/<package>/` subdirectory to the
package of that name:

- [`ci/patches/gcc/`](ci/patches/gcc) -- m68k `fold-mem-offsets` miscompile
  breaking `sendfile`/`splice`.
- [`ci/patches/elfutils/`](ci/patches/elfutils) -- mips64el MIPS unwinding bug
  breaking `strace -k`/`-kk`.
- [`ci/patches/linux/`](ci/patches/linux) -- hppa (parisc) `__get_user` kernel
  bug that evaluates its pointer argument twice.

## Running locally

The `ci/` scripts are plain POSIX shell:

```sh
export STRACE_SRC=$PWD/strace BUILDROOT_SRC=$PWD/buildroot OUTPUT_BASE=$PWD/output
./ci/build-buildroot s390x                       # toolchain, kernel, rootfs (slow)
STRACE_PATCHES=ci/strace-patches ./ci/build-strace s390x   # cross-compile (patches optional)
./ci/run-qemu        s390x                       # boot under QEMU, run make check
```

## License

GPL-2.0-or-later.  See [LICENSE](LICENSE).
