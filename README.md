[![License](https://img.shields.io/badge/license-Apache--2.0-lightgrey?style=flat-square)](LICENSE)
[![Status](https://img.shields.io/badge/status-alpha-red?style=flat-square)](docs/architecture.md)

# Meyrin

> 开源、单机分发的纯安全垂类 agent：单二进制、跨平台，专注部署完成之后的持续安全维护。

## 简介

Meyrin 是一个安全垂类 AI agent，不做通用 agent，也不规划多租户 / 企业版。它面向未知第三方环境部署——用户拿到二进制装到自己的机器上跑，因此工程上以「单机、单二进制、跨平台」为核心约束。运维不是与安全并列的独立品类：运维侧的检查本质也是运维安全。

核心能力分两大类：

- **巡检**：固定脚本产出原始数据，LLM 在此基础上做查漏补缺式分析；LLM 可基于脚本结果追加工具调用（读文件 / 跑命令）进一步核实。
- **类 Hermes**：LLM 默认主动找漏洞、在沙箱内模拟攻击、产出完整攻击路径报告 + 危险等级 + 修复方案；按学习到的使用者习惯优先排定查找顺序，并随用户能力提升动态调整重点。

部署过程本身不是产品重点（部署随便一个通用 LLM 都能教），重点在部署完成之后的**持续安全维护**：防火墙配置、CVE 修复、版本落后检测、各类服务安全性。

## 特性

- 四个薄壳入口（CLI / TUI / GUI / Web UI）统一调 `internal/core` 包，业务逻辑不重复实现
- Rust 沙箱隔离，白名单安全模型 + Workspace 隔离
  - Linux：原生 namespace + seccomp + Landlock
  - macOS / Windows：Go 拉起精简 Linux microVM，Rust 沙箱在 VM 内以 Linux 原生方式运行，host↔guest 走 virtio-vsock
- 存储用 SQLite 单文件，零外部依赖，跟"单二进制分发"的定位对齐
- 构建矩阵：2 种架构（x86_64 / arm64）× 3 个平台（Linux / macOS / Windows）= 6 个构建产物

## 架构 / 技术栈

```text
CLI（cobra）──────────────────┐
TUI（bubbletea）──────────────┼──→ core（Go 内部包，唯一逻辑权威）
GUI（wails + TS/React）───────┤
Web UI（内嵌 HTTP）───────────┘
                            │
                            ├── scheduler  → Agent 决策调度 / 状态机 / 子代理 fan-out
                            ├── runner     → 模型 API 调用（OpenAI / Anthropic 官方 Go SDK）
                            ├── toolproxy  → 工具调用分发
                            └── db         → SQLite 连接与生命周期管理
                                   ↕ gRPC
                            Rust 沙箱（4 个独立二进制）
                                   ↕
                            SQLite（单文件，随程序数据目录）
```

| 层 | 技术选型 | 说明 |
| --- | --- | --- |
| CLI | Go + cobra | 核心分发形态，单二进制 |
| TUI | Go + bubbletea（配 lipgloss / glamour） | 纯 Go，无额外运行时，跟 CLI 编进同一二进制 |
| GUI | Wails（Go 后端 + webview，TS/React 前端） | 可选构建目标；TS 仅用于界面展示 |
| Web UI | 同一二进制内的子命令模式，内嵌 HTTP server | 面向无头服务器的远程管理面板 |

## 命令

```text
$ go build ./...      # 构建
$ go test ./...       # 测试
```

具体命令与产物待构建矩阵定型后补充。

## 安全

Meyrin 是一个安全 agent，安全敏感面包括：

- **沙箱隔离**：所有工具执行（读文件 / 跑命令 / 模拟攻击）都在 Rust 沙箱内进行，白名单安全模型 + Workspace 隔离。Linux 用 namespace + seccomp + Landlock；macOS / Windows 用 Go 拉起的精简 Linux microVM。
- **网络端点**：Web UI 默认绑定 `localhost`，面向无头服务器的远程管理面板；可选开启 mesh 访问（默认接入 EasyTier），即使在内网也需鉴权。不提供公网直连方案。
- **持久化状态**：SQLite 单文件存于程序数据目录，含巡检报告、攻击路径记录与本地状态。
- **信任边界**：LLM 输出为分析与修复建议，不自动执行；所有修改性操作由用户确认后执行。
- **LLM 数据流向**：`runner` 将上下文发送给 OpenAI / Anthropic 模型 API。涉密环境请自行评估此数据流向。

## 文档

| 文档 | 用途 |
| --- | --- |
| [docs/architecture.md](docs/architecture.md) | 系统架构（定位、拓扑、交互层、跨平台、安全模型） |

## 已知局限

- 当前为 alpha 阶段，核心逻辑（scheduler / runner / toolproxy / db）仍在开发中，暂不可用
- Linux 原生路径先行；macOS / Windows 的 microVM 方案后期再实现
- Web UI 不提供公网直连方案，公网暴露不在产品职责内

## License & Disclaimer

本项目基于 [Apache License 2.0](LICENSE) 发布。

本软件按「现状」（as-is）提供，不附带任何明示或暗示的担保。作者不对因使用本软件造成的任何损害承担责任，包括但不限于数据丢失、系统损坏或安全事件。完整条款见 [LICENSE](LICENSE) 文件。

Meyrin 会接触、存储或暴露用户的系统状态、网络端点与持久化数据。请仅在你有权审计的机器上运行，并在执行 LLM 建议的修复操作前自行核对。

任何商业实体使用本软件，须自行负责遵守适用的法律法规，包括但不限于欧盟《网络弹性法案》（EU Cyber Resilience Act, CRA）及其他区域性要求。
