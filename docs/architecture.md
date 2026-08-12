# Meyrin 系统架构（安全垂类）

> v0.1 · draft · 2026-08-13

## 1. 定位与目标

**定位**：开源、单机分发的纯安全垂类 agent——单二进制、跨平台（Linux / macOS / Windows × x86_64 / arm64），面向未知第三方环境部署，用户拿到对应平台的单个可执行文件即可运行，不需要 Docker、不需要额外起服务、不需要外部数据库。不做通用 agent，不规划多租户 / 企业版；运维与安全不并列为两个品类，运维侧的检查本质也是运维安全。

**目标**：产品重点不是部署过程本身，而是部署完成之后的持续安全维护（防火墙配置、CVE 修复、版本落后检测、各类服务安全性），围绕两个安全域展开：

| 安全域 | 引擎 | 覆盖内容 |
| --- | --- | --- |
| 运维安全 | 巡检引擎 | 宿主机 + 远程机器（SSH 添加）的无人值守巡检：脚本集产出原始数据 → LLM 查漏补缺式分析（可追加核心工具调用进一步核实）→ 报告落盘 + 异常报警 |
| 软件产出安全 | 深挖引擎 | 用户软件的审计：代码层面 L1→L3 渐进式扫描 + 运行层面部署进 VM 模拟攻击，产出攻击路径报告、危险等级与修复方案 |

两域之上是学习引擎：从任务过程与用户交互中蒸馏记忆与 skill，沉淀用户画像，随用户能力提升动态调整巡检 / 深挖重点（见「引擎层」）。

支撑定位的工程约束：

- **单机单二进制**：四个薄壳入口（CLI / TUI / GUI / Web UI）统一调 `internal/core`（唯一逻辑权威，业务逻辑唯一实现处）；SQLite 单文件存储，零外部依赖（见「整体拓扑」「目录结构」「存储与 data-dir」「跨平台打包」）
- **无人值守**：复用系统原生调度设施（systemd timer / launchd / 任务计划程序）触发一次性执行，通知渠道主动报警，产品无常驻 daemon（见「定时任务与通知」）
- **沙箱隔离**：agent 自身的工具调用走执行层沙箱（Linux 原生 namespace + seccomp + Landlock，macOS / Windows 走 microVM），模拟攻击的被测软件跑部署层 VM；安全模型 = judge + 四维白名单 + workspace 隔离，对全部工具调用统一生效（见「沙箱」「安全模型」）

## 2. 整体拓扑

```text
CLI / TUI / GUI / Web UI ──→ internal/core（唯一逻辑权威，目录落点见「目录结构」）
                              ├── scheduler（inspect / dig / learn 三引擎）
                              ├── runner（模型接入）
                              ├── toolproxy（官方工具集）── gRPC（Go 主动调用）──→ Rust 沙箱执行层
                              ├── profile（文档读取层）
                              └── db（SQLite）
                                     （Go 侧 db 包统一管理，Rust 不直连 SQLite）

Rust 沙箱执行层：sandbox-main（gRPC server）+ 三个子沙箱，常驻子进程
Rust 沙箱部署层：VM（Firecracker / Virtualization.framework / Hyper-V），
                 模拟攻击时才拉起、一次性，与执行层平行存在、不常驻
```

- 四个薄壳入口统一调 `internal/core`，业务逻辑不重复实现
- 核心工具（read_file / write_file / run_terminal_command）调用链：toolproxy → gRPC → sandbox-main → 对应子沙箱；gRPC 由 Go 侧发起，db 不直连 Rust（见「存储与 data-dir」章）
- 部署层 VM 模拟攻击时才拉起，与执行层平行存在、不常驻

## 3. 目录结构

> **规划布局**：目标架构布局（顶层起步实现为 `runner/`），正文描述规划结构、非当前目录快照。

```text
cmd/             # 四个薄壳入口（进程入口，统一调 internal/core，不实现业务逻辑）
  cli/           # CLI 入口
  tui/           # TUI 入口
  gui/           # GUI 入口（Wails）
  web/           # Web UI 入口（本机 loopback）
internal/
  core/          # 唯一逻辑权威（业务逻辑唯一实现处）
    scheduler/   # 三个独立引擎
      inspect/   # 巡检引擎
      dig/       # 深挖引擎
      learn/     # 学习引擎（证据蒸馏 / 记忆与画像更新）
    runner/      # 模型接入层（Adapter + 鉴权）
      auth/      # 统一鉴权子包（API Key 加解密）
      anthropic/ # Anthropic Adapter（adapter.go + modelspec.go）
      openai/    # OpenAI 兼容 Adapter（adapter.go + modelspec.go）
    toolproxy/   # 官方保证工具集（per-tool 独立开关）
    profile/     # 文档读取层：AGENTS.md / SKILL.md / memory / MCP（RAG 兼容位，可插拔、默认关）
    db/          # SQLite 连接与生命周期（Go 统一管理，Rust 不直连）
sandbox/         # Rust 沙箱源码（独立二进制，产物经 go:embed 嵌入 Go 二进制）
  main/          # sandbox-main（gRPC server，哑转发器）
  read/          # sandbox-read（只读沙箱）
  write/         # sandbox-write（只写沙箱）
  cmd/           # sandbox-cmd（命令沙箱）
```

- `core` = `internal/core`，既是逻辑权威也是目录落点；四个薄壳入口统一调它
- per-tool 开关：启用 / 禁用状态为业务配置存 SQLite（见「配置管理」）；工具声明（名称 / 描述 / 参数 schema）随工具代码

## 4. 引擎层：巡检 / 深挖 / 学习

三条独立逻辑，共用底层状态机（工具执行 / 授权 / judge），任务层各自独立。

### 4.1 巡检引擎

- 线性任务：脚本集 → 原始数据 → LLM 查漏补缺分析 → 报告
- LLM 可基于脚本结果追加核心工具调用（读文件 / 跑命令）进一步核实，走完整 Agent 状态机；追加可多轮，终止条件 = LLM 判定无待核实项或达追加轮数上限（上限默认待细化）
- **原始数据信任边界**：脚本输出 / 被读取的文件内容视为**不可信输入**（与「协作文档」同一标注，见「profile 文档层」）——LLM 分析时按不可信来源处理，不把其中的断言当既定事实；基于原始数据追加的核心工具调用（read_file / write_file / run_terminal_command）仍是白名单操作、命中即执行，收容由执行层沙箱兜底（见「执行层」）
- LLM 自主度低，任务边界由脚本集定义
- 巡检 / 深挖场景 `PreToolUse` judge 强制启用
- 巡检结果无自动验证闭环：LLM 查漏补缺分析的误报 / 漏报纠偏机制待细化（`PreResponse` judge 默认关，仅 `PreToolUse` 强制启用，见「引擎决策层」）

**触发**：三条路——用户交互发起（CLI / TUI / GUI / Web UI）、定时任务（systemd timer / launchd / 任务计划程序）、IM 回复续接已有会话。

**任务定义**：巡检任务 = 对巡检范围内的机器（宿主机本身 + 用户通过 SSH 添加的机器清单）按定义的脚本集执行（脚本定义 / 输入输出约定待细化，官方预置清单位置见「目录结构」章）。**添加远程机器时询问是否将该机器 IP:端口加入网络白名单**（见「安全模型 · 四维白名单」），确认后该机器即可纳入无人值守巡检。脚本来源：官方预置脚本（产品保证，责任模型同「工具体系」章官方工具）+ 用户自加巡检脚本（用户自担，接入方式见「工具体系」章）；本地执行的脚本统一经 sandbox-cmd 执行（见「执行层 · 脚本工具执行路径」），远端执行的脚本经 SSH 推送至目标机器（见下方「远端机器巡检的执行模型」）。脚本默认并行执行（默认并行数 4，可配置）；执行前 LLM 预检脚本集——检查脚本间冲突或重复（判定标准待细化），交互模式告知用户并给出建议（关一个 / 调整），无人值守模式默认跳过冲突脚本并汇报（跳过策略可配置：跳过并汇报 / 全部跳过静默 / 终止任务）。

**无人值守行为**：定时巡检（无 UI）中遇到需要授权的操作（judge 不确定）默认跳过并汇报，不卡死任务。两处「跳过」语义不同——预检跳过 = 冲突 / 重复脚本整体不执行；执行中跳过 = 单次授权操作未执行；报告均标注「未做」及原因。

**远端机器巡检的执行模型**：远程机器（SSH 添加的机器）的巡检 = 检查逻辑经 SSH 推送到远端、**以远端用户身份在远端执行**（fleet-audit 同款模式：`declare -f` 打包函数体 + `ssh ... bash -s --`），本地只是接收 stdout 汇总。因此**远端执行不受 Meyrin 执行层沙箱约束**——沙箱（seccomp / Landlock / Mount Namespace）只约束本地 `ssh` 进程，远端命令以目标机器 SSH 用户的权限裸跑；远端需特权操作（如查 `iptables` / `sshd -T`）时按「授权与放权」手动 sudo。安全边界：本地收容由执行层沙箱兜底，远端收容依赖「以受控 SSH 用户执行 + 该机器是用户信任清单内机器」这一前提。

**产物**：巡检报告，md 文档，随数据目录存放；自动清理，默认保留 7 天（可配置）。报告默认落盘；通知默认报警才推送，可配置为报告也推送；通知渠道默认内置支持 Email / Telegram / Webhook 三个（可同时启用，也可只开其中一个，配置凭据后生效，见「定时任务与通知」）。

**学习开关**：巡检原始数据与工具调用日志是否进入历史轨迹（证据层）供学习引擎蒸馏，由用户配置，默认开启。（与报告清理策略不同：报告是消费产物、默认 7 天清理；历史轨迹原始数据是证据层、长期保留供蒸馏——用途不同，策略因而不同）无人值守场景（systemd timer / launchd / 任务计划程序，无 UI）下同样按此开关执行——数据默认进历史轨迹，用户可通过配置在定时任务中关闭。

### 4.2 深挖引擎

- 用户发起（唯一触发），与巡检引擎相互独立，不自动联动
- 服务软件产出安全，从两个层面看用户软件的问题，**两个层面由用户自由选择执行哪个、执行次序**：
  - **代码层面**：直接看代码——三层渐进式扫描，按 L1 → L2 → L3 递进执行（每层以上一层结果为基础）；用户可选择执行哪些层、从哪一层起（如只跑 L2，或 L1+L3 跳过 L2）：

    | 层级 | 强度 | 扫描内容 |
    | --- | --- | --- |
    | L1 静态检查 | 模式匹配 | 高危模式（`eval()` / 密钥硬编码） |
    | L2 轻量扫描 | 语义理解 | SQL 注入 / 远程命令执行 / 敏感信息泄露 |
    | L3 深度扫描 | 跨文件数据流分析 | 跨文件跨函数追踪完整数据流 |
  - **运行层面**：实际跑起来看问题——软件部署进部署层 VM，模拟攻击验证，见「部署层 · 模拟攻击拓扑」（模拟攻击拓扑细节依赖 §7.2，尚未细化，此处仅立前提）
- 收敛条件：LLM 自主判断——覆盖了它能想到的攻击面即收敛；扫描层级与轮数由用户选择
- 按学习到的用户画像优先排定查找顺序；检查前先向用户提示已知漏洞、测试其意识（见「记忆与画像」）
- **产物**：与巡检同构——md 报告落盘（深挖耗时长，用户可能离开 / 休息，报告需完整落盘）+ 通知；代码层面与运行层面**分开报告**

### 4.3 学习引擎

- 触发时机：每次巡检 / 深挖任务结束后，以及和用户对话时
- **观察**：跟踪多步任务过程（工具调用、决策分支、用户纠正）写入工作层记忆；观察原始证据（快照 diff / 工具调用日志）

**记忆**（发生过什么），分层存储：

- **策展层**：唯一注入 system prompt 的记忆层，只收两类内容——① 程序采集事实（快照 diff / 工具日志等事件记录）；② 用户显式确认的内容（`USER.md` 用户画像——偏好、沟通风格、习惯与盲点，由用户提供或确认）。`MEMORY.md` 存环境事实与约定（程序采集 + 用户确认）；**LLM 蒸馏的「学到的东西」默认只进工作层，不进策展层**（投毒防御：注入面不含 LLM 产物，见「后台整合」）。会话开始时冻结快照注入 system prompt；有预算上限（注入副本上限，默认待细化；超限时磁盘完整保留、只截断注入副本——见下方「后台整合」）
- **工作层**：`memory/YYYY-MM-DD.md` 日笔记——详细记录与观察，只索引不注入；索引走 SQLite FTS5 全文索引（零外部依赖），搜索可达
- **后台整合**：任务结束后执行（与触发时机统一——产品无常驻 daemon，无独立周期）；从工作层材料提炼注入内容，**经投毒门过滤**——程序采集事实与用户显式确认的内容进策展层，LLM 蒸馏的推论默认留在工作层（可搜索、不注入）。对话时捕捉的内容先落工作层、一并过滤。工作层蒸馏触发点：任务结束（巡检 / 深挖收尾）或对话连续 N 轮未蒸馏时补蒸馏一次（N 可配置，默认待细化）；策展层超预算时磁盘完整保留、只截断注入副本，并提示把详细材料迁移回工作层——不报错不丢弃

**Skills**（下次怎么查）：`SKILL.md` 程序性知识，按需加载（progressive disclosure）。同一任务模式重复 3+ 次后蒸馏生成（流程、坑、验证步骤）——判定由学习引擎在每次任务结束后统计（同任务模式的标准：任务类型 + 主要工具序列相似度，具体阈值待细化），跨会话累计；使用中改进（发现更好方法时打补丁），失败路径剪枝、成功路径强化。

**行动敏感记忆**：影响「之后该做什么」的笔记记录行动边界——审批 / 权限要求、临时约束、过期条件、安全行动时机、「什么不能碰」。同受投毒门约束：程序采集（授权事件等）与用户确认的内容可进策展层；LLM 推断的行动边界只进工作层。

**坏习惯**：软件产出坏习惯结构化存储，LLM 不可读原始记录，只拿脱敏后信息。

**审计**：记忆与 skill 的所有变更写入审计日志（见「日志体系」）。

## 5. 工具体系

三类工具来源，责任模型不同：

| 来源 | 位置 | 接入 | 责任 |
| --- | --- | --- | --- |
| 官方工具 | `toolproxy/` | 内置，per-tool 独立开关 | 产品保证 |
| 标准 MCP | `profile/mcp/` | 市面通用 MCP 协议 | 用户自担 |
| 用户 / LLM 自写 | 任意目录 | 文档（SKILL.md / AGENTS.md，agent 按文档找）或 MCP 接入，都接受 | 用户自担 |

- 官方工具集每个工具独立开关，用户可灵活禁用（不整体一刀切）；**开关粒度 = 整个工具（原子）**，工具内子能力（只读 / 读写等）由四维白名单承担（见「安全模型」）
- **巡检脚本不是独立来源**：巡检脚本 = 工具按「批量执行」用途的视图——官方预置巡检脚本 = 官方工具中标记 batch 用途的子集；用户自加巡检脚本 = 用户自写工具（按文档接入）以批量模式运行；脚本定义 / 输入输出约定见「巡检引擎」
- **文档即入口，作用域限定**：agent 不扫描文件系统，只按文档指示访问工具；「无文档即无工具」只约束用户 / LLM 自写源（文档 = 发现机制）；官方工具（内置声明）与 MCP 工具（协议清单）走结构化发现，不适用文档门槛。所有工具调用统一过四维白名单 + judge
- RAG：可插拔 adapter，默认关闭，预留接口不实现（兼容位，不强求，不投入核心路径）

## 6. profile 文档层

agent 读取「喂给它的文档」的统一入口，按文档找、不越界。

### 6.1 文档形态

- `AGENTS.md`：用户写的 system prompt——机器 / 环境约定（这台机器有什么特殊配置、哪些不能碰）；agent 不主动写，用户让改才改；内容属「用户显式确认」类，可入策展层注入（同「学习引擎」策展层投毒门）
- `SKILL.md`：技能与工具说明（工具用法、参数、前置条件），按需加载
- `memory/`：记忆（策展层 `MEMORY.md` / `USER.md`、工作层日笔记、坏习惯结构化存储，见「记忆与画像」）

### 6.2 项目文件夹内其他 Agent 协作文档探测（可选）

巡检 / 深挖某个项目文件夹时，探测该文件夹内是否存在其他 AI agent 工具留下的协作文档，读取作为参考素材，帮助理解项目背景以提升安全判断针对性。与「工具体系」章「文档即入口」不冲突：探测只发生在任务目标文件夹内，不是扫描整个文件系统。

- **默认关闭**（配置开启）
- **内容处理**：读取内容在 prompt 中明确标注为「不可信参考材料」，不当指令执行——这些文档不是本 agent 自己写的，理论上可被投毒（攻击者只需污染目标仓库的协作文档，不需要碰这个 agent 本身）。基于不可信材料发起的工具调用仍属白名单操作、命中即执行，收容由执行层沙箱兜底（与「巡检引擎」原始数据信任边界同一规则）
- **交互**：发现协作文档后主动告知用户，询问是否据此调整本次排查重点，不自动采信
- **检测清单**：内置默认清单（`CLAUDE.md`、`AGENTS.md`、`.cursorrules` / `.cursor/rules/*`、`.windsurfrules`、`.github/copilot-instructions.md` 等，随二进制内置）+ 用户生效清单（存 data-dir，用户可编辑，路径见「目录结构」）；agent 升级导致默认清单变化时，对比用户当前生效清单与新默认清单生成 diff 落盘待采纳，询问用户是否采纳新增项，重复项自动跳过

### 6.3 记忆与画像

记忆分层，存储各归各：

- **策展层**（唯一注入面，只收程序采集事实 + 用户显式确认，见「学习引擎」策展层投毒门）：
  - `MEMORY.md`：环境事实与约定（程序采集 + 用户确认），巡检 / 深挖 / 学习通用
  - `USER.md`：用户画像——偏好、沟通风格、操作习惯、宿主机历史盲点（用户提供 / 确认）→ 服务巡检引擎
- **工作层**：`memory/YYYY-MM-DD.md` 日笔记——详细记录与观察（含 LLM「学到的东西」），只索引不注入
- **坏习惯**：编码 / 部署习惯中的安全问题模式 → 服务深挖引擎，结构化存储、LLM 不可读原始记录、只拿脱敏后信息

- 共用机制：采集 / 蒸馏 / 读写框架（薄薄一层），三个引擎共用
- 画像数据层零共用：运维画像与软件产出坏习惯各归各，互不读写
- 历史轨迹各归各：快照 diff 归运维体系，工具调用日志跟着各自会话走
- 记忆机制（注入 / 后台整合 / 行动敏感边界 / 审计）见「学习引擎」

## 7. 沙箱

沙箱分两层，对应两个安全领域的隔离需求：

| 层 | 用途 | 形态 |
| --- | --- | --- |
| **执行层** | agent 自己的工具调用（`read_file` / `write_file` / `run_terminal_command`） | 三个独立子沙箱（read / write / cmd） |
| **部署层** | 模拟软件运行环境，专门跑「被测软件」本身，不是 agent 自己的工具调用 | 完整 VM |

两层沙箱不是按安全领域一对一划分的：**执行层承载 agent 自身的全部工具调用**，运维安全（巡检）与软件产出安全（深挖：代码审计与模拟攻击，含攻击方执行）都要用到它；**部署层只服务软件产出安全领域**，承载被测软件本身的隔离运行与被攻击目标。

### 7.1 执行层

agent 自身的全部工具调用都在执行层沙箱内运行：`read_file` / `write_file` / `run_terminal_command`。Linux 上原生运行；macOS / Windows 上由 Go 拉起精简 Linux microVM，同一份 Rust 代码原封不动跑在 VM 内 Linux 环境中（gRPC over virtio-vsock）。Go 是唯一调用方。

主调度器 + 独立子沙箱的扁平隔离架构，每个子沙箱为独立二进制、独立进程：

```text
sandbox-main（gRPC server，极简调度器）
  │
  │  match tool_type → spawn 对应子沙箱
  │
  ├── sandbox-read（独立进程）
  ├── sandbox-write（独立进程）
  └── sandbox-cmd（独立进程）

三个子沙箱平级关系，互不感知，不存在嵌套；每次工具调用新 spawn 对应类型的子沙箱，用完即毁（不做进程池常驻，见「主进程安全说明」）。性能代价已知接受：spawn 一次性、无状态——进程池 / 只读沙箱缓存均不采用，缓存即持久状态，违背「无持久反打面」原则
```

| 二进制 | 职责 |
| --- | --- |
| `sandbox-main` | gRPC server，接收 Go 请求，按工具类型 dispatch 到对应子沙箱（读取 / 修改 / 命令三类）；子沙箱异常时立即禁止并留档通知 |
| `sandbox-read` | 只读沙箱，执行 `read_file`，seccomp 白名单仅开放读操作与 stdout 通道 |
| `sandbox-write` | 只写沙箱，执行 `write_file`，seccomp 白名单仅开放写操作与 stdin 通道 |
| `sandbox-cmd` | 命令沙箱，执行 `run_terminal_command`，seccomp 黑名单约束 + 可执行文件白名单由 Mount Namespace 实现（见下） |

sandbox-main 与子沙箱之间 IPC 通过 stdin/stdout 管道通信。

**脚本工具执行路径**：官方工具集（toolproxy）中的脚本（巡检 / 审计类）在**本地执行**时作为 `run_terminal_command` 经 sandbox-cmd 执行，不另设本地脚本执行路径——脚本本身就是命令序列，与普通命令同一套护栏（白名单 / seccomp / Landlock）。远端机器巡检的脚本不在此列：经 SSH 推送至目标机器、以远端用户身份执行（见「巡检引擎 · 远端机器巡检的执行模型」）。脚本的可读路径、可执行文件、网络出口由该工具的 per-tool 配置定义安全上下文，随工具走。**巡检远程机器时 `ssh` 需加入脚本的可执行文件白名单**（per-tool 配置放行，随工具走）；SSH 凭据存储见「凭证安全」。脚本类代码经解释器（python3 / node / bash 等）执行，**可执行文件白名单约束的是解释器二进制是否可被调用，脚本文件本身是数据**——受 workspace 边界（Mount Namespace）+ Landlock 路径粒度约束，不单独作为挂载白名单项。

#### sandbox-main 职责

- 接收 Go gRPC 请求（含安全上下文：白名单/黑名单规则快照、一次性授权描述符、workspace 路径）
- 按工具类型 dispatch（读取 / 修改 / 命令 三类）到对应子沙箱（纯 match，无业务逻辑）
- 通过 stdin/stdout 管道将安全上下文单向下发至子沙箱；子沙箱只能上行结果数据，不能上行指令或控制信息
- 等待子沙箱返回结果，收口返回给 Go（返回给 LLM 的路径唯一）
- **打穿响应**：执行层子沙箱被打穿、或部署层 VM 失控（逃逸 / 资源滥用 / 反向控制）→ 立即禁止（终止 / 隔离）→ 审计日志留档 → 告知用户

**sandbox-main 不做白名单检查、不做路径校验、不做来源判断**——安全判断全部在子沙箱内部（白名单四维 + seccomp），主要判断在 `sandbox-cmd`。主沙箱是纯分发收口与安全边界，判断「命令是否恶意」「攻击是否成功」都不是这层的职责——攻击路径记录由深挖引擎产出，main 不参与。

#### 子沙箱 seccomp 规则

每个子沙箱拥有独立的最小 seccomp 规则集。`sandbox-read` / `sandbox-write` 采用**白名单模式**（未列出的 syscall 一律拒绝；危险 syscall 仍显式列出以强调）；`sandbox-cmd` 例外——采用**黑名单式约束**（默认放行，显式禁止危险 syscall），原因见下：

| 子沙箱 | 允许的 syscall | 明确禁止 |
| --- | --- | --- |
| `sandbox-read` | `openat(O_RDONLY)` / `read` / `fstat` / `close` / `write(仅 stdout)` / `exit_group` | 所有写文件、execve、fork、socket、`unshare` / `setns` / `keyctl` |
| `sandbox-write` | `openat(O_WRONLY\|O_CREAT)` / `write` / `fstat` / `close` / `read(仅 stdin)` / `exit_group` | 所有读任意文件、execve、fork、socket、`unshare` / `setns` / `keyctl` |
| `sandbox-cmd` | 常用 syscall 默认放行（`execve` / `fork` / `clone` / `read` / `write` / `socket` / `exit_group` 等） | 白名单外 execve（由 Mount Namespace 物理拦截）、`ptrace` / `mount` / `pivot_root` / `chroot` / `unshare` / `setns` / `keyctl` / `io_uring_setup` / `mknod` / `setuid` / `setgid` |

`sandbox-cmd` 的 seccomp 必须为外部程序（python3 / node / git 等）留出空间，是三者中最难收紧的部分，因此对 `sandbox-cmd` 采用黑名单式约束（禁危险 syscall，其余交给 Mount Namespace / Landlock / 网络 namespace / cgroups 约束），不追求逐 syscall 白名单；可执行文件白名单不依赖 seccomp 路径过滤（seccomp BPF 无法检查 execve 的路径参数），而是通过 Mount Namespace 实现：白名单以外的二进制不挂载进沙箱，物理上不可达。

sandbox-cmd 的文件系统隔离（Mount Namespace + pivot_root）由 sandbox-main 在 spawn 时建立、用户命令执行前完成；此后 seccomp 禁止再次 `mount` / `pivot_root`，防止解释器重新挂载或切换根目录逃逸。

#### sandbox-cmd 多层防御

sandbox-read / sandbox-write 由 Rust 进程直接执行，文件路径白名单在应用层全程可控。

sandbox-cmd 允许解释器（python3 / node / bash 等）运行，解释器内部的文件操作绕过 Rust 应用层，仅受 Mount Namespace + pivot_root（workspace 边界）约束，文件路径白名单的粒度规则（如「允许写 .py，禁止写 .sh」）对解释器子进程无效。为此引入多层内核机制叠加：

| 层 | 机制 | 作用范围 |
| --- | --- | --- |
| Mount Namespace + pivot_root | 文件系统隔离 | workspace 物理边界（白名单外路径不挂载，pivot_root 切换根），全部子进程继承 |
| seccomp | syscall 类型过滤 | sandbox-cmd 进程及其所有子进程（含解释器派生进程） |
| Landlock LSM | 内核级文件访问规则（Linux ≥ 5.13） | sandbox-cmd 及其所有子进程（含解释器派生进程） |
| Network Namespace | 网络出口隔离 | 仅允许白名单 IP:端口可达（nftables 过滤），非白名单目标在路由层不可达；规则可写域名，解析为 IP 后下发，域名仅配置便利；DDNS 域名走「DDNS 域名检测」机制 |
| cgroups | 资源配额限制 | 进程数上限（防 fork 炸弹）、内存上限、CPU 时间配额 |

Landlock 规则从安全上下文的文件白名单生成，在 exec 目标命令前配置，并由所有子进程继承；解释器内部的文件操作同样受内核级约束，无法绕过路径白名单粒度规则。

**Landlock 版本依赖降级**：Landlock 要求内核 ≥ 5.13，LTS / 企业发行版常见低内核（Debian 11 = 5.10、Ubuntu 20.04 = 5.4、RHEL 8 = 4.18）。启动时探测可用性：不可用 → **fail-closed，拒绝 `sandbox-cmd`**（解释器子进程的路径粒度规则失效时不做静默降级），并明确告知用户当前内核不满足要求；`sandbox-read` / `sandbox-write` 不受影响（Rust 进程直接执行，文件路径白名单在应用层可控）。**决策：保持 fail-closed，不提供降级路径**——已知代价：低内核机器上巡检脚本 / `run_terminal_command` 全部不可用，用户需升级内核或接受不可用；与部署层「不做弱隔离降级」口径一致。

**安全模型差异**：sandbox-read / sandbox-write 属于**拒绝型防御**（不符合规则的操作不执行）；sandbox-cmd 属于**收容型防御**（假设被执行的代码可能有恶意行为，通过多层内核机制限制其影响范围）。收容型防御天然弱于拒绝型，`sandbox-cmd` 是整个沙箱体系中攻击面最大的组件，其安全边界依赖上述多层机制的组合而非任意单层。

**审计粒度**：

| 工具类型 | 审计粒度 |
| --- | --- |
| `sandbox-read` / `sandbox-write` | 每次文件操作均经 Rust 拦截层记录 |
| `sandbox-cmd`（Rust 直接操作） | 同上 |
| `sandbox-cmd`（解释器子进程内操作） | 工具调用级记录（执行了哪条命令），内部文件操作不单独记录；Landlock 阻断越权行为，合规操作细节不入审计日志 |

这是安全性（内核级强制阻断）与可观测性（细粒度日志）的已知权衡。

#### 主进程安全说明

**定位：哑转发器**——只转不智：所有安全判断在子沙箱 / 规则层，main 只做 match 分发、管道下发、结果收口。功能面刻意最小：

- **无状态**：不做缓存、不记会话、不保留任何跨请求状态
- **接口最小集**：gRPC 仅 spawn / 收结果 / 返回三件事，无管理、调试、配置热更新接口
- **无动态加载**：静态二进制，无解释器、无 dlopen，RCE 后可用语言原语极少
- **通信面仅两条通道**：Go→main 的 gRPC、main→子沙箱的单向管道，无第三连接
- **子沙箱每调用一实例、用完即毁**：不做进程池常驻，无持久反打面

**身份分离**：sandbox-main 以用户 uid 运行；子沙箱进程降权到独立低权限 uid（userns 内才是虚拟 root）。子沙箱被打穿后无法操作 main——不同 uid，无法 ptrace、无法写其 fd、连 SIGKILL 都没有权限。

**自身白名单 seccomp**：启动即锁死——初始化完成后一次性设置白名单，运行时不再新增任何 syscall。白名单 = main 真实 syscall 面，刻意收窄：不碰文件系统（安全上下文从 gRPC 接收、日志外抛给 Go 侧，不经手磁盘）、网络仅本地 gRPC socket。与非特权 uid 互补：白名单管「能用哪些 syscall」，非特权管「用了也没权限提权」。

**管道协议当数据**：子沙箱 stdout 按长度前缀 + 二进制严格解析，畸形 / 越界输入 → 终止该子沙箱（不猜、不恢复）。子沙箱只能上行结果数据，不能上行指令或控制信息。

**快照只校验、不解释**：规则快照到达时做格式 / 边界校验（字段完整性、引用路径形态、长度上限），防畸形快照打 main 自身；语义正确性归规则库，main 不解释。

**不持久化用户数据、不持有凭据**：工具结果数据只在 main 过手转发，不停留、不存储；进程无密钥、无 token；gRPC 仅监听本地（unix socket / vsock），不对局域网 / 公网开口。

**无旁路原则**：执行层不接受规则库之外的命令来源——任何进入执行层的请求，其安全上下文必须由受控的规则构造层（toolproxy）生成，不存在绕过规则库直接下达命令的路径。

**Namespace 隔离策略**：sandbox-main 采用 User Namespace + Mount Namespace 方案 spawn 子沙箱——子进程在新的 user namespace 内获得虚拟 root 权限，在新的 mount namespace 内完成 workspace bind mount 和 `pivot_root` 切换根目录；sandbox-main 自身无需宿主机 `CAP_SYS_ADMIN`，可运行于非特权容器。

**userns 可用性探测**：unprivileged user namespace 本身是内核攻击面（近年容器逃逸 / 内核提权 CVE 有相当一部分入口在 userns 打开的代码路径），加固发行版默认限制或禁用（如 `kernel.unprivileged_userns_clone`、AppArmor userns restriction）。启动时探测可用性：不可用 → 明确告知用户 tradeoff——放开 userns 会扩大自身机器的内核攻击面，换取沙箱执行层可用；由用户显式选择放行或拒绝运行核心工具，不静默降级、不默默拒绝。

### 7.2 部署层

服务软件产出安全领域，专门用于跑「被测软件」本身（用户自己的代码/软件），不是 agent 自己的工具调用。风险模型与执行层不同——执行层跑的是 agent 的工具调用（受白名单约束的固定命令），部署层跑的是任意第三方软件，因此不适用执行层「Linux 原生隔离已够用」的结论。

#### 技术形态

- **不分平台统一走完整 VM**（包括 Linux 在内），不用 namespace + seccomp + Landlock 机制——跑任意未知软件的风险面与「执行几条审计命令」不是一个量级，参考 GitHub CI 每个 job 用一个全新 VM 的逻辑
- **生命周期**：一次性，类 GitHub CI——跑完销毁，不留状态、不做常驻复用；与执行层 VM（macOS/Windows，按需拉起 + 空闲超时销毁）的生命周期策略相互独立，即使底层技术组件相同也不共用生命周期管理

| 平台 | 部署层 VM 技术 | 与执行层的关系 |
| --- | --- | --- |
| macOS | 复用执行层已用的 `Virtualization.framework`（`github.com/Code-Hex/vz`） | 共用同一套管理组件 |
| Windows | 复用执行层已用的 Hyper-V（`github.com/Microsoft/hcsshim`） | 共用同一套管理组件 |
| Linux | **Firecracker**（KVM-based microVM，Go 侧 `firecracker-go-sdk`） | 执行层无 VM 组件可复用，Linux 部署层单独引入 |

选 Firecracker 的理由：专为「一次性、跑完销毁」的短生命周期场景设计（AWS Lambda / CI runner 同类场景的标准选型），启动速度快（约 125ms 级），与部署层「类 GitHub CI 每个 job 一个新 VM」的生命周期策略天然契合；同为 KVM-based，不需要额外的完整 QEMU 设备模拟开销。

三个平台的部署层对 Go 上层暴露统一接口，屏蔽底层技术差异，与执行层「Rust 沙箱代码跨平台复用、只是运行容器不同」的设计思路一致。

**安全边界**：部署层 VM 纳入主沙箱统一安全管理——生命周期、状态监控（停止 / 卡住 / 失控）、打穿响应（逃逸 / 资源滥用 / 反向控制 → 禁止 + 留档 + 通知）与执行层子沙箱同一套逻辑。VM 内被测软件被攻破（模拟攻击成功）是任务结果，不是主沙箱的异常判断职责；攻击路径报告由深挖引擎产出。

**降级路径与执行层保持一致**：部署层所需的 VM 组件（Firecracker / Virtualization.framework / Hyper-V）因权限或系统策略不可用时，直接拒绝执行涉及部署层的功能（软件产出安全的模拟攻击环节直接不可用），明确告知用户；不做弱隔离降级——弱隔离下跑模拟攻击类操作风险跟不隔离没有本质区别。Linux 上 Firecracker 依赖 KVM（`/dev/kvm`）：无 KVM 的云 VPS / 容器环境部署层直接不可用，为已知依赖前提，不提供用户态替代——qemu-user 等非全虚拟化方案不足以承担「跑任意未知软件」的隔离要求。

#### 模拟攻击拓扑

深挖阶段对部署层跑起来的软件发起模拟攻击时：

- **攻击方运行在执行层沙箱内**，复用现有 `sandbox-cmd` 等执行层机制，不为攻击方单独建一套沙箱
- 攻击方所在的执行层沙箱此时**网络出口白名单临时收窄为只对目标部署层 VM 开放**，攻击流量走网络层打进部署层 VM，贴近真实攻击场景
- 该临时收窄规则仅作用于本次模拟攻击的会话/任务范围；其余场景下执行层沙箱的网络规则维持「四维白名单」所定义的正常规则不变
- **时序不变量**：收窄规则先下发并**确认生效**（从攻击方沙箱做连通性验证：目标 VM 可达、原白名单目标不可达），再放行攻击方开始执行——不允许收窄与攻击开始异步并发，避免收窄生效前攻击方仍持有原白名单窗口期；模拟攻击任务结束后恢复原四维白名单规则

具体的临时白名单收窄机制（生效范围、与四维白名单规则体系的关系）待细化。

【待细化】被测物如何进 VM、网络平面、与任务状态机的关系、产物如何出 VM。

## 8. 安全模型

### 8.1 护栏总则

judge、白名单、沙箱这套护栏是引擎级强制约束，对所有工具调用一律生效，不因工具来源（官方工具 / 标准 MCP / 用户自写文档工具）而有差别对待。**注：本条约束针对「本地执行」——远端机器巡检（§4.1）的远端执行段不受执行层沙箱约束，其安全边界由「受控 SSH 用户 + 机器在信任清单内」承担，见「巡检引擎」远端执行模型。**

信任边界总则：

| 边界 | 信任判定 | 详述 |
| --- | --- | --- |
| 本机 UI（CLI / TUI / GUI / Web UI） | 本机即信任：进程内调用（同一二进制）、无网络协议；Web UI 仅监听本机 loopback | 「整体拓扑」「通信协议」 |
| IM 接口 | 外部网络入口：凭据校验 + 会话归属校验，默认不可信 | 「定时任务与通知」 |
| LLM API | 出站网络：凭据由用户配置，模型输出视为不可信输入 | 「模型接入 runner」 |
| MCP 工具 | 用户自担责任的外部工具，调用统一过白名单 + judge | 「工具体系」 |
| 远程机器（SSH 添加） | 用户自担的受信机器：远端执行不受执行层沙箱约束（以目标机器 SSH 用户权限运行），信任前提 = 机器在用户显式添加的清单内 + 凭据加密存储 | 「巡检引擎」 |
| 被测软件（部署层 VM 内） | 任意未知软件：VM 强隔离，攻破 = 任务结果而非主沙箱异常 | 「部署层」 |
| 下载的 kernel / rootfs | 供应链风险：下载通道与签名校验（信任根） | 「跨平台打包」 |

### 8.2 四维白名单

| 维度 | 控制内容 | 通配符支持 |
| --- | --- | --- |
| 文件路径 | 读/写/执行 分开控制，限定在 Workspace 内；单文件大小上限（实现期定义） | `*` / `**` |
| 网络出口 | IP:端口白名单（nftables 按 IP/端口过滤）、流量上限（实现期定义）；规则可写域名，系统解析为 IP 后下发（域名仅配置便利，安全边界是解析后的 IP:端口）。**用户添加巡检机器（SSH）时询问是否将该机器 IP:端口加入白名单，确认后写入规则**（加机器是交互动作，当场询问；白名单内的远程机器无人值守巡检可直连，无需每轮授权） | 精确匹配（IP 段支持 CIDR） |
| 可执行文件 | 允许运行的二进制白名单，通过 Mount Namespace 实现（非 seccomp 路径过滤） | 精确匹配 |
| 系统调用 | seccomp 级限制 | 精确匹配 |

通配符匹配在 `canonicalize()` 之后进行，防止路径穿越绕过规则。可执行文件和 syscall 不支持通配符，防止 `py*` 等模糊匹配引入攻击面。DDNS 域名走单独检测机制，见下方「DDNS 域名检测」。

预设规则包含禁止访问云 metadata 接口（`169.254.0.0/16`）等通用防护，具体预设清单见「授权与放权」节的白名单管理。

**网络白名单边界声明**：白名单是目的地级（IP:端口），不承诺内容级检测——放行域名内的合法流量（如 pip / npm / git 拉取）中夹带的恶意内容不在本层防护范围，依赖包供应链安全；缓解手段为流量上限（cgroups）与网络行为审计信号（上传体积异常等，具体信号待细化）。

**白名单信任模型（姿态 A）**：白名单 = 预授权，命中即执行——白名单内操作不设内容级 / 来源级阻断（如投毒内容诱导的白名单内调用不在此层拦截），收容由执行层沙箱兜底（见「执行层」收容型防御）；残余风险：白名单内操作的恶意利用不落入 judge / 白名单防护范围，依赖沙箱边界与事后审计。此姿态为有意设计——产品核心是沙箱收容而非阻止模型发起调用。

**DNS 解析白名单**：规则可写域名、解析为 IP 后下发，解析请求本身也纳入白名单——只允许向内置固定 resolver 发起 DNS 查询（限流限包），防止自由 DNS 查询被当作隐蔽信道（DNS tunneling）绕过目的地级白名单。

#### DDNS 域名检测

域名白名单按「解析为 IP 后下发」处理，但 DDNS（动态域名）的 IP 会漂移，静态快照会过期：新 IP 不在规则内导致访问被拒（可用性），旧 IP 易主后规则仍放行（安全性）。为此对 DDNS 域名启用专门检测：

- **判定**：内置常见动态域名后缀表（`duckdns.org` / `no-ip.com` / `ddns.net` / `freedns` 等）作默认判据；添加域名白名单时若命中自动提示「是否为 DDNS 分类」，用户可确认或归入普通域名；其余域名默认按普通处理，用户可手动标记为 DDNS。DDNS 分类命中后该规则进入检测流程，普通域名不启用检测、零噪音
- **检测**：会话建立时解析下发；会话存续期内每 3 分钟重解析比对 IP 集合（检测仅覆盖会话内漂移，跨会话每次重新解析下发，天然刷新，无持久规则）
- **IP 变更处理**：检测到 IP 变化 → 宿主机本地 UI 询问「是否为正常变更」。确认 → 更新规则；否认 → 维持旧规则并告警
- **无人值守场景**（systemd timer / launchd / 任务计划程序，无 UI）：**默认拒绝新 IP 连接、旧规则维持、任务照常跑完**，变更留待下次交互会话确认——绝不静默跟随 IP 变更（安全向 agent 权限可能很高，自动放行 DNS 劫持不可接受）

#### 防逃逸措施

| 攻击向量 | 防御措施 |
| --- | --- |
| 路径穿越（`../../etc/passwd`） | 所有路径先 `canonicalize()` 解析为绝对路径，再比对白名单 |
| 符号链接 / 硬链接攻击 | follow symlink / 解析硬链接后校验目标路径是否在白名单内 |
| `/proc/self/fd` 引用绕过 | `/proc` 限制挂载（pid namespace + hidepid），已打开的 fd 无法越过白名单指向白名单外文件 |
| `/proc/1/root`（宿主根文件系统） | pid namespace 隔离，沙箱内 pid 1 不是宿主 init；`/proc` 仅挂载受限视图 |
| 环境变量注入（`LD_PRELOAD` / `LD_LIBRARY_PATH` / `PATH`） | 沙箱启动时清理环境变量，只保留白名单内的变量 |
| `unshare` / `setns`（进入宿主 namespace） | seccomp 禁止 |
| `keyctl`（内核 keyring 窃取凭据） | seccomp 禁止 |
| `io_uring`（绕过 seccomp 过滤） | seccomp 禁止 `io_uring_setup` |
| `mount` / `pivot_root` / `chroot`（篡改挂载边界） | seccomp 禁止 |
| `/proc` / `/sys` 信息泄露 | mount namespace 内屏蔽或 remount 为空 |
| 设备节点 / FIFO 伪造 | 白名单外设备不挂载进沙箱；seccomp 禁止 `mknod` |
| 子进程逃逸 | 子进程继承同等沙箱限制，不可通过 fork/exec 突破 |
| fork 炸弹 / 内存耗尽 / 磁盘填满 | cgroups 进程数上限、内存上限、CPU 配额；文件大小上限 |
| 内核漏洞（CVE） | 沙箱不承诺防御未知内核漏洞，依赖系统内核及时更新；被攻破后的处置见「sandbox-main 职责」打穿响应 |

#### 判断流程

Go toolproxy 从 SQLite 实时读取规则构造安全上下文（规则快照 + workspace 路径 + 一次性授权描述符），随 gRPC 请求传入 sandbox-main，再透传至子沙箱；子沙箱内依次执行路径规范化、一次性授权校验、四维白名单检查，通过后在 seccomp 受限环境下执行，拒绝则返回原因触发用户授权流程。

### 8.3 授权与放权

Agent 请求的操作不在白名单内时，系统向用户发起授权请求。

交互模式（用户可配置）：

| 模式 | 选项 |
| --- | --- |
| 简洁模式 | 允许 / 拒绝 |
| 完整模式 | 允许（此次）/ 拒绝 / 加白名单并允许 / 加黑名单并拒绝 |

**「允许（此次）」的处理**：Go 在重新发起的 gRPC 请求中携带一次性授权描述符，描述符绑定本次操作的工具名和资源标识；子沙箱校验描述符与当前请求一致后才跳过白名单检查，不一致则按正常流程处理；描述符仅在单次请求生命周期内有效，不写入 SQLite，不影响规则体系。

**完全放权模式（本次对话级）**：面向巡检 / 深挖场景（以及其他任何场景）设计的独立机制，与白名单/黑名单体系并列，不是「加白名单」的变体。只有一条触发路径：

- 用户在授权弹窗中选择「本次对话完全放权」
- 需连续两道警告确认才生效：**第一次弹窗默认高亮「确认」，第二次弹窗默认高亮「取消」**（防止用户无意识连续点同一位置直接放权）
- 两次确认之间必须间隔一定时间，间隔阈值可配置，**默认 3 秒**（防手抖 / 防脚本化连续点击）
- 生效后 UI 用红色持续显示「当前为全放权模式」，提醒用户 LLM 也会犯错，需自行留意安全问题
- 作用域为当前对话（session 级），对话结束自动失效。全放权期间写入的文件按正常 workspace 规则落盘（session workspace 内），**生命周期归 workspace 清理策略管、与放权模式无关**——需要长期保留的由用户显式移入全局 workspace；放权只豁免白名单检查，不豁免 workspace 边界（见「Workspace 隔离」）

白名单 / 黑名单管理：

| 项目 | 说明 |
| --- | --- |
| 预设白名单 | 内置通用安全规则（基础工具、常见路径） |
| 用户白名单 | 用户自行添加的信任规则 |
| 黑名单 | 命中即拒绝，不再弹窗询问 |
| 作用域 | 每条规则可独立设置为 session 级或全局级 |
| 默认作用域 | 黑名单和白名单均可配置默认作用域（session 级或全局级） |
| 外部管理 | 白名单和黑名单均支持外部增删改（API / 配置文件） |

匹配优先级：

```text
黑名单（命中即拒） → 白名单（命中即放行） → 未匹配（触发用户授权弹窗）
```

### 8.4 Workspace 隔离

#### 目录结构

```text
/sandbox/workspaces/default/                       ← 全局级，跨 session 持久化
/sandbox/workspaces/sessions/{session_id}/          ← session 级，任务结束可清理
```

单机单用户场景不按 `user_id` 分目录，只有全局 workspace 与 session workspace 两级。

#### 访问规则

| 场景 | 行为 |
| --- | --- |
| Agent 默认操作 | 限定在当前 session workspace 内 |
| 访问全局 workspace | 需通过白名单授权 |
| 访问 workspace 外路径 | Rust 直接拒绝，无论白名单如何配置 |

层级关系：**workspace 边界绝对优先（Rust 强制）> workspace 内白名单细化 > 未命中触发授权**——白名单只在 workspace 内生效，任何白名单配置都不能越过 workspace 边界；「完全放权」同样豁免不了 workspace 边界（见「授权与放权」）。

#### 生命周期

| 级别 | 创建时机 | 清理策略 |
| --- | --- | --- |
| 全局 workspace | 首次使用时创建 | 用户主动删除 |
| session workspace | 新建 session 时创建 | 可配置：session 结束后自动清理 / 保留 N 天 / 手动清理 |

#### 挂载方式

sandbox-main 在新的 User Namespace + Mount Namespace 内将 workspace bind mount 后执行 `pivot_root`，子沙箱进程的文件系统根目录即为 workspace；白名单内的可执行文件同样通过 bind mount 显式挂载，不在白名单内的二进制物理上不可达。

## 9. 状态机

```text
                    用户输入
                       ↓
    ┌───────────── IDLE ←──────────────────────────────┐
    │                  ↓                                │
    │             THINKING                              │
    │           （模型推理中）                            │
    │            ↙        ↘                             │
    │     生成文本响应   需要工具调用                      │
    │        ↓              ↓                            │
    │   RESPONDING    TOOL_CALLING                       │
    │        ↓        （两类工具分发：见下方说明）            │
    │        ↓              ↓                            │
    │        ↓        白名单未命中？                      │
    │        ↓         ↙        ↘                       │
    │        ↓    命中放行   AWAITING_AUTH                │
    │        ↓        ↓     （等待用户授权）              │
    │        ↓        ↓      ↙        ↘                 │
    │        ↓        ↓   授权通过   授权拒绝             │
    │        ↓        ↓      ↓         ↓                │
    │        ↓        ↓      ↓    返回拒绝原因给模型      │
    │        ↓        ↓      ↓         ↓                │
    │        ↓     EXECUTING  ←────────↓                │
    │        ↓    （子沙箱 / Go内置 执行）                 │
    │        ↓         ↓                                │
    │        ↓    AGGREGATING                            │
    │        ↓   （结果汇聚，判断是否继续）                │
    │        ↓      ↙        ↘                          │
    │        ↓  需要继续    本轮完成                      │
    │        ↓     ↓           ↓                         │
    │        ↓  THINKING    RESPONDING                   │
    │        ↓                  ↓                        │
    │        └──────────────→ IDLE ──────────────────────┘
    │
    │  *** 任意状态均可被打断 ***
    └──────── INTERRUPTED ←── 用户发送 interrupt
                   ↓
              所有子 goroutine 级联取消
                   ↓
                 IDLE
```

| 状态 | 说明 | 退出条件 |
| --- | --- | --- |
| `IDLE` | 空闲，等待用户输入 | 收到用户消息 |
| `THINKING` | 模型推理中，流式接收 token | 模型返回完整响应（文本或工具调用） |
| `TOOL_CALLING` | 解析工具调用请求，维护工具调用队列，串行处理；按工具类型分发（见下），队列清空后转 AGGREGATING | 分发完成，进入对应路径 |
| `AWAITING_AUTH` | 操作未命中白名单，等待用户授权 | 用户响应（允许/拒绝/加白名单/加黑名单）；或超时 → 按「拒绝」处理（fail-closed，见下） |
| `EXECUTING` | 工具执行中（子沙箱 / Go 内置，取决于工具类型） | 执行完成或超时 |
| `AGGREGATING` | 汇聚工具结果，判断是否需要继续调用模型 | scheduler 决策：继续 → THINKING / 完成 → RESPONDING |
| `RESPONDING` | 生成最终响应，推送给 CLI/TUI/GUI/Web UI | 推送完成 |
| `INTERRUPTED` | 用户打断，级联取消所有子任务 | 取消完成，回到 IDLE |

**TOOL_CALLING 两路分发**：

```text
TOOL_CALLING
  ├── 辅助工具（get_time / get_quota 等）
  │     → Go toolproxy 内置直接执行
  │     → 无白名单检查，无用户授权
  │     → 直接进入 EXECUTING → AGGREGATING
  │     → 能力边界锁定：能力固定、只读、无用户可控参数、返回不含宿主机敏感信息
  │       （侦察面受工具自身定义约束，投毒最多让 LLM 多调用几次，无增量信息增益）
  │
  └── 核心工具（read_file / write_file / run_terminal_command）
        → Go toolproxy → Rust sandbox-main → 对应子沙箱
        → 白名单四维检查
          → 命中放行 → EXECUTING（子沙箱执行）
          → 未命中 →（PreToolUse 启用）judge 裁决
                      → allow → EXECUTING（子沙箱执行）
                      → deny / 不确定 → AWAITING_AUTH
                   →（PreToolUse 未启用）→ AWAITING_AUTH
                      → 授权通过 → EXECUTING
                      → 授权拒绝 → 返回拒绝原因给模型
```

辅助工具能力边界原则：不读文件、不跑命令、不访问网络、返回固定只读信息（当前时间 / 用量状态等）；凡是可能携带宿主机敏感信息的工具（如系统信息）不纳入辅助工具集，需此类能力一律走核心工具进沙箱 + 白名单。新加入的辅助工具需经同类审查（能力是否固定只读）。

巡检 / 深挖复用核心工具同一套状态机（`TOOL_CALLING` / `EXECUTING`），可基于中间结果追加工具调用；具体工具集见「工具体系」章。

**judge 检查点与状态机的关系**：`PreToolUse`（核心工具未命中白名单分支）与 `PreResponse`（`AGGREGATING` 后、`RESPONDING` 前）均为转移内的同步子步骤、无外部等待，不引入新状态；区别于 `AWAITING_AUTH`——后者等待人工响应，需 `session_state` 持久化与超时，八状态定义与转移条件不变，judge 仅决定既有转移走哪条分支。

**AWAITING_AUTH 超时**：默认 **fail-closed**——超时按「授权拒绝」处理，返回拒绝原因给模型，不静默放行、不卡死任务；无人值守场景与「巡检引擎」跳过语义一致（跳过并汇报）。超时时长可配置（默认值待细化）。

## 10. 引擎决策层

scheduler 的决策逻辑统一由 Go 驱动：状态机流转、工具调度、上下文管理、主/子代理协调。

### 10.1 模型绑定

按实际调用场景直接绑定模型：

| 场景 | 触发方 | 模型绑定方式 |
| --- | --- | --- |
| 主任务（巡检分析 / 深挖对话） | scheduler 主流程 | 全局默认模型，用户可在设置中覆盖 |
| judge 检查点（`PreToolUse` / `PreResponse`） | scheduler judge 子步骤 | 独立绑定，见「配置管理」`review.pre_tool.model` / `review.pre_response.model` |
| 子代理（fan-out） | scheduler 子代理调度 | 继承全局默认模型，用户可单独覆盖子代理模型 |

用户可随时在 session 内手动切换当前对话使用的模型，覆盖上述默认绑定。切换作用于本 session 的主任务模型绑定；judge 独立绑定不受影响；子代理继承当前 session 的主任务模型；持久化为 session 级，不覆盖全局默认。具体绑定项的 SQLite 字段与配置 UI 待细化。

**Failover 策略**（用户可配置）：

| 策略 | 行为 | 适用场景 |
| --- | --- | --- |
| 立即停止 | 模型 API 失败后直接返回错误 | 用户希望严格控制模型选择 |
| 告警后自动切换 | 失败后告警，等待 15 秒，自动切换至备用模型 | 兼顾用户感知与可用性 |
| 直接切换 | 失败后静默切换至备用模型，继续执行 | 优先保证任务不中断 |

- 备用模型由用户在设置中配置（failover 链，全局共享）
- scheduler 读取 failover 链和当前任务所需能力后随调用传入 runner，runner 内部完成切换决策，scheduler 不感知 failover 内部过程
- 切换时向 CLI/TUI/GUI/Web UI 推送 `model_failover` 事件

### 10.2 上下文管理

#### 加载策略

上下文恢复时按 `position DESC` 读取消息，累加 `token_count`，达到**模型窗口的 80%** 停止，剩余 20% 留给新消息和模型响应。`token_count` 在消息入库时一次性计算，采用语言感知估算，可替换为精确分词器（如 tiktoken-go）；估算系数待细化。基准为模型声明窗口（`ContextWindow`）的 80%，非「窗口减固定开销」——固定开销（system prompt / 工具定义等）在加载时计入 token_count，统一从预算内扣除。

#### 截断与压缩

上下文超出 80% 窗口时，按用户配置（`context_strategy`）处理：

| 策略 | 行为 |
| --- | --- |
| 截断 | 保留头部（系统 prompt + 最早几条）+ 尾部（最近 N 条），丢弃中间段 |
| 压缩 | 将中间段发给模型压成摘要，摘要替换原消息，额外消耗一次模型调用 |

默认截断，用户可切换为压缩。压缩调用失败时 fallback 截断（按截断策略处理）并附 `error` 事件告知用户，不阻塞任务。

#### 工具结果分块

`read_file` 读取大文件时默认分块返回：首次调用返回第一块 + 元信息（`total_size` / `has_more`），模型按需携带 `offset` 参数再次调用取下一块，复用原工具，无需新增接口。用户可切换为截断模式（超过阈值直接截断并告知模型）。

#### max_turns

Agent 执行轮次上限，防止模型陷入死循环。

**轮次定义**：一次模型 `Complete` 调用为一个 turn（状态机中每回到一次 `THINKING` 即新的一轮）。单轮问答（无工具调用）耗 1 turn；带工具调用的深挖耗多个 turn。以下内容**不计入** turn 数：failover / 重试（属容错，非决策轮次）、judge 检查点（同步子步骤）。

**默认值**：39（13 × 3），可配置范围 **26～52**（13 × 2 ～ 13 × 4），39 为中位数。13 为作者偏好基数，无功能含义。

**达到上限**：若正在 `EXECUTING`（工具执行中），等当前工具跑完再收尾，不留半截副作用；随后把当前已有结果推给用户，附 `error` 事件说明原因，不直接崩掉任务。

**子代理预算**：子代理独立计数，继承主代理配置但使用子代理默认值 **13**（13 × 1），可配置范围 13～26；与主代理预算互不占用，避免单次 fan-out 多个子任务时总消耗失控。

### 10.3 子代理派发

Go scheduler 判断需要并行任务后，从完整对话提取全局约束并显式注入，拆解子任务，fan-out 启动独立 goroutine；每个子代理上下文四部分：① System Prompt ② 全局约束 ③ fan-out 前最近 N 条消息（占剩余窗口 20%）④ 子代理自身工作记录。

- 子代理为 Go scheduler 内独立 goroutine，天然支持后期拆成独立进程扩展
- 子代理上下文不继承主代理完整上下文；全局约束由主代理显式注入，不依赖物理位置截断
- 每个子代理默认使用全局默认模型，用户可单独覆盖（见「模型绑定」）
- 用户打断 → Go 向所有子 goroutine 广播 cancel
- gRPC 连接从池中获取，不与特定子代理绑定
- judge 检查点（`PreToolUse` / `PreResponse`）不是 fan-out 子代理：只复用 scheduler 的 goroutine 并发与 runner 的单次 `Complete` 设施，不继承上述四段子代理上下文，也不参与结果汇聚

子代理失败处理：fan-out 中某个子代理失败时，收集所有成功子代理的结果，将失败子代理的错误原因一并交回主代理，主代理根据具体失败原因自行决策（重试、跳过或报错给用户），不在架构层硬编码处理策略。

### 10.4 配置管理

所有运行时**业务**配置存入 SQLite，不依赖配置文件；启动引导仅处理两件文件态前提——数据目录路径与主密钥生成 / 读取（本地文件，首次运行自动生成，见「凭证安全」）。引导项是进程启动前提（SQLite 文件本身位于数据目录内），不属于「运行时配置」，与本节「配置入 SQLite」不冲突。

配置来源与优先级：

```text
启动引导（仅进程启动必需：数据目录路径、主密钥文件读取/生成）

本地配置（运行时配置，所有业务逻辑相关，存 SQLite）
  → Provider API Key（本地加密文件存储，见「凭证安全」）
  → 模型绑定（场景 → 模型映射，见「模型绑定」）
  → Failover 备用链
  → 限流 / 重试策略参数
  → 白名单 / 黑名单规则
  → 用户偏好设置（context_strategy / max_turns / auth_timeout_sec 等）
  → 执行审查配置（review 块）
```

配置读取与缓存：SQLite 是本地文件，读取延迟低，缓存策略是否有必要待细化；安全相关规则（白名单/黑名单、API Key）无论如何都应保持实时读取，不缓存。

**执行审查配置（review）**：`review` 配置块存 SQLite，控制两个 judge 检查点，两者默认关闭：

| 配置键 | 默认值 | 说明 |
| --- | --- | --- |
| `review.pre_tool.enabled` | `false` | `PreToolUse` 检查点开关，关闭时维持「未命中即弹窗」行为 |
| `review.pre_tool.model` | 空（继承全局默认模型） | `PreToolUse` 裁决所用 judge 模型；空 = 用全局默认模型，保证强制启用时有模型裁决 |
| `review.pre_response.enabled` | `false` | `PreResponse` 检查点开关 |
| `review.pre_response.model` | 空（继承全局默认模型） | `PreResponse` 裁决所用 judge 模型；空 = 用全局默认模型 |
| `review.pre_response.on_fail` | `warn` | 产出不合格的处置：`retry`（回 `THINKING` 重试，受 `max_turns` 约束）/ `warn`（附告警直接 `RESPONDING`） |

- `enabled` / `on_fail` 即时生效（每次进入检查点时实时读取）
- 例外：巡检 / 深挖场景下 `PreToolUse` 强制启用，`review.pre_tool.enabled` 的配置值对该场景不生效
- `auth_timeout_sec`（授权超时，见「状态机」AWAITING_AUTH 超时）默认值待细化，实现时必须有默认值——fail-closed 行为依赖它

## 11. 模型接入 runner

两套 Adapter：Anthropic + OpenAI（含 OpenAI 兼容 Provider）。未来若要接自托管模型，走 OpenAI 兼容 `baseURL` 复用 OpenAIAdapter，不单独开 Adapter。Go runner 内部按 Adapter 分组，统一对上层暴露相同接口。

每个 Adapter 目录包含两个文件：

- `adapter.go`：初始化、调用实现；通过调用 `runner/auth/` 子包获取凭证，不自行处理鉴权逻辑
- `modelspec.go`：该 Provider 下所有模型的能力声明

`runner/auth/` 是统一鉴权子包，提供 `APIKey` 相关基础函数；所有 Adapter 均调用此层获取凭证，不自行实现鉴权。

### 11.1 Adapter 统一接口

所有 Adapter 实现同一个 interface，`runner.go` 通过此接口调用，不感知具体 Provider 实现。

| 方法 | 说明 |
| --- | --- |
| `Complete` | 核心方法：将统一格式的请求翻译为 Provider 特定的 API 调用，将 Provider 的 SSE 响应翻译为统一事件流返回；所有调用均为流式，非流式响应视为单 chunk 流 |
| `Models` | 数据访问方法：返回本 Adapter 的静态能力声明（单个 Provider 的能力来源）；跨 Adapter 的汇聚与统一查询见「能力声明」节 |

每次 `Complete` 同时返回响应元信息（速率余量、配额重置时间、429 建议等待秒数），从 Provider 响应头中提取，`runner.go` 据此追踪各 Provider 的速率状态。

### 11.2 能力声明

各 Adapter 的 `modelspec.go` 是内部数据源，声明本 Provider 下所有模型的能力；`runner.go` 汇聚所有 Adapter 的声明，对外暴露查询接口；`scheduler` 只通过 `runner.go` 获取模型信息，不直接接触各 Adapter。

`runner.go` 对外暴露两个查询方法：按 ModelID 查单个模型信息、列出所有模型；`scheduler` 只感知模型能力（是否支持 Streaming / ToolCall / Vision，以及 ContextWindow 大小），不感知 Provider 身份——换模型、加模型、failover 切模型，`scheduler` 决策逻辑零改动。

### 11.3 鉴权方式

| 类型 | 说明 | 适用 Provider |
| --- | --- | --- |
| `api_key` | API Key，本地加密文件存储（见「凭证安全」） | Anthropic / OpenAI（含 OpenAI 兼容 Provider） |

### 11.4 OpenAI 兼容 Provider

新增 OpenAI 兼容 Provider 仅需注册其 baseURL，不改 Adapter 代码；`modelspec.go` 仅声明 OpenAI 官方模型能力，与 Provider 路由解耦。非 OpenAI 官方模型的能力声明：**保守默认 + 手动覆盖**——默认按最低能力（保守上下文窗口、无 ToolCall / Vision 假设），用户在配置中添加模型时按模型 ID 声明实际能力；能力声明跟随模型 ID 而非 Provider，同一模型跨 Provider 复用同一声明。不采用运行时探测（费 token、Provider 行为不一致、能力随版本漂移）。

| 提供商 | baseURL | 鉴权 |
| --- | --- | --- |
| OpenAI | `api.openai.com/v1` | `api_key` |
| DeepSeek | `api.deepseek.com/v1` | `api_key` |
| Kimi（Moonshot） | `api.moonshot.cn/v1` | `api_key` |
| MiniMax | `api.minimax.chat/v1` | `api_key` |
| GLM（智谱） | `open.bigmodel.cn/api/paas/v4` | `api_key` |
| xAI Grok | `api.x.ai/v1` | `api_key` |
| Mistral | `api.mistral.ai/v1` | `api_key` |
| Groq | `api.groq.com/openai/v1` | `api_key` |
| Perplexity | `api.perplexity.ai` | `api_key` |
| Ollama | `localhost:11434/v1` | `api_key`（可选，有值走鉴权，无值跳过） |
| SiliconFlow | `api.siliconflow.cn/v1` | `api_key` |
| OpenRouter | `openrouter.ai/api/v1` | `api_key` |
| Together | `api.together.xyz/v1` | `api_key` |
| Qwen（通义） | `dashscope.aliyuncs.com/compatible-mode/v1` | `api_key` |
| Doubao（豆包） | `ark.cn-beijing.volces.com/api/v3` | `api_key` |
| Hunyuan（混元） | `api.hunyuan.cloud.tencent.com/v1` | `api_key` |
| Baichuan（百川） | `api.baichuan-ai.com/v1` | `api_key` |
| Yi（零一万物） | `api.lingyiwanwu.com/v1` | `api_key` |

除 Ollama 外，其余 Provider 的 `api_key` 均为必填；Ollama 本地模型无鉴权时留空即可。

## 12. 存储与 data-dir

| 组件 | 用途 |
| --- | --- |
| SQLite | 会话历史、任务状态、配置、白名单/黑名单规则、审计日志，单文件随程序数据目录存放（API Key 走独立本地加密文件方案，见「凭证安全」，不入 SQLite） |
| 磁盘文件 | 记忆层 `AGENTS.md`、Skills 层 `SKILL.md`、运行日志（轮转文件），均不入 SQLite |
| 读写入口 | SQLite 统一由 Go `db` 包管理，Rust 不直连；磁盘文件读写路径走 profile 层 |

- 记忆 / skills / 运行日志三类高频或需人工可读的内容单独走磁盘文件，不挤占核心数据库的备份与完整性校验开销

**`<data-dir>` 整树布局（初稿，待细化）**：

```text
<data-dir>/
  config/      # 用户可编辑配置（白名单/黑名单规则、检测清单等）
  db/          # SQLite 单文件（会话/消息/任务/审计/规则/报告）
  memory/      # 记忆层：MEMORY.md / USER.md / YYYY-MM-DD.md 日笔记
  skills/      # SKILL.md
  workspaces/  # workspace 目录（全局 default + sessions/，见「Workspace 隔离」）
  logs/        # 运行日志（轮转文件）
  secrets/     # 主密钥文件（0600）+ API Key 加密文件；不随备份走（见「凭证安全」）
  backups/     # SQLite 备份
  vm-cache/    # 下载版内核 + rootfs 缓存
```

**schema 概览（初稿，待细化）**：

| 表 | 关键字段 | 关系 |
| --- | --- | --- |
| `session` | id、session_state、created_at | 1:N message / task |
| `message` | id、session_id、role、content、token_count、position | N:1 session |
| `task` | id、session_id、type（inspect / dig）、status | N:1 session |
| `audit` | id、ts、actor、action、detail | 独立追加写 |
| `rule` | id、type（whitelist / blacklist）、dimension、pattern、scope（session / global） | 独立表 |
| `report` | id、task_id、path、generated_at | N:1 task；md 落盘、此表记元数据 |

### 凭证安全

API Key 涉及用户真实金钱消耗，泄露直接造成经济损失，必须在存储、传输、使用全链路做到最小暴露。

单机分发场景下不用系统 keychain（macOS Keychain / Windows Credential Manager / Linux Secret Service）——keychain 分平台实现，且 Linux Secret Service 依赖桌面环境的 daemon，无头服务器上通常不可用，跟本产品「服务器为主力部署场景、需要支持无人值守定时跑」的定位冲突。确定为**本地加密文件统一方案**：不依赖平台差异，跨平台实现一致：

- 算法：AES-256-GCM（认证加密，同时保证机密性和完整性）
- 密文采用单字段存储：`base64(nonce || ciphertext || tag)`，nonce 12 字节、tag 16 字节（Go `crypto/cipher` GCM 默认长度），`Seal`/`Open` 对称完成拼接与拆分，整体走标准 base64 编码后写入存储字段
- key 长度校验（AES-256 要求 32 字节）统一放在调用方（`runner/auth/` 读取主密钥之后），不下沉到加解密函数内部，避免每次加解密调用重复校验
- **主密钥来源**：首次运行自动生成，写入本地文件（权限 `0600`），之后自动读取，无需用户交互。排除「用户口令派生主密钥」方案——巡检需要支持 systemd timer / launchd / Windows 任务计划程序这类无人值守定时触发，口令派生会导致定时任务卡在解锁步骤，与「能自动化报警」的产品目标冲突。安全性代价：主密钥安全性完全依赖文件系统权限，钥匙和锁在同一台机器上——**此风险明确接受**：root 攻破场景下产品在宿主机上的全部防护均已失守，密钥加密只防「磁盘 / 备份副本离线泄露」层面的风险，不防 root 现场读取。缓解：`secrets/` 目录不随备份走（备份文件不含主密钥，泄露备份 ≠ 泄露密钥）；密钥与业务数据分文件存放

**runner/auth/ 读取链路**：每次读取密文，解密后返回 `[]byte`，请求完成后主动清零；不缓存，API Key 变更立即生效。

**通知渠道凭据**（SMTP 密码 / Telegram bot token / webhook secret）：走同一套本地加密文件方案（AES-256-GCM + 主密钥），不入 SQLite、不落明文（见「定时任务与通知」）。

**远程机器 SSH 凭据**（巡检通过 SSH 添加的机器时使用）：走同一套本地加密文件方案（AES-256-GCM + 主密钥），不入 SQLite、不落明文。添加远程机器时登记凭据（密钥 / 口令，密钥以加密文件存 `secrets/`、口令同样加密存储），随该机器的 per-tool 配置在沙箱内取用；与通知渠道凭据同一加密链路，风险模型一致（见本段开头）。

## 13. 通信协议

| 链路 | 协议 | 方向 | 连接策略 |
| --- | --- | --- | --- |
| CLI / TUI / GUI / Web UI ↔ core | 进程内直接调用（同一二进制） | 双向 | 不需要网络协议 |
| Go ↔ Rust（Linux） | gRPC（双向 stream） | Go 主动调用 | 连接池 + 自动重连 |
| Go ↔ Rust（macOS / Windows） | gRPC over virtio-vsock（VM 内外） | Go 主动调用 | 待设计 |
| Go ↔ SQLite | 进程内文件 I/O | Go 读写 | 待细化（连接管理方式） |
| Go → 外部模型 API | HTTPS / SSE | Go 发起 | HTTP 连接池 + 重试 |

### gRPC 连接池

- Go 维护 Rust 连接池
- 请求从池中获取连接，用完归还；连接池管理连接复用，不缓存查询结果
- 连接异常时自动重建，上层调用无感知

### 事件流协议（Go 内部 pub/sub 事件总线）

`runner` / `scheduler` 发布事件到统一的内部 pub/sub 总线（进程内 channel，不需要 WebSocket 断线重连这类网络层设计），各消费方各自订阅、按自己的机制处理，互不感知：

| 消费方 | 处理方式 |
| --- | --- |
| CLI | 订阅后直接同步渲染（简单场景可以是 `switch event.Type` + 打印） |
| TUI | 订阅后桥接为 `tea.Msg` 塞进 bubbletea 消息循环 |
| GUI（Wails） | 订阅后调用 Wails `EventsEmit` 桥给 JS 前端 |
| 通知 dispatcher | 订阅后按事件类型过滤（如 `error`/`auth_request`），推送到已配置的通知渠道（见「定时任务与通知」） |
| 审计日志 | 订阅后无条件全量写 SQLite |

总线本身只定义三样共用的东西：事件类型全集（已见 `tool_call` / `tool_output` / `auth_request` / `model_failover` / `rate_limited` / `error` 等，完整清单与总数实现时统一登记、待细化）、发布接口、订阅注册机制；事件被消费后如何处理完全是各订阅方自己的事，新增消费方只需新写一个订阅者，不用改总线或其他订阅者的代码。无人值守定时任务场景下没有 UI 挂载，但通知 dispatcher 与审计日志两个订阅方始终在线，事件依然完整可用。

**事件 → 状态机映射（已知事件）**：

| 事件 | 触发点 / 关联状态 |
| --- | --- |
| `tool_call` | `TOOL_CALLING`（工具调用请求） |
| `tool_output` | `AGGREGATING`（结果汇聚） |
| `auth_request` | `AWAITING_AUTH`（授权请求，通知 dispatcher 过滤用） |
| `model_failover` | 模型调用过程（THINKING 内，切换时推送） |
| `rate_limited` | 限流策略「通知并停止」时推送，暂停当前任务 |
| `error` | 任意状态出错 |

实现时以本表为登记基线，新增事件同步补表，保证总线事件与状态机转移可对账。

## 14. 容错策略

### 模型 API 容错（runner 内部职责）

runner 内部处理三层容错：限流 → 重试 → failover，scheduler 只感知最终结果。

#### 限流（Rate Limiting）

runner.go 持续追踪各 Provider 的速率状态，数据来源为每次 `Complete` 返回的 `ResponseMeta`（从 Provider 响应头 `x-ratelimit-remaining` / `x-ratelimit-reset` / `retry-after` 提取）。

**接近限额时的预防策略**（用户可配置）：

| 策略 | 行为 | 适用场景 |
| --- | --- | --- |
| 提前切换 | 剩余配额低于阈值（可配置，默认 10%）时，主动切换至备用 Provider——**与 failover 同一机制**，走同一备用链、消耗链上模型（见「模型绑定」Failover） | 优先保证任务不中断 |
| 减速继续 | 降低发送频率，等配额自然恢复 | 不想切换 Provider |
| 不干预 | 不做预防，等 429 真正触发后再处理 | 简单场景 / 单 Provider |

**429 触发后的处理策略**（用户可配置）：

| 策略 | 行为 | 适用场景 |
| --- | --- | --- |
| 等待后重发 | 按 `retry-after` 等待配额重置，再将排队请求一次性发出 | 不在意延迟，想省 failover |
| 通知并停止 | 向 CLI/TUI/GUI/Web UI 推送 `rate_limited` 事件，暂停当前任务，等待用户决策 | 用户希望手动控制 |
| 直接 failover | 不等待，立即触发 failover 切换至备用 Provider | 优先保证任务不中断 |

#### 重试（Retry）

单次请求失败后，runner 在当前 Provider 内重试，重试耗尽后才考虑 failover。

**可重试错误**：

| 错误类型 | 重试 | 说明 |
| --- | --- | --- |
| 5xx | 是 | 服务端临时故障 |
| 超时 | 是 | 网络抖动或 Provider 响应慢 |
| 网络断开 | 是 | 临时网络中断 |
| 429 | 否 | 交给限流策略处理，不在重试层处理 |
| 401 / 403 | 否 | 鉴权问题，重试无意义 |
| 400 | 否 | 请求格式错误，重试无意义 |

**重试策略**：

- 重试间隔：指数退避，可配置基础间隔和最大间隔；默认值待细化（实现时必须有默认值——退避是重试路径的必需参数，不能留空）
- 最大重试次数：默认 3 次（可配置）
- 重试期间不切换 Provider，在当前 Adapter 内完成

#### Failover

重试耗尽 或 限流策略决定切换时，触发 failover，策略同「模型绑定」的 Failover 表。

**触发条件**（从重试层升级到 failover 层的边界）：

| 条件 | 行为 |
| --- | --- |
| 重试次数耗尽（连续失败） | 触发 failover |
| 限流策略决定切换 | 触发 failover |
| 401 / 403 鉴权错误 | 直接 failover，不经重试 |
| Provider 连续 N 次超时（可配置，默认 N = 3，与最大重试次数一致，待细化） | 触发 failover |

**备用链**：

- 备用模型由用户在配置中设置（failover 链，全局共享），scheduler 读取后随调用传入 runner
- 切换时 runner 内部通过模型接入层的查询接口查询备用模型能力，按 `ModelCapability` 匹配当前任务所需能力；不满足则跳过，取下一个
- 切换时向 CLI/TUI/GUI/Web UI 推送 `model_failover` 事件（附带原模型与目标模型信息）
- 备用链耗尽后返回错误给 scheduler

优先级：成功直接返回 → 429 走限流策略 → 5xx/超时/网络断开走重试 → 重试耗尽走 failover → 401/403 直接 failover → 400 直接返回错误。

### Rust 沙箱故障

| 阶段 | 行为 |
| --- | --- |
| 第一次失败 | 在当前沙箱实例内重试一次 |
| 第二次失败 | 重启沙箱进程，重试一次 |
| 第三次失败 | 返回错误给 Agent，Agent 可选择放弃或调整策略 |
| gRPC 连接断开（Go 崩溃或网络中断） | sandbox-main 级联终止所有正在运行的子沙箱；sandbox-main 自身保持运行，等待 Go 重连后接受新请求 |

沙箱重启期间，Go toolproxy 排队等待（可配置超时时间），不丢弃请求。

### SQLite 故障

日志模式启用 **WAL**（相比默认 rollback journal 模式，崩溃恢复能力和并发读写性能都更好）。

内置本地备份机制：保留 **3 份**滚动备份，作为产品自身的救急手段（不依赖用户自行搭建备份链路，但也不替代用户自己的完整备份方案）。备份触发时机为**进程启动时检查上次备份时间，超过间隔就补一次**，不绑定巡检任务、不走独立固定周期。多进程并发启动（定时任务多进程场景，见「部署与 MVP 分期」）时备份加进程级互斥（文件锁），同刻仅一个备份执行，其余跳过本次（下次启动再补）。

| 场景 | 行为 |
| --- | --- |
| 文件不可写 | 重试几次（带退避），仍失败才报错退出；不做只读降级——适合磁盘满这种可能瞬间恢复的场景 |
| 数据库损坏 | 启动时跑 `PRAGMA integrity_check`；失败则自动从最新一份备份恢复，并明确告警用户发生了什么 |

### Go 进程崩溃

| 场景 | 行为 |
| --- | --- |
| 崩溃检测 | Go 重启后扫描 session_state，心跳时间戳超过阈值的记录为孤儿任务 |
| 孤儿处理 | 将孤儿任务标记为 INTERRUPTED，在消息记录中写入系统消息记录中断原因（含中断时的状态和工具名） |
| 清理 | 删除对应 session_state 记录，会话可正常重连使用 |
| 不自动重试 | 工具执行可能有副作用（文件写入、命令执行），自动重试存在重复执行风险；由用户重连后自行决定是否重新发起 |

### judge 检查点容错

judge 检查点本身可能失败或超时，按「安全优先于可用、可用优先于阻塞」分别降级：

| 检查点 | 失败/超时行为 | 说明 |
| --- | --- | --- |
| `PreToolUse` | fail-closed | judge 失败或超时降级为 `AWAITING_AUTH`（升级人工），绝不静默放行 |
| `PreResponse` | fail-open | judge 失败附告警直接 `RESPONDING`，不因 judge 故障阻塞用户结果 |

- `PreResponse` 的 `on_fail = retry` 在裁决判定产出不合格时回退至 `THINKING` 重试。judge retry **不计入 turn 计数**（同「重试属容错、非决策轮次」），但任务整体仍受 `max_turns` 上限兜底（达上限按既有规则把已有结果推给用户）；子代理同理——judge retry 不计入子代理预算（默认 13），防无限循环靠同一兜底
- judge 降级行为同样写入审计日志

### 通用原则

- 所有重试和降级行为均写入审计日志
- 向 CLI/TUI/GUI/Web UI 推送对应事件（`error` / `model_failover`），用户可感知
- 不静默吞掉错误：即使自动容错成功，也通知用户发生了什么

## 15. 日志体系

两类日志用途、写入频率、存储介质都不同，分开设计：审计日志管「安全相关操作留痕」，运行日志管「排障用的运行细节」。

### 审计日志

#### 设计原则

安全产品的核心要求：**用户可以回查 AI 对系统做了什么**。所有涉及安全和资源消耗的操作均需记录。

#### 记录范围

| 类别 | 记录内容 |
| --- | --- |
| 沙箱操作 | 每次工具调用的请求内容、白名单判断结果（通过/拒绝/待授权）、执行结果、耗时；`sandbox-cmd` 中解释器子进程内的文件操作不单独记录（见「执行层」） |
| 用户授权 | 每次授权弹窗的触发原因、用户选择（允许/拒绝/加白名单/加黑名单）、作用域（session/全局） |
| 白名单/黑名单变更 | 规则的增删改操作、操作来源（用户手动/授权弹窗/API）、变更前后的值 |
| Skill / 记忆变更 | Skill 的创建/修改/批准/驳回（含 `approved_for_unattended` 状态变化）、记忆层 `AGENTS.md` 的写入，操作来源与变更前后内容 |
| 模型调用 | 使用的模型、Provider、token 用量（input/output）、延迟、是否触发 failover |
| 容错事件 | 沙箱重试/重启、模型 failover、SQLite 访问异常 |
| 执行审查（judge） | `tool_review`（`PreToolUse` 裁决、理由、所用 judge 模型）、`output_review`（`PreResponse` 裁决与处置） |
| 会话生命周期 | session 创建/恢复/结束、workspace 创建/清理 |

> **token 口径说明**：审计日志的 token 用量取自 Provider 响应（计费真实值），与消息表中的 token_count（本地语言感知估算，仅用于上下文窗口管理）口径不同，数值不一致属正常，不可直接对账。

#### 存储与查询

- 审计日志写入 SQLite（Go `db` 包统一管理）
- 按时间范围、事件类型建索引，支持查询
- 保留策略可配置（默认保留 90 天，可调整）
- CLI：内置 `audit` 子命令，支持按 session / 时间 / 操作类型过滤（过滤字段对齐「存储与 data-dir」schema 概览的 `audit` 表：ts / actor / action）
- GUI（可选）：审计日志面板
- **与备份关系**：审计 90 天保留针对活库（到期清行）；备份轮换只轮换备份副本、不清活库审计行；从备份恢复时审计数据回退到快照时刻（任何备份恢复的固有损失，非本层引入）

### 运行日志

排障用的运行时细节（例如某次巡检/深挖具体加载了哪些 skill、skill 内容如何影响本轮推理），跟审计日志分开管理：

- **存储**：独立轮转日志文件，不进 SQLite——高频、单条价值低的写入不该跟审计日志抢同一条写入路径，也不该拖累 SQLite 的备份与 `PRAGMA integrity_check` 开销
- **不进云端加密备份**：只在本机保留——已接受的取舍：本机损坏则运行日志全失；取证价值集中在审计日志（已入 SQLite 且随备份走），运行日志仅排障用、丢失可接受。进阶方案（跟随备份）为后续方向，不在实现范围
- **保留策略**：比审计日志短，轮转大小 / 周期 / 压缩待细化（暂定：单文件 10 MB 轮转、按天分片、旧文件 gzip 压缩）
- **查看方式**：待细化，倾向自制 CLI/Rust 小工具直接读日志目录，不追加数据库依赖

## 16. 定时任务与通知

垂类定位要求巡检 / 深挖能无人值守自动化运行并主动报警，不只是交互式工具，因此需要一套独立于 CLI/TUI/GUI 的调度与通知机制。

### 定时任务执行模式

产品自身不常驻 daemon，复用系统原生调度设施触发一次性执行（跟 fleet-audit 现行的 `.service`(oneshot) + `.timer` 模式一致）：

| 平台 | 调度设施 |
| --- | --- |
| Linux | systemd timer |
| macOS | launchd |
| Windows | 任务计划程序（Task Scheduler） |

产品提供跨平台的注册 / 卸载定时任务子命令，封装对应平台调度 API 的差异；实际巡检执行是外部调度器拉起的一次性子进程，跑完退出，不需要产品自己实现进程守护、开机自启动、崩溃重启这套逻辑。

### 通知渠道

插件化多渠道 adapter 架构：定义统一的「发送通知」接口，底层挂多个具体实现（Telegram / 飞书 / 钉钉 / Email / Webhook 等）。**默认内置支持 Email / Telegram / Webhook 三个渠道，三者可同时启用（并发推送），也可只开其中一个**；各渠道需配置对应凭据后生效（Email 需 SMTP 凭据、Telegram 需 bot token、Webhook 需接收端），不强制自建 webhook 接收端。

**配置读取链路**：通知渠道配置（启用哪些渠道、目标地址）是业务配置存 SQLite；渠道凭据（SMTP 密码 / bot token / webhook secret）走「凭证安全」的本地加密文件方案。定时任务拉起的一次性进程按「引导项（数据目录路径 + 主密钥）→ 读 SQLite 通知配置 → 解密凭据 → 发送通知」读取，无需额外引导项。

### 双向交互（IM 回复）

巡检 / 深挖报警推送到 IM 渠道后，用户可以在 IM 里回复继续深挖，但边界严格限定：

- IM 回复只能路由回**触发本次报警的那个已存在会话**，作为该会话的下一轮输入（例如「继续深挖那个端口」）；不能从 IM 发起全新任务——发起新任务仍需回到 CLI/TUI/GUI
- **会话定位**：报警通知携带触发会话的标识（session_id），IM 回复附该标识路由回对应会话。开启 IM 回复功能时，产品运行一个专属轻量 listener 进程接收回复（仅该功能启用时存在，非常驻全局 daemon）；未开启则不运行——与「产品无常驻 daemon」不冲突，listener 生命周期绑定 IM 回复功能开关
- **默认关闭**，用户需显式开启才能使用 IM 回复功能（更保守的默认值）
- 授权边界：IM 回复只能推进白名单内、本来就不需要授权的操作；一旦涉及需要新授权的操作（写文件、执行不在白名单内的命令等），一律打回「完全放权模式」的双重确认机制，且**只能在宿主机本地界面确认**，IM 侧无法完成授权确认——安全原则是「只相信宿主机」，防止远程凭据泄露后被用来篡改系统。用户不在宿主机时，需要新授权的操作按「状态机」AWAITING_AUTH 超时 fail-closed 处理（超时按拒绝、任务跳过并汇报继续），不会永久卡住
- 两层强制防护（不可关闭）：
  1. **发送方身份硬绑定**：只信任配置阶段绑定的具体 chat ID / 用户 ID，其他任何账号发消息给 bot 一律忽略，不进入指令路由
  2. **来源单独打标审计**：IM 触发的操作在审计日志中单独标注来源，与宿主机本地触发的操作区分开，便于事后审查

## 17. 跨平台打包

### 构建矩阵

2 种架构（x86_64 / arm64）× 3 个平台（Linux / macOS / Windows）= 6 个构建产物，CI 用矩阵构建。

### Rust 沙箱嵌入 Go 二进制

- 构建顺序：先按目标平台交叉编译 Rust 沙箱产物，放入约定目录，再执行 `go build`；`go:embed` 本身不做跨语言交叉编译
- 用 `//go:build` 平台标签分别 embed 对应目录的产物，不把全部平台产物塞进单一构建，避免体积膨胀
- Linux 上可选 `memfd_create` + `execveat`（`golang.org/x/sys/unix`）免落盘执行沙箱二进制，运行时不留可篡改的落盘文件；macOS / Windows 无此机制，走「释放到临时目录（`0700`）→ 执行 → 退出清理」的通用路径。**威胁模型**：篡改面来自「磁盘上可被单独替换的沙箱二进制文件」——`go:embed` 内嵌使沙箱二进制与主二进制同源同签名、运行时无独立落盘文件可替换（篡改面从两个文件收窄到一个）；免落盘进一步消除「运行期间落盘副本被替换」的持久化攻击面。此机制不防「主进程已沦陷」场景（此时沙箱二进制是否落盘已无关紧要）——防的是主进程仍可信时、磁盘上独立沙箱文件被持久化替换
- macOS / Windows 的沙箱产物是配合 microVM 的 Linux 原生二进制（跑在 VM 内），不是各平台专属实现

### 跨平台虚拟化方案（执行层，macOS / Windows）

| 平台 | 方案 | Go 侧库 |
| --- | --- | --- |
| macOS | `Virtualization.framework`（macOS 11+ 原生，Apple Silicon 上接近原生性能） | `github.com/Code-Hex/vz` |
| Windows | Hyper-V | `github.com/Microsoft/hcsshim`（与 Docker/containerd 在 Windows 上用的同一套库），若检测到已装 WSL2 可考虑复用 |

已知代价与决策：

- **打包体积**：分两个发行变体——下载版（二进制保持小体积，首次运行时下载精简 Linux 内核 + rootfs）和自带版（内核 + rootfs 随二进制嵌入，离线可用，体积更大）；用户按场景选
- **下载版供应链校验**：下载走 HTTPS 固定源（官方发布地址 / 分发 CDN），下载后校验内置哈希 / 签名——信任根为产品发布密钥，与产品二进制升级同一信任根；校验失败拒绝使用。自带版（go:embed 内嵌）无下载供应链问题，随二进制签名走
- **降级路径**：Hyper-V / Virtualization.framework 因权限或系统策略不可用时，**直接拒绝运行**，核心工具与攻击模拟直接不可用，明确告知用户当前环境不满足要求；不做弱隔离降级——弱隔离下跑攻击模拟类操作，风险跟不隔离没有本质区别，不符合「沙箱内安全模拟攻击」的产品前提。Windows 上以虚拟化平台组件（Virtual Machine Platform / Windows Hypervisor Platform）可用为判据，与 WSL2 / Docker Desktop 共享同一栈；不可用时直接拒绝运行并明确告知
- **VM 常驻策略**：按需拉起 + 空闲超时销毁；`sandbox-read` / `sandbox-write` / `sandbox-cmd` 三个子沙箱各自独立 VM，隔离粒度维持跟 Linux 原生方案一致，不退化为单 VM 共享三个子沙箱；空闲达到超时阈值后统一销毁释放内存，避免桌面场景下长期空占资源
- Linux 执行层保持原生 namespace + seccomp + Landlock 方案，不额外套 VM——Linux 本身就有强隔离机制可用，加一层 VM 是给主力部署平台增加不必要的开销（多一层 hypervisor、多几十上百 MB 常驻内存），VM 只用于 macOS/Windows 执行层缺原生隔离机制的场景。**此结论仅适用于执行层**；部署层不分平台统一走 VM，包括 Linux 在内——见「部署层」

### 部署规划

单二进制分发，不做 Docker Compose 多容器服务端部署。

| 产物 | 构建方式 | 说明 |
| --- | --- | --- |
| CLI / TUI 二进制 | Go 交叉编译，`go:embed` 嵌入对应平台 Rust 沙箱产物 | 6 个目标（linux/darwin/windows × amd64/arm64） |
| GUI 二进制（可选） | Wails 构建 | 依赖平台 webview 运行时 |
| macOS / Windows 沙箱依赖 | 精简 Linux microVM 内核 + rootfs，分下载版（首次运行时下载）与自带版（随二进制嵌入）两个发行变体 | 见本节 |
| Linux 部署层 VM 依赖 | Firecracker 内核 + rootfs（与 macOS / Windows 执行层共享同一套下载 / 嵌入机制与校验） | 见「部署层」 |

用户拿到对应平台的单个可执行文件即可运行，不需要 Docker、不需要额外起服务、不需要外部数据库。

## 18. 部署与 MVP 分期

分期事项按依赖序排定（两安全域决定 MVP，MVP 决定并发，巡检能力决定漏洞源）：

| 优先级 | 事项 | 依赖 |
| --- | --- | --- |
| 1 | 两个安全域 v1 承诺（两安全域同属 v1，软件产出安全仅覆盖代码层面 L1→L3；运行层面模拟攻击与部署层 VM 属 v2；深挖的画像排序、坏习惯素材耦合属 v2） | 最上游——决定 MVP 范围与后续所有分期 |
| 2 | MVP 第一条可跑通路径（A 手动巡检闭环打底：触发巡检 → 脚本集经沙箱执行层运行 → LLM 查漏补缺 → 报告落盘 → 查看，一次验证执行层 / 白名单 / SQLite / runner 全链路；C 定时无人值守闭环收口：A 之上注册系统原生调度 + 通知渠道，依赖 A；深挖 L1 可并行、不作第一条） | 取决于 1（见上） |
| 3 | 单机并发模型 | 取决于 1 / 2——v1 是否含定时任务多进程决定模型（见下） |
| 4 | 漏洞情报源（CVE / 版本检测数据） | 取决于 1 / 2——巡检「版本落后检测」类脚本的数据依赖；若 v1 巡检不含该能力，此源推迟到 v1 后 |
| 5 | 升级与迁移（schema / 默认 skill 清单 / 完整性校验） | 发布前才需要 |

**单机并发模型：单实例互斥**——同一 data-dir 同时只允许一个 Meyrin 进程（进程锁）；定时任务拉起时若锁被占（交互会话活跃）则跳过本次并汇报，下次调度再跑。无 memory/ 日笔记写冲突、SQLite 保持单写者简单性；「产品无常驻 daemon、复用系统原生调度触发一次性执行」不变，只是同一时刻至多一个实例。

**【待交叉审查】**IM 回复 listener（§16）与单实例互斥的持锁交互：listener 持锁则 IM 回复期间所有定时任务跳过（无人值守停摆）；不持锁则并发碰 data-dir（单实例决策想消灭的冲突复活）。初稿阶段不锁方案，下次交叉审查时定（需联动 §14 备份文件锁）。

**漏洞情报源与巡检的依赖**：巡检的「版本落后检测」脚本需要 CVE / 版本数据作判断依据，此源未定时该能力不可落地；离线可用性（内网 / 无外网服务器也能用）是硬约束，源需支持本地数据导入或离线包。

**升级与迁移**：schema / 默认 skill 清单 / 嵌入沙箱与下载 rootfs 的完整性校验，与「跨平台打包」供应链校验同一信任根（产品发布密钥）。

## 19. 变更记录

### v0.1（2026-08-13 ~ 08-14）

**Established the initial architecture definition** — a single-machine, single-binary, cross-platform security agent built on three engines (inspect / dig / learn), a two-layer sandbox (execution + deployment), a whitelist security model, and unattended scheduled inspection.
**建立首版架构定义** — 单机、单二进制、跨平台的安全垂类 agent，以三引擎（巡检 / 深挖 / 学习）、两层沙箱（执行层 + 部署层）、白名单安全模型与无人值守定时巡检为骨架。

- **新增：** 定位与目标——纯安全垂类、单机单二进制、两安全域（运维安全巡检 + 软件产出安全深挖）
- **新增：** 三引擎——巡检（无人值守 + 远端机器 SSH 执行模型）、深挖（代码 L1→L3 + VM 模拟攻击）、学习（记忆分层 + skill 蒸馏 + 用户画像）
- **新增：** 两层沙箱——执行层（sandbox-main + read / write / cmd 三子沙箱）与部署层（Firecracker / Virtualization.framework / Hyper-V 完整 VM）
- **新增：** 安全模型——judge + 四维白名单 + workspace 隔离、DDNS 域名检测、授权与放权（含完全放权模式）
- **新增：** 状态机与引擎决策层（模型绑定 / 上下文管理 / 子代理 / 配置管理）、模型接入 runner、存储与凭证安全、通信协议、容错策略、日志体系
- **新增：** 定时任务与通知（Email / Telegram / Webhook）、跨平台打包、部署与 MVP 分期
