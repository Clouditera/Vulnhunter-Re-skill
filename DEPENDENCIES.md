# 依赖清单（DEPENDENCIES）

本文件汇总 vulnhunter-reverse 插件（skill 层）与 REA 工具层的全部依赖，分为**必装项**、可选项、系统自带项、建议项与按场景项，必装项均给出安装方式与安装命令；另附 Windows 环境使用说明。

## 一、必装项（开工前必须就位）

| 依赖 | 版本要求 | 用途 | 自检命令 |
|---|---|---|---|
| Node.js | **≥ 24.22.1**（REA 6.8.0 声明的运行时要求，低于此版本 REA 不可运行） | REA CLI 的运行时 | `node --version` |
| REA（npm 包 `rea-agents`） | **固定 6.8.0** | 分析工具层（全部静态解析能力） | `command -v rea && rea --version`（应输出 6.8.0） |
| C/C++ 工具链 | 任意现代版本 | reconstruct 原生函数的最小重建与验证 | `cc --version` 或 `g++ --version` |
| TypeScript（Node/TS） | 任意现代版本 | reconstruct JS/TS 模块的最小重建与验证 | `tsc --version` |
| .NET SDK | 8.0 | reconstruct .NET 程序集（C#）的最小重建与验证 | `dotnet --info` |

### Node.js ≥ 24.22.1

```bash
# 方式一：apt（NodeSource 官方仓库，推荐）
# 注意：Ubuntu 默认 apt 源里的 Node.js 版本过旧（远低于 24.22.1），不可用；需先加 NodeSource 源
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
node --version    # 确认 ≥ 24.22.1

# 方式二：nvm（需要多版本并存时用；安装脚本见 nvm 官方仓库）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash
nvm install 24        # 安装 24.x 最新版（须 ≥ 24.22.1）

# macOS：brew install node@24，或官网安装包 https://nodejs.org/
```

### REA（npm 包 `rea-agents`，固定 6.8.0）

```bash
npm install -g rea-agents@6.8.0    # 全局安装（推荐），版本固定，保证结果可复现
rea --version                      # 应输出 6.8.0

# 临时使用可免安装（首次会下载，需网络）：
npx -y rea-agents@6.8.0 --version
```

> 版本固定纪律：所有 SKILL.md 与派发指令中的 `{REA}` 调用均按 6.8.0 写死；升级 REA 是一个显式决策（评估变更日志后统一更新文档与技能文件），不允许环境间版本漂移。

### C/C++ 工具链

```bash
# Debian / Ubuntu
sudo apt update && sudo apt install -y build-essential

# macOS（Xcode 命令行工具）
xcode-select --install

cc --version
```

### TypeScript（Node/TS）

```bash
npm install -g typescript
tsc --version
```

### .NET SDK 8.0

```bash
# 方式一：apt（Ubuntu 22.04+ 官方源已内置 .NET 8，直接装）
sudo apt update && sudo apt install -y dotnet-sdk-8.0

# 方式二：跨平台官方安装脚本（Linux / macOS，非 Ubuntu 22.04+ 时用）
curl -sSL https://dot.net/v1/dotnet-install.sh | bash /dev/stdin --channel 8.0

# macOS 也可用 Homebrew：brew install dotnet-sdk

dotnet --info     # 确认 8.0 SDK 就位
```

## 二、可选项：Ghidra（12.1.x）+ JDK

Ghidra 是**可选依赖**：只有需要对原生 ELF/PE/Mach-O 做函数级反编译（deep 深度）时才需要。不装 Ghidra，插件照常工作，仅原生二进制的函数级能力降级（降级范围见下）。

| 依赖 | 版本要求 | 用途 | 自检命令 |
|---|---|---|---|
| JDK（64-bit） | 版本以 Ghidra 12.1.x 官方发布说明为准（历史参考：Ghidra 11.x 要求 JDK 21） | Ghidra 的运行时 | `java -version` |
| Ghidra | 12.1.x | 原生 ELF/PE/Mach-O 的函数级反编译（deep 深度 provider） | `rea doctor --provider ghidra --json` |

### JDK 安装

```bash
# Debian / Ubuntu
sudo apt update && sudo apt install -y openjdk-21-jdk

# macOS（Homebrew）
brew install openjdk@21

java -version     # 确认 64-bit JDK
```

### Ghidra 安装

> **为什么不是 apt/npm**：Ghidra 在 apt 与 npm 均**没有官方包**（官方只以 zip 压缩包发布），因此必须手动下载解压；第三方 snap 等非官方渠道版本滞后且来源不可控，不建议用于分析环境。

```bash
# 1. 从官方 releases 下载 12.1.x 并解压
#    （具体构建号以 https://github.com/NationalSecurityAgency/ghidra/releases 为准，<构建日期> 按实际发布替换）
curl -L -o ghidra.zip https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_12.1_build/ghidra_12.1_PUBLIC_<构建日期>.zip
sudo unzip ghidra.zip -d /opt/

# 2. 配置环境变量（写入 shell 配置文件持久化）
export GHIDRA_INSTALL_DIR=/opt/ghidra_12.1_PUBLIC

# 3. 核实 REA 侧 provider 可用
rea doctor --provider ghidra --json
```

### 未安装 Ghidra 时的能力范围（降级说明）

不装 Ghidra 时插件以 static 深度运行，**以下能力完全不受影响**（REA 内置静态解析器，零外部引擎）：

- JS/Electron（目录/ASAR）：模块、imports、IPC 边界、source maps
- .NET PE/CLI：元数据 + CIL 检查（`inspect-managed-artifact` / `inspect-managed-members`）
- Go / Rust 二进制：编译器版本、模块、构建设置、cargo-auditable 依赖
- Unity IL2CPP metadata、Godot PCK、SQLite、GGUF/safetensors 等模型文件
- 全类型的基础事实采集（file magic、哈希、大小、字符串）

**以下能力不可用（降级）**：

- 原生 ELF/PE/Mach-O 的函数级分析：`search` / `function` / `decompile` / `xrefs` / `trace`（这些命令在 deep 深度下走 Ghidra provider）
- 原生目标退化为仅基础事实（file/readelf/Go 元数据等轻量事实），画像中标注「深度证据待补」
- 涉及原生函数级证据的调查项如实记 `unresolved`，原因写「需要 Ghidra 深度分析」，**不得编造伪代码或函数级结论**

降级决策由主会话在交互确认时明示给用户，并记录在 `scope.md` 的能力缺口中；后续补装 Ghidra 后可对未决项重新派单。

## 三、系统自带项（Linux/macOS，无需安装）

| 工具 | 用途 | 自检命令 |
|---|---|---|
| `file` / `sha256sum` / `du` / `ls` | triage 基础事实采集（magic、哈希、大小） | `file <目标>` |
| `readelf` | 大目标 Ghidra 建库受阻时的 ELF 轻量事实兜底（binutils 自带） | `readelf -h <目标>` |

## 四、建议项

| 依赖 | 用途 | 安装命令 |
|---|---|---|
| `jq` | REA 大输出落盘后的视图投影（evidence view JSON 构造）；缺失时可手写 view JSON 降级 | Debian/Ubuntu：`sudo apt install -y jq`；macOS：`brew install jq`；Windows：`winget install jqlang.jq` |

## 五、按场景项

| 依赖 | 用途 | 说明 |
|---|---|---|
| 隔离执行环境（沙箱/VM/容器） | dynamic 深度：执行目标做动态观测或重建对拍 | 用户自备；**必须用户明确批准**才提供此选项，默认绝不执行目标 |

## 六、Windows 环境说明

### 推荐用法：WSL2

**推荐在 WSL2（Ubuntu）内运行本插件**：技能文件中的全部命令按 POSIX shell（bash）编写，WSL2 内可直接套用本文第一至四节的 Linux 安装命令，获得与文档完全一致的行为，无需任何替代方案。

### 原生 Windows（不用 WSL）的可用性

REA 是纯 Node.js CLI，**核心分析能力在原生 Windows 可用**（Node + `rea-agents@6.8.0` + Ghidra 均有原生 Windows 支持）。但技能工作流依赖 POSIX 工具链，以下命令在原生 Windows（PowerShell/cmd）**不可用或有差异**，按右列替代：

| POSIX 命令（技能文件中） | 原生 Windows 状态 | 替代命令（PowerShell） |
|---|---|---|
| `file <目标>` | 不可用 | 无等价物；用 `Get-Item <目标>` 取大小/时间，magic 识别交由 `{REA} inspect-artifact` 兜底，画像中标注「file magic 未采集」 |
| `sha256sum <目标>` | 不可用 | `Get-FileHash <目标> -Algorithm SHA256` |
| `du -sh <目录>` | 不可用 | `(Get-ChildItem <目录> -Recurse -File \| Measure-Object Length -Sum).Sum / 1MB` |
| `readelf -h <目标>` | 不可用 | ELF 轻量事实跳过（记 unknown）；PE 目标可用 `dumpbin /headers <目标>`（随 VS Build Tools 安装） |
| `curl -fsSL ... \| bash` 安装脚本 | 不可用 | 一律改用下方的 Windows 安装命令 |
| `jq` | 可用（需安装） | `winget install jqlang.jq`，或 PowerShell `ConvertFrom-Json` 兜底 |
| bash 复合命令（重定向、`&&`、`xargs` 等） | 语法不兼容 | 在 PowerShell 语义下逐条改写；`>` 重定向可用（注意 PowerShell 5.x 默认 UTF-16，建议 `out-file -Encoding utf8`） |

### 原生 Windows 安装命令

```powershell
# Node.js ≥ 24.22.1：官网安装包 https://nodejs.org/ ，或 nvm-windows / fnm
winget install OpenJS.NodeJS          # 确认装到 ≥ 24.22.1，否则改用官网安装包
node --version

# REA 6.8.0 / TypeScript（npm 原生支持 Windows）
npm install -g rea-agents@6.8.0
npm install -g typescript

# .NET SDK 8.0
winget install Microsoft.DotNet.SDK.8

# JDK 21（可选，仅 Ghidra 需要）
winget install Microsoft.OpenJDK.21

# Ghidra 12.1.x（可选）：官网 zip 解压后设置环境变量
#   setx GHIDRA_INSTALL_DIR "C:\tools\ghidra_12.1_PUBLIC"
#   然后重开终端，rea doctor --provider ghidra --json 核实

# C/C++ 工具链：Visual Studio Build Tools（含 MSVC 与 dumpbin）
winget install Microsoft.VisualStudio.2022.BuildTools
```

**Windows 不可用的能力**：除上述命令差异外，`readelf` 类 ELF 轻量解析无原生替代（记 unknown 或由 REA 兜底）；隔离执行环境在 Windows 上需自备 Hyper-V VM / Sandbox。需要完整、无替代命令的体验时，请回到 WSL2。

## 七、开工前自检清单（主会话环境预检）

对应 `skills/reverse/SKILL.md` 的「环境预检」步骤：

```bash
node --version                       # ≥ 24.22.1
command -v rea && rea --version      # REA 就位，版本应为 6.8.0
cc --version                         # C/C++ 工具链
tsc --version                        # TypeScript
dotnet --info                        # .NET SDK 8.0
file <目标>                           # 初步识别目标形态
# 可选（仅需要 deep 深度时）：
java -version                        # JDK（Ghidra 运行时）
rea doctor --provider ghidra --json  # Ghidra provider 可用（需已设 GHIDRA_INSTALL_DIR）
# 能力核实（可选）：
rea capabilities --json && rea providers --json
```

预检结论写入工作区 `scope.md`。安装动作仍遵循插件纪律：**执行安装前先征得用户同意**；某项暂不可用时如实记录能力缺口并降级（降级范围见第二节），不假装完成。

## 八、维护约定

- 本清单是插件依赖的唯一权威汇总；各 `skills/*/SKILL.md` 中的操作卡只描述用法，不重复维护安装细节。
- REA 版本固定为 6.8.0；升级时需同步更新本文件、README 与各技能文件中的 `@6.8.0` 引用。
- 新增依赖（如未来接入动态沙箱、新 provider）时同步更新本表与 README 的「安装与依赖」一节。
- REA 自身的能力演进以其上游文档为准：[REA CLI 指南](https://github.com/morluto/rea/blob/main/docs/cli.md)。
- Ghidra / JDK 版本要求以 Ghidra 官方发布说明为准（本文以 12.1.x 为基线）。
