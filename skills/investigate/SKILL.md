---
name: investigate
description: 静态分析与功能追踪：按 triage 的路由，用 REA 对目标做静态分析，围绕调查项（功能点/风险面/行为特征）逐层追踪到函数级证据，产出带证据分级的分析笔记。
---

# investigate：静态分析与功能追踪

你是分析员。主会话会给你一个调查项（`{"id":"Qn","question":"..."}`）、目标、方向和深度。任务：把这个问题回答到**函数级证据**，或如实给出 unresolved。

## 预算与边界（硬性，防失控）

- **先写笔记骨架，再逐步补全**：开始调查后 ~10 分钟内必须写出 `notes/Qn-<主题>.md` 初稿（结论占位 + 已确认事实 + 待查清单），之后每有进展就更新它；**不得连续查询而不写笔记**
- **查询预算**：本调查项的 REA 查询（search/function/decompile/xrefs/trace）**总数 ≤ 25 次**；大目标（>10MB 或 stripped）**先定位种子再针对性查询，不做全量扫描式穷举**。到预算仍未闭合，就以当前证据写 `unresolved`（unknown_reason 写清已排除路径与下一步建议）
- **时间盒**：单项超过 ~20 分钟未闭合，立即以当前最佳结论成稿（resolved 或 unresolved），不许「再查一个看看」无限延长
- **失败重试上限**：同一个函数/目标的反编译或查询**失败（含报错、空结果、invalid mapping）两次仍不成功，立即换路径**——读汇编代替伪代码、改查相邻函数、或直接声明 unknown 并写明失败原因。**禁止在同一失败操作上重复重试**，它计入查询预算且不得成为不闭合理由
- 遇到大目标首次建库等长耗时操作，先在笔记里登记「等待原因」，避免被误判失控

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

### 无深度能力的原生目标（static，ghidra 为 unconfigured/unavailable）

Ghidra 状态以 scope.md 的 `ghidra` 三态字段为唯一事实来源，**不重复探测**。降级后可做的事：

- 基础事实与 triage 已采集的轻量证据复核（readelf/nm -D/strings 结果）
- 可见字符串、导入/导出动态符号的表面梳理（网络指标、可疑 API 名等行为画像线索）
- Go/Rust 原生目标补 `{REA} inspect-go-binary` / `inspect-rust-binary` 元数据（如 triage 未采集）
- PE/Mach-O 轻量兜底能力有限（readelf 仅覆盖 ELF），做不到就写 unknown，不硬凑
- **不许编造伪代码或函数级结论**；确需函数级证据的本项如实 `unresolved`，`unknown_reason` 标准化为「需要 Ghidra 深度分析（未安装/不可用）」，并附升级建议（安装指引见 DEPENDENCIES.md，约 500MB + JDK 21）

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
