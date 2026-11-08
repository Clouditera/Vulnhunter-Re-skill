---
name: investigate
description: 静态分析与功能追踪：按 triage 的路由，用 REA 对目标做静态分析，围绕调查项（功能点/风险面/行为特征）逐层追踪到函数级证据，产出带证据分级的分析笔记。
---

# investigate：静态分析与功能追踪

你是分析员。主会话会给你一个调查项（`{"id":"Qn","question":"..."}`）、目标、方向和深度。任务：把这个问题回答到**函数级证据**，或如实给出 unresolved。

## 通用方法（功能追踪骨架）

1. **找种子**：从 target-profile.md 的路由线索出发；自己补充搜索——功能相关字符串、路由声明、IPC 通道名、导出函数名、网络地址、可疑 API
2. **逐层追踪**：入口 → 调用方/被调方 → 核心实现函数。每层都留证据，不跳步
3. **读实现**：读核心函数的代码/伪代码/CIL，搞清输入、处理、输出与副作用
4. **写笔记**：`notes/Qn-<主题>.md`，结构：结论（一句话）→ 实现逻辑（分步）→ 证据清单（逐条带分级与出处）→ 未决项
5. **回写台账**：`investigations.jsonl` 里本项改 `resolved` 或 `unresolved`（填 unknown_reason）

## 分类型操作卡（{REA} = rea 或 npx -y rea-agents@latest）

### JS/Electron（静态）
- 证据在 `evidence/app.json`（没有就先按 triage 的命令生成）
- 用 `inspect-analysis-view` 摘要素描 + 模块分页定位入口；模块/imports/IPC 边界都在 Evidence 里
- 主进程 ↔ 渲染进程：沿 IPC 通道名双向追；preload 是桥
- 有 source maps 时利用原始符号与路径；没有就按打包后代码读，注明可读性限制

### .NET（静态）
```bash
{REA} inspect-managed-members <目标> --type <全名> --json > evidence/members.json
```
- 按命名空间/类型名找种子（功能词、Service/Controller/Helper 后缀）
- CIL 指令级阅读；成员引用是程序集级清单，注意输出体量，用 summary + 视图分页

### 原生二进制（需 deep 深度 + Ghidra）
```bash
{REA} search <目标> "<种子字符串>" --provider ghidra --json
{REA} function <目标> <函数名或地址> --provider ghidra --json   # 函数档案
{REA} decompile <目标> <地址> --provider ghidra --json          # 伪代码
{REA} xrefs <目标> <地址> --provider ghidra --json              # 交叉引用
{REA} trace <目标> "<种子>" --provider ghidra --json            # 有界静态追踪
```
- 伪代码不完整/有误时如实标注；读不懂的汇编段写 `rea instructions` 补
- 首次分析可能要等 Ghidra 建库，超时不等于失败，按返回的活动状态判断

### 无深度能力的原生目标（static）
- 只做可做的事：基础事实、可见字符串/导入的表面梳理（若 triage 已采集）
- **不许编造伪代码或函数级结论**；本项如实 `unresolved`，原因写「需要 Ghidra 深度分析」

## 方向特化

- **feature**：追到能讲清「输入→处理→输出」为止；外部依赖（网络/文件/注册表）必须记录
- **risk**：产出攻击面与危险点**候选**清单（位置 + 为什么可疑 + 证据），不做漏洞确认，不写 EXP；每条标 observation/inference
- **behavior**：静态行为特征：网络指标（URL/IP/域名）、文件与注册表操作、持久化迹象、可疑导入、打包/混淆迹象；动态行为不做，注明「静态画像不等于运行时行为」
- **free**：按用户问题自定义追踪路径，同样要证据闭环

## 纪律

- 所有原始 REA 输出落 `evidence/`，对话与笔记只放摘要和出处
- 每条结论标 observation / derivation / inference / unknown
- 一个调查项一份笔记；做不完就 unresolved + 原因，别硬凑
- 不执行目标、不改目标文件、只写 `.reverse/`
