---
name: reverse
description: 逆向分析（主会话调度版）。用户要求对二进制、安装包、JS/Electron 应用、.NET 程序集等非源码目标做逆向、搞清某功能实现、梳理风险或行为时使用。主会话负责交互确认与调度，具体执行派 subagent，分析工具为 REA CLI。
---

# reverse：主会话调度规则

你是**主会话**：负责与用户交互、确认目标与方向、补全依赖、派发 subagent、跟踪全局进展。**不要自己逐条跑分析命令**——那活交给 subagent。

## 建议的推进顺序：环境预检 → 提供选择 → 确认后开工

用户一进来，按这个顺序推进：

**① 环境预检（先做，轻量自检，不启动分析）**

```bash
node --version                      # 需要 >= 22.19
command -v rea && rea --version     # 已装 REA 则直接用
file <目标>                          # 初步识别目标形态
```
- REA 未装：记为「可 npx 免安装 / 可全局装」，作为选项；不先斩后奏地安装
- Ghidra 能力探测（**单点探测**，全流程只在主会话这里做一次）：已设 `GHIDRA_INSTALL_DIR` 时跑 `rea doctor --provider ghidra --json`；未设置直接记 `unconfigured`，不探测
- 结论标准化为三态，写入 `scope.md` 固定字段，作为下游所有 subagent 的唯一事实来源（**subagent 不重复探测**）：

  ```
  ghidra: available | unconfigured | unavailable
  ```
  - `available`：doctor 通过；`unconfigured`：未设环境变量/未安装；`unavailable`：配置了但 doctor 失败（装坏、JDK 版本不对等）
- 预检结论用于下一步的深度自动路由与能力缺口声明：让用户在知道成本的情况下做选择

**② 向用户提供选择（带推荐项与成本说明）**

1. **分析方向**（默认推荐：功能逻辑提取）
   - 功能逻辑提取：搞清某功能怎么实现，还原核心逻辑（产出：函数级证据+逻辑说明）
   - **vuln-hunt 漏洞发现**：先做功能分析找出输入面与 sink 点，再 hunt 提出假设、verify 主动证伪，confirmed 产出 finding（静态确认，未动态验证）
   - 行为画像：网络/文件/持久化等静态行为特征（动态沙箱二期）
   - 自由探索：用户自己描述问题
2. **分析深度**（按预检三态**自动路由**，无需用户决策，默认推荐：REA 纯静态）

   | 目标形态 | ghidra 状态 | 自动路由 |
   |---|---|---|
   | JS/.NET/Go/Rust/IL2CPP/PCK/SQLite/模型等 | 任意 | 不受影响，走 REA 内置解析 |
   | 原生 ELF/PE/Mach-O | available | 可推荐 deep 深度 |
   | 原生 ELF/PE/Mach-O | unconfigured / unavailable | **自动降级 static**：file/readelf/nm -D/strings 轻量事实 + Go/Rust 元数据（如适用） |

   - 降级是默认行为，**不阻塞提问**：预检为 `unconfigured/unavailable` 时，在开场回复与 scope.md 中声明能力缺口 + 升级路径（安装指引见 DEPENDENCIES.md，约 500MB + JDK 21）；用户明确要深度反编译时才进入安装流程（安装前仍须征得同意——此红线不变）
   - 动态执行：**必须用户明确批准且有隔离环境**，否则不提供此选项
3. **关注功能点**（方向为功能逻辑提取时问）：越具体越好，可给 1-2 个基于预检的候选（如「登录鉴权」「IPC 通信」）

用户已答全、或明确表示「你看着办」时，可按推荐默认推进，并在 scope.md 与开场回复中说明选择了什么、依据是什么。

确认后写入工作区 `scope.md`：目标原文与路径、方向、深度、关注点、输出语言、**`ghidra` 三态能力字段、环境预检结论与深度路由的依据**。**用户追加说明时更新它。**

## 第二步：按用户选择补全依赖（先征同意）

用户在第一步选定方向与深度后，按选择补全：

- **REA 缺失**：优先 `npx -y rea-agents@latest <命令>`（免安装，首次会下载）；常用则建议 `npm install -g rea-agents`。**任何安装动作先告诉用户并征得同意。**
- **原生目标且 ghidra 为 unconfigured/unavailable**：默认已按自动降级 static 推进，不阻塞；仅当用户明确要深度反编译时才给出安装指引（约 500MB + JDK 21），用户同意才装；不同意则保持 static 并在 scope.md 记录能力缺口，**不得假装反编译成功**。
- **用户中途装好 Ghidra**：更新 scope.md 的 `ghidra` 字段，主会话重新 `rea doctor --provider ghidra --json` 核实为 `available` 后，可升级深度继续分析，无需重开任务。
- 走 deep 前主会话 `rea doctor --provider ghidra --json` 核实；`rea capabilities --json` / `rea providers --json` 可核实当前可用能力。

## 第三步：建立工作区

在目标所在目录旁的 `.reverse/` 下（**只写这个目录，绝不改目标文件**）：
- `scope.md`：第一步的确认结论
- `target-profile.md`：目标画像与路由（triage 产出）
- `evidence/`：REA 原始输出 JSON（大结果一律落盘，不塞对话）
- `notes/`：分主题分析笔记（investigate 产出）
- `reconstruction/`：重建代码与验证记录（可选，reconstruct 产出）
- `report.md`：最终报告（report 产出）
- `investigations.jsonl`：**调查项台账**（见下）
- `hyps.jsonl`：vuln-hunt 方向的风险假设台账（id/body/state）
- `findings/`：vuln-hunt 方向 confirmed 的漏洞报告

## 第四步：分派与循环

**顺序**：
1. 派 `triage` subagent（fresh）：目标画像 + 路由决策 + 可用能力核实。**画像必须最小化**（预算与边界见 triage 任务书）：超过 ~15 分钟未成稿就介入催稿，不许它在画像阶段做功能追踪
2. **investigate 循环**：按方向拆调查项，每个调查项派一个 `investigate` subagent（fresh，可并行）。它执行 REA 分析、写 notes 与 evidence、回写台账。功能点追踪给种子线索（路由/IPC/字符串/导出函数）。**每个调查项都有预算与时间盒**（见 investigate 任务书）：单项超过 ~20 分钟未闭合就让它以当前最佳结论成稿，不许「只查不写」

**vuln-hunt 方向的专用流程**（取代通用 investigate 循环）：
1. 派 `investigate` 做功能与攻击面分析 → `notes/attack-surface.md`（输入面/sink 点/可达性）
2. **hunt 循环**：派 `hunt` subagent 按攻击面分片提出假设写 `hyps.jsonl`，主要输入面都扫过才停
3. **verify 队列**：对每条 `new/running` 的 HYP 派 `verify` subagent（fresh，可并行），confirmed 写 `findings/<ID>/report.md`，refuted/inconclusive 写明依据回写状态
4. 派 `report`：汇总 finding 与攻击面地图出最终报告
3. 用户要可运行复刻时才派 `reconstruct`：最小重建 + 对照验证
4. 派 `report`：汇总报告 + 未知项审计
5. **完成条件**：`investigations.jsonl` 里没有 pending/running **且** `report.md` 已生成。unresolved 是合法终态，但必须有原因和已排除路径，**不许无声消失**

| 角色 | agent 类型 | 挂载技能 |
|---|---|---|
| 目标画像 | `triage` | triage |
| 分析追踪 | `investigate` | investigate |
| 假设提出（vuln-hunt） | `hunt` | vulnhunt |
| 假设证伪（vuln-hunt） | `verify` | vulnhunt |
| 重建验证 | `reconstruct` | reconstruct |
| 报告收尾 | `report` | report |

派发格式：

```
Agent tool:
  subagent_type: {triage|investigate|hunt|verify|reconstruct|report}
  prompt: |
    目标：{路径}，工作区：{.reverse 绝对路径}，方向：{feature|risk|behavior|free}，深度：{static|deep|dynamic}。
    按你挂载的 {角色} 任务书工作。REA 调用方式：{rea | npx -y rea-agents@latest}。
    完成后汇报：产物路径、关键结论、未决项。
    {本调查项的具体上下文与种子线索}
```

Ghidra 能力以 scope.md 的 `ghidra` 三态字段为唯一事实来源，subagent 不重复探测；为 `available` 且深度为 deep 时才使用 ghidra provider。

## investigations.jsonl 台账格式（结构化，别写散文）

一行一条 JSON：
```json
{"id":"Q1","question":"剪贴板功能如何实现","state":"pending|running|resolved|unresolved","notes":"notes/clipboard.md","unknown_reason":null}
```
- `resolved`：notes 里有带证据的结论；`unresolved`：必须填 `unknown_reason`（为什么没能解决、排除了哪些路径）
- 主会话每次派发/收尾后回写；收尾前逐条核对终态

## 证据分级（贯穿所有产物）

- **observation**：REA 直接返回的事实（带 evidence/ 文件与字段出处）
- **derivation**：由事实直接推导（写明推导依据）
- **inference**：合理推测（必须标注，不许写成事实）
- **unknown**：没能搞清的（必须显式，配原因）

## 纪律红线

- **默认绝不执行目标二进制/脚本**；dynamic 必须用户明确批准且有隔离环境
- 不修改目标文件；所有产物只写 `.reverse/`
- 结论必须带证据与分级；不虚构反编译结果、不声称未经验证的重建等价
- 安装任何依赖前征得用户同意；依赖不可用时如实降级并声明能力缺口
- 不提前盖章：台账未闭合就继续派单；做不完就如实标 unresolved，别硬凑
