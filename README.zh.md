# Node.js 24 on GNU/Hurd x86_64（中文说明）

这是把 **Node.js v24.22.2**（含 npm 11.19.1）交叉编译到
**GNU/Hurd x86_64** 的实验性补丁和构建记录。

仓库只包含 **write-up + 按组件拆分的 diff**，不包含构建脚本；完整工具链、
configure 参数和常见问题见 [`docs/BUILD-NOTES.md`](docs/BUILD-NOTES.md)，
每份 diff 的说明见 [`docs/CHANGES.md`](docs/CHANGES.md)。

实际运行中哪些功能可用、哪些不可用，见
[`docs/RUNTIME-NOTES.zh.md`](docs/RUNTIME-NOTES.zh.md)。

不是 Node.js 官方 port，也不是 Debian 官方包。

## 编译信息

- Host：Ubuntu 26.04 x86_64，Clang 21.1.8
- Target：`x86_64-unknown-gnu`（Debian GNU/Hurd 0.9，GNU-Mach 1.8）
- Node.js：**v24.22.2**
- npm：**11.19.1**

## 当前结果

已启用：

- ICU：**small-icu**
- SQLite：启用，`node:sqlite` 可用

未启用：

- Amaro（TypeScript 剥离）
- inspector
- corepack
- node snapshot
- OpenSSL 汇编
- QUIC / HTTP3

`small-icu` 只有英文数据，中文等非英文 locale 的日期、数字、排序会退化；
需要 `full-icu`。`Intl.Segmenter` 另有崩溃问题。具体见
[`docs/RUNTIME-NOTES.zh.md`](docs/RUNTIME-NOTES.zh.md)。

已测试：

- `fs`、`crypto`、`zlib`、`child_process`、`worker_threads`
- `http`、`https`、`net`、`dns`
- `Intl.DateTimeFormat`、Unicode property escape（仅英文数据）
- `node:sqlite`
- `fs.watchFile`

## 复现

补丁针对上游 `node-v24.22.2.tar.xz`：

```sh
tar -xf node-v24.22.2.tar.xz
cd node-v24.22.2
for p in /path/to/patches/*.diff; do
  patch -p1 < "$p"
done
```

或者用 Git：`git apply /path/to/patches/*.diff`。然后按
[`docs/BUILD-NOTES.md`](docs/BUILD-NOTES.md) 设置 Hurd sysroot、
Clang target、configure 参数和 host toolset。

## 补丁结构

8 份 diff，14 个文件（新增 2、修改 12）：

1. build：让 `configure.py` / `common.gypi` 认识 Hurd
2. v8：交叉编译时 host toolset 继续用 Linux
3. libuv：加入 Hurd 源文件并修复 `uv_cpu_info()`
4. c-ares：新增 Hurd 的 `ares_config.h`
5. Abseil：识别 Hurd 平台宏
6. v8：加入 `V8_OS_HURD` 和 `platform-hurd.cc`
7. v8：在 Hurd 上启用 POSIX trap handler
8. v8：兼容 Hurd libc 缺项（`PTHREAD_STACK_MIN`、`madvise`）

每份 diff 做了什么见 [`docs/CHANGES.md`](docs/CHANGES.md)，文件清单见
[`docs/PATCHES.md`](docs/PATCHES.md)。

## 运行时简述

完整说明和复现代码见
[`docs/RUNTIME-NOTES.zh.md`](docs/RUNTIME-NOTES.zh.md)。

两个比较严重的问题：

- **`fetch` / undici 不可用**：连 `127.0.0.1` 都返回 `ENOPROTOOPT`。
  `http.get()` / `https.get()`、`curl`、`wget` 正常。
- **`fs.watch` 返回 `ENOSYS`**：HMR、`nodemon`、Vite watch 模式不可用。
  `fs.watchFile`（轮询）可用。

原生扩展可以编译，但需要手动处理：

```sh
node-gyp rebuild --nodedir=/opt/node
```

并在 `binding.gyp` 里加 `cflags: [ "-fPIC" ]`，否则链接阶段会报
`R_X86_64_PC32 ... recompile with -fPIC`。原因是 gyp 还不认识 `hurd`。

## 其他限制

- 目前是 Clang 交叉编译，Debian 打包用的原生 GCC 还没跑
- `--without-amaro`、`--without-inspector`、`--without-corepack`、
  `--without-node-snapshot`、`--openssl-no-asm` 是主动关闭的构建选项
- 原生 addon 的整体生态（sharp、node-pty、koffi 等）没有系统性验证
- `process.platform` 是 `hurd`，只检查 `linux` 的包可能走错分支
- 这些 diff 只是阅读和复现用的，不是可直接提交的 patch queue（没有 DEP-3
  头和 `debian/patches/series`）
