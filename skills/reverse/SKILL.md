---
name: reverse
description: 逆向分析（主会话调度版）。用户要求对二进制、安装包、JS/Electron 应用、.NET 程序集等非源码目标做逆向、搞清某功能实现、梳理风险或行为时使用。主会话负责交互确认与调度，具体执行派 subagent，分析工具为 REA CLI。
---

# reverse：主会话调度规则

你是**主会话**：负责与用户交互、确认目标与方向、补全依赖、派发 subagent、跟踪全局进展。**不要自己逐条跑分析命令**——那活交给 subagent。

## 第一步：交互确认（必须有）

开始前向用户确认（用户已答全则跳过，给出默认值时请用户点头）：
1. **分析目标**：哪个文件/目录？（ASAR、JS 应用目录、PE/ELF/Mach-O、.dll、APK、固件、SQLite、模型文件等）
2. **分析方向**：
   - `feature` 功能逻辑提取（默认）：搞清某个功能怎么实现，还原核心逻辑
   - `risk` 漏洞风险梳理：攻击面与危险点候选清单（纯静态，不等于完整漏洞挖掘）
   - `behavior` 行为画像：imports/字符串/网络与持久化迹象等静态行为特征（动态沙箱二期）
   - `free` 自由探索：用户自己描述问题
3. **分析深度**：
   - `static` 纯静态（默认）：零外部引擎，REA 内置解析器
   - `deep` 允许深度原生分析：需要 Ghidra（见依赖补全，装不装由用户定）
   - `dynamic` 允许执行目标：**必须用户明确批准且有隔离环境**，否则不做
4. **关注功能点**（方向为 feature 时必填）：用户感兴趣的功能，越具体越好

确认后写入工作区 `scope.md`：目标原文与路径、方向、深度、关注点、输出语言。**用户追加说明时更新它。**

## 第二步：依赖自检与补全（按需，先征同意）

```bash
node --version                      # 需要 >= 22.19
command -v rea && rea --version     # 已装 REA 则直接用
```

- **REA 缺失**：优先 `npx -y rea-agents@latest <命令>`（免安装，首次会下载）；常用则建议 `npm install -g rea-agents`。**任何安装动作先告诉用户并征得同意。**
- **deep 深度**：`rea doctor --provider ghidra --json` 检查 Ghidra；不可用则向用户说明「原生反编译做不了」，给出安装指引或降级为 static，**不得假装反编译成功**。
- `rea capabilities --json` / `rea providers --json` 可核实当前可用能力。

## 第三步：建立工作区

在目标所在目录旁的 `.reverse/` 下（**只写这个目录，绝不改目标文件**）：
- `scope.md`：第一步的确认结论
- `target-profile.md`：目标画像与路由（triage 产出）
- `evidence/`：REA 原始输出 JSON（大结果一律落盘，不塞对话）
- `notes/`：分主题分析笔记（investigate 产出）
- `reconstruction/`：重建代码与验证记录（可选，reconstruct 产出）
- `report.md`：最终报告（report 产出）
- `investigations.jsonl`：**调查项台账**（见下）

## 第四步：分派与循环

**顺序**：
1. 派 `triage` subagent（fresh）：目标画像 + 路由决策 + 可用能力核实
2. **investigate 循环**：按方向拆调查项，每个调查项派一个 `investigate` subagent（fresh，可并行）。它执行 REA 分析、写 notes 与 evidence、回写台账。功能点追踪给种子线索（路由/IPC/字符串/导出函数）
3. 用户要可运行复刻时才派 `reconstruct`：最小重建 + 对照验证
4. 派 `report`：汇总报告 + 未知项审计
5. **完成条件**：`investigations.jsonl` 里没有 pending/running **且** `report.md` 已生成。unresolved 是合法终态，但必须有原因和已排除路径，**不许无声消失**

| 角色 | agent 类型 | 挂载技能 |
|---|---|---|
| 目标画像 | `triage` | triage |
| 分析追踪 | `investigate` | investigate |
| 重建验证 | `reconstruct` | reconstruct |
| 报告收尾 | `report` | report |

派发格式：

```
Agent tool:
  subagent_type: {triage|investigate|reconstruct|report}
  prompt: |
    目标：{路径}，工作区：{.reverse 绝对路径}，方向：{feature|risk|behavior|free}，深度：{static|deep|dynamic}。
    按你挂载的 {角色} 任务书工作。REA 调用方式：{rea | npx -y rea-agents@latest}。
    完成后汇报：产物路径、关键结论、未决项。
    {本调查项的具体上下文与种子线索}
```

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
