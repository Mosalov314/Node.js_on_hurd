# Diff inventory

There are **8 plain unified diffs**, touching **14 files**: **2 new**,
**12 modified**. Apply them in numeric order.

| Diff | Files touched |
|---|---|
| `01-build-configure-gyp-hurd.diff` | `configure.py`, `common.gypi` |
| `02-v8-host-toolset-linux.diff` | `tools/v8_gypfiles/v8.gyp` |
| `03-libuv-hurd-sources.diff` | `deps/uv/uv.gyp`, `deps/uv/src/unix/hurd.c` |
| `04-cares-hurd-config.diff` | `deps/cares/cares.gyp`, `deps/cares/config/hurd/ares_config.h` *(new)* |
| `05-abseil-hurd-macros.diff` | `deps/v8/third_party/abseil-cpp/absl/base/config.h` |
| `06-v8-hurd-platform.diff` | `deps/v8/include/v8config.h`, `deps/v8/src/base/platform/platform-hurd.cc` *(new)*, `tools/v8_gypfiles/v8.gyp` |
| `07-v8-trap-handler-hurd.diff` | `deps/v8/src/trap-handler/trap-handler.h`, `handler-inside-posix.h`, `handler-inside-posix.cc`, `tools/v8_gypfiles/v8.gyp` |
| `08-v8-libc-gaps.diff` | `deps/v8/src/base/platform/platform-posix.cc` |

## New files (2)

| File | Purpose |
|---|---|
| `deps/v8/src/base/platform/platform-hurd.cc` | Hurd platform hooks: `CreateTimezoneCache`, `GetSharedLibraryAddresses`, `SignalCodeMovingGC`, `AdjustSchedulingParams` |
| `deps/cares/config/hurd/ares_config.h` | c-ares configuration for Hurd, derived from `config/linux` with epoll and getrandom disabled |

## Modified files (12)

### Build system (3)

| File | Change |
|---|---|
| `configure.py` | Accept `--dest-os=hurd`; gyp derives `OS=hurd` from the generator flavor |
| `common.gypi` | Hurd compile flags/defines: `gnu++20`, pthread, `_GNU_SOURCE`, `PATH_MAX`, `-latomic` |
| `tools/v8_gypfiles/v8.gyp` | Host/target toolset separation; Hurd platform, stack trace, trap handler sources |

### libuv (2)

| File | Change |
|---|---|
| `deps/uv/uv.gyp` | Add Hurd sources: `hurd.c`, `posix-poll.c`, `bsd-ifaddrs.c`, etc. |
| `deps/uv/src/unix/hurd.c` | Fill `uv_cpu_info()` model string to avoid NULL deref in `os.cpus()` |

### V8 (5)

| File | Change |
|---|---|
| `deps/v8/include/v8config.h` | Detect `__GNU__` as `V8_OS_HURD` / `V8_OS_POSIX`; define `V8_TARGET_OS_LINUX` as `1` |
| `deps/v8/src/base/platform/platform-posix.cc` | `PTHREAD_STACK_MIN` fallback; tolerate `madvise` `ENOSYS`/`EINVAL` |
| `deps/v8/src/trap-handler/trap-handler.h` | Enable trap handler for Hurd |
| `deps/v8/src/trap-handler/handler-inside-posix.h` | Use `SIGSEGV` on Hurd |
| `deps/v8/src/trap-handler/handler-inside-posix.cc` | Read registers through `ucontext.gregs` on Hurd |

### Third-party (2)

| File | Change |
|---|---|
| `deps/v8/third_party/abseil-cpp/absl/base/config.h` | Enable `ABSL_HAVE_MMAP` and `ABSL_HAVE_PTHREAD_GETSCHEDPARAM` for Hurd |
| `deps/cares/cares.gyp` | Add Hurd config branch using `config/hurd/ares_config.h` |

See [`CHANGES.md`](CHANGES.md) for what each diff does.
