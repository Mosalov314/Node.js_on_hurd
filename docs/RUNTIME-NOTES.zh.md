# 运行时说明（GNU/Hurd）

这份文档记录 **Node.js v24.22.2 Hurd 交叉编译版** 在 Debian GNU/Hurd
x86_64 上实际能做什么、不能做什么，以及可用的规避方式。

测试环境：

```text
GNU debian 0.9
GNU-Mach 1.8+git20261028-up-amd64/Hurd-0.9 x86_64
Node.js v24.22.2
```

构建配置（small-icu、SQLite、未启用组件）见 README 的“当前结果”和
[`BUILD-NOTES.md`](BUILD-NOTES.md)。这份文档只讲运行时行为。

## 汇总

| 功能 | 状态 | 说明 |
|---|---|---|
| `http` / `https` / `net` / `dns` | 可用 | 回环 `http.get()` 正常 |
| `fetch` / undici | 不可用 | 连 `127.0.0.1` 都返回 `ENOPROTOOPT` |
| `fs.watch` | 不可用 | 返回 `ENOSYS` |
| `fs.watchFile` | 可用 | 基于 `stat` 轮询 |
| `Intl` | 部分可用 | small-icu 只有有限的英文数据；非英文 locale 的日期、数字、相对时间退化 |
| `Intl.Segmenter` | 崩溃 | `segment()` 触发 SIGSEGV（`JSSegments::Create`） |
| 原生扩展 | 可用，但要手动加参数 | 需要 `--nodedir` 和 `-fPIC` |

Amaro、inspector、corepack、snapshot、OpenSSL 汇编、QUIC/HTTP3 等构建期
功能见 README 的“当前结果”和 [`BUILD-NOTES.md`](BUILD-NOTES.md)。

## 1. `fetch` 不可用（undici / ENOPROTOOPT）

不是网络问题，回环也一样失败。

复现：

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

实际输出：

```text
http.get: pong
fetch error: undefined fetch failed ENOPROTOOPT
```

外层是 `TypeError: fetch failed`，cause 是：

```text
connect ENOPROTOOPT 127.0.0.1:43804
```

说明：

- `http.get()` 走同一套协议栈是正常的，所以不是网络、路由或防火墙问题
- undici 的 connect 路径触发了一个 Hurd `pfinet` 不支持的 socket 选项
- 具体是哪个选项目前没有定位到；这里只记录现象和规避方式

影响：

- 所有基于 `fetch` / undici 的功能，包括 DSH 的 `web_fetch`
- `node-gyp` 下载 headers（它也走这套 HTTP 栈）
- 一些用 `fetch` 做本地 HTTP 调用的 npm 包

替代方案：

- 用 `http.get()` / `https.get()`
- 调用外部的 `curl` 或 `wget`（Hurd 上都有）
- `node-gyp` 用 `--nodedir=/opt/node` 跳过 headers 下载

## 2. `Intl` 只有 small-icu 数据

构建参数是 `--with-intl=small-icu`，只带有限的英文数据。中文 formatter
会退化成英文格式：

```text
                       Hurd (small-icu)        full-icu
zh 日期                2026 M01 15, Thu        2026年1月15日星期四
相对时间               -2 d                    2天前
货币                   CN¥ 1,234,567.89        ¥1,234,567.89
```

复现：

```js
console.log(new Intl.DateTimeFormat('zh-CN', { dateStyle: 'full' })
  .format(new Date('2026-01-15')));
console.log(new Intl.RelativeTimeFormat('zh-CN').format(-2, 'day'));
console.log(new Intl.NumberFormat('zh-CN', {
  style: 'currency',
  currency: 'CNY',
}).format(1234567.89));
```

### 不止中文

`small-icu` 不是“只有中文缺”，而是除了英文之外基本都缺。用同一份测试
分别在 Hurd（small-icu）和 full-icu 上跑：

| locale | supportedLocalesOf | small-icu 日期 | full-icu 日期 |
|---|---|---|---|
| en-US | `["en-US"]` | Jan 15, 2026 | Jan 15, 2026 |
| en-GB | `["en-GB"]` | Jan 15, 2026 | 15 Jan 2026 |
| zh-CN | `[]` | 2026 M01 15 | 2026年1月15日 |
| ja-JP | `[]` | 2026 M01 15 | 2026/01/15 |
| de-DE | `[]` | 2026 M01 15 | 15.01.2026 |
| ru-RU | `[]` | 2026 M01 15 | 15 янв. 2026 г. |
| ar-EG | `[]` | 2026 M01 15 | ١٥‏/٠١‏/٢٠٢٦ |

日期用的是 `dateStyle: 'medium'`、`timeZone: 'UTC'`。full-icu 列来自另一套
full-icu 构建（Linux 上的 Node v22），不是这台 Hurd 的输出，只作为对照。

几个结论：

- `supportedLocalesOf` 对 zh-CN、ja-JP、de-DE、ru-RU、ar-EG 全部返回 `[]`
- 连 en-GB 的区域日期模式也没有，退化成 en-US 的 `Jan 15, 2026`
- 数字和货币分组也走 root 规则：

```text
de-DE:  1,234.5   -> full-icu: 1.234,5
fr-CA:  1,234.5   -> full-icu: 1 234,5
ar-EG:  1,234.5   -> full-icu: ١٬٢٣٤٫٥
```

- `Intl.PluralRules` 在 ru、pl、ar 上仍然能给出正确结果（例如 ru 的
  `one / few / many`），这是 ICU 数据里恰好保留的部分，不建议依赖
- 排序规则也退化成 root：`sv-SE` 下 `ä` 和 `z` 比较返回 `-1`，
  full-icu 返回 `1`（瑞典语里 `ä` 排在 `z` 之后）

可选方案：

- 重新构建时使用 `--with-intl=full-icu`（本仓库没有在 Hurd 上验证过）
- 或者通过 `NODE_ICU_DATA` 提供外部 ICU 数据（同样没有验证）
- 对中文等非英文日期、排序、货币格式有硬性要求的应用，不要使用
  small-icu 构建

### `Intl.Segmenter` 会直接崩溃

`Intl.Segmenter.segment()` 在这个构建上会触发 SIGSEGV，旧二进制和新二进制
都一样：

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

连 `en-US` 的 `granularity: 'grapheme'` 也会崩，所以这不是“只缺中文数据”
那么简单。具体是 small-icu 缺少 break iterator 数据，还是 V8/Hurd 上的
其他问题，目前没有定位。

规避：不要使用 `Intl.Segmenter`，改用正则、按字符切分或应用自己的分词
逻辑。

## 3. `fs.watch` 返回 `ENOSYS`

```js
fs.watch('.', () => {});
// Error: ENOSYS: function not implemented, watch '.'
```

Hurd 当前这套 libuv 配置没有 inotify / kqueue 类的文件事件后端。

影响：

- Vite / webpack 的 HMR 和 watch 模式
- `nodemon`
- 任何直接调用 `fs.watch()` 的库

规避方式：

- `fs.watchFile()` 可用，它基于 `stat` 轮询；实测文件从 1 字节变成 6 字节
  能收到事件
- chokidar：`usePolling: true`
- Vite：`server.watch.usePolling: true`
- nodemon：使用 legacy / polling 模式（`nodemon -L`）
- 轮询成本太高时，直接关闭 watch 模式

## 4. 原生扩展：先解决 headers，再解决 `-fPIC`

有两个独立的问题。

第一，`node-gyp rebuild` 会尝试下载 headers，而下载走 `fetch`，于是失败：

```text
node-gyp rebuild
# 下载 headers 失败
```

用已经安装的 headers：

```sh
node-gyp rebuild --nodedir=/opt/node
```

这样可以过下载这一步，但链接会失败：

```text
ld: relocation R_X86_64_PC32 ... can not be used when making a shared object;
    recompile with -fPIC
```

原因是 gyp 不认识 `hurd` 这个平台，没有给 addon target 加上 `-fPIC`。
在 `binding.gyp` 里手动加：

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

实测只加 `cflags` 就够了；如果某个链接仍然报错，再加
`ldflags: [ "-fPIC" ]`。生成的 8.7 KB `.node` 可以正常加载。

这是 workaround，不是完整修复。真正的修复需要让 gyp 认识 Hurd，并在
shared library / addon target 上自动加 `-fPIC`。

## 5. 构建配置

small-icu、Amaro、inspector、corepack、node snapshot、OpenSSL 汇编和
QUIC/HTTP3 都是构建时的选择，不是 Hurd 本身缺功能。完整列表和 configure
参数见 README 的“当前结果”和 [`BUILD-NOTES.md`](BUILD-NOTES.md)。
它们对运行时的影响已经在上面的 `Intl` 等章节说明，或者只是功能缺失。

## 6. `process.platform` 是 `hurd`

`process.platform` 返回 `hurd`。只检查 `linux` 的 npm 包可能走错分支，
或者在安装阶段直接拒绝。这个值是故意保留的，但对生态兼容性有影响。

## 已经验证可用的功能

在 Hurd VM 上实际跑过：

- `fs`、`crypto`、`zlib`、`child_process`、`worker_threads`
- `http`、`https`、`net`、`dns`
- `Intl.DateTimeFormat`、Unicode property escape（英文数据）
- `node:sqlite`
- `fs.watchFile`
- npm 11.19.1

npm 的小提示：如果直接运行 `/opt/node/bin/npm` 报找不到 node，是因为脚本
shebang 用的是 `/usr/bin/env node`；把 `/opt/node/bin` 加进 `PATH`，或者
用 `/opt/node/bin/node /opt/node/lib/node_modules/npm/bin/npm-cli.js`。

## 结论

这个移植已经能跑 Node 核心的大部分功能。目前有两个 Hurd 层面的严重问题
会影响投产：

- 依赖 `fetch` / undici 的 HTTP 客户端
- 依赖 `fs.watch` 的开发工具链（HMR、nodemon、Vite watch）

非英文 `Intl` 数据缺失、`Intl.Segmenter` 崩溃，以及 Amaro、corepack、
inspector、QUIC 等未启用项，更多是构建选项带来的限制，不是 Hurd 本身
缺功能；具体见上文。

原生扩展可以用，但需要手动处理 headers 和 `-fPIC`。
