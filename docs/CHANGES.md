# What was changed

This is a working port of Node.js **v24.22.2** to **GNU/Hurd x86_64**. The
changes are split into eight plain unified diffs so that each part can be read
on its own instead of one 900-line diff.

The diffs have no mail headers, dates, or sign-offs. They are a reading and
reproduction aid, written after the build worked, not a submission proposal.

- Base: upstream `node-v24.22.2.tar.xz` (`v24.22.2`)
- Result: 14 files, 698 insertions, 21 deletions
- Each diff applies on top of the previous one; the tree after
  `08-v8-libc-gaps.diff` is the tree that was cross-compiled and tested on
  Debian GNU/Hurd x86_64

## Applying

```sh
tar -xf node-v24.22.2.tar.xz
cd node-v24.22.2
for p in /path/to/patches/*.diff; do
  patch -p1 < "$p"
done
```

Or with Git:

```sh
git apply /path/to/patches/*.diff
```

Both routes were tested against a pristine `v24.22.2` tree.

## Diffs at a glance

| # | Diff | Files | Purpose |
|---|---|---|---|
| 1 | `01-build-configure-gyp-hurd.diff` | `configure.py`, `common.gypi` | Register `hurd` as a destination OS and add Hurd to the POSIX flag groups |
| 2 | `02-v8-host-toolset-linux.diff` | `tools/v8_gypfiles/v8.gyp` | Do not apply `OS=hurd` to host tools (`torque`, `mksnapshot`, `node_js2c`) |
| 3 | `03-libuv-hurd-sources.diff` | `deps/uv/uv.gyp`, `deps/uv/src/unix/hurd.c` | Build the vendored Hurd backend; fix `uv_cpu_info()` model string |
| 4 | `04-cares-hurd-config.diff` | `deps/cares/cares.gyp`, `deps/cares/config/hurd/ares_config.h` | New c-ares config derived from Linux, epoll/getrandom disabled |
| 5 | `05-abseil-hurd-macros.diff` | `absl/base/config.h` | `ABSL_HAVE_MMAP` and `ABSL_HAVE_PTHREAD_GETSCHEDPARAM` for `__GNU__` |
| 6 | `06-v8-hurd-platform.diff` | `deps/v8/include/v8config.h`, `platform-hurd.cc`, `v8.gyp` | `V8_OS_HURD` / `V8_OS_POSIX`, minimal platform file |
| 7 | `07-v8-trap-handler-hurd.diff` | `trap-handler/*`, `v8.gyp` | Use the existing POSIX handler with `ucontext.gregs` |
| 8 | `08-v8-libc-gaps.diff` | `deps/v8/src/base/platform/platform-posix.cc` | `PTHREAD_STACK_MIN` fallback; tolerate `madvise` `ENOSYS`/`EINVAL` |

## Diff notes

### 01 — build system

`configure.py` has a whitelist of accepted `--dest-os` values. Adding `hurd`
to that list is enough: `--dest-os=hurd` sets gyp's generator flavor, and the
make generator derives `OS=hurd` from the flavor. Android is different because
gyp deliberately maps the Android flavor to `OS=linux` for legacy reasons, so
only Android needs the explicit override in `configure_node`.

`common.gypi` adds Hurd to two POSIX flag lists, plus three Hurd-only settings:
`_GNU_SOURCE`, `PATH_MAX=4096`, and `-latomic`. The last two compensate for
Hurd libc and toolchain gaps rather than describing a platform feature.

### 02 — host toolset

`v8.gyp` applies `OS=hurd` to both the target and the host toolset. The host
toolset runs on the build machine, so it has to keep the Linux platform and
stack-trace implementation. This diff narrows that one condition to
`(OS=="hurd" and _toolset=="host")`, which only affects cross builds.

### 03 — libuv

`deps/uv/src/unix/hurd.c` already exists in the vendored libuv, but `uv.gyp`
has no `uv_sources_hurd` list, so it was never compiled. The diff adds the
source list and the `OS=="hurd"` branches, including host/target conditions
analogous to diff 2.

The Hurd branch only sets `_GNU_SOURCE`. `common.gypi` already provides it
globally for Hurd, and glibc's `_GNU_SOURCE` implies the POSIX and XOPEN
feature sets, so the branch does not repeat `_POSIX_C_SOURCE` or
`_XOPEN_SOURCE`; it stays for parity with the Linux branch and keeps the
vendored libuv target self-contained.

`uv_cpu_info()` on Hurd fills CPUs but leaves `model` unset; Node's
`os.cpus()` then reads a NULL pointer. The diff fills in a placeholder string
and handles allocation failure. Hurd does not expose per-CPU data yet, so
every CPU reports `GNU/Hurd CPU`.

### 04 — c-ares

c-ares requires an `ares_config.h` per platform and has no Hurd directory.
The new header is a copy of the Linux configuration with `HAVE_EPOLL` and
`HAVE_GETRANDOM` disabled, because Hurd does not provide those interfaces.

Most of the 573 lines are generated-style feature macros. The meaningful
content is the two feature decisions; the rest can be compared against
`config/linux/ares_config.h` rather than read line by line.

### 05 — Abseil

Abseil's `config.h` lists platforms that have `mmap()` and
`pthread_getschedparam()`. Hurd is POSIX and has both, but `__GNU__` is not in
either list, so `ABSL_HAVE_MMAP` stays undefined and `poison.cc` fails to
compile.

### 06 — V8 platform

`v8config.h` gains an `__GNU__` branch that defines `V8_OS_HURD` and
`V8_OS_POSIX`. `V8_TARGET_OS_LINUX` changes from an empty define to `1`,
because V8 code uses it inside `#if` expressions where an empty macro is
invalid.

`platform-hurd.cc` is deliberately small: POSIX supplies almost everything,
and V8 only needs four target hooks (`CreateTimezoneCache`,
`GetSharedLibraryAddresses`, `SignalCodeMovingGC`, `AdjustSchedulingParams`).
`v8.gyp` selects it for the target toolset, together with
`stack_trace_posix.cc`. Shared-library addresses are empty, which is enough
for a working runtime but limits profiler symbolization.

### 07 — trap handler

V8 supports the x64 WebAssembly trap handler on Linux, Windows, macOS and
FreeBSD, and otherwise emits `#error`. Hurd provides `SIGSEGV` and
`ucontext.gregs`, so the existing POSIX implementation works after adding Hurd
to the platform conditions and the gyp source list.

This is the largest behaviour change in the port. It turns on a signal handler
that was previously absent on Hurd. It was exercised by the runtime tests
listed in the README.

### 08 — libc gaps

Two small compatibility changes in `platform-posix.cc`:

- `PTHREAD_STACK_MIN` is not defined by Hurd's `<pthread.h>`; the fallback is
  16384 bytes.
- `madvise(MADV_DONTNEED)` is not implemented everywhere on Hurd and may
  return `ENOSYS` or `EINVAL`. Discarding pages is only an optimization, so
  those two errors are treated as success.

## Not part of these diffs

- Build automation and machine-specific paths. These are described in
  [`BUILD-NOTES.md`](BUILD-NOTES.md) but intentionally not shipped.
- Configure options that remove functionality: `--without-amaro`
  (TypeScript stripping), `--without-inspector`, `--without-corepack`,
  `--without-node-snapshot`, `--openssl-no-asm`. QUIC/HTTP3 is also not built.
  These affect what was tested, not what Hurd needs to compile. See
  [`RUNTIME-NOTES.md`](RUNTIME-NOTES.md) for the runtime consequences.
- The `-DV8_HAVE_TARGET_OS -DV8_TARGET_OS_LINUX=1` host define used to work
  around V8's pre-`v8config.h` checks during cross-compilation. It is a build
  environment workaround, documented in `BUILD-NOTES.md`.

## Verification

- Diffs apply with `git apply` and with `patch -p1` on a pristine
  `node-v24.22.2` tree.
- Resulting tree matches the source tree that was built and tested.
- Runtime tests: `fs`, `crypto`, `zlib`, `child_process`, `worker_threads`,
  `http`, `https`, `net`, `dns`, `Intl.DateTimeFormat`, `node:sqlite`,
  `fs.watchFile`. Known runtime gaps are documented in
  [`RUNTIME-NOTES.md`](RUNTIME-NOTES.md).

## Status of this port

There is no native GCC build yet. Debian packaging would need one.
`process.platform` currently reports `hurd`.
