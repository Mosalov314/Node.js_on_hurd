# Node.js 24 for GNU/Hurd x86_64

Experimental port notes and plain diffs for cross-compiling **Node.js v24.22.2**
(including npm 11.19.1) for **GNU/Hurd x86_64**.

This is an experience write-up plus a set of plain diffs. Build automation is intentionally not
included because the local toolchain paths and sysroot are machine-specific.
For the full toolchain assumptions, manual configure command, and pitfalls,
see [`docs/BUILD-NOTES.md`](docs/BUILD-NOTES.md).

This is not an official Node.js or Debian port.

## Status

- Node.js: **v24.22.2**
- npm: **11.19.1**
- Target: **x86_64-unknown-gnu**
- Build host: Ubuntu 26.04 x86_64, Clang 21.1.8
- ICU: **small-icu**
- SQLite: **enabled** (`node:sqlite` works)
- Amaro: **disabled** (`process.features.typescript` is false)
- inspector: **disabled** (`process.features.inspector` is false)
- corepack: **disabled**
- node snapshot: **disabled** (`process.features.cached_builtins` is false)
- OpenSSL assembly: **disabled**
- QUIC / HTTP3: **not built**
- Native GCC build: **not yet validated**
- Debian package: not built; Debian Hurd currently ships only `nodejs-doc`

Basic runtime tests have passed on Debian GNU/Hurd x86_64:

- `fs`, `crypto`, `zlib`, `child_process`, `worker_threads`
- `http`, `https`, `net`, `dns`
- `Intl.DateTimeFormat`, Unicode property escapes (English data)
- `node:sqlite`
- `fs.watchFile`

## Repository layout

```text
.
├── README.md
├── README.zh.md
├── patches/                        # 8 plain unified diffs, apply in order
│   ├── 01-build-configure-gyp-hurd.diff
│   ├── ...
│   └── 08-v8-libc-gaps.diff
└── docs/
    ├── BUILD-NOTES.md              # toolchain, configure, pitfalls
    ├── CHANGES.md                  # what each diff does
    ├── PATCHES.md                  # diff/file inventory
    ├── RUNTIME-NOTES.md            # what works at runtime
    └── RUNTIME-NOTES.zh.md         # Chinese version
```

## Reproducing

The diffs apply to the upstream `node-v24.22.2.tar.xz` source tree:

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

Then follow [`docs/BUILD-NOTES.md`](docs/BUILD-NOTES.md) for the Hurd sysroot,
Clang target, configure options, and host toolset variables.

## What the diffs change

8 diffs, 14 files: **2 new**, **12 modified**. Each diff does one thing:

1. build: teach `configure.py` and `common.gypi` about Hurd
2. v8: keep the host toolset on Linux while targeting Hurd
3. libuv: add the Hurd source set and fix `uv_cpu_info()`
4. c-ares: add a Hurd `ares_config.h`
5. Abseil: recognize Hurd platform macros
6. v8: add `V8_OS_HURD` and `platform-hurd.cc`
7. v8: enable the POSIX trap handler on Hurd
8. v8: tolerate Hurd libc gaps (`PTHREAD_STACK_MIN`, `madvise`)

See [`docs/CHANGES.md`](docs/CHANGES.md) for what each diff does,
and [`docs/PATCHES.md`](docs/PATCHES.md) for the file inventory.

## Runtime notes

See [`docs/RUNTIME-NOTES.md`](docs/RUNTIME-NOTES.md) for details and
reproduction scripts.

Two serious problems:

- **`fetch` / undici is broken**: even `127.0.0.1` returns `ENOPROTOOPT`.
  `http.get()` / `https.get()`, `curl` and `wget` work.
- **`fs.watch` returns `ENOSYS`**: HMR, `nodemon` and Vite watch mode do not
  work. `fs.watchFile()` (polling) works.

Native addons compile, but need manual flags:

```sh
node-gyp rebuild --nodedir=/opt/node
```

and `cflags: [ "-fPIC" ]` in `binding.gyp`, otherwise linking fails with
`R_X86_64_PC32 ... recompile with -fPIC`. gyp does not recognize `hurd` yet.

`small-icu` only contains English data; non-English `Intl` dates, numbers and
collation fall back. `Intl.Segmenter` crashes. See the runtime notes for
details.

## Known limitations

- The build is cross-compiled with Clang; a native GCC build on Hurd is still
  needed before this can be considered for Debian packaging.
- `--without-amaro`, `--without-inspector`, `--without-corepack`,
  `--without-node-snapshot` and `--openssl-no-asm` are deliberate build
  options.
- Native addon ecosystem coverage (`sharp`, `node-pty`, `koffi`, Playwright,
  etc.) has not been systematically verified.
- `process.platform` reports `hurd`, which may affect packages that only check
  for `linux`.
- The diffs are a reading aid and a reproduction aid, not a submission-ready
  patch queue; there are no DEP-3 headers or `debian/patches/series` yet.

## License

The diffs apply to Node.js, which is MIT-licensed. This repository's notes
are provided under the same terms.
