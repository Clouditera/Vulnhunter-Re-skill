# 依赖清单（DEPENDENCIES）

本文件汇总 vulnhunter-reverse 插件（skill 层）与 REA 工具层的全部依赖，按「必需 / 按深度可选 / 按任务可选」分层，并给出自检命令。安装任何依赖前**先征得用户同意**；深度能力不可用时如实降级，不假装完成。

## 一、依赖总览

| 依赖 | 用途 | 必需性 | 安装方式 | 自检命令 |
|---|---|---|---|---|
| Node.js ≥ 22.19 | REA CLI 的运行时 | 必需 | 系统包管理器 / nvm | `node --version` |
| REA（npm 包 `rea-agents`） | 分析工具层（全部静态解析能力） | 必需（可免安装） | 免安装：`npx -y rea-agents@latest <命令>`；常用：`npm install -g rea-agents` | `command -v rea && rea --version` |
| Ghidra（约 500MB）+ JDK | 原生 ELF/PE/Mach-O 的函数级反编译（deep 深度 provider） | 按深度可选（仅 deep） | 用户同意后安装，配置 `GHIDRA_INSTALL_DIR` | `rea doctor --provider ghidra --json`（需已设 `GHIDRA_INSTALL_DIR`） |
| `file` / `sha256sum` / `du` | 目标基础事实采集（magic、哈希、大小） | 必需（类 Unix 自带） | 系统自带 | `file <目标>` |
| `readelf` | 大目标 Ghidra 建库受阻时的 ELF 轻量事实兜底 | 建议（Linux 自带） | 系统自带（binutils） | `readelf -h <目标>` |
| `jq` | REA 大输出落盘后的视图投影（evidence view JSON 构造） | 建议 | 系统包管理器 | `jq --version` |
| 编译工具链（C/C++、Node/TS、.NET SDK 等） | reconstruct 最小重建与验证（按重建目标语言选用） | 按任务可选（仅用户要求可运行复刻时） | 按场景安装 | 随重建 README 说明 |
| 隔离执行环境（沙箱/VM/容器） | dynamic 深度：执行目标做动态观测或重建对拍 | 按深度可选（**必须用户明确批准**） | 用户自备 | 人工确认 |

## 二、分层说明

### 1. 必需层（开工前必须就位）

- **Node.js ≥ 22.19**：REA 的硬性运行时要求，低于此版本 REA 不可运行。
- **REA（`rea-agents`）**：全部分析能力的来源。
  - 首选免安装方式 `npx -y rea-agents@latest`（首次会下载，需网络）；
  - 高频使用建议 `npm install -g rea-agents`；
  - **任何安装动作先告知用户并征得同意**，不先斩后奏。
- **基础系统工具**（`file`/`sha256sum`/`du`/`ls`）：triage 的 Step 1 基础事实采集依赖，类 Unix 系统自带，通常无需处理。

### 2. 按深度可选层（用户选定深度后才补）

- **Ghidra + JDK**：仅当用户在交互确认中选择「Ghidra 深度（原生反编译）」且预检未配置时才需要安装。
  - 成本约 500MB + JDK，必须向用户明示成本、由用户决定；
  - 用户不同意 → 降级为 static 深度，在 `scope.md` 记录能力缺口，原生目标只保留基础事实（file/readelf/Go 元数据等轻量事实），**不得假装反编译成功**；
  - 配置方式：安装后设置环境变量 `GHIDRA_INSTALL_DIR`，再用 `rea doctor --provider ghidra --json` 核实。
- **隔离执行环境**：仅 dynamic 深度需要。**必须用户明确批准且有隔离环境**，否则不提供此选项（默认绝不执行目标二进制/脚本）。

### 3. 按任务可选层

- **重建工具链**：仅当用户要求可运行复刻（reconstruct 阶段）时按语言准备——原生函数用 C/C++ 工具链，JS 模块用 Node/TypeScript，.NET 用 C#/.NET SDK。具体构建与运行依赖写在每个重建主题的 README 里。
- **`jq` / `readelf`**：非硬性阻塞项——缺 `jq` 时可手写 view JSON；缺 `readelf` 时 ELF 轻量事实降级为 file + REA 基础输出，均在产物中如实标注。

## 三、开工前自检清单（主会话环境预检）

对应 `skills/reverse/SKILL.md` 的「环境预检」步骤：

```bash
node --version                      # 需要 >= 22.19
command -v rea && rea --version     # 已装 REA 则直接用；否则记「可 npx 免安装 / 可全局装」
file <目标>                          # 初步识别目标形态
# Ghidra（仅 deep 深度候选时）：已设 GHIDRA_INSTALL_DIR 则
rea doctor --provider ghidra --json # 未设置则记「未配置」，向用户给安装选项
# 能力核实（选定深度后）：
rea capabilities --json && rea providers --json
```

预检结论写入工作区 `scope.md`（含深度选择的依据），随后按用户选择补全依赖。

## 四、维护约定

- 本清单是插件依赖的唯一权威汇总；各 `skills/*/SKILL.md` 中的操作卡只描述用法，不重复维护安装细节。
- 新增依赖（如未来接入动态沙箱、新 provider）时同步更新本表与 README 的「安装与依赖」一节。
- REA 自身的能力演进以其上游文档为准：[REA CLI 指南](https://github.com/morluto/rea/blob/main/docs/cli.md)。
