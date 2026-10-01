# NOTICE

i-trove
Copyright 2026 i-trove contributors

本仓库（内核，Rust，二进制名 `trove`）以 **Apache License 2.0** 发布，全文见 [`LICENSE`](./LICENSE)。

## 与上游的关系（重要，别搞混）

- **本仓库不是任何产品的分支**：全部代码为本项目所写。
- **界面在另一个仓库** `i-trove-ui`，它是 [`zai-org/ZCode`](https://github.com/zai-org/ZCode)（Apache License 2.0）的**修改版（fork）**：
  - 上游 `LICENSE`、`NOTICE.md`、`THIRD-PARTY-NOTICES.md` 在那个仓库里**原样保留**；
  - 改动声明记录在它的 `CHANGES.i-trove.md`（Apache-2.0 §4(b) 要求）；
  - 不使用上游的名称、Logo、图标作为本产品的品牌（Apache-2.0 §6 不授予商标权）。
- **分发目录** `dist/i-trove-<版本>/licenses/` 里带齐四类东西（这是 Apache-2.0 与其它许可的义务，见 [`docs/PACKAGING.md`](./docs/PACKAGING.md)）：
  ① 界面上游的 `LICENSE` / `NOTICE.md` / `NOTICE.i-trove.md`；② 界面 fork 的**改动声明** `CHANGES.i-trove.md`（§4(b)）；
  ③ 本仓自己的 `LICENSE` / `NOTICE.md`；④ 随包 Node 的许可（本机缺上游 LICENSE 文件时给的是指向上游的说明）。

## 第三方组件

- 内核依赖的 Rust crates 在 `Cargo.lock` 中锁定，各自许可见 crates.io 对应页面。
- 界面随包带 Node、`node-pty`、`ssh2` 等，其许可与声明随分发目录的 `licenses/` 一起给出。

### 发布前必须确认的一条（唯一需要动作的）

- **`option-ext` 是 MPL-2.0**（我们唯一一个会被编进二进制、且非 MIT/Apache 系的依赖）：
  MPL-2.0 允许随二进制分发，但 §3.2 要求**告诉接收方去哪拿它的源码**。所以每次发布时，
  发布说明（或 README 的「第三方组件」一节）里要带上：
  <https://github.com/soc/option-ext>（或 crates.io 上的同名页面）。

- 完整的依赖清单、许可证分布与结论见 [`docs/THIRD-PARTY.md`](./docs/THIRD-PARTY.md)（732 条，含「发布前怎么复核」的命令）。
