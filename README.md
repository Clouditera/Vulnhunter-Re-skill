# vulnhunter-reverse

VulnHunter 逆向分析插件（zcode 插件形态）：主会话交互与调度 + 按角色挂载技能的 subagent 执行，以 [REA](https://github.com/morluto/rea) CLI 作为分析工具层。

用户启用插件后：agent 确认逆向目标与分析方向 → 按需补全依赖 → 目标画像与路由 → 分主题静态分析/功能追踪 → 可选最小重建与验证 → 带证据分级与局限性的报告。

与 [vulnhunter-skill](https://github.com/Clouditera/vulnhunter-skill)（白盒审计）同族：audit 的眼睛是 Read/Grep，reverse 的眼睛是 REA。

## 能做什么

| 方向 | 内容 | 深度 |
|---|---|---|
| 功能逻辑提取（feature） | 搞清某功能的实现：入口 → 调用链 → 核心函数 → 逻辑说明，函数级证据 | 静态 / 深度原生（Ghidra） |
| 漏洞发现（vuln-hunt） | 先做功能分析找出输入面与 sink 点，再 hunt 提出假设、verify 主动证伪，confirmed 产出 finding（静态确认，未动态验证） | 静态 / 深度原生（Ghidra） |
| 行为画像（behavior） | 静态行为特征：网络指标、文件/注册表操作、持久化与混淆迹象 | 静态（动态沙箱为后续方向） |
| 自由探索（free） | 用户自定义问题的逆向追踪，同样要求证据闭环 | 静态 / 深度原生 |
| 最小重建（可选） | 已查清逻辑的可运行复刻 + 用例对照验证，如实记录差异 | 依赖已 resolved 的调查项 |

上表「深度」列中的「深度原生（Ghidra）」仅在 Ghidra 可用时提供；Ghidra 为**可选依赖**，未安装/不可用时原生 ELF/PE/Mach-O **自动降级为纯静态**（file/readelf/nm/strings 轻量事实兜底），不阻塞流程，能力缺口如实标注。

目标类型（REA 内置静态解析，零外部引擎）：JS/Electron（目录/ASAR）、.NET PE/CLI、Go/Rust 二进制、Unity IL2CPP metadata、Godot PCK、SQLite、GGUF/safetensors 等。原生 ELF/PE/Mach-O 的函数级反编译需要 Ghidra（深度模式，可选）。

上表为速览；各场景的详细输入输出与工具用法见下节。

## 支持场景

插件支持 5 种场景：feature / vuln-hunt / behavior / free 四种分析方向，外加可选的 reconstruct。所有场景共享同一入口（`/reverse <目标路径>`）与 `.reverse/` 产物结构。

### 场景 1：功能逻辑提取（feature，默认推荐）

搞清某个功能是怎么实现的，还原核心逻辑。

- **输入**：目标路径 + 关注功能点（如「登录鉴权」「IPC 通信」）+ 分析深度（static / deep）
- **输出**：函数级证据 + 逻辑说明（入口 → 调用链 → 核心函数 → 输入/处理/输出）；产物为 `notes/Qn-<主题>.md`（结论、实现逻辑分步、逐条带分级的证据清单、未决项），最终汇入 `report.md`
- **主要工具**：
  - REA：`analyze-javascript-application`（JS/Electron）、`inspect-managed-artifact` / `inspect-managed-members`（.NET）、`inspect-go-binary` / `inspect-rust-binary`、`inspect-il2cpp-metadata`（Unity IL2CPP）等
  - deep 深度下 Ghidra provider：`search` / `function` / `decompile` / `xrefs` / `trace`（有界静态追踪）
  - 大输出管理：`inspect-analysis-view`（摘要视图 + 分页投影，避免全量进对话）

### 场景 2：漏洞发现（vuln-hunt）

先做功能逻辑分析找出输入面与 sink 点，再基于输入→sink 提出假设并逐条证伪。

- **输入**：目标路径 + 分析深度
- **流程**：功能与攻击面分析（`notes/attack-surface.md`）→ hunt 提出假设（`hyps.jsonl`）→ verify 主动证伪（confirmed 写 `findings/<ID>/report.md`，refuted/inconclusive 写明依据）
- **主要工具**：与场景 1 相同的 REA 静态解析命令 + deep 下的 `decompile` / `xrefs` / `trace`（输入面与 sink 点定位）
- **边界**：confirmed = 静态证据链确认，报告必须声明「未动态验证」；不写武器化 EXP；动态 PoC 复现为后续方向

### 场景 3：行为画像（behavior）

静态行为特征梳理。

- **输入**：目标路径 + 分析深度
- **输出**：静态行为画像——网络指标（URL/IP/域名）、文件与注册表操作、持久化迹象、可疑导入、打包/混淆迹象；每条带证据分级
- **主要工具**：REA 静态解析 + 字符串/导入表搜索（`search`）；.NET 走 `inspect-managed-*`，模型文件走 `inspect-model-artifact`，数据库走 `inspect-sqlite-database`
- **边界**：动态沙箱为后续方向当前不做；静态画像不等于运行时行为

### 场景 4：自由探索（free）

用户自定义问题的逆向追踪（如「这个配置项在哪解析」）。

- **输入**：目标路径 + 用户自己描述的问题
- **输出**：同 investigate 笔记结构（结论 → 实现逻辑 → 证据清单 → 未决项），同样要求证据闭环
- **主要工具**：按问题自定义追踪路径，复用上述全部 REA 命令

### 场景 5（可选）：最小重建与对照验证（reconstruct）

把已查清的函数级逻辑复刻成可运行代码并验证一致性。仅在用户明确要求可运行复刻时触发，且依赖已 resolved 的调查项。

- **输入**：已 resolved 的调查项（notes 中有完整函数级证据）
- **输出**：`reconstruction/<主题>/` 下的重建代码 + `SPEC.md`（输入/输出/处理步骤/明确不覆盖的部分）+ 测试用例 + `verification.md`（用例数、通过数、逐处差异原因；结论分「已验证等价（范围限定）/ 部分一致 / 未通过」）
- **主要工具**：按场景选语言的工具链——原生函数用 C/C++（build-essential）、JS 模块用 TypeScript、.NET 用 C#（.NET SDK 8.0）；验证方法含伪代码/CIL 逐分支比对、（用户批准 dynamic 时）与原程序对拍、字节级对照
- **纪律**：不声称「完全等价」，只说「在 N 个用例/某范围内一致」

### 工具层总览

| 类别 | 工具 | 用途 |
|---|---|---|
| 核心分析引擎 | REA CLI（`rea-agents`，Node.js ≥ 22.19） | 约 28 类目标的静态解析：analyze / inspect-* / search / function / decompile / xrefs / trace / inspect-analysis-view |
| 深度反编译（可选） | Ghidra（+ JDK 21，`GHIDRA_INSTALL_DIR`） | 原生 ELF/PE/Mach-O 的函数级反编译（deep 深度 provider）；未安装时自动降级 static，不阻塞 |
| 系统工具 | `file`、`sha256sum`、`ls`/`du` | triage 阶段基础事实采集 |
| 重建工具链 | C/C++（build-essential）、TypeScript、.NET SDK 8.0 | reconstruct 场景的最小重建与验证 |
| 自检 | `rea doctor --provider ghidra --json`、`rea capabilities --json` | 开工前环境与能力核实 |

### 通用输入输出与贯穿纪律

- **统一输入**：目标路径 + 分析方向（feature/risk/behavior/free）+ 分析深度（static/deep/dynamic，dynamic 需用户明确批准且有隔离环境）+ 可选关注功能点
- **统一输出**：目标旁的 `.reverse/` 工作区（结构详见下文「产物结构」）
- **贯穿纪律**：目标只读、默认不执行；结论按 observation / derivation / inference / unknown 分级；安装依赖先征得同意；能力不可用时如实降级不假装完成；unresolved 必须带原因与已排除路径

## 安装与依赖

完整依赖清单（含各项安装命令与开工前自检清单）见 [DEPENDENCIES.md](DEPENDENCIES.md)。必装项：

- Node.js ≥ 22.19（REA 运行要求）
- REA：`npm install -g rea-agents`（临时使用可 `npx -y rea-agents@latest` 免安装）
- 重建工具链：C/C++（build-essential）、TypeScript（`npm install -g typescript`）、.NET SDK 8.0

按场景可选项：

- JDK 21 + Ghidra（约 500MB）：仅原生 ELF/PE/Mach-O 的深度反编译需要；配置 `GHIDRA_INSTALL_DIR` 后 `rea doctor --provider ghidra --json` 自检。未安装时原生目标**自动降级 static**（readelf/nm/strings 轻量事实兜底），不阻塞流程

**主会话会在开工前自检依赖，Ghidra 结论标准化为 `ghidra: available | unconfigured | unavailable` 写入 scope.md；任何安装动作都会先征得你的同意；深度能力不可用时如实降级，不会假装完成。**

## 快速开始

```
/reverse <目标路径>
```
agent 会依次确认：分析方向（功能提取/风险梳理/行为画像）、深度（纯静态/深度原生/动态）、关注功能点，然后开始工作。

## 产物结构

所有产物写入目标旁的 `.reverse/`（不修改目标文件，默认不执行目标）：

```
.reverse/
  scope.md               # 交互确认：目标/方向/深度/关注点
  target-profile.md      # 画像与路由（triage）
  evidence/              # REA 原始输出 JSON（大结果落盘）
  notes/                 # 分主题分析笔记（investigate）
  reconstruction/        # 重建代码 + SPEC + 用例 + verification.md（可选）
  report.md              # 最终报告（含证据索引、未决项与局限性）
  investigations.jsonl   # 调查项台账（pending/running/resolved/unresolved）
```

## 设计要点

- **主会话只调度不分析**：交互确认、依赖补全、派发与台账、完成条件把关
- **按角色挂载技能**：每个 subagent 只带自己的任务书（triage / investigate / hunt / verify / reconstruct / report）
- **证据分级**：observation / derivation / inference / unknown，每条结论有出处
- **完成条件闭合**：台账无 pending/running 且 report.md 生成；unresolved 必须有原因和已排除路径，不许无声消失
- **红线**：默认不执行目标、不改目标、安装先征同意、不虚构反编译与重建结论

## 边界（诚实声明）

- 纯静态形态不等于运行时行为确认；vuln-hunt 的 confirmed 是静态证据链确认，报告必须声明「未动态验证」
- 完成闸门靠主会话纪律执行（与 audit-lite 同取舍）；大目标的覆盖口径是「分析过的面」，不等于理解深度
- 动态行为沙箱与动态 PoC 复现为后续方向

## 参考与致谢

### REA —— 核心参考项目与工具层

本插件的分析能力由 **[REA (morluto/rea)](https://github.com/morluto/rea)** 提供（npm 包 `rea-agents`，MIT License）。REA 是一个本地运行的逆向分析引擎（MCP server + CLI），覆盖原生二进制、JS/Electron、.NET、APK、固件等约 28 类目标。感谢 REA 作者及其贡献者。

我们不仅使用 REA 作为运行时工具（通过 npm 安装，**未复制其代码**），方法设计也大量借鉴了它的现成机制：

- **目标路由**：triage 的路由表借鉴其官方 skill `reverse-engineer-anything` 的目标类型决策树
- **功能追踪（investigate）**：借鉴其 guided prompt `investigate_feature` 的调查骨架
- **重建验证（reconstruct）**：借鉴其 guided prompt `verify_reconstruction` 与 DX-Ball 案例的对照验证方法学
- **未知项审计（report）**：借鉴其 guided prompt `audit_residual_unknowns`
- **证据规范**：observation / derivation / inference / unknown 的分级与局限性声明，沿用其 Evidence 模型

参考文档：[REA CLI 与 Evidence 指南](https://github.com/morluto/rea/blob/main/docs/cli.md)、[REA guided MCP prompts](https://github.com/morluto/rea/blob/main/docs/mcp-prompts.md)、[REA showcases](https://rea.tools/showcase/)。

### 同族项目

- [vulnhunter-skill](https://github.com/Clouditera/vulnhunter-skill)：白盒审计姊妹插件，插件结构（清单/角色/命令）与主会话调度范式参照自它
- [agent-loom](https://github.com/Clouditera/agent-loom)：VulnHunter 引擎 SDK（本插件为其能力的轻量分发形态）

License: Apache-2.0（REA 为 MIT，仅作运行时依赖与方法参考，不产生代码许可冲突）
