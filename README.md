# i-trove 发布产物

> **这里只放打好的二进制**（见 [Releases](../../releases/latest)），**没有源码** —— 源码在私有仓库 `zhangkeke123/i-trove`。

i-trove 是一个**本地跑**的「agent 工作台 + 记忆 + 治理」内核（Rust，命令 `trove`）：
一个工作台跑着你所有的 agent，并且记住、守住它们做的一切。模型接你自己的 OpenAI 兼容 `/v1` 端点，数据都在本机。

## 下载哪一件

| 资产 | 用途 |
|---|---|
| `i-trove-＜版本＞-setup.exe` | **要装就用这个**：per-user，不弹 UAC，开始菜单出现 `i-trove`，卸载默认保留你的数据 |
| `i-trove-＜版本＞-windows-x64-portable.exe` | 便携单文件：自包含，第一次运行自解到 `%LOCALAPPDATA%\i-trove\cache` |
| `i-trove-＜版本＞-windows-x64-kernel.exe` | 只要内核：命令行 + 内核自带 Web GUI（8 个视图），没有原生窗口 |
| `SHA256SUMS.txt` | 上面这几件的 sha256（**先校验再装**） |

校验（PowerShell）：

```powershell
Get-FileHash .\i-trove-＜版本＞-setup.exe -Algorithm SHA256
```

## 装之前该知道的

- 只在 **Windows 10 / 11 x64** 上实测过。
- 原生窗口要 **WebView2** 运行时（Win11 自带；Win10 多数已带；缺了会打印指引并自动退回系统浏览器）。
- **没有代码签名**：首次运行会被 SmartScreen 拦 →「更多信息」→「仍要运行」。要真签名得买代码签名证书。
- 数据都在本机 `%APPDATA%\i-trove`；卸载默认保留（配置 / 密钥 / 会话库）。
- **哈希只对「那一次构建」有效**：同源两次构建不会逐字节一致（PE 头里有链接时间戳）。
  核对「装出来的对不对」请看 `trove --version` 与行为（`doctor` / 界面 8 个视图）。

## 许可与第三方

- i-trove 本体以 **Apache License 2.0** 发布：见 [`LICENSE`](./LICENSE) 与 [`NOTICE.md`](./NOTICE.md)。
- 随包第三方组件（含界面 fork、随包 Node）的许可清单与复核命令见 [`THIRD-PARTY.md`](./THIRD-PARTY.md)；
  其中 **MPL-2.0 的 `option-ext`** 按该许可 §3.2 在文档里给出源码位置。
- 界面部件是 [`zai-org/ZCode`](https://github.com/zai-org/ZCode)（Apache-2.0）的**修改版**：
  上游 `LICENSE` / `NOTICE` 与改动声明都随产物一起分发（产物里 `licenses\`）。

> 这个仓库**不接收 issue / PR**：它只是产物镜像。问题请找 i-trove 的维护者。
