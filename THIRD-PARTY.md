# 第三方依赖与许可清单

目标：让「对外发」这件事在**许可合规上站得住**，而且是**可复核**的 ——
一张从本机生成的 Rust 依赖/许可清单、一段明确的「有没有 copyleft 风险」结论、
随包第三方资产的核对结果，以及发布前能一条条重跑的复核命令。

本文件的依赖表**不是手抄的**：从本仓 `Cargo.lock` + 本机 cargo registry 缓存生成（确切命令见第 1 节），
仓库自己的包（`i-trove`）不计入。内核许可 = Apache-2.0，全文见 [`LICENSE`](../LICENSE)，声明见 [`NOTICE.md`](../NOTICE.md)。
本文只覆盖**内核（Rust）**这一侧；界面侧（`i-trove-ui`，另一个仓库，fork 自 `zai-org/ZCode`）的 npm 依赖不在这里，
那部分随包在分发目录的 `licenses\` 里（第 5 节核对）。

> 口径：「许可证」一列是 crate manifest 里 `license` 字段的**原样字符串**（SPDX 表达式）。
> `A OR B` = 二选一（发二进制时**选最宽松的那支**即可），`A AND B` = 两支都得满足。
> 判断能不能发，看的是**选定的那支**，不是整串字符串。

---

## 0. 结论先行

- **发布产物里没有 GPL / AGPL / SSPL / CC-BY-NC /「未声明」这类会挡住分发的东西。**
- 唯一「会编进 Windows 二进制、又不是标准宽松许可」的是 **`option-ext` 0.2.0（MPL-2.0，弱 copyleft，文件级）**。
  发二进制没问题，但按 MPL-2.0 §3.2 要**告诉接收者从哪里能拿到对应源码**（4.1 给结论、第 6 节给做法）。
- `self_cell`（`Apache-2.0 OR GPL-2.0-only`）和 `r-efi`（`MIT OR Apache-2.0 OR LGPL-2.1-or-later`）是**多许可**里各带了一个 copyleft 分支：
  我们**选宽松那支**（`Apache-2.0` / `MIT`），copyleft 分支不适用（4.1）。
- 6 个 `MPL-2.0` 包里只有 `option-ext` 会进产物；其余 5 个（`cssparser` / `cssparser-macros` / `selectors` / `thin-slice` / `dtoa-short`）
  走 `wry → kuchikiki` 的 `cfg(target_os = "android")` 分支，**Windows 构建根本不编**（4.1）。
- 2 个包本地取不到许可证（`arbitrary` / `derive_arbitrary`）：它们是 `zip` 的 `cfg(fuzzing)` 依赖，**正常构建不编译**（4.3）。
- 随包 `licenses\` 目录核对结果：3 个文件与界面仓库原件**逐字节一致**；但**内核自己的 `LICENSE` / `NOTICE.md` 没随包**、
  **Node 运行时的 `LICENSE` 也没随包** —— 见第 5 节的「建议改法」。

---

## 1. 怎么重新生成这张表（Rust 依赖）

### 1.1 首选：一条 PowerShell 管道（`cargo metadata`）

前提：本地 registry 缓存够用（至少跑过一次 `cargo fetch` 或 `cargo build --release`）。
`source` 非空的才是第三方 crate；本仓自己的包 `source` 为空，正好被滤掉。

```powershell
# 完整依赖表（名字 / 版本 / 许可证 / 上游链接）
cargo metadata --format-version 1 --offline |
  ConvertFrom-Json |
  ForEach-Object { $_.packages } |
  Where-Object { $_.source } |
  Sort-Object name, version |
  ForEach-Object { "| ``$($_.name)`` | $($_.version) | $($_.license) | $($_.repository) |" }

# 许可证分布统计
cargo metadata --format-version 1 --offline |
  ConvertFrom-Json |
  ForEach-Object { $_.packages } |
  Where-Object { $_.source } |
  Group-Object license |
  Sort-Object Count -Descending |
  Format-Table Count, Name -AutoSize
```

### 1.2 本机的实际情况（诚实记录，别以为照抄 1.1 就一定能跑通）

`cargo metadata --format-version 1 --offline` 在本机**直接失败**：

```
error: failed to download `arbitrary v1.4.2`
Caused by:
  attempting to make an HTTP request, but --offline was specified
```

原因：`arbitrary 1.4.2` / `derive_arbitrary 1.4.2` 的 `.crate` **不在本地缓存**里
（它们是 `zip` 的 `cfg(fuzzing)` 依赖，正常构建从不去下），而 `cargo metadata` 要读每个包的 manifest，
于是要联网抓这两份 → 加了 `--offline` 就报错。去掉 `--offline` 会**联网**，本任务不允许。

所以**本文件的表用下面这段纯离线脚本生成**：数据源和 cargo 一致 ——
`Cargo.lock` 的包集合 × `~/.cargo/registry/src/index.crates.io-*/<name>-<version>/Cargo.toml` 里 `[package]` 段的
`license` / `repository`。不联网、不写文件。**在本仓根目录跑。**

```powershell
$srcRoot = Join-Path $env:USERPROFILE '.cargo\registry\src\index.crates.io-1949cf8c6b5b557f'
$pkgs = @(); $cur = $null
foreach ($l in [IO.File]::ReadAllLines('Cargo.lock')) {
  if ($l -eq '[[package]]') { if ($cur) { $pkgs += $cur }; $cur = @{n='';v='';s=''}; continue }
  if ($null -eq $cur) { continue }
  if     ($l -match '^name = "(.*)"$')    { $cur.n = $Matches[1] }
  elseif ($l -match '^version = "(.*)"$') { if (-not $cur.v) { $cur.v = $Matches[1] } }
  elseif ($l -match '^source = "(.*)"$')  { $cur.s = $Matches[1] }
  elseif ($l -match '^\[')                { $pkgs += $cur; $cur = $null }
}
if ($cur) { $pkgs += $cur }
$rows = foreach ($p in $pkgs) {
  if ($p.s) {                                    # 有 source = 第三方；本仓的包没有 source
    $toml = Join-Path $srcRoot ("{0}-{1}\Cargo.toml" -f $p.n, $p.v)
    $lic = ''; $repo = ''
    if (Test-Path -LiteralPath $toml) {
      $inPkg = $false
      foreach ($t in [IO.File]::ReadAllLines($toml)) {
        if ($t -match '^\s*\[([^\]]+)\]') { $inPkg = ($Matches[1] -eq 'package'); continue }
        if (-not $inPkg) { continue }
        if     ($t -match '^license\s*=\s*"(.*)"')    { $lic  = $Matches[1] }
        elseif ($t -match '^repository\s*=\s*"(.*)"') { $repo = $Matches[1] }
      }
    }
    [pscustomobject]@{ name=$p.n; version=$p.v; license=$lic; repository=$repo }
  }
}
$rows | Sort-Object name, version |
  ForEach-Object { "| ``$($_.name)`` | $($_.version) | $($_.license) | $($_.repository) |" }
```

**与 `cargo metadata` 的差异**：本机这套离线脚本覆盖 lock 里 **732 条第三方**中的 **730 条**
（本地有 manifest）；缺的 2 条 `arbitrary` / `derive_arbitrary` 在表里标成「未声明」。
缓存齐全时 `cargo metadata` 会把这 2 条也补上 —— 预期值是 `MIT OR Apache-2.0`，
但**本机拿不到，所以本文不写、不猜**（见 4.3）。

> 额外说明：`crates/i-trove/Cargo.toml` 的 `version` 走 `workspace.package` = `0.2.0`，
> 而提交在库里的 `Cargo.lock` 仍写着 `i-trove 0.1.3`。本文只统计第三方包，这处 lock/manifest 版本差
> **与本次清单无关**；但若 `cargo metadata` 报「lock 需要更新」，那是这条引起的。

---

## 2. 许可证分布统计（732 条第三方依赖）

原始分组统计（`Group-Object license` 输出，原样）：

```
327  MIT OR Apache-2.0
185  MIT
 52  Apache-2.0 OR MIT
 31  MIT/Apache-2.0
 21  Apache-2.0
 18  Unicode-3.0
 16  Zlib OR Apache-2.0 OR MIT
 12  Unlicense OR MIT
  8  Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT
  7  MIT OR Apache-2.0 OR Zlib
  6  MPL-2.0
  5  BSD-3-Clause
  4  Apache-2.0/MIT
  3  ISC
  3  Zlib
  2  MIT OR Zlib OR Apache-2.0
  2  Unlicense/MIT
  2  Apache-2.0 OR ISC OR MIT
  2  BSD-3-Clause OR Apache-2.0
  2  BSD-3-Clause OR MIT OR Apache-2.0
  2  MIT OR Apache-2.0 OR LGPL-2.1-or-later
  2  BSL-1.0
  2  MIT / Apache-2.0
  2  （未取到 manifest → 未声明）
  2  BSD-2-Clause OR Apache-2.0 OR MIT
  1  Apache-2.0 OR BSL-1.0
  1  0BSD OR MIT OR Apache-2.0
  1  CDLA-Permissive-2.0
  1  MIT OR Apache-2.0 OR BSD-1-Clause
  1  (MIT OR Apache-2.0) AND Unicode-3.0
  1  Apache-2.0 OR GPL-2.0-only
  1  Apache-2.0 WITH LLVM-exception
  1  CC0-1.0 OR MIT-0 OR Apache-2.0
  1  (MIT OR Apache-2.0) AND OFL-1.1 AND Ubuntu-font-1.0
  1  CC0-1.0
  1  Apache-2.0 AND MIT
  1  Apache-2.0 AND ISC
  1  BSD-2-Clause
  1  MIT AND BSD-3-Clause
```

几个直接结论：

- **690 条**（约 94%）的许可证串里提到 `MIT` 和/或 `Apache-2.0`（`MIT OR Apache-2.0` 这种双许可是 Rust 生态的默认惯例）。
- **706 条**落在「纯标准宽松许可」里（不涉及 MPL / GPL / LGPL / OFL / CDLA / LLVM-exception / BSD-1-Clause / BSL）。
- 许可证串里出现 `GPL` / `LGPL` / `MPL` 字样的只有 **9 条**（4.1 逐条给结论）；`AGPL` / `SSPL` / `CC-BY-NC` **0 条**。
- **2 条**取不到 `license` 字段（4.3）。

---

## 3. 完整依赖表（732 条，按 crate 名排序）

表就是 1.2 那段脚本的原样输出；`upstream` 是 manifest 的 `repository` 字段（取不到时写 `-`；
没 manifest 时许可证写「未声明」）。**完整、无抽样。**

<details>
<summary>点开：732 条第三方依赖（名字 / 版本 / 许可证 / 上游链接）</summary>

| crate | version | license | upstream |
|---|---|---|---|
| `ab_glyph` | 0.2.32 | Apache-2.0 | https://github.com/alexheretic/ab-glyph |
| `ab_glyph_rasterizer` | 0.1.10 | Apache-2.0 | https://github.com/alexheretic/ab-glyph |
| `accesskit` | 0.19.0 | MIT OR Apache-2.0 | https://github.com/AccessKit/accesskit |
| `accesskit_atspi_common` | 0.12.0 | MIT OR Apache-2.0 | https://github.com/AccessKit/accesskit |
| `accesskit_consumer` | 0.28.0 | MIT OR Apache-2.0 | https://github.com/AccessKit/accesskit |
| `accesskit_macos` | 0.20.0 | MIT OR Apache-2.0 | https://github.com/AccessKit/accesskit |
| `accesskit_unix` | 0.15.0 | MIT OR Apache-2.0 | https://github.com/AccessKit/accesskit |
| `accesskit_windows` | 0.27.0 | MIT OR Apache-2.0 | https://github.com/AccessKit/accesskit |
| `accesskit_winit` | 0.27.0 | Apache-2.0 | https://github.com/AccessKit/accesskit |
| `adler2` | 2.0.1 | 0BSD OR MIT OR Apache-2.0 | https://github.com/oyvindln/adler2 |
| `aead` | 0.5.2 | MIT OR Apache-2.0 | https://github.com/RustCrypto/traits |
| `age` | 0.10.1 | MIT OR Apache-2.0 | https://github.com/str4d/rage |
| `age-core` | 0.10.0 | MIT OR Apache-2.0 | https://github.com/str4d/rage |
| `agent-client-protocol` | 2.2.0 | Apache-2.0 | https://github.com/agentclientprotocol/rust-sdk |
| `agent-client-protocol-derive` | 2.2.0 | Apache-2.0 | https://github.com/agentclientprotocol/rust-sdk |
| `agent-client-protocol-schema` | 1.9.1 | Apache-2.0 | https://github.com/agentclientprotocol/agent-client-protocol |
| `ahash` | 0.8.12 | MIT OR Apache-2.0 | https://github.com/tkaitchuck/ahash |
| `aho-corasick` | 1.1.5 | Unlicense OR MIT | https://github.com/BurntSushi/aho-corasick |
| `android_system_properties` | 0.1.6 | MIT OR Apache-2.0 | https://github.com/nical/android_system_properties |
| `android-activity` | 0.6.1 | MIT OR Apache-2.0 | https://github.com/rust-mobile/android-activity |
| `android-properties` | 0.2.2 | MIT | https://github.com/miklelappo/android-properties |
| `anstream` | 1.0.0 | MIT OR Apache-2.0 | https://github.com/rust-cli/anstyle.git |
| `anstyle` | 1.0.14 | MIT OR Apache-2.0 | https://github.com/rust-cli/anstyle.git |
| `anstyle-parse` | 1.0.0 | MIT OR Apache-2.0 | https://github.com/rust-cli/anstyle.git |
| `anstyle-query` | 1.1.5 | MIT OR Apache-2.0 | https://github.com/rust-cli/anstyle.git |
| `anstyle-wincon` | 3.0.11 | MIT OR Apache-2.0 | https://github.com/rust-cli/anstyle.git |
| `anyhow` | 1.0.104 | MIT OR Apache-2.0 | https://github.com/dtolnay/anyhow |
| `arbitrary` | 1.4.2 | 未声明（本地缓存没有该包的 manifest） | - |
| `arboard` | 3.6.1 | MIT OR Apache-2.0 | https://github.com/1Password/arboard |
| `arc-swap` | 1.9.2 | MIT OR Apache-2.0 | https://github.com/vorner/arc-swap |
| `arrayref` | 0.3.9 | BSD-2-Clause | https://github.com/droundy/arrayref |
| `arrayvec` | 0.7.8 | MIT OR Apache-2.0 | https://github.com/bluss/arrayvec |
| `ash` | 0.38.0+1.3.281 | MIT OR Apache-2.0 | https://github.com/ash-rs/ash |
| `as-raw-xcb-connection` | 1.0.1 | MIT OR Apache-2.0 | https://github.com/psychon/as-raw-xcb-connection |
| `async-broadcast` | 0.7.2 | MIT OR Apache-2.0 | https://github.com/smol-rs/async-broadcast |
| `async-channel` | 2.5.0 | Apache-2.0 OR MIT | https://github.com/smol-rs/async-channel |
| `async-executor` | 1.14.0 | Apache-2.0 OR MIT | https://github.com/smol-rs/async-executor |
| `async-io` | 2.6.0 | Apache-2.0 OR MIT | https://github.com/smol-rs/async-io |
| `async-lock` | 3.4.2 | Apache-2.0 OR MIT | https://github.com/smol-rs/async-lock |
| `async-process` | 2.5.0 | Apache-2.0 OR MIT | https://github.com/smol-rs/async-process |
| `async-recursion` | 1.1.1 | MIT OR Apache-2.0 | https://github.com/dcchut/async-recursion |
| `async-signal` | 0.2.14 | Apache-2.0 OR MIT | https://github.com/smol-rs/async-signal |
| `async-task` | 4.7.1 | Apache-2.0 OR MIT | https://github.com/smol-rs/async-task |
| `async-trait` | 0.1.92 | MIT OR Apache-2.0 | https://github.com/dtolnay/async-trait |
| `atk` | 0.18.2 | MIT | https://github.com/gtk-rs/gtk3-rs |
| `atk-sys` | 0.18.2 | MIT | https://github.com/gtk-rs/gtk3-rs |
| `atomic-waker` | 1.1.2 | Apache-2.0 OR MIT | https://github.com/smol-rs/atomic-waker |
| `atspi` | 0.25.0 | Apache-2.0 OR MIT | https://github.com/odilia-app/atspi |
| `atspi-common` | 0.9.0 | Apache-2.0 OR MIT | https://github.com/odilia-app/atspi |
| `atspi-connection` | 0.9.0 | Apache-2.0 OR MIT | https://github.com/odilia-app/atspi/ |
| `atspi-proxies` | 0.9.0 | Apache-2.0 OR MIT | https://github.com/odilia-app/atspi |
| `autocfg` | 1.5.1 | Apache-2.0 OR MIT | https://github.com/cuviper/autocfg |
| `axum` | 0.8.9 | MIT | https://github.com/tokio-rs/axum |
| `axum-core` | 0.5.6 | MIT | https://github.com/tokio-rs/axum |
| `base64` | 0.21.7 | MIT OR Apache-2.0 | https://github.com/marshallpierce/rust-base64 |
| `base64` | 0.22.1 | MIT OR Apache-2.0 | https://github.com/marshallpierce/rust-base64 |
| `base64` | 0.23.1 | MIT OR Apache-2.0 | https://github.com/marshallpierce/rust-base64 |
| `basic-toml` | 0.1.10 | MIT OR Apache-2.0 | https://github.com/dtolnay/basic-toml |
| `bech32` | 0.9.1 | MIT | https://github.com/rust-bitcoin/rust-bech32 |
| `bitflags` | 1.3.2 | MIT/Apache-2.0 | https://github.com/bitflags/bitflags |
| `bitflags` | 2.13.2 | MIT OR Apache-2.0 | https://github.com/bitflags/bitflags |
| `bit-set` | 0.8.0 | Apache-2.0 OR MIT | https://github.com/contain-rs/bit-set |
| `bit-vec` | 0.8.0 | Apache-2.0 OR MIT | https://github.com/contain-rs/bit-vec |
| `block` | 0.1.6 | MIT | http://github.com/SSheldon/rust-block |
| `block2` | 0.5.1 | MIT | https://github.com/madsmtm/objc2 |
| `block2` | 0.6.2 | MIT | https://github.com/madsmtm/objc2 |
| `block-buffer` | 0.10.4 | MIT OR Apache-2.0 | https://github.com/RustCrypto/utils |
| `block-buffer` | 0.12.1 | MIT OR Apache-2.0 | https://github.com/RustCrypto/utils |
| `blocking` | 1.7.0 | Apache-2.0 OR MIT | https://github.com/smol-rs/blocking |
| `bs58` | 0.5.1 | MIT/Apache-2.0 | https://github.com/Nullus157/bs58-rs |
| `bstr` | 1.13.1 | MIT OR Apache-2.0 | https://github.com/BurntSushi/bstr |
| `bumpalo` | 3.20.3 | MIT OR Apache-2.0 | https://github.com/fitzgen/bumpalo |
| `bytemuck` | 1.25.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/Lokathor/bytemuck |
| `bytemuck_derive` | 1.12.1 | Zlib OR Apache-2.0 OR MIT | https://github.com/Lokathor/bytemuck |
| `byteorder` | 1.5.0 | Unlicense OR MIT | https://github.com/BurntSushi/byteorder |
| `byteorder-lite` | 0.1.0 | Unlicense OR MIT | https://github.com/image-rs/byteorder-lite |
| `bytes` | 1.12.1 | MIT | https://github.com/tokio-rs/bytes |
| `cairo-rs` | 0.18.5 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `cairo-sys-rs` | 0.18.2 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `calloop` | 0.13.0 | MIT | https://github.com/Smithay/calloop |
| `calloop` | 0.14.4 | MIT | https://github.com/Smithay/calloop |
| `calloop-wayland-source` | 0.3.0 | MIT | https://github.com/smithay/calloop-wayland-source |
| `calloop-wayland-source` | 0.4.1 | MIT | https://github.com/smithay/calloop-wayland-source |
| `cc` | 1.4.7 | MIT OR Apache-2.0 | https://github.com/rust-lang/cc-rs |
| `cesu8` | 1.1.0 | Apache-2.0/MIT | https://github.com/emk/cesu8-rs |
| `cfg_aliases` | 0.2.2 | MIT | https://github.com/katharostech/cfg_aliases |
| `cfg-expr` | 0.15.8 | MIT OR Apache-2.0 | https://github.com/EmbarkStudios/cfg-expr |
| `cfg-if` | 1.0.5 | MIT OR Apache-2.0 | https://github.com/rust-lang/cfg-if |
| `cgl` | 0.3.2 | MIT / Apache-2.0 | https://github.com/servo/cgl-rs |
| `chacha20` | 0.10.2 | MIT OR Apache-2.0 | https://github.com/RustCrypto/stream-ciphers |
| `chacha20` | 0.9.1 | Apache-2.0 OR MIT | https://github.com/RustCrypto/stream-ciphers |
| `chacha20poly1305` | 0.10.1 | Apache-2.0 OR MIT | https://github.com/RustCrypto/AEADs/tree/master/chacha20poly1305 |
| `chrono` | 0.4.45 | MIT OR Apache-2.0 | https://github.com/chronotope/chrono |
| `cipher` | 0.4.4 | MIT OR Apache-2.0 | https://github.com/RustCrypto/traits |
| `clap` | 4.6.7 | MIT OR Apache-2.0 | https://github.com/clap-rs/clap |
| `clap_builder` | 4.6.7 | MIT OR Apache-2.0 | https://github.com/clap-rs/clap |
| `clap_derive` | 4.6.7 | MIT OR Apache-2.0 | https://github.com/clap-rs/clap |
| `clap_lex` | 1.1.1 | MIT OR Apache-2.0 | https://github.com/clap-rs/clap |
| `clipboard-win` | 5.4.1 | BSL-1.0 | https://github.com/DoumanAsh/clipboard-win |
| `codespan-reporting` | 0.12.0 | Apache-2.0 | https://github.com/brendanzab/codespan |
| `colorchoice` | 1.0.5 | MIT OR Apache-2.0 | https://github.com/rust-cli/anstyle.git |
| `combine` | 4.6.8 | MIT | https://github.com/Marwes/combine |
| `concurrent-queue` | 2.5.0 | Apache-2.0 OR MIT | https://github.com/smol-rs/concurrent-queue |
| `const-oid` | 0.10.2 | Apache-2.0 OR MIT | https://github.com/RustCrypto/formats |
| `convert_case` | 0.10.0 | MIT | https://github.com/rutrum/convert-case |
| `convert_case` | 0.4.0 | MIT | https://github.com/rutrum/convert-case |
| `cookie` | 0.18.2 | MIT OR Apache-2.0 | https://github.com/SergioBenitez/cookie-rs |
| `cookie-factory` | 0.3.3 | MIT | https://github.com/rust-bakery/cookie-factory |
| `core-foundation` | 0.10.1 | MIT OR Apache-2.0 | https://github.com/servo/core-foundation-rs |
| `core-foundation` | 0.9.4 | MIT OR Apache-2.0 | https://github.com/servo/core-foundation-rs |
| `core-foundation-sys` | 0.8.7 | MIT OR Apache-2.0 | https://github.com/servo/core-foundation-rs |
| `core-graphics` | 0.23.2 | MIT OR Apache-2.0 | https://github.com/servo/core-foundation-rs |
| `core-graphics` | 0.25.0 | MIT OR Apache-2.0 | https://github.com/servo/core-foundation-rs |
| `core-graphics-types` | 0.1.3 | MIT OR Apache-2.0 | https://github.com/servo/core-foundation-rs |
| `core-graphics-types` | 0.2.0 | MIT OR Apache-2.0 | https://github.com/servo/core-foundation-rs |
| `cpufeatures` | 0.2.17 | MIT OR Apache-2.0 | https://github.com/RustCrypto/utils |
| `cpufeatures` | 0.3.1 | MIT OR Apache-2.0 | https://github.com/RustCrypto/utils |
| `crc32fast` | 1.5.2 | MIT OR Apache-2.0 | https://github.com/srijs/rust-crc32fast |
| `crossbeam-channel` | 0.5.17 | MIT OR Apache-2.0 | https://github.com/crossbeam-rs/crossbeam |
| `crossbeam-utils` | 0.8.23 | MIT OR Apache-2.0 | https://github.com/crossbeam-rs/crossbeam |
| `crunchy` | 0.2.4 | MIT | https://github.com/eira-fransham/crunchy |
| `crypto-common` | 0.1.7 | MIT OR Apache-2.0 | https://github.com/RustCrypto/traits |
| `crypto-common` | 0.2.2 | MIT OR Apache-2.0 | https://github.com/RustCrypto/traits |
| `cssparser` | 0.27.2 | MPL-2.0 | https://github.com/servo/rust-cssparser |
| `cssparser-macros` | 0.6.1 | MPL-2.0 | https://github.com/servo/rust-cssparser |
| `cursor-icon` | 1.2.0 | MIT OR Apache-2.0 OR Zlib | https://github.com/rust-windowing/cursor-icon |
| `curve25519-dalek` | 4.1.3 | BSD-3-Clause | https://github.com/dalek-cryptography/curve25519-dalek/tree/main/curve25519-dalek |
| `curve25519-dalek-derive` | 0.1.1 | MIT/Apache-2.0 | https://github.com/dalek-cryptography/curve25519-dalek |
| `darling` | 0.24.1 | MIT | https://github.com/TedDriggs/darling |
| `darling_core` | 0.24.1 | MIT | https://github.com/TedDriggs/darling |
| `darling_macro` | 0.24.1 | MIT | https://github.com/TedDriggs/darling |
| `dashmap` | 5.5.3 | MIT | https://github.com/xacrimon/dashmap |
| `defmt` | 1.1.1 | MIT OR Apache-2.0 | https://github.com/knurling-rs/defmt |
| `defmt-macros` | 1.1.1 | MIT OR Apache-2.0 | https://github.com/knurling-rs/defmt |
| `defmt-parser` | 1.0.0 | MIT OR Apache-2.0 | https://github.com/knurling-rs/defmt |
| `deranged` | 0.5.8 | MIT OR Apache-2.0 | https://github.com/jhpratt/deranged |
| `derive_arbitrary` | 1.4.2 | 未声明（本地缓存没有该包的 manifest） | - |
| `derive_more` | 0.99.20 | MIT | https://github.com/JelteF/derive_more |
| `derive_more` | 2.1.1 | MIT | https://github.com/JelteF/derive_more |
| `derive_more-impl` | 2.1.1 | MIT | https://github.com/JelteF/derive_more |
| `digest` | 0.10.7 | MIT OR Apache-2.0 | https://github.com/RustCrypto/traits |
| `digest` | 0.11.3 | MIT OR Apache-2.0 | https://github.com/RustCrypto/traits |
| `directories` | 6.0.0 | MIT OR Apache-2.0 | https://github.com/soc/directories-rs |
| `dirs-sys` | 0.5.0 | MIT OR Apache-2.0 | https://github.com/dirs-dev/dirs-sys-rs |
| `dispatch` | 0.2.0 | MIT | http://github.com/SSheldon/rust-dispatch |
| `dispatch2` | 0.3.1 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `displaydoc` | 0.2.7 | MIT OR Apache-2.0 | https://github.com/yaahc/displaydoc |
| `dlib` | 0.5.3 | MIT | https://github.com/elinorbgr/dlib |
| `dlopen2` | 0.8.2 | MIT | https://github.com/OpenByteDev/dlopen2 |
| `dlopen2_derive` | 0.4.3 | MIT | https://github.com/OpenByteDev/dlopen2 |
| `document-features` | 0.2.12 | MIT OR Apache-2.0 | https://github.com/slint-ui/document-features |
| `downcast-rs` | 1.2.1 | MIT/Apache-2.0 | https://github.com/marcianx/downcast-rs |
| `dpi` | 0.1.2 | Apache-2.0 AND MIT | https://github.com/rust-windowing/winit |
| `dtoa` | 1.0.11 | MIT OR Apache-2.0 | https://github.com/dtolnay/dtoa |
| `dtoa-short` | 0.3.5 | MPL-2.0 | https://github.com/upsuper/dtoa-short |
| `dunce` | 1.0.5 | CC0-1.0 OR MIT-0 OR Apache-2.0 | https://gitlab.com/kornelski/dunce |
| `dyn-clone` | 1.0.20 | MIT OR Apache-2.0 | https://github.com/dtolnay/dyn-clone |
| `ecolor` | 0.32.3 | MIT OR Apache-2.0 | https://github.com/emilk/egui |
| `eframe` | 0.32.3 | MIT OR Apache-2.0 | https://github.com/emilk/egui/tree/main/crates/eframe |
| `egui` | 0.32.3 | MIT OR Apache-2.0 | https://github.com/emilk/egui |
| `egui_glow` | 0.32.3 | MIT OR Apache-2.0 | https://github.com/emilk/egui/tree/main/crates/egui_glow |
| `egui-wgpu` | 0.32.3 | MIT OR Apache-2.0 | https://github.com/emilk/egui/tree/main/crates/egui-wgpu |
| `egui-winit` | 0.32.3 | MIT OR Apache-2.0 | https://github.com/emilk/egui/tree/main/crates/egui-winit |
| `emath` | 0.32.3 | MIT OR Apache-2.0 | https://github.com/emilk/egui/tree/main/crates/emath |
| `endi` | 1.1.1 | MIT | https://github.com/zeenix/endi |
| `enumflags2` | 0.7.12 | MIT OR Apache-2.0 | https://github.com/meithecatte/enumflags2 |
| `enumflags2_derive` | 0.7.12 | MIT OR Apache-2.0 | https://github.com/meithecatte/enumflags2 |
| `epaint` | 0.32.3 | MIT OR Apache-2.0 | https://github.com/emilk/egui/tree/main/crates/epaint |
| `epaint_default_fonts` | 0.32.3 | (MIT OR Apache-2.0) AND OFL-1.1 AND Ubuntu-font-1.0 | https://github.com/emilk/egui/tree/main/crates/epaint_default_fonts |
| `equivalent` | 1.0.2 | Apache-2.0 OR MIT | https://github.com/indexmap-rs/equivalent |
| `errno` | 0.3.14 | MIT OR Apache-2.0 | https://github.com/lambda-fairy/rust-errno |
| `error-code` | 3.4.0 | BSL-1.0 | https://github.com/DoumanAsh/error-code |
| `event-listener` | 5.4.2 | Apache-2.0 OR MIT | https://github.com/smol-rs/event-listener |
| `event-listener-strategy` | 0.5.4 | Apache-2.0 OR MIT | https://github.com/smol-rs/event-listener-strategy |
| `fallible-iterator` | 0.3.0 | MIT/Apache-2.0 | https://github.com/sfackler/rust-fallible-iterator |
| `fallible-streaming-iterator` | 0.1.9 | MIT/Apache-2.0 | https://github.com/sfackler/fallible-streaming-iterator |
| `fastrand` | 2.5.0 | Apache-2.0 OR MIT | https://github.com/smol-rs/fastrand |
| `fax` | 0.2.7 | MIT | https://github.com/pdf-rs/fax |
| `fdeflate` | 0.3.7 | MIT OR Apache-2.0 | https://github.com/image-rs/fdeflate |
| `fiat-crypto` | 0.2.9 | MIT OR Apache-2.0 OR BSD-1-Clause | https://github.com/mit-plv/fiat-crypto |
| `field-offset` | 0.3.6 | MIT OR Apache-2.0 | https://github.com/Diggsey/rust-field-offset |
| `find-crate` | 0.6.3 | Apache-2.0 OR MIT | https://github.com/taiki-e/find-crate |
| `find-msvc-tools` | 0.1.13 | MIT OR Apache-2.0 | https://github.com/rust-lang/cc-rs |
| `fixedbitset` | 0.5.7 | MIT OR Apache-2.0 | https://github.com/petgraph/fixedbitset |
| `flate2` | 1.1.10 | MIT OR Apache-2.0 | https://github.com/rust-lang/flate2-rs |
| `fluent` | 0.16.1 | Apache-2.0 OR MIT | https://github.com/projectfluent/fluent-rs |
| `fluent-bundle` | 0.15.3 | Apache-2.0 OR MIT | https://github.com/projectfluent/fluent-rs |
| `fluent-langneg` | 0.13.1 | Apache-2.0 OR MIT | https://github.com/projectfluent/fluent-langneg-rs |
| `fluent-syntax` | 0.11.1 | Apache-2.0 OR MIT | https://github.com/projectfluent/fluent-rs |
| `foldhash` | 0.1.5 | Zlib | https://github.com/orlp/foldhash |
| `foreign-types` | 0.5.0 | MIT/Apache-2.0 | https://github.com/sfackler/foreign-types |
| `foreign-types-macros` | 0.2.4 | MIT/Apache-2.0 | https://github.com/sfackler/foreign-types |
| `foreign-types-shared` | 0.3.1 | MIT/Apache-2.0 | https://github.com/sfackler/foreign-types |
| `form_urlencoded` | 1.2.2 | MIT OR Apache-2.0 | https://github.com/servo/rust-url |
| `futf` | 0.1.5 | MIT / Apache-2.0 | https://github.com/servo/futf |
| `futures` | 0.3.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/futures-rs |
| `futures-channel` | 0.3.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/futures-rs |
| `futures-concurrency` | 7.7.1 | MIT OR Apache-2.0 | https://github.com/yoshuawuyts/futures-concurrency |
| `futures-core` | 0.3.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/futures-rs |
| `futures-executor` | 0.3.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/futures-rs |
| `futures-io` | 0.3.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/futures-rs |
| `futures-lite` | 2.6.1 | Apache-2.0 OR MIT | https://github.com/smol-rs/futures-lite |
| `futures-macro` | 0.3.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/futures-rs |
| `futures-sink` | 0.3.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/futures-rs |
| `futures-task` | 0.3.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/futures-rs |
| `futures-util` | 0.3.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/futures-rs |
| `fxhash` | 0.2.1 | Apache-2.0/MIT | https://github.com/cbreeden/fxhash |
| `gdk` | 0.18.2 | MIT | https://github.com/gtk-rs/gtk3-rs |
| `gdk-pixbuf` | 0.18.5 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `gdk-pixbuf-sys` | 0.18.0 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `gdk-sys` | 0.18.2 | MIT | https://github.com/gtk-rs/gtk3-rs |
| `gdkwayland-sys` | 0.18.2 | MIT | https://github.com/gtk-rs/gtk3-rs |
| `generic-array` | 0.14.7 | MIT | https://github.com/fizyk20/generic-array.git |
| `gethostname` | 1.1.0 | Apache-2.0 | https://codeberg.org/swsnr/gethostname.rs.git |
| `getrandom` | 0.1.16 | MIT OR Apache-2.0 | https://github.com/rust-random/getrandom |
| `getrandom` | 0.2.17 | MIT OR Apache-2.0 | https://github.com/rust-random/getrandom |
| `getrandom` | 0.3.4 | MIT OR Apache-2.0 | https://github.com/rust-random/getrandom |
| `getrandom` | 0.4.3 | MIT OR Apache-2.0 | https://github.com/rust-random/getrandom |
| `gio` | 0.18.4 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `gio-sys` | 0.18.1 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `gl_generator` | 0.14.0 | Apache-2.0 | https://github.com/brendanzab/gl-rs/ |
| `glib` | 0.18.5 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `glib-macros` | 0.18.5 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `glib-sys` | 0.18.1 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `globset` | 0.4.20 | Unlicense OR MIT | https://github.com/BurntSushi/ripgrep/tree/master/crates/globset |
| `glow` | 0.16.0 | MIT OR Apache-2.0 OR Zlib | https://github.com/grovesNL/glow |
| `glutin` | 0.32.3 | Apache-2.0 | https://github.com/rust-windowing/glutin |
| `glutin_egl_sys` | 0.7.1 | Apache-2.0 | https://github.com/rust-windowing/glutin |
| `glutin_glx_sys` | 0.6.1 | Apache-2.0 | https://github.com/rust-windowing/glutin |
| `glutin_wgl_sys` | 0.6.1 | Apache-2.0 | https://github.com/rust-windowing/glutin |
| `glutin-winit` | 0.5.0 | MIT | https://github.com/rust-windowing/glutin |
| `gobject-sys` | 0.18.0 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `gpu-alloc` | 0.6.2 | MIT OR Apache-2.0 | https://github.com/zakarumych/gpu-alloc |
| `gpu-allocator` | 0.27.0 | MIT OR Apache-2.0 | https://github.com/Traverse-Research/gpu-allocator |
| `gpu-alloc-types` | 0.3.1 | MIT OR Apache-2.0 | https://github.com/zakarumych/gpu-alloc |
| `gpu-descriptor` | 0.3.2 | MIT OR Apache-2.0 | https://github.com/zakarumych/gpu-descriptor |
| `gpu-descriptor-types` | 0.2.0 | MIT OR Apache-2.0 | https://github.com/zakarumych/gpu-descriptor |
| `gtk` | 0.18.2 | MIT | https://github.com/gtk-rs/gtk3-rs |
| `gtk3-macros` | 0.18.2 | MIT | https://github.com/gtk-rs/gtk3-rs |
| `gtk-sys` | 0.18.2 | MIT | https://github.com/gtk-rs/gtk3-rs |
| `half` | 2.7.1 | MIT OR Apache-2.0 | https://github.com/VoidStarKat/half-rs |
| `hashbrown` | 0.12.3 | MIT OR Apache-2.0 | https://github.com/rust-lang/hashbrown |
| `hashbrown` | 0.14.5 | MIT OR Apache-2.0 | https://github.com/rust-lang/hashbrown |
| `hashbrown` | 0.15.5 | MIT OR Apache-2.0 | https://github.com/rust-lang/hashbrown |
| `hashbrown` | 0.17.1 | MIT OR Apache-2.0 | https://github.com/rust-lang/hashbrown |
| `hashlink` | 0.10.0 | MIT OR Apache-2.0 | https://github.com/kyren/hashlink |
| `heck` | 0.4.1 | MIT OR Apache-2.0 | https://github.com/withoutboats/heck |
| `heck` | 0.5.0 | MIT OR Apache-2.0 | https://github.com/withoutboats/heck |
| `hermit-abi` | 0.5.3 | MIT OR Apache-2.0 | https://github.com/hermit-os/hermit-rs |
| `hex` | 0.4.3 | MIT OR Apache-2.0 | https://github.com/KokaKiwi/rust-hex |
| `hexf-parse` | 0.2.1 | CC0-1.0 | https://github.com/lifthrasiir/hexf |
| `hkdf` | 0.12.4 | MIT OR Apache-2.0 | https://github.com/RustCrypto/KDFs/ |
| `hmac` | 0.12.1 | MIT OR Apache-2.0 | https://github.com/RustCrypto/MACs |
| `html5ever` | 0.26.0 | MIT OR Apache-2.0 | https://github.com/servo/html5ever |
| `http` | 1.5.0 | MIT OR Apache-2.0 | https://github.com/hyperium/http |
| `httparse` | 1.10.1 | MIT OR Apache-2.0 | https://github.com/seanmonstar/httparse |
| `http-body` | 1.1.0 | MIT | https://github.com/hyperium/http-body |
| `http-body-util` | 0.1.5 | MIT | https://github.com/hyperium/http-body |
| `httpdate` | 1.0.3 | MIT OR Apache-2.0 | https://github.com/pyfisch/httpdate |
| `hybrid-array` | 0.4.15 | MIT OR Apache-2.0 | https://github.com/RustCrypto/hybrid-array |
| `hyper` | 1.11.1 | MIT | https://github.com/hyperium/hyper |
| `hyper-rustls` | 0.27.10 | Apache-2.0 OR ISC OR MIT | https://github.com/rustls/hyper-rustls |
| `hyper-util` | 0.1.21 | MIT | https://github.com/hyperium/hyper-util |
| `i18n-config` | 0.4.8 | MIT | https://github.com/kellpossible/cargo-i18n/tree/master/i18n-config |
| `i18n-embed` | 0.14.1 | MIT | https://github.com/kellpossible/cargo-i18n/tree/master/i18n-embed |
| `i18n-embed-fl` | 0.7.0 | MIT | - |
| `i18n-embed-impl` | 0.8.4 | MIT | https://github.com/kellpossible/cargo-i18n/tree/master/i18n-embed |
| `iana-time-zone` | 0.1.65 | MIT OR Apache-2.0 | https://github.com/strawlab/iana-time-zone |
| `iana-time-zone-haiku` | 0.1.2 | MIT OR Apache-2.0 | https://github.com/strawlab/iana-time-zone |
| `icu_collections` | 2.3.0 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `icu_locale_core` | 2.3.0 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `icu_normalizer` | 2.3.0 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `icu_normalizer_data` | 2.3.0 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `icu_properties` | 2.3.0 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `icu_properties_data` | 2.3.0 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `icu_provider` | 2.3.1 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `ident_case` | 1.0.1 | MIT/Apache-2.0 | https://github.com/TedDriggs/ident_case |
| `idna` | 1.1.0 | MIT OR Apache-2.0 | https://github.com/servo/rust-url/ |
| `idna_adapter` | 1.2.2 | Apache-2.0 OR MIT | https://github.com/hsivonen/idna_adapter |
| `image` | 0.25.10 | MIT OR Apache-2.0 | https://github.com/image-rs/image |
| `indexmap` | 1.9.3 | Apache-2.0 OR MIT | https://github.com/bluss/indexmap |
| `indexmap` | 2.14.2 | Apache-2.0 OR MIT | https://github.com/indexmap-rs/indexmap |
| `inout` | 0.1.4 | MIT OR Apache-2.0 | https://github.com/RustCrypto/utils |
| `intl_pluralrules` | 7.0.2 | Apache-2.0/MIT | https://github.com/zbraniecki/pluralrules |
| `intl-memoizer` | 0.5.3 | Apache-2.0 OR MIT | https://github.com/projectfluent/fluent-rs |
| `io_tee` | 0.1.1 | MIT OR Apache-2.0 | https://github.com/TheOnlyMrCat/io_tee |
| `ipnet` | 2.12.2 | MIT OR Apache-2.0 | https://github.com/krisprice/ipnet |
| `is_terminal_polyfill` | 1.70.2 | MIT OR Apache-2.0 | https://github.com/polyfill-rs/is_terminal_polyfill |
| `itoa` | 0.4.8 | MIT OR Apache-2.0 | https://github.com/dtolnay/itoa |
| `itoa` | 1.0.18 | MIT OR Apache-2.0 | https://github.com/dtolnay/itoa |
| `jiff` | 0.2.37 | Unlicense OR MIT | https://github.com/BurntSushi/jiff |
| `jiff-core` | 0.1.1 | Unlicense OR MIT | https://github.com/BurntSushi/jiff |
| `jiff-static` | 0.2.37 | Unlicense OR MIT | https://github.com/BurntSushi/jiff |
| `jiff-tzdb` | 0.1.8 | Unlicense OR MIT | https://github.com/BurntSushi/jiff |
| `jiff-tzdb-platform` | 0.1.3 | Unlicense OR MIT | https://github.com/BurntSushi/jiff |
| `jni` | 0.21.1 | MIT/Apache-2.0 | https://github.com/jni-rs/jni-rs |
| `jni` | 0.22.4 | MIT OR Apache-2.0 | https://github.com/jni-rs/jni-rs |
| `jni-macros` | 0.22.4 | MIT OR Apache-2.0 | https://github.com/jni-rs/jni-rs |
| `jni-sys` | 0.3.1 | MIT OR Apache-2.0 | https://github.com/jni-rs/jni-sys |
| `jni-sys` | 0.4.1 | MIT OR Apache-2.0 | https://github.com/jni-rs/jni-sys |
| `jni-sys-macros` | 0.4.1 | MIT OR Apache-2.0 | https://github.com/jni-rs/jni-sys |
| `jobserver` | 0.1.35 | MIT OR Apache-2.0 | https://github.com/rust-lang/jobserver-rs |
| `js-sys` | 0.3.105 | MIT OR Apache-2.0 | https://github.com/wasm-bindgen/wasm-bindgen/tree/master/crates/js-sys |
| `khronos_api` | 3.1.0 | Apache-2.0 | https://github.com/brendanzab/gl-rs/ |
| `khronos-egl` | 6.0.0 | MIT/Apache-2.0 | https://github.com/timothee-haudebourg/khronos-egl |
| `kuchikiki` | 0.8.2 | MIT | https://github.com/brave/kuchikiki |
| `lazy_static` | 1.5.0 | MIT OR Apache-2.0 | https://github.com/rust-lang-nursery/lazy-static.rs |
| `libc` | 0.2.189 | MIT OR Apache-2.0 | https://github.com/rust-lang/libc |
| `libloading` | 0.8.9 | ISC | https://github.com/nagisa/rust_libloading/ |
| `libm` | 0.2.16 | MIT | https://github.com/rust-lang/compiler-builtins |
| `libredox` | 0.1.25 | MIT | https://gitlab.redox-os.org/redox-os/libredox.git |
| `libsqlite3-sys` | 0.35.0 | MIT | https://github.com/rusqlite/rusqlite |
| `linux-raw-sys` | 0.12.1 | Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT | https://github.com/sunfishcode/linux-raw-sys |
| `linux-raw-sys` | 0.4.15 | Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT | https://github.com/sunfishcode/linux-raw-sys |
| `litemap` | 0.8.3 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `litrs` | 1.0.0 | MIT OR Apache-2.0 | https://github.com/LukasKalbertodt/litrs |
| `lock_api` | 0.4.14 | MIT OR Apache-2.0 | https://github.com/Amanieu/parking_lot |
| `log` | 0.4.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/log |
| `lru-slab` | 0.1.3 | MIT OR Apache-2.0 OR Zlib | https://github.com/Ralith/lru-slab |
| `mac` | 0.1.1 | MIT/Apache-2.0 | https://github.com/reem/rust-mac.git |
| `malloc_buf` | 0.0.6 | MIT | https://github.com/SSheldon/malloc_buf |
| `markup5ever` | 0.11.0 | MIT OR Apache-2.0 | https://github.com/servo/html5ever |
| `matchers` | 0.2.0 | MIT | https://github.com/hawkw/matchers |
| `matches` | 0.1.10 | MIT | https://github.com/SimonSapin/rust-std-candidates |
| `matchit` | 0.8.4 | MIT AND BSD-3-Clause | https://github.com/ibraheemdev/matchit |
| `memchr` | 2.8.3 | Unlicense OR MIT | https://github.com/BurntSushi/memchr |
| `memmap2` | 0.9.11 | MIT OR Apache-2.0 | https://github.com/RazrFalcon/memmap2-rs |
| `memoffset` | 0.9.1 | MIT | https://github.com/Gilnaa/memoffset |
| `metal` | 0.31.0 | MIT OR Apache-2.0 | https://github.com/gfx-rs/metal-rs |
| `mime` | 0.3.17 | MIT OR Apache-2.0 | https://github.com/hyperium/mime |
| `mime_guess` | 2.0.5 | MIT | https://github.com/abonander/mime_guess |
| `minimal-lexical` | 0.2.1 | MIT/Apache-2.0 | https://github.com/Alexhuszagh/minimal-lexical |
| `miniz_oxide` | 0.8.9 | MIT OR Zlib OR Apache-2.0 | https://github.com/Frommi/miniz_oxide/tree/master/miniz_oxide |
| `miniz_oxide` | 0.9.1 | MIT OR Zlib OR Apache-2.0 | https://github.com/Frommi/miniz_oxide/tree/master/miniz_oxide |
| `mio` | 1.2.3 | MIT | https://github.com/tokio-rs/mio |
| `moxcms` | 0.8.1 | BSD-3-Clause OR Apache-2.0 | https://github.com/awxkee/moxcms.git |
| `naga` | 25.0.1 | MIT OR Apache-2.0 | https://github.com/gfx-rs/wgpu/tree/trunk/naga |
| `ndk` | 0.9.0 | MIT OR Apache-2.0 | https://github.com/rust-mobile/ndk |
| `ndk-context` | 0.1.1 | MIT OR Apache-2.0 | https://github.com/rust-windowing/android-ndk-rs |
| `ndk-sys` | 0.5.0+25.2.9519653 | MIT OR Apache-2.0 | https://github.com/rust-mobile/ndk |
| `ndk-sys` | 0.6.0+11769913 | MIT OR Apache-2.0 | https://github.com/rust-mobile/ndk |
| `new_debug_unreachable` | 1.0.6 | MIT | https://github.com/mbrubeck/rust-debug-unreachable |
| `nodrop` | 0.1.14 | MIT/Apache-2.0 | https://github.com/bluss/arrayvec |
| `nohash-hasher` | 0.2.0 | Apache-2.0 OR MIT | https://github.com/paritytech/nohash-hasher |
| `nom` | 7.1.3 | MIT | https://github.com/Geal/nom |
| `nu-ansi-term` | 0.50.3 | MIT | https://github.com/nushell/nu-ansi-term |
| `num_enum` | 0.7.6 | BSD-3-Clause OR MIT OR Apache-2.0 | https://github.com/illicitonion/num_enum |
| `num_enum_derive` | 0.7.6 | BSD-3-Clause OR MIT OR Apache-2.0 | https://github.com/illicitonion/num_enum |
| `num-conv` | 0.2.2 | MIT OR Apache-2.0 | https://github.com/jhpratt/num-conv |
| `num-traits` | 0.2.19 | MIT OR Apache-2.0 | https://github.com/rust-num/num-traits |
| `objc` | 0.2.7 | MIT | http://github.com/SSheldon/rust-objc |
| `objc2` | 0.5.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2` | 0.6.4 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-app-kit` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-app-kit` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-cloud-kit` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-cloud-kit` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-contacts` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-core-data` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-core-data` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-core-foundation` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-core-graphics` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-core-image` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-core-image` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-core-location` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-core-location` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-core-text` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-encode` | 4.1.0 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-foundation` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-foundation` | 0.3.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-io-surface` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-link-presentation` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-metal` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-quartz-core` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-quartz-core` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-symbols` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-ui-kit` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-ui-kit` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-uniform-type-identifiers` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-user-notifications` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc2-user-notifications` | 0.3.2 | Zlib OR Apache-2.0 OR MIT | https://github.com/madsmtm/objc2 |
| `objc2-web-kit` | 0.2.2 | MIT | https://github.com/madsmtm/objc2 |
| `objc-sys` | 0.3.5 | MIT | https://github.com/madsmtm/objc2 |
| `once_cell` | 1.21.4 | MIT OR Apache-2.0 | https://github.com/matklad/once_cell |
| `once_cell_polyfill` | 1.70.2 | MIT OR Apache-2.0 | https://github.com/polyfill-rs/once_cell_polyfill |
| `opaque-debug` | 0.3.1 | MIT OR Apache-2.0 | https://github.com/RustCrypto/utils |
| `option-ext` | 0.2.0 | MPL-2.0 | https://github.com/soc/option-ext.git |
| `orbclient` | 0.3.55 | MIT | https://gitlab.redox-os.org/redox-os/orbclient |
| `ordered-float` | 4.6.0 | MIT | https://github.com/reem/rust-ordered-float |
| `ordered-stream` | 0.2.0 | MIT OR Apache-2.0 | https://github.com/danieldg/ordered-stream |
| `owned_ttf_parser` | 0.25.1 | Apache-2.0 | https://github.com/alexheretic/owned-ttf-parser |
| `pango` | 0.18.3 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `pango-sys` | 0.18.0 | MIT | https://github.com/gtk-rs/gtk-rs-core |
| `parking` | 2.2.1 | Apache-2.0 OR MIT | https://github.com/smol-rs/parking |
| `parking_lot` | 0.12.5 | MIT OR Apache-2.0 | https://github.com/Amanieu/parking_lot |
| `parking_lot_core` | 0.9.12 | MIT OR Apache-2.0 | https://github.com/Amanieu/parking_lot |
| `paste` | 1.0.15 | MIT OR Apache-2.0 | https://github.com/dtolnay/paste |
| `pbkdf2` | 0.12.2 | MIT OR Apache-2.0 | https://github.com/RustCrypto/password-hashes/tree/master/pbkdf2 |
| `percent-encoding` | 2.3.2 | MIT OR Apache-2.0 | https://github.com/servo/rust-url/ |
| `phf` | 0.10.1 | MIT | https://github.com/sfackler/rust-phf |
| `phf` | 0.8.0 | MIT | https://github.com/sfackler/rust-phf |
| `phf_codegen` | 0.10.0 | MIT | https://github.com/sfackler/rust-phf |
| `phf_codegen` | 0.8.0 | MIT | https://github.com/sfackler/rust-phf |
| `phf_generator` | 0.10.0 | MIT | https://github.com/sfackler/rust-phf |
| `phf_generator` | 0.11.3 | MIT | https://github.com/rust-phf/rust-phf |
| `phf_generator` | 0.8.0 | MIT | https://github.com/sfackler/rust-phf |
| `phf_macros` | 0.8.0 | MIT | https://github.com/sfackler/rust-phf |
| `phf_shared` | 0.10.0 | MIT | https://github.com/sfackler/rust-phf |
| `phf_shared` | 0.11.3 | MIT | https://github.com/rust-phf/rust-phf |
| `phf_shared` | 0.8.0 | MIT | https://github.com/sfackler/rust-phf |
| `pin-project` | 1.1.13 | Apache-2.0 OR MIT | https://github.com/taiki-e/pin-project |
| `pin-project-internal` | 1.1.13 | Apache-2.0 OR MIT | https://github.com/taiki-e/pin-project |
| `pin-project-lite` | 0.2.17 | Apache-2.0 OR MIT | https://github.com/taiki-e/pin-project-lite |
| `piper` | 0.2.5 | MIT OR Apache-2.0 | https://github.com/smol-rs/piper |
| `pkg-config` | 0.3.34 | MIT OR Apache-2.0 | https://github.com/rust-lang/pkg-config-rs |
| `plain` | 0.2.3 | MIT/Apache-2.0 | https://github.com/randomites/plain |
| `png` | 0.18.1 | MIT OR Apache-2.0 | https://github.com/image-rs/image-png |
| `polling` | 3.11.0 | Apache-2.0 OR MIT | https://github.com/smol-rs/polling |
| `poly1305` | 0.8.0 | Apache-2.0 OR MIT | https://github.com/RustCrypto/universal-hashes |
| `portable-atomic` | 1.15.0 | Apache-2.0 OR MIT | https://github.com/taiki-e/portable-atomic |
| `portable-atomic-util` | 0.2.8 | Apache-2.0 OR MIT | https://github.com/taiki-e/portable-atomic-util |
| `potential_utf` | 0.1.6 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `powerfmt` | 0.2.0 | MIT OR Apache-2.0 | https://github.com/jhpratt/powerfmt |
| `ppv-lite86` | 0.2.21 | MIT OR Apache-2.0 | https://github.com/cryptocorrosion/cryptocorrosion |
| `precomputed-hash` | 0.1.1 | MIT | https://github.com/emilio/precomputed-hash |
| `presser` | 0.3.1 | MIT OR Apache-2.0 | https://github.com/EmbarkStudios/presser |
| `proc-macro2` | 1.0.107 | MIT OR Apache-2.0 | https://github.com/dtolnay/proc-macro2 |
| `proc-macro-crate` | 1.3.1 | MIT OR Apache-2.0 | https://github.com/bkchr/proc-macro-crate |
| `proc-macro-crate` | 2.0.2 | MIT OR Apache-2.0 | https://github.com/bkchr/proc-macro-crate |
| `proc-macro-crate` | 3.5.0 | MIT OR Apache-2.0 | https://github.com/bkchr/proc-macro-crate |
| `proc-macro-error` | 1.0.4 | MIT OR Apache-2.0 | https://gitlab.com/CreepySkeleton/proc-macro-error |
| `proc-macro-error-attr` | 1.0.4 | MIT OR Apache-2.0 | https://gitlab.com/CreepySkeleton/proc-macro-error |
| `proc-macro-hack` | 0.5.20+deprecated | MIT OR Apache-2.0 | https://github.com/dtolnay/proc-macro-hack |
| `profiling` | 1.0.18 | MIT OR Apache-2.0 | https://github.com/aclysma/profiling |
| `pxfm` | 0.1.30 | BSD-3-Clause OR Apache-2.0 | https://github.com/awxkee/pxfm |
| `quick-error` | 2.0.1 | MIT/Apache-2.0 | http://github.com/tailhook/quick-error |
| `quick-xml` | 0.41.0 | MIT | https://github.com/tafia/quick-xml |
| `quinn` | 0.11.12 | MIT OR Apache-2.0 | https://github.com/quinn-rs/quinn |
| `quinn-proto` | 0.11.18 | MIT OR Apache-2.0 | https://github.com/quinn-rs/quinn |
| `quinn-udp` | 0.5.15 | MIT OR Apache-2.0 | https://github.com/quinn-rs/quinn |
| `quote` | 1.0.47 | MIT OR Apache-2.0 | https://github.com/dtolnay/quote |
| `rand` | 0.10.3 | MIT OR Apache-2.0 | https://github.com/rust-random/rand |
| `rand` | 0.7.3 | MIT OR Apache-2.0 | https://github.com/rust-random/rand |
| `rand` | 0.8.8 | MIT OR Apache-2.0 | https://github.com/rust-random/rand |
| `rand_chacha` | 0.2.2 | MIT OR Apache-2.0 | https://github.com/rust-random/rand |
| `rand_chacha` | 0.3.1 | MIT OR Apache-2.0 | https://github.com/rust-random/rand |
| `rand_core` | 0.10.1 | MIT OR Apache-2.0 | https://github.com/rust-random/rand_core |
| `rand_core` | 0.5.1 | MIT OR Apache-2.0 | https://github.com/rust-random/rand |
| `rand_core` | 0.6.4 | MIT OR Apache-2.0 | https://github.com/rust-random/rand |
| `rand_hc` | 0.2.0 | MIT/Apache-2.0 | https://github.com/rust-random/rand |
| `rand_pcg` | 0.10.2 | MIT OR Apache-2.0 | https://github.com/rust-random/rngs |
| `rand_pcg` | 0.2.1 | MIT OR Apache-2.0 | https://github.com/rust-random/rand |
| `range-alloc` | 0.1.5 | MIT OR Apache-2.0 | https://github.com/gfx-rs/range-alloc |
| `raw-window-handle` | 0.6.2 | MIT OR Apache-2.0 OR Zlib | https://github.com/rust-windowing/raw-window-handle |
| `redox_syscall` | 0.4.1 | MIT | https://gitlab.redox-os.org/redox-os/syscall |
| `redox_syscall` | 0.5.18 | MIT | https://gitlab.redox-os.org/redox-os/syscall |
| `redox_syscall` | 0.9.4 | MIT | https://gitlab.redox-os.org/redox-os/kernel |
| `redox_users` | 0.5.3 | MIT | https://gitlab.redox-os.org/redox-os/users |
| `ref-cast` | 1.0.27 | MIT OR Apache-2.0 | https://github.com/dtolnay/ref-cast |
| `ref-cast-impl` | 1.0.27 | MIT OR Apache-2.0 | https://github.com/dtolnay/ref-cast |
| `r-efi` | 5.3.0 | MIT OR Apache-2.0 OR LGPL-2.1-or-later | https://github.com/r-efi/r-efi |
| `r-efi` | 6.0.0 | MIT OR Apache-2.0 OR LGPL-2.1-or-later | https://github.com/r-efi/r-efi |
| `regex-automata` | 0.4.18 | MIT OR Apache-2.0 | https://github.com/rust-lang/regex |
| `regex-syntax` | 0.8.11 | MIT OR Apache-2.0 | https://github.com/rust-lang/regex |
| `renderdoc-sys` | 1.1.0 | MIT OR Apache-2.0 | https://github.com/ebkalderon/renderdoc-rs |
| `reqwest` | 0.12.28 | MIT OR Apache-2.0 | https://github.com/seanmonstar/reqwest |
| `ring` | 0.17.14 | Apache-2.0 AND ISC | https://github.com/briansmith/ring |
| `rusqlite` | 0.37.0 | MIT | https://github.com/rusqlite/rusqlite |
| `rustc_version` | 0.4.1 | MIT OR Apache-2.0 | https://github.com/djc/rustc-version-rs |
| `rustc-hash` | 1.1.0 | Apache-2.0/MIT | https://github.com/rust-lang-nursery/rustc-hash |
| `rustc-hash` | 2.1.3 | Apache-2.0 OR MIT | https://github.com/rust-lang/rustc-hash |
| `rust-embed` | 8.12.0 | MIT | https://pyrossh.dev/repos/rust-embed |
| `rust-embed-impl` | 8.12.0 | MIT | https://pyrossh.dev/repos/rust-embed |
| `rust-embed-utils` | 8.12.0 | MIT | https://pyrossh.dev/repos/rust-embed |
| `rustix` | 0.38.44 | Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT | https://github.com/bytecodealliance/rustix |
| `rustix` | 1.1.5 | Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT | https://github.com/bytecodealliance/rustix |
| `rustls` | 0.23.45 | Apache-2.0 OR ISC OR MIT | https://github.com/rustls/rustls |
| `rustls-pki-types` | 1.15.1 | MIT OR Apache-2.0 | https://github.com/rustls/pki-types |
| `rustls-webpki` | 0.103.15 | ISC | https://github.com/rustls/webpki |
| `rustversion` | 1.0.23 | MIT OR Apache-2.0 | https://github.com/dtolnay/rustversion |
| `ryu` | 1.0.23 | Apache-2.0 OR BSL-1.0 | https://github.com/dtolnay/ryu |
| `salsa20` | 0.10.2 | MIT OR Apache-2.0 | https://github.com/RustCrypto/stream-ciphers |
| `same-file` | 1.0.6 | Unlicense/MIT | https://github.com/BurntSushi/same-file |
| `schemars` | 0.9.0 | MIT | https://github.com/GREsau/schemars |
| `schemars` | 1.2.2 | MIT | https://github.com/GREsau/schemars |
| `schemars_derive` | 1.2.2 | MIT | https://github.com/GREsau/schemars |
| `scoped-tls` | 1.0.1 | MIT/Apache-2.0 | https://github.com/alexcrichton/scoped-tls |
| `scopeguard` | 1.2.0 | MIT OR Apache-2.0 | https://github.com/bluss/scopeguard |
| `scrypt` | 0.11.0 | MIT OR Apache-2.0 | https://github.com/RustCrypto/password-hashes/tree/master/scrypt |
| `sctk-adwaita` | 0.10.1 | MIT | https://github.com/PolyMeilex/sctk-adwaita |
| `secrecy` | 0.8.0 | Apache-2.0 OR MIT | https://github.com/iqlusioninc/crates/tree/main/secrecy |
| `selectors` | 0.22.0 | MPL-2.0 | https://github.com/servo/servo |
| `self_cell` | 0.10.3 | Apache-2.0 | https://github.com/Voultapher/self_cell |
| `self_cell` | 1.3.0 | Apache-2.0 OR GPL-2.0-only | https://github.com/Voultapher/self_cell |
| `semver` | 1.0.28 | MIT OR Apache-2.0 | https://github.com/dtolnay/semver |
| `serde` | 1.0.229 | MIT OR Apache-2.0 | https://github.com/serde-rs/serde |
| `serde_core` | 1.0.229 | MIT OR Apache-2.0 | https://github.com/serde-rs/serde |
| `serde_derive` | 1.0.229 | MIT OR Apache-2.0 | https://github.com/serde-rs/serde |
| `serde_derive_internals` | 0.30.0 | MIT OR Apache-2.0 | https://github.com/serde-rs/serde |
| `serde_json` | 1.0.151 | MIT OR Apache-2.0 | https://github.com/serde-rs/json |
| `serde_path_to_error` | 0.1.20 | MIT OR Apache-2.0 | https://github.com/dtolnay/path-to-error |
| `serde_repr` | 0.1.21 | MIT OR Apache-2.0 | https://github.com/dtolnay/serde-repr |
| `serde_spanned` | 0.6.9 | MIT OR Apache-2.0 | https://github.com/toml-rs/toml |
| `serde_urlencoded` | 0.7.1 | MIT/Apache-2.0 | https://github.com/nox/serde_urlencoded |
| `serde_with` | 3.23.0 | MIT OR Apache-2.0 | https://github.com/jonasbb/serde_with/ |
| `serde_with_macros` | 3.23.0 | MIT OR Apache-2.0 | https://github.com/jonasbb/serde_with/ |
| `servo_arc` | 0.1.1 | MIT/Apache-2.0 | https://github.com/servo/servo |
| `sha2` | 0.10.9 | MIT OR Apache-2.0 | https://github.com/RustCrypto/hashes |
| `sha2` | 0.11.0 | MIT OR Apache-2.0 | https://github.com/RustCrypto/hashes |
| `sharded-slab` | 0.1.7 | MIT | https://github.com/hawkw/sharded-slab |
| `shell-words` | 1.1.1 | MIT/Apache-2.0 | https://github.com/tmiasko/shell-words |
| `shlex` | 2.0.1 | MIT OR Apache-2.0 | https://github.com/comex/rust-shlex |
| `signal-hook-registry` | 1.4.8 | MIT OR Apache-2.0 | https://github.com/vorner/signal-hook |
| `simd_cesu8` | 1.2.0 | Apache-2.0 OR MIT | https://github.com/seancroach/simd_cesu8 |
| `simd-adler32` | 0.3.10 | MIT | https://github.com/mcountryman/simd-adler32 |
| `simdutf8` | 0.1.5 | MIT OR Apache-2.0 | https://github.com/rusticstuff/simdutf8 |
| `siphasher` | 0.3.11 | MIT/Apache-2.0 | https://github.com/jedisct1/rust-siphash |
| `siphasher` | 1.0.4 | MIT OR Apache-2.0 | https://github.com/jedisct1/rust-siphash |
| `slab` | 0.4.12 | MIT | https://github.com/tokio-rs/slab |
| `slotmap` | 1.1.1 | Zlib | https://github.com/orlp/slotmap |
| `smallvec` | 1.16.1 | MIT OR Apache-2.0 | https://github.com/servo/rust-smallvec |
| `smithay-client-toolkit` | 0.19.2 | MIT | https://github.com/smithay/client-toolkit |
| `smithay-client-toolkit` | 0.20.0 | MIT | https://github.com/smithay/client-toolkit |
| `smithay-clipboard` | 0.7.3 | MIT | https://github.com/smithay/smithay-clipboard |
| `smol_str` | 0.2.2 | MIT OR Apache-2.0 | https://github.com/rust-analyzer/smol_str |
| `socket2` | 0.6.5 | MIT OR Apache-2.0 | https://github.com/rust-lang/socket2 |
| `spirv` | 0.3.0+sdk-1.3.268.0 | Apache-2.0 | https://github.com/gfx-rs/rspirv |
| `stable_deref_trait` | 1.2.1 | MIT OR Apache-2.0 | https://github.com/storyyeller/stable_deref_trait |
| `static_assertions` | 1.1.0 | MIT OR Apache-2.0 | https://github.com/nvzqz/static-assertions-rs |
| `strict-num` | 0.1.1 | MIT | https://github.com/RazrFalcon/strict-num |
| `string_cache` | 0.8.9 | MIT OR Apache-2.0 | https://github.com/servo/string-cache |
| `string_cache_codegen` | 0.5.4 | MIT OR Apache-2.0 | https://github.com/servo/string-cache |
| `strsim` | 0.10.0 | MIT | https://github.com/dguo/strsim-rs |
| `strsim` | 0.11.1 | MIT | https://github.com/rapidfuzz/strsim-rs |
| `strum` | 0.26.3 | MIT | https://github.com/Peternator7/strum |
| `strum` | 0.28.0 | MIT | https://github.com/Peternator7/strum |
| `strum_macros` | 0.26.4 | MIT | https://github.com/Peternator7/strum |
| `strum_macros` | 0.28.0 | MIT | https://github.com/Peternator7/strum |
| `subtle` | 2.6.1 | BSD-3-Clause | https://github.com/dalek-cryptography/subtle |
| `syn` | 1.0.109 | MIT OR Apache-2.0 | https://github.com/dtolnay/syn |
| `syn` | 2.0.119 | MIT OR Apache-2.0 | https://github.com/dtolnay/syn |
| `syn` | 3.0.6 | MIT OR Apache-2.0 | https://github.com/dtolnay/syn |
| `sync_wrapper` | 1.0.2 | Apache-2.0 | https://github.com/Actyx/sync_wrapper |
| `synstructure` | 0.14.0 | MIT | https://github.com/mystor/synstructure |
| `system-deps` | 6.2.2 | MIT OR Apache-2.0 | https://github.com/gdesmott/system-deps |
| `tao` | 0.36.0 | Apache-2.0 | https://github.com/tauri-apps/tao |
| `tao-macros` | 0.1.4 | MIT OR Apache-2.0 | https://github.com/tauri-apps/tao |
| `target-lexicon` | 0.12.16 | Apache-2.0 WITH LLVM-exception | https://github.com/bytecodealliance/target-lexicon |
| `tempfile` | 3.27.0 | MIT OR Apache-2.0 | https://github.com/Stebalien/tempfile |
| `tendril` | 0.4.3 | MIT/Apache-2.0 | https://github.com/servo/tendril |
| `termcolor` | 1.4.1 | Unlicense OR MIT | https://github.com/BurntSushi/termcolor |
| `thin-slice` | 0.1.1 | MPL-2.0 | https://github.com/heycam/thin-slice |
| `thiserror` | 1.0.69 | MIT OR Apache-2.0 | https://github.com/dtolnay/thiserror |
| `thiserror` | 2.0.21 | MIT OR Apache-2.0 | https://github.com/dtolnay/thiserror |
| `thiserror-impl` | 1.0.69 | MIT OR Apache-2.0 | https://github.com/dtolnay/thiserror |
| `thiserror-impl` | 2.0.21 | MIT OR Apache-2.0 | https://github.com/dtolnay/thiserror |
| `thread_local` | 1.1.10 | MIT OR Apache-2.0 | https://github.com/Amanieu/thread_local-rs |
| `tiff` | 0.11.3 | MIT | https://github.com/image-rs/image-tiff |
| `time` | 0.3.55 | MIT OR Apache-2.0 | https://github.com/time-rs/time |
| `time-core` | 0.1.9 | MIT OR Apache-2.0 | https://github.com/time-rs/time |
| `time-macros` | 0.2.32 | MIT OR Apache-2.0 | https://github.com/time-rs/time |
| `tiny-skia` | 0.11.4 | BSD-3-Clause | https://github.com/RazrFalcon/tiny-skia |
| `tiny-skia-path` | 0.11.4 | BSD-3-Clause | https://github.com/RazrFalcon/tiny-skia/tree/master/path |
| `tinystr` | 0.8.4 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `tinyvec` | 1.13.3 | Zlib OR Apache-2.0 OR MIT | https://github.com/Lokathor/tinyvec |
| `tokio` | 1.53.1 | MIT | https://github.com/tokio-rs/tokio |
| `tokio-macros` | 2.7.2 | MIT | https://github.com/tokio-rs/tokio |
| `tokio-rustls` | 0.26.5 | MIT OR Apache-2.0 | https://github.com/rustls/tokio-rustls |
| `toml` | 0.5.11 | MIT/Apache-2.0 | https://github.com/toml-rs/toml |
| `toml` | 0.8.2 | MIT OR Apache-2.0 | https://github.com/toml-rs/toml |
| `toml_datetime` | 0.6.3 | MIT OR Apache-2.0 | https://github.com/toml-rs/toml |
| `toml_datetime` | 1.1.1+spec-1.1.0 | MIT OR Apache-2.0 | https://github.com/toml-rs/toml |
| `toml_edit` | 0.19.15 | MIT OR Apache-2.0 | https://github.com/toml-rs/toml |
| `toml_edit` | 0.20.2 | MIT OR Apache-2.0 | https://github.com/toml-rs/toml |
| `toml_edit` | 0.25.15+spec-1.1.0 | MIT OR Apache-2.0 | https://github.com/toml-rs/toml |
| `toml_parser` | 1.1.3+spec-1.1.0 | MIT OR Apache-2.0 | https://github.com/toml-rs/toml |
| `tower` | 0.5.3 | MIT | https://github.com/tower-rs/tower |
| `tower-http` | 0.6.11 | MIT | https://github.com/tower-rs/tower-http |
| `tower-layer` | 0.3.3 | MIT | https://github.com/tower-rs/tower |
| `tower-service` | 0.3.3 | MIT | https://github.com/tower-rs/tower |
| `tracing` | 0.1.44 | MIT | https://github.com/tokio-rs/tracing |
| `tracing-attributes` | 0.1.31 | MIT | https://github.com/tokio-rs/tracing |
| `tracing-core` | 0.1.36 | MIT | https://github.com/tokio-rs/tracing |
| `tracing-log` | 0.2.0 | MIT | https://github.com/tokio-rs/tracing |
| `tracing-subscriber` | 0.3.23 | MIT | https://github.com/tokio-rs/tracing |
| `try-lock` | 0.2.5 | MIT | https://github.com/seanmonstar/try-lock |
| `ttf-parser` | 0.25.1 | MIT OR Apache-2.0 | https://github.com/harfbuzz/ttf-parser |
| `type-map` | 0.5.1 | MIT/Apache-2.0 | https://github.com/kardeiz/type-map |
| `typenum` | 1.20.1 | MIT OR Apache-2.0 | https://github.com/paholg/typenum |
| `uds_windows` | 1.2.1 | MIT | https://github.com/haraldh/rust_uds_windows |
| `unicase` | 2.9.0 | MIT OR Apache-2.0 | https://github.com/seanmonstar/unicase |
| `unic-langid` | 0.9.6 | MIT OR Apache-2.0 | https://github.com/zbraniecki/unic-locale |
| `unic-langid-impl` | 0.9.6 | MIT OR Apache-2.0 | https://github.com/zbraniecki/unic-locale |
| `unicode-ident` | 1.0.26 | (MIT OR Apache-2.0) AND Unicode-3.0 | https://github.com/dtolnay/unicode-ident |
| `unicode-segmentation` | 1.13.3 | MIT OR Apache-2.0 | https://github.com/unicode-rs/unicode-segmentation |
| `unicode-width` | 0.2.2 | MIT OR Apache-2.0 | https://github.com/unicode-rs/unicode-width |
| `unicode-xid` | 0.2.6 | MIT OR Apache-2.0 | https://github.com/unicode-rs/unicode-xid |
| `universal-hash` | 0.5.1 | MIT OR Apache-2.0 | https://github.com/RustCrypto/traits |
| `untrusted` | 0.9.0 | ISC | https://github.com/briansmith/untrusted |
| `url` | 2.5.8 | MIT OR Apache-2.0 | https://github.com/servo/rust-url |
| `utf-8` | 0.7.6 | MIT OR Apache-2.0 | https://github.com/SimonSapin/rust-utf8 |
| `utf8_iter` | 1.0.4 | Apache-2.0 OR MIT | https://github.com/hsivonen/utf8_iter |
| `utf8parse` | 0.2.2 | Apache-2.0 OR MIT | https://github.com/alacritty/vte |
| `uuid` | 1.26.1 | Apache-2.0 OR MIT | https://github.com/uuid-rs/uuid |
| `valuable` | 0.1.1 | MIT | https://github.com/tokio-rs/valuable |
| `vcpkg` | 0.2.15 | MIT/Apache-2.0 | https://github.com/mcgoo/vcpkg-rs |
| `version_check` | 0.9.5 | MIT/Apache-2.0 | https://github.com/SergioBenitez/version_check |
| `version-compare` | 0.2.1 | MIT | https://gitlab.com/timvisee/version-compare |
| `walkdir` | 2.5.0 | Unlicense/MIT | https://github.com/BurntSushi/walkdir |
| `want` | 0.3.1 | MIT | https://github.com/seanmonstar/want |
| `wasi` | 0.11.1+wasi-snapshot-preview1 | Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT | https://github.com/bytecodealliance/wasi |
| `wasi` | 0.9.0+wasi-snapshot-preview1 | Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT | https://github.com/bytecodealliance/wasi |
| `wasip2` | 1.0.4+wasi-0.2.12 | Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT | https://github.com/bytecodealliance/wasi-rs |
| `wasm-bindgen` | 0.2.128 | MIT OR Apache-2.0 | https://github.com/wasm-bindgen/wasm-bindgen |
| `wasm-bindgen-futures` | 0.4.78 | MIT OR Apache-2.0 | https://github.com/wasm-bindgen/wasm-bindgen/tree/master/crates/futures |
| `wasm-bindgen-macro` | 0.2.128 | MIT OR Apache-2.0 | https://github.com/wasm-bindgen/wasm-bindgen/tree/master/crates/macro |
| `wasm-bindgen-macro-support` | 0.2.128 | MIT OR Apache-2.0 | https://github.com/wasm-bindgen/wasm-bindgen/tree/main/crates/macro-support |
| `wasm-bindgen-shared` | 0.2.128 | MIT OR Apache-2.0 | https://github.com/wasm-bindgen/wasm-bindgen/tree/master/crates/shared |
| `wayland-backend` | 0.3.17 | MIT | https://github.com/smithay/wayland-rs |
| `wayland-client` | 0.31.15 | MIT | https://github.com/smithay/wayland-rs |
| `wayland-csd-frame` | 0.3.0 | MIT | https://github.com/rust-windowing/wayland-csd-frame |
| `wayland-cursor` | 0.31.14 | MIT | https://github.com/smithay/wayland-rs |
| `wayland-protocols` | 0.32.13 | MIT | https://github.com/smithay/wayland-rs |
| `wayland-protocols-experimental` | 20250721.0.1 | MIT | https://github.com/smithay/wayland-rs |
| `wayland-protocols-misc` | 0.3.12 | MIT | https://github.com/smithay/wayland-rs |
| `wayland-protocols-plasma` | 0.3.12 | MIT | https://github.com/smithay/wayland-rs |
| `wayland-protocols-wlr` | 0.3.12 | MIT | https://github.com/smithay/wayland-rs |
| `wayland-scanner` | 0.31.11 | MIT | https://github.com/smithay/wayland-rs |
| `wayland-sys` | 0.31.11 | MIT | https://github.com/smithay/wayland-rs |
| `webbrowser` | 1.2.4 | MIT OR Apache-2.0 | https://github.com/amodm/webbrowser-rs |
| `webpki-roots` | 1.0.9 | CDLA-Permissive-2.0 | https://github.com/rustls/webpki-roots |
| `web-sys` | 0.3.105 | MIT OR Apache-2.0 | https://github.com/wasm-bindgen/wasm-bindgen/tree/master/crates/web-sys |
| `web-time` | 1.1.0 | MIT OR Apache-2.0 | https://github.com/daxpedda/web-time |
| `webview2-com` | 0.34.0 | MIT | https://github.com/wravery/webview2-rs |
| `webview2-com-macros` | 0.8.1 | MIT | https://github.com/wravery/webview2-rs |
| `webview2-com-sys` | 0.34.0 | MIT | https://github.com/wravery/webview2-rs |
| `weezl` | 0.1.12 | MIT OR Apache-2.0 | https://github.com/image-rs/weezl |
| `wgpu` | 25.0.2 | MIT OR Apache-2.0 | https://github.com/gfx-rs/wgpu |
| `wgpu-core` | 25.0.2 | MIT OR Apache-2.0 | https://github.com/gfx-rs/wgpu |
| `wgpu-core-deps-apple` | 25.0.0 | MIT OR Apache-2.0 | https://github.com/gfx-rs/wgpu |
| `wgpu-core-deps-emscripten` | 25.0.0 | MIT OR Apache-2.0 | https://github.com/gfx-rs/wgpu |
| `wgpu-core-deps-windows-linux-android` | 25.0.0 | MIT OR Apache-2.0 | https://github.com/gfx-rs/wgpu |
| `wgpu-hal` | 25.0.2 | MIT OR Apache-2.0 | https://github.com/gfx-rs/wgpu |
| `wgpu-types` | 25.0.0 | MIT OR Apache-2.0 | https://github.com/gfx-rs/wgpu |
| `winapi` | 0.3.9 | MIT/Apache-2.0 | https://github.com/retep998/winapi-rs |
| `winapi-i686-pc-windows-gnu` | 0.4.0 | MIT/Apache-2.0 | https://github.com/retep998/winapi-rs |
| `winapi-util` | 0.1.11 | Unlicense OR MIT | https://github.com/BurntSushi/winapi-util |
| `winapi-x86_64-pc-windows-gnu` | 0.4.0 | MIT/Apache-2.0 | https://github.com/retep998/winapi-rs |
| `windows` | 0.58.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows` | 0.61.3 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_aarch64_gnullvm` | 0.42.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_aarch64_gnullvm` | 0.52.6 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_aarch64_msvc` | 0.42.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_aarch64_msvc` | 0.52.6 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_i686_gnu` | 0.42.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_i686_gnu` | 0.52.6 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_i686_gnullvm` | 0.52.6 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_i686_msvc` | 0.42.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_i686_msvc` | 0.52.6 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_x86_64_gnu` | 0.42.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_x86_64_gnu` | 0.52.6 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_x86_64_gnullvm` | 0.42.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_x86_64_gnullvm` | 0.52.6 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_x86_64_msvc` | 0.42.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows_x86_64_msvc` | 0.52.6 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-collections` | 0.2.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-core` | 0.58.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-core` | 0.61.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-core` | 0.62.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-future` | 0.2.1 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-implement` | 0.58.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-implement` | 0.60.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-interface` | 0.58.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-interface` | 0.59.3 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-link` | 0.1.3 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-link` | 0.2.1 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-numerics` | 0.2.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-result` | 0.2.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-result` | 0.3.4 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-result` | 0.4.1 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-strings` | 0.1.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-strings` | 0.4.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-strings` | 0.5.1 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-sys` | 0.45.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-sys` | 0.52.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-sys` | 0.59.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-sys` | 0.61.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-targets` | 0.42.2 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-targets` | 0.52.6 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-threading` | 0.1.0 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `windows-version` | 0.1.7 | MIT OR Apache-2.0 | https://github.com/microsoft/windows-rs |
| `winit` | 0.30.13 | Apache-2.0 | https://github.com/rust-windowing/winit |
| `winnow` | 0.5.40 | MIT | https://github.com/winnow-rs/winnow |
| `winnow` | 1.0.4 | MIT | https://github.com/winnow-rs/winnow |
| `wit-bindgen` | 0.57.1 | Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT | https://github.com/bytecodealliance/wit-bindgen |
| `writeable` | 0.6.4 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `wry` | 0.48.1 | Apache-2.0 OR MIT | https://github.com/tauri-apps/wry |
| `x11-dl` | 2.21.0 | MIT | https://github.com/AltF02/x11-rs.git |
| `x11rb` | 0.13.2 | MIT OR Apache-2.0 | https://github.com/psychon/x11rb |
| `x11rb-protocol` | 0.13.2 | MIT OR Apache-2.0 | https://github.com/psychon/x11rb |
| `x25519-dalek` | 2.0.1 | BSD-3-Clause | https://github.com/dalek-cryptography/curve25519-dalek/tree/main/x25519-dalek |
| `xcursor` | 0.3.11 | MIT | https://github.com/esposm03/xcursor-rs |
| `xkbcommon-dl` | 0.4.2 | MIT | https://github.com/rust-windowing/xkbcommon-dl |
| `xkeysym` | 0.2.1 | MIT OR Apache-2.0 OR Zlib | https://github.com/notgull/xkeysym |
| `xml-rs` | 0.8.29 | MIT | https://github.com/kornelski/xml-rs |
| `yoke` | 0.8.3 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `yoke-derive` | 0.8.3 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `zbus` | 5.19.0 | MIT | https://github.com/z-galaxy/zbus/ |
| `zbus_macros` | 5.19.0 | MIT | https://github.com/z-galaxy/zbus/ |
| `zbus_names` | 4.3.4 | MIT | https://github.com/z-galaxy/zbus/ |
| `zbus_xml` | 5.2.1 | MIT | https://github.com/z-galaxy/zbus/ |
| `zbus-lockstep` | 0.5.2 | MIT | https://github.com/luukvanderduim/zbus-lockstep |
| `zbus-lockstep-macros` | 0.5.2 | MIT | https://github.com/luukvanderduim/zbus-lockstep |
| `zcheapstr` | 1.1.0 | MIT | https://github.com/z-galaxy/zcheapstr/ |
| `zerocopy` | 0.8.58 | BSD-2-Clause OR Apache-2.0 OR MIT | https://github.com/google/zerocopy |
| `zerocopy-derive` | 0.8.58 | BSD-2-Clause OR Apache-2.0 OR MIT | https://github.com/google/zerocopy |
| `zerofrom` | 0.1.8 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `zerofrom-derive` | 0.1.8 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `zeroize` | 1.9.0 | Apache-2.0 OR MIT | https://github.com/RustCrypto/utils |
| `zeroize_derive` | 1.5.0 | Apache-2.0 OR MIT | https://github.com/RustCrypto/utils |
| `zerotrie` | 0.2.5 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `zerovec` | 0.11.8 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `zerovec-derive` | 0.11.6 | Unicode-3.0 | https://github.com/unicode-org/icu4x |
| `zip` | 4.6.1 | MIT | https://github.com/zip-rs/zip2.git |
| `zlib-rs` | 0.6.8 | Zlib | https://github.com/trifectatechfoundation/zlib-rs |
| `zmij` | 1.0.23 | MIT | https://github.com/dtolnay/zmij |
| `zopfli` | 0.8.3 | Apache-2.0 | https://github.com/zopfli-rs/zopfli |
| `zune-core` | 0.5.3 | MIT OR Apache-2.0 OR Zlib | https://github.com/etemesi254/zune-image |
| `zune-jpeg` | 0.5.15 | MIT OR Apache-2.0 OR Zlib | https://github.com/etemesi254/zune-image/tree/dev/crates/zune-jpeg |
| `zvariant` | 5.15.0 | MIT | https://github.com/z-galaxy/zbus/ |
| `zvariant_derive` | 5.15.0 | MIT | https://github.com/z-galaxy/zbus/ |
| `zvariant_utils` | 4.2.0 | MIT | https://github.com/z-galaxy/zbus/ |

</details>

---

## 4. copyleft / 非标准 / 未知许可 风险段

判断「会不会进 Windows 产物」用了两条依据（都在本机可复核）：

1. **manifest 的 target 门控**：某个依赖写在哪个 `[target...dependencies]` 段里，决定它在 Windows 构建里编不编；
2. **release 产物的 `.d` 文件**：`target\release\deps\<crate>-*.d` 存在 = 这个 crate 真的编过。

> 注意：`target\release` 的 mtime（2026-09-25）比 `Cargo.lock`（2026-09-30）旧，
> 所以「会进产物」以第 1 条（manifest 门控）为准，第 2 条只作佐证。

### 4.1 涉及 copyleft 的（逐条结论）

| crate | 版本 | license（原样） | 依赖路径 | 构建/运行 | 会进 Windows 产物吗 | 能不能发二进制 |
|---|---|---|---|---|---|---|
| `option-ext` | 0.2.0 | `MPL-2.0` | `i-trove` → `directories` → `dirs-sys` → `option-ext`（三段都**无 target 门控**） | **运行期** | **会**（`dirs-sys-0.5.0/Cargo.toml` 的 `[dependencies.option-ext]` 不带 target 条件；`target\release\deps\option_ext-*.d` 也在） | **能**。MPL-2.0 是**文件级**弱 copyleft：只要不改 `option-ext` 的源文件，把它编进更大的二进制没问题。**义务**：MPL-2.0 §3.2 要求分发 Executable Form 时告诉接收者怎么拿到对应 Source Code Form —— 给个 crates.io / GitHub 链接即可（第 6 节） |
| `cssparser` | 0.27.2 | `MPL-2.0` | `wry` → `kuchikiki` → `cssparser` | Android 分支（不编） | **不会**（`wry-*/Cargo.toml` 里 `kuchikiki` 挂在 `[target.'cfg(target_os = "android")'.dependencies]`；deps 里也没有它） | 能（根本不在产物里） |
| `cssparser-macros` | 0.6.1 | `MPL-2.0` | `cssparser` → `cssparser-macros` | Android 分支（不编） | **不会** | 能 |
| `selectors` | 0.22.0 | `MPL-2.0` | `wry` → `kuchikiki` → `selectors` | Android 分支（不编） | **不会** | 能 |
| `thin-slice` | 0.1.1 | `MPL-2.0` | `selectors` → `thin-slice` | Android 分支（不编） | **不会** | 能 |
| `dtoa-short` | 0.3.5 | `MPL-2.0` | `cssparser` → `dtoa-short` | Android 分支（不编） | **不会** | 能 |
| `self_cell` | 1.3.0 | `Apache-2.0 OR GPL-2.0-only` | `age` → `i18n-embed` → `fluent-bundle` → `self_cell 0.10.3` → `self_cell 1.3.0` | **运行期** | **会**（`self_cell-*.d` 在） | **能**。「OR」= 二选一，**选 `Apache-2.0` 那支**，GPL 分支不适用 |
| `self_cell` | 0.10.3 | `Apache-2.0` | 同上 | **运行期** | **会** | 能（纯 Apache-2.0） |
| `r-efi` | 5.3.0 | `MIT OR Apache-2.0 OR LGPL-2.1-or-later` | `getrandom` → `r-efi` | UEFI 分支（不编） | **不会**（`getrandom-0.3.4/Cargo.toml` 门控 `[target.'cfg(all(target_os = "uefi", getrandom_backend = "efi_rng"))'.dependencies.r-efi]`） | 能。即便编，也**选 `MIT` / `Apache-2.0`** 那支，LGPL 分支不适用 |
| `r-efi` | 6.0.0 | `MIT OR Apache-2.0 OR LGPL-2.1-or-later` | 同上 | UEFI 分支（不编） | **不会** | 能 |

**结论：不存在「必须开源整个产品」的风险。** 唯一要动的是给 `option-ext` 附一条源码获取途径（4.1 第一行 / 第 6 节）。

### 4.2 非标准、但明确允许的（列出并给结论）

这些许可不是「标准宽松五件套」，但都**允许闭源分发**，只是各有小义务 / 小坑：

| crate | 版本 | license（原样） | 结论 |
|---|---|---|---|
| `rustix` | 0.38.44 / 1.1.5 | `Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT` | 三选一，选 `Apache-2.0` 或 `MIT` 即可。**运行期**（`rustix-*.d` 在）。能发 |
| `linux-raw-sys` | 0.4.15 / 0.12.1 | `Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT` | 同上。Linux target 才编，Windows 不编。能发 |
| `wasi` | 0.9.0 / 0.11.1 | `Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT` | wasm target 才编，**不在 Windows 产物里**。能发 |
| `wasip2` | 1.0.4 | `Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT` | 同上（`getrandom` 的 `target_os = "wasi"` 分支）。能发 |
| `wit-bindgen` | 0.57.1 | `Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT` | 同上。能发 |
| `target-lexicon` | 0.12.16 | `Apache-2.0 WITH LLVM-exception` | 只有 LLVM-exception 这一支；`system-deps` / `cfg-expr`（Linux GTK 链）用，**Windows 不编**。能发 |
| `fiat-crypto` | 0.2.9 | `MIT OR Apache-2.0 OR BSD-1-Clause` | 三选一。`curve25519-dalek` 只在 `cfg(curve25519_dalek_backend = "fiat")` 时用；x86_64 默认 u64 backend → **不在 Windows 产物里**。能发 |
| `epaint_default_fonts` | 0.32.3 | `(MIT OR Apache-2.0) AND OFL-1.1 AND Ubuntu-font-1.0` | **会进产物**（egui 内嵌字体）。代码 MIT/Apache-2.0；**字体**是 SIL OFL-1.1 + Ubuntu Font Licence，允许随产品一起发，义务是「不得用原字体名发布被修改后的版本」—— 我们没改字体，无需动作。能发 |
| `unicode-ident` | 1.0.26 | `(MIT OR Apache-2.0) AND Unicode-3.0` | **会进产物**（proc-macro 链用）。MIT/Apache-2.0 + Unicode 数据许可，都允许。能发 |
| `webpki-roots` | 1.0.9 | `CDLA-Permissive-2.0` | 数据（根证书集）许可，宽松。能发 |
| `clipboard-win` / `error-code` | 5.4.1 / 3.4.0 | `BSL-1.0` | Boost Software License，宽松。能发 |
| `adler2` | 2.0.1 | `0BSD OR MIT OR Apache-2.0` | 三选一。能发 |
| `dunce` | 1.0.5 | `CC0-1.0 OR MIT-0 OR Apache-2.0` | 三选一。能发 |
| `hexf-parse` | 0.2.1 | `CC0-1.0` | 公共领域奉献。能发 |
| `foldhash` / `slotmap` / `zlib-rs` | 0.1.5 / 1.1.1 / 0.6.8 | `Zlib` | Zlib 许可，宽松（含「不得用作者名背书」条款）。能发 |
| `icu_*` / `zerovec*` / `yoke*` 等 18 个 | — | `Unicode-3.0` | Unicode 数据许可，宽松。能发 |
| `Unlicense` / `Unlicense OR MIT` / `Unlicense/MIT` 等 | — | 公共领域 | 能发 |
| `Apache-2.0 AND MIT` / `Apache-2.0 AND ISC` / `MIT AND BSD-3-Clause` 等 | — | 组合，全部标准宽松 | 门槛提高了但仍是宽松，能发 |

### 4.3 取不到许可证的（明确写「未声明」，不猜）

| crate | 版本 | 为什么在这 | 情况 |
|---|---|---|---|
| `arbitrary` | 1.4.2 | `zip-4.6.1` 的 `[target."cfg(fuzzing)".dependencies.arbitrary]` | 本地 registry **没有**这个包的 manifest（`.crate` 从没下过），`license` 字段**取不到** |
| `derive_arbitrary` | 1.4.2 | `arbitrary` 的 proc-macro | 同上 |

**结论 / 影响**：只有给 fuzz 插桩（`--cfg fuzzing`）时才会编它们；正常 `cargo build --release` 根本不碰
（`target\release\deps` 里没有 `arbitrary-*` / `derive_arbitrary-*`，印证了）。
**它们不随产物分发，不构成发布阻断项。**
若想把这两行也填上：`cargo fetch` 之后用 1.1 的 `cargo metadata` 命令重跑一次就有（前提是允许联网）。

---

## 5. 随包第三方资产核对（只读）

### 5.1 `dist\i-trove-0.1.3\licenses\` —— 到底有什么

实测就 **3 个文件**（由 `scripts\package-ui.ps1` 第 530–534 行的固定清单复制，见 `docs\PACKAGING.md` §1）：

| 文件 | 大小 | 是什么 | 与界面仓库原件 |
|---|---|---|---|
| `i-trove-ui-LICENSE` | 11,546 B | 界面仓库根的 `LICENSE`（Apache-2.0 全文） | SHA-256 **逐字节一致** |
| `i-trove-ui-NOTICE.md` | 27,791 B | 上游 ZCode 的 `NOTICE.md`（原样保留） | SHA-256 **逐字节一致** |
| `i-trove-ui-NOTICE.i-trove.md` | 757 B | 本项目的 fork 声明 | SHA-256 **逐字节一致** |

对 Apache-2.0 §4 的义务逐条对：

- **§4(a) 给接收者一份 License**：`licenses\i-trove-ui-LICENSE` 提供了 Apache-2.0 全文 —— 对得上 ✅
- **§4(b) 修改过的文件要带醒目的改动声明**：界面仓库里 `CHANGES.i-trove.md`（43,765 B）在，**但它不在 `licenses\` 里** ⚠️
  （严格说 §4(b) 落在「被修改的文件」上；不过既然是随包分发，把这行也放进 `licenses\` 更稳）
- **§4(c) 保留版权 / 专利 / 商标 / 归属声明**：上游 `NOTICE.md` 原样在 —— 对得上 ✅
- **§4(d) 若含 NOTICE 文件，衍生作品要带一份可读的归属声明**：`i-trove-ui-NOTICE.md` 在 —— 对得上 ✅

### 5.2 界面仓库（`i-trove-ui`，只读核对）

| 资产 | 状态 |
|---|---|
| `LICENSE`（11,546 B） | 在 ✅ |
| `NOTICE.md`（27,791 B，上游 ZCode 的） | 在 ✅ |
| `NOTICE.i-trove.md`（757 B） | 在 ✅ |
| `THIRD-PARTY-NOTICES.md`（1,978,939 B：npm 依赖 + 内嵌源 / 资产的声明） | 在 ✅ |
| `CHANGES.i-trove.md`（43,765 B，**Apache-2.0 §4(b) 的改动声明**） | 在 ✅ |
| `package.json` 的 `license` | `"Apache-2.0"` ✅ |
| `third-party\inventory.json` / `third-party\native-search\sources.json` | 在（`THIRD-PARTY-NOTICES.md` 里点名引用）✅ |

另外，界面仓库的 `THIRD-PARTY-NOTICES.md` 也**随包**进了产物：
`dist\i-trove-0.1.3\i-trove-ui\packages\web\dist\THIRD-PARTY-NOTICES.md`、
`...\packages\server\dist\THIRD-PARTY-NOTICES.md`、
`...\packages\server\dist\remote\THIRD-PARTY-NOTICES.md`（都是 1,978,939 B）✅

### 5.3 随包的 Node 运行时与 `node_modules`

- `dist\i-trove-0.1.3\node\` 里**只有 `node.exe`**（89.4 MB），**没有 Node 的 `LICENSE`**。
  `scripts\package-ui.ps1` 第 483–485 行只在 node 源目录里**恰好存在** `LICENSE` / `LICENSE.md` / `README.md` 时才顺手复制 ——
  本机这份 node 目录里没有，所以**漏了**。Node.js 是 MIT，**分发 node.exe 应当附它的 LICENSE**
  （Node 的二进制里还内嵌 OpenSSL / V8 等一堆声明）。这是**本次核对发现的主要缺口**。
- `dist\i-trove-0.1.3\i-trove-ui\node_modules\` 有 10 个包，**每个都自带 LICENSE**（`ssh2` / `asn1` / `bcrypt-pbkdf` /
  `buildcheck` / `cpu-features` / `nan` / `node-addon-api` / `node-pty` / `safer-buffer` / `tweetnacl`）✅ ——
  但这是**各包目录里的**许可，成体系的汇总声明还是靠界面仓库那份 `THIRD-PARTY-NOTICES.md`。
- 单文件形态 `dist\i-trove-0.1.3-single\` 里只有 `trove.exe` + `i-trove.ico`；`licenses\` 是运行时自解到缓存目录的
  （`scripts\package-ui.ps1` §5.7），两种形态**用的是同一批产物**。

### 5.4 建议改法（本次**没动**任何其它文件；`NOTICE.md` 也没动）

以下都是「发布前值得补」的，但超出「只建 `docs\THIRD-PARTY.md`」的授权范围，**留给你定**：

1. **把内核自己的 `LICENSE` + `NOTICE.md` 也放进 `licenses\`**（例如命名 `i-trove-LICENSE` / `i-trove-NOTICE.md`）：
   现在 `licenses\` 全是界面侧的；内核自己也是 Apache-2.0，理应同样随包（否则「一份 License」只覆盖了界面那半边）。
   改法：`scripts\package-ui.ps1` 第 530–534 行那个 `$licenseFiles` 数组里加两项，源指向仓库根的 `LICENSE` / `NOTICE.md`。
2. **把 Node 的 `LICENSE` 放进 `licenses\`**（`node-LICENSE`）。改法：往同一个数组加一项，源指向随包 node 的来源目录。
3. **把 `CHANGES.i-trove.md` 也放进 `licenses\`**：§4(b) 的改动声明随包最稳。
4. 若同意 1–3，建议顺手把 `NOTICE.md` 第 15–16 行那句「`licenses\` 里同时带上游的 `LICENSE` / `NOTICE.md` / `NOTICE.i-trove.md`」
   补成「上游的 + 本仓自己的 + Node 的」，并提一句 `docs\THIRD-PARTY.md`（本文）作为完整 Rust 依赖清单的入口。

（1–4 都**没有**在本任务里执行 —— 那会改到 `scripts\package-ui.ps1` / `NOTICE.md`，越过授权边界。）

---

## 6. 发布前怎么复核（在本机可跑）

发版前依次跑这 5 步，任何一条不对就别发：

```powershell
# 1) 许可证分布（期望：总 732 条上下；无 AGPL/SSPL/CC-BY-NC；MPL-2.0 恰好 6 条）
#    缓存齐全时用这条；本机缓存缺包时用第 1.2 节的离线脚本
cargo metadata --format-version 1 --offline |
  ConvertFrom-Json | ForEach-Object { $_.packages } | Where-Object { $_.source } |
  Group-Object license | Sort-Object Count -Descending | Format-Table Count, Name -AutoSize

# 2) 找 copyleft / 未知（期望：只有 MPL-2.0 x6、self_cell 的 GPL 可选支、r-efi 的 LGPL 可选支，以及 0 条 AGPL/SSPL/CC-BY-NC）
cargo metadata --format-version 1 --offline |
  ConvertFrom-Json | ForEach-Object { $_.packages } | Where-Object { $_.source } |
  Where-Object { $_.license -match 'GPL|MPL|AGPL|SSPL|BY-NC' -or -not $_.license } |
  Select-Object name, version, license

# 3) 确认 MPL-2.0 的 option-ext 真的会进 Windows 产物（期望：列出一行 option_ext-*.d）
Get-ChildItem target\release\deps -Filter 'option_ext-*.d' | Select-Object Name

# 4) 对随包 licenses 目录：期望就是 3 个文件，且与界面仓库原件逐字节一致
$ui = '..\..\i-trove-ui'          # 按实际路径改
Get-ChildItem dist\i-trove-0.1.3\licenses | Select-Object Name, Length
foreach ($p in @(@('LICENSE','i-trove-ui-LICENSE'), @('NOTICE.md','i-trove-ui-NOTICE.md'), @('NOTICE.i-trove.md','i-trove-ui-NOTICE.i-trove.md'))) {
  $a = (Get-FileHash (Join-Path $ui $p[0]) -Algorithm SHA256).Hash
  $b = (Get-FileHash (Join-Path 'dist\i-trove-0.1.3\licenses' $p[1]) -Algorithm SHA256).Hash
  "{0}: {1}" -f $p[0], ($a -eq $b)
}

# 5) 确认 MPL-2.0 源码获取途径还在（option-ext）；把它随 MPL 义务写进发布说明
Start-Process 'https://crates.io/crates/option-ext'   # 只读确认，别装东西
```

---

## 附：本文的口径与限制

- 表 = `Cargo.lock`（共 733 条，含本仓 `i-trove`）去掉本仓后的 **732 条第三方**；字段来自本地 registry manifest。
- 「会进 Windows 产物吗」= manifest 的 target 门控（主）+ `target\release\deps\*.d` 佐证（次，因为它比 lock 旧）。
- 本机 **`cargo metadata --offline` 跑不通**（缺 `arbitrary` 缓存），所以第 1.2 节那套离线脚本才是**本机实测的生成路径**；
  第 1.1 节那两条 `cargo metadata` 命令在**缓存齐全的机器上**可直接用。两条路径的字段来源相同（cargo 也是读同样的 manifest）。
- 全程**没有联网、没有增删任何依赖、没改 `Cargo.toml` / `Cargo.lock` / `NOTICE.md` / `scripts\*` / 界面仓库**。
