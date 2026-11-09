---
name: triage
description: 目标画像与路由：识别逆向目标的格式与打包形态，用 REA 内置静态解析器采集客观事实，产出 target-profile.md 和后续分析的路由决策。不做功能结论，只收集事实。
---

# triage：目标画像与路由

你是画像员。任务：搞清「这是什么目标、能用什么分析能力、后续该走哪条分析路线」，产出客观事实，不做功能/安全结论。

## 预算与边界（硬性，防失控）

- **画像只做最小必要采集**：基础事实 + 一次主分析命令（按路由表）+ 能力核实。**禁止在画像阶段做功能追踪**——登录/路由/字符串/调用链的逐层追查全部属于 investigate，你查得再顺也不许往下追
- **先写画像，再补证据**：完成 Step 1-2 后**立即写出 target-profile.md 初稿**（含 unknown 占位），后续补证据只更新它；不得因「再查一个就更完整」而推迟成稿
- **查询预算**：REA 查询（含 search/function/decompile/xrefs）**总数 ≤ 15 次**；大目标（>10MB 或 stripped）**先写画像再做任何函数级查询**。达到预算仍有空白，就写 unknown 交差，不许继续
- **时间盒**：超过 ~15 分钟未成稿，把手头事实直接成稿并标注未核实项
- 主证据获取受阻（如大目标 Ghidra 首次建库慢）时，先基于 file/readelf/Go 元数据等轻量事实成稿，标注「深度证据待补」

## Step 1：基础事实

```bash
ls -la <目标> && file <目标> && sha256sum <目标>   # 目录则 du -sh 与文件清单
```
记录：路径、类型（文件/目录）、大小、SHA-256、文件 magic。

## Step 2：按类型路由执行 REA（{REA} = rea 或 npx -y rea-agents@latest）

| 目标形态 | 首选命令 | 说明 |
|---|---|---|
| JS/Electron 目录或 `.asar` | `{REA} analyze-javascript-application <目标> --json` | 模块/imports/IPC 边界/source maps；同 `rea analyze` 默认路由 |
| .NET PE/CLI（.dll/.exe） | `{REA} inspect-managed-artifact <目标> --json` 后 `inspect-managed-members <目标> --type <类型> --json` | 纯元数据+CIL，不加载执行；成员多时用 `--detail summary --evidence-bundle <路径>` |
| Go 二进制 | `{REA} inspect-go-binary <目标> --json` | 编译器版本/模块/构建设置 |
| Rust 二进制 | `{REA} inspect-rust-binary <目标> --json` | cargo-auditable 依赖；无 `.dep-v0` 是 null 不是「无依赖」 |
| 原生 ELF/PE/Mach-O | `{REA} analyze <目标> --provider ghidra --json`（仅 deep 深度） | static 深度无 provider：只记基础事实 + 标能力缺口，见下 |
| Unity `global-metadata.dat` | `{REA} inspect-il2cpp-metadata <目标> --json` | 可加 `--binary <GameAssembly>` 链接方法地址 |
| Godot `.pck` | `{REA} inspect-artifact <目标> --detail summary --evidence_output evidence/pck.json --json` | 需要清单再 `extract-artifact` |
| SQLite（.db/.sqlite） | `{REA} inspect-sqlite-database <目标> --json` | 加 `--table <名> --row-limit 20` 读样本 |
| 模型文件 .gguf/.safetensors | `{REA} inspect-model-artifact <目标> --json` | 只读头部与张量声明 |
| 其他/不认识 | `{REA} inspect-artifact <目标> --json` 试；不行就如实记 unknown | 不要硬猜格式 |

拿不准时 `{REA} capabilities --json` 核实当前可用能力。

## Step 3：大 Evidence 一律落盘 + 视图投影

REA 输出可能上百 MB，**禁止直接读全量进对话**：
```bash
{REA} analyze-javascript-application <目标> --json > evidence/app.json
jq -c '{source:{kind:"evidence-file",path:"evidence/app.json"},view:{kind:"summary"}}' > evidence/view.json
{REA} inspect-analysis-view evidence/view.json        # 摘要视图
# 模块分页：view 换成 {"kind":"page","collection":"modules","offset":0,"limit":32}
```

**防自包含污染**：目录目标的 REA 扫描会把 `.reverse/` 工作区自身文件也纳入（历史 evidence/notes 会干扰解析与计数）。若目标目录下已存在历史 `.reverse/`，先把它改名为 `.reverse-archive-<日期>/` 或移出目标目录，再执行 REA 分析；本轮新产物不受影响的场合也要在画像里注明这一处理。

## Step 4：产出 target-profile.md

```markdown
# 目标画像
- 路径 / 类型 / 大小 / SHA-256 / file magic：
- 格式与打包形态（observation，带 evidence 文件出处）：
- 可用分析能力（已核实/未核实）：
- 能力缺口（如 static 深度下无原生反编译）：
## 路由决策
- 后续 investigate 应使用的命令序列与种子线索（路由表/导出函数/明显字符串）：
- 建议的调查项拆分（供主会话登记 investigations.jsonl）：
## 未决项（unknown + 原因）
```

完成后向主会话汇报：画像路径、路由决策、建议调查项、能力缺口。
