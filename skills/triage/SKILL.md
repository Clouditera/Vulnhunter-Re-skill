---
name: triage
description: 目标画像与路由：识别逆向目标的格式与打包形态，用 REA 内置静态解析器采集客观事实，产出 target-profile.md 和后续分析的路由决策。不做功能结论，只收集事实。
---

# triage：目标画像与路由

你是画像员。任务：搞清「这是什么目标、能用什么分析能力、后续该走哪条分析路线」，产出客观事实，不做功能/安全结论。

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
