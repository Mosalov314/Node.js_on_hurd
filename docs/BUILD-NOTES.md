# Build notes and pitfalls

Target: Node.js v24.22.2 for `x86_64-unknown-gnu`.

## Toolchain

There is no packaged cross GCC for Hurd in Debian/Ubuntu. This build uses
Clang's Hurd toolchain and `binutils-x86_64-gnu`:

```sh
clang   --target=x86_64-unknown-gnu --sysroot=$SYSROOT -fuse-ld=/usr/bin/x86_64-gnu-ld
clang++ --target=x86_64-unknown-gnu --sysroot=$SYSROOT -fuse-ld=/usr/bin/x86_64-gnu-ld
```

Host tools must stay Linux binaries:

```sh
CC_host=gcc
CXX_host=g++
LINK_host=g++
AR_host=ar
```

## Configure

```sh
./configure \
  --dest-os=hurd \
  --dest-cpu=x64 \
  --cross-compiling \
  --with-intl=small-icu \
  --without-node-snapshot \
  --without-amaro \
  --without-inspector \
  --without-corepack \
  --openssl-no-asm \
  --prefix=/opt/node
```

The `CC`/`CXX`/`AR` variables must be exported for every `make` call.

### Options that remove functionality

These are deliberate build choices, not Hurd limitations:

| Option | Effect |
|---|---|
| `--without-amaro` | no TypeScript stripping (`process.features.typescript=false`) |
| `--without-inspector` | no `--inspect` / inspector protocol |
| `--without-corepack` | no corepack |
| `--without-node-snapshot` | `cached_builtins=false`, slower startup |
| `--openssl-no-asm` | portable OpenSSL, slower crypto |

QUIC/HTTP3 is also not built: `process.versions.nghttp3` and `ngtcp2` are
empty, and `require('node:quic')` throws `ERR_UNKNOWN_BUILTIN_MODULE`.
Runtime consequences are documented in [`RUNTIME-NOTES.md`](RUNTIME-NOTES.md).

## Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| gyp does not know Hurd | `valid_os` lacks `hurd` | Add `hurd` to `valid_os`; gyp derives `OS=hurd` from `--dest-os` |
| C++20/pthread/`_GNU_SOURCE` missing | flags only added for Linux | Add `hurd` to the relevant gyp OS lists |
| `PATH_MAX` undeclared | Hurd glibc does not define it | `PATH_MAX=4096` |
| `PTHREAD_STACK_MIN` undeclared | not provided on Hurd | fallback to 16384 |
| Host tools compile Hurd sources | `OS=hurd` also applies to host toolset | use `_toolset=="host"`/`"target"` |
| V8 does not recognize Hurd | no `__GNU__` branch | add `V8_OS_HURD`/`V8_OS_POSIX` |
| `operator && has no right operand` | `V8_TARGET_OS_LINUX` is an empty macro | define it as `1`; pass host defines explicitly |
| Abseil `data` undeclared | `ABSL_HAVE_MMAP` does not include Hurd | add `__GNU__` |
| c-ares `ares_config.h` missing | no Hurd config | add `config/hurd/ares_config.h` |
| libuv Hurd sources not built | `uv.gyp` does not include `hurd.c` | add Hurd sources/conditions |
| `os.cpus()` crashes | `uv_cpu_info()` leaves model NULL | fill a model string |
| `CreateTimezoneCache` undefined | V8 only has a Linux platform file | add `platform-hurd.cc` |
| `madvise` aborts | V8 treats failure as fatal | tolerate `ENOSYS`/`EINVAL` |
| trap handler `#error` | V8 only supports Linux/macOS/BSD | add Hurd + `ucontext` registers |
| host link uses cross libc | exported `LD=x86_64-gnu-ld` | do not export `LD`; use clang `-fuse-ld` |
| `__atomic_compare_exchange` undefined | gyp only adds `-latomic` on Linux | add `-latomic` for Hurd |
| `\p{...}` regex fails | built with `--without-intl` | rebuild with `small-icu` |
| pnpm needs `node:sqlite` | built with `--without-sqlite` | rebuild with SQLite enabled |
| `fetch` fails with `ENOPROTOOPT` | undici reaches a socket option pfinet does not implement | use `http.get`/`https.get`, `curl` or `wget` |
| `fs.watch` returns `ENOSYS` | no file-event backend in this libuv configuration | use `fs.watchFile` or polling |
| non-English `Intl` falls back to English | `--with-intl=small-icu` only ships English data | rebuild with `--with-intl=full-icu` |
| `Intl.Segmenter.segment()` segfaults | ICU break-iterator integration, not diagnosed yet | do not use `Intl.Segmenter` |
| `node-gyp` cannot download headers | download path uses `fetch` | `node-gyp rebuild --nodedir=/opt/node` |
| addon link fails with `R_X86_64_PC32` | gyp does not add `-fPIC` for Hurd | add `cflags: [ "-fPIC" ]` |
| `node:quic` is missing | QUIC/HTTP3 not built in this configuration | expected; no workaround |

## Native addons

`node-gyp` needs two adjustments on Hurd:

```sh
node-gyp rebuild --nodedir=/opt/node
```

and in `binding.gyp`:

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [ "addon.cc" ],
      "cflags": [ "-fPIC" ]
    }
  ]
}
```

Without `--nodedir`, `node-gyp` tries to download headers through `fetch`,
which fails with `ENOPROTOOPT`. Without `-fPIC`, the link fails with
`R_X86_64_PC32 ... recompile with -fPIC` because gyp does not recognize
`hurd` as a platform. A test addon built and loaded successfully this way.

## Runtime issues

Build-time problems are in the table above. Runtime behaviour is documented
separately in [`RUNTIME-NOTES.md`](RUNTIME-NOTES.md).

## Cross-build-only workaround

`CPPFLAGS_host` contains:

```sh
-DV8_HAVE_TARGET_OS -DV8_TARGET_OS_LINUX=1
```

V8 sources test `V8_TARGET_OS_LINUX` before including `v8config.h`, and the host
toolset is a Linux toolset. This is a cross-build workaround, not part of the
patch series.
