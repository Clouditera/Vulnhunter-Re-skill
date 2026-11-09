# vulnhunter-reverse

VulnHunter 逆向分析插件（zcode 插件形态）：主会话交互与调度 + 按角色挂载技能的 subagent 执行，以 [REA](https://github.com/morluto/rea) CLI 作为分析工具层。

用户启用插件后：agent 确认逆向目标与分析方向 → 按需补全依赖 → 目标画像与路由 → 分主题静态分析/功能追踪 → 可选最小重建与验证 → 带证据分级与局限性的报告。

与 [vulnhunter-skill](https://github.com/Clouditera/vulnhunter-skill)（白盒审计）同族：audit 的眼睛是 Read/Grep，reverse 的眼睛是 REA。

## 能做什么

| 方向 | 内容 | 深度 |
|---|---|---|
| 功能逻辑提取（feature） | 搞清某功能的实现：入口 → 调用链 → 核心函数 → 逻辑说明，函数级证据 | 静态 / 深度原生（Ghidra） |
| 漏洞风险梳理（risk） | 攻击面与危险点候选清单（位置+疑点+证据），不做漏洞确认与 EXP | 静态 |
| 行为画像（behavior） | 静态行为特征：网络指标、文件/注册表操作、持久化与混淆迹象 | 静态（动态沙箱为后续方向） |
| 最小重建（可选） | 已查清逻辑的可运行复刻 + 用例对照验证，如实记录差异 | 依赖已 resolved 的调查项 |

目标类型（REA 内置静态解析，零外部引擎）：JS/Electron（目录/ASAR）、.NET PE/CLI、Go/Rust 二进制、Unity IL2CPP metadata、Godot PCK、SQLite、GGUF/safetensors 等。原生 ELF/PE/Mach-O 的函数级反编译需要 Ghidra（深度模式）。

## 安装与依赖

完整依赖清单（含各项安装命令与开工前自检清单）见 [DEPENDENCIES.md](DEPENDENCIES.md)。必装项：

- Node.js ≥ 22.19（REA 运行要求）
- REA：`npm install -g rea-agents`（临时使用可 `npx -y rea-agents@latest` 免安装）
- JDK 21 + Ghidra（深度原生分析；配置 `GHIDRA_INSTALL_DIR` 后 `rea doctor --provider ghidra --json` 自检）
- 重建工具链：C/C++（build-essential）、TypeScript（`npm install -g typescript`）、.NET SDK 8.0

**主会话会在开工前自检依赖，任何安装动作都会先征得你的同意；深度能力不可用时如实降级，不会假装完成。**

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
- **按角色挂载技能**：每个 subagent 只带自己的任务书（triage / investigate / reconstruct / report）
- **证据分级**：observation / derivation / inference / unknown，每条结论有出处
- **完成条件闭合**：台账无 pending/running 且 report.md 生成；unresolved 必须有原因和已排除路径，不许无声消失
- **红线**：默认不执行目标、不改目标、安装先征同意、不虚构反编译与重建结论

## 边界（诚实声明）

- 纯静态形态不等于完整漏洞挖掘或运行时行为确认；risk 方向产出的是候选清单
- 完成闸门靠主会话纪律执行（与 audit-lite 同取舍）；大目标的覆盖口径是「分析过的面」，不等于理解深度
- 动态行为沙箱、完整二进制漏洞挖掘（hunt/verify 流程）为后续方向

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
