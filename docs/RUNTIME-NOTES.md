# Runtime notes (GNU/Hurd)

What the cross-built **Node.js v24.22.2 Hurd** binary can and cannot do on
Debian GNU/Hurd x86_64, with workarounds where they exist.

Test environment:

```text
GNU debian 0.9
GNU-Mach 1.8+git20261028-up-amd64/Hurd-0.9 x86_64
Node.js v24.22.2
```

Build configuration (small-icu, SQLite, disabled components) is listed in
the README Status and [`BUILD-NOTES.md`](BUILD-NOTES.md). This document covers
runtime behaviour.

## Summary

| Feature | Status | Notes |
|---|---|---|
| `http` / `https` / `net` / `dns` | works | loopback `http.get()` returns correctly |
| `fetch` / undici | broken | fails with `ENOPROTOOPT`, even for `127.0.0.1` |
| `fs.watch` | broken | returns `ENOSYS` |
| `fs.watchFile` | works | `stat`-based polling |
| `Intl` | partial | small-icu contains only limited English data; non-English dates, numbers and relative time degrade |
| `Intl.Segmenter` | crashes | `segment()` triggers SIGSEGV (`JSSegments::Create`) |
| native addons | works with flags | needs `--nodedir` and `-fPIC` |

Build-time features (Amaro, inspector, corepack, snapshot, OpenSSL assembly,
QUIC/HTTP3) are listed in the README Status and [`BUILD-NOTES.md`](BUILD-NOTES.md).

## 1. `fetch` fails with `ENOPROTOOPT`

This is not a network problem; loopback fails the same way.

Reproduce:

```js
const http = require('http');

const srv = http.createServer((req, res) => res.end('pong'))
  .listen(0, '127.0.0.1', async () => {
    const port = srv.address().port;

    http.get(`http://127.0.0.1:${port}`, (res) => {
      let body = '';
      res.on('data', (chunk) => body += chunk);
      res.on('end', async () => {
        console.log('http.get:', body);
        try {
          const r = await fetch(`http://127.0.0.1:${port}`);
          console.log('fetch:', await r.text());
        } catch (e) {
          console.log('fetch error:', e.code, e.message,
                      e.cause && e.cause.code);
        }
        srv.close();
      });
    });
  });
```

Observed output:

```text
http.get: pong
fetch error: undefined fetch failed ENOPROTOOPT
```

The outer error is `TypeError: fetch failed`, with cause:

```text
connect ENOPROTOOPT 127.0.0.1:43804
```

Notes:

- `http.get()` works over the same stack, so this is not routing or a firewall
- undici's connect path reaches a socket option that Hurd's `pfinet` does not
  implement
- the exact option has not been identified; this section records the symptom
  and workarounds only

Impact:

- anything built on `fetch` / undici, including DSH's `web_fetch`
- `node-gyp` header downloads (same HTTP stack)
- npm packages that use `fetch` for local HTTP calls

Workarounds:

- use `http.get()` / `https.get()`
- shell out to `curl` or `wget` (both are present on Hurd)
- use `node-gyp --nodedir=/opt/node` to skip the header download

## 2. `Intl` only has small-icu data

The build uses `--with-intl=small-icu`, which carries limited English data.
Chinese formatters fall back to English patterns:

```text
                       Hurd (small-icu)        full-icu
zh date                2026 M01 15, Thu        2026年1月15日星期四
relative time          -2 d                    2天前
currency               CN¥ 1,234,567.89        ¥1,234,567.89
```

Reproduce:

```js
console.log(new Intl.DateTimeFormat('zh-CN', { dateStyle: 'full' })
  .format(new Date('2026-01-15')));
console.log(new Intl.RelativeTimeFormat('zh-CN').format(-2, 'day'));
console.log(new Intl.NumberFormat('zh-CN', {
  style: 'currency',
  currency: 'CNY',
}).format(1234567.89));
```

### Not just Chinese

small-icu is not missing only Chinese data; it is missing almost everything
except English. The same test on Hurd (small-icu) and a full-icu build:

| locale | supportedLocalesOf | small-icu date | full-icu date |
|---|---|---|---|
| en-US | `["en-US"]` | Jan 15, 2026 | Jan 15, 2026 |
| en-GB | `["en-GB"]` | Jan 15, 2026 | 15 Jan 2026 |
| zh-CN | `[]` | 2026 M01 15 | 2026年1月15日 |
| ja-JP | `[]` | 2026 M01 15 | 2026/01/15 |
| de-DE | `[]` | 2026 M01 15 | 15.01.2026 |
| ru-RU | `[]` | 2026 M01 15 | 15 янв. 2026 г. |
| ar-EG | `[]` | 2026 M01 15 | ١٥‏/٠١‏/٢٠٢٦ |

Dates use `dateStyle: 'medium'` and `timeZone: 'UTC'`. The full-icu column
comes from a separate full-icu build (Node v22 on Linux), not from this Hurd
machine; it is only a reference.

Conclusions:

- `supportedLocalesOf` returns `[]` for zh-CN, ja-JP, de-DE, ru-RU and ar-EG
- even en-GB loses its regional date pattern and falls back to the en-US
  `Jan 15, 2026`
- number and currency grouping also use root rules:

```text
de-DE:  1,234.5   -> full-icu: 1.234,5
fr-CA:  1,234.5   -> full-icu: 1 234,5
ar-EG:  1,234.5   -> full-icu: ١٬٢٣٤٫٥
```

- `Intl.PluralRules` still returns correct results for ru, pl and ar
  (e.g. `one / few / many` for ru); this is data that happens to be present
  and should not be relied on
- collation also falls back to root: for `sv-SE`, comparing `ä` with `z`
  returns `-1`, while full-icu returns `1` (Swedish sorts `ä` after `z`)

Options:

- rebuild with `--with-intl=full-icu` (not validated on Hurd in this
  repository)
- or supply external ICU data through `NODE_ICU_DATA` (also not validated)
- applications that require non-English dates, collation or currency
  formatting should not use the small-icu build

### `Intl.Segmenter` crashes

`Intl.Segmenter.segment()` triggers SIGSEGV on this build, in both the old and
new binaries:

```js
const s = new Intl.Segmenter('en-US', { granularity: 'word' });
[...s.segment('hello world')];
```

```text
Thread 2 received signal SIGSEGV, Segmentation fault.
0x0000000001072c2d in v8::internal::JSSegments::Create(...)
#0 JSSegments::Create
#1 Builtin_SegmenterPrototypeSegment
```

`en-US` with `granularity: 'grapheme'` crashes too, so this is not simply
"Chinese data is missing". Whether it is small-icu lacking break-iterator data
or another V8/Hurd problem has not been determined.

Workaround: do not use `Intl.Segmenter`; use regular expressions, per-code-point
splitting, or application-level tokenization.

## 3. `fs.watch` returns `ENOSYS`

```js
fs.watch('.', () => {});
// Error: ENOSYS: function not implemented, watch '.'
```

This libuv configuration has no inotify/kqueue-style file event backend on
Hurd.

Impact:

- Vite / webpack HMR and watch mode
- `nodemon`
- any library that calls `fs.watch()`

Workarounds:

- `fs.watchFile()` works (it polls with `stat`); verified with a file growing
  from 1 to 6 bytes
- chokidar: `usePolling: true`
- Vite: `server.watch.usePolling: true`
- nodemon: legacy/polling mode (`nodemon -L`)
- disable watch mode where polling is too expensive

## 4. Native addons: headers first, then `-fPIC`

There are two independent failures.

First, `node-gyp rebuild` tries to download headers through `fetch` and fails:

```text
node-gyp rebuild
# header download fails
```

Use the installed headers instead:

```sh
node-gyp rebuild --nodedir=/opt/node
```

That gets past the download, but linking fails:

```text
ld: relocation R_X86_64_PC32 ... can not be used when making a shared object;
    recompile with -fPIC
```

gyp does not know the `hurd` platform, so it does not add `-fPIC` to the addon
target. Add it in `binding.gyp`:

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

`cflags` alone was enough in testing; add `ldflags: [ "-fPIC" ]` if a
particular link still complains. The resulting 8.7 KB `.node` file compiled
and loaded successfully.

This is a workaround, not a complete fix. A proper port would teach gyp about
Hurd so shared-library and addon targets get `-fPIC` automatically.

## 5. Build configuration

small-icu, Amaro, inspector, corepack, node snapshot, OpenSSL assembly and
QUIC/HTTP3 are build-time choices, not Hurd limitations. The full list and
the configure flags are in the README Status and
[`BUILD-NOTES.md`](BUILD-NOTES.md). Their runtime consequences are covered
above (the `Intl` section) or are simply missing features.

## 6. `process.platform` is `hurd`

`process.platform` returns `hurd`. npm packages that only check for `linux`
may take a wrong branch or refuse to install. This is intentional, but it is a
compatibility risk for the ecosystem.

## What works

Verified on the Hurd VM:

- `fs`, `crypto`, `zlib`, `child_process`, `worker_threads`
- `http`, `https`, `net`, `dns`
- `Intl.DateTimeFormat`, Unicode property escapes (English data)
- `node:sqlite`
- `fs.watchFile`
- npm 11.19.1

npm note: running `/opt/node/bin/npm` directly may fail to find node because
the script shebang uses `/usr/bin/env node`; add `/opt/node/bin` to `PATH`, or
run `/opt/node/bin/node /opt/node/lib/node_modules/npm/bin/npm-cli.js`.

## Conclusion

The port runs most of Node's core. Two Hurd-level problems block production
use for now:

- HTTP clients built on `fetch` / undici
- development tooling that depends on `fs.watch` (HMR, nodemon, Vite watch)

Missing non-English `Intl` data, the `Intl.Segmenter` crash, and the disabled
Amaro, corepack, inspector and QUIC components are mostly consequences of
build options, not missing Hurd functionality. See the sections above.

Native addons work, but headers and `-fPIC` have to be handled manually.
