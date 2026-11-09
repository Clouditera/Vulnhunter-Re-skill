# 依赖清单（DEPENDENCIES）

本文件汇总 vulnhunter-reverse 插件（skill 层）与 REA 工具层的全部依赖，分为**必装项**、系统自带项、建议项与按场景项，必装项均给出安装方式与安装命令。

## 一、必装项（开工前必须就位）

| 依赖 | 用途 | 自检命令 |
|---|---|---|
| Node.js ≥ 22.19 | REA CLI 的运行时（硬性要求，低于此版本 REA 不可运行） | `node --version` |
| REA（npm 包 `rea-agents`） | 分析工具层（全部静态解析能力） | `command -v rea && rea --version` |
| JDK 21（64-bit） | Ghidra 的运行时（Ghidra 11.x 要求 JDK 21；10.x 为 JDK 17） | `java -version` |
| Ghidra | 原生 ELF/PE/Mach-O 的函数级反编译（deep 深度 provider） | `rea doctor --provider ghidra --json` |
| C/C++ 工具链 | reconstruct 原生函数的最小重建与验证 | `cc --version` 或 `g++ --version` |
| TypeScript（Node/TS） | reconstruct JS/TS 模块的最小重建与验证 | `tsc --version` |
| .NET SDK 8.0 | reconstruct .NET 程序集（C#）的最小重建与验证 | `dotnet --info` |

### Node.js ≥ 22.19

```bash
# 方式一：apt（NodeSource 官方仓库，推荐）
# 注意：Ubuntu 默认 apt 源里的 Node.js 版本过旧（远低于 22.19），不可用；需先加 NodeSource 源
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node --version    # 确认 ≥ 22.19

# 方式二：nvm（需要多版本并存时用；安装脚本见 nvm 官方仓库）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash
nvm install 22        # 安装 22.x 最新版（须 ≥ 22.19）

# macOS：brew install node@22，或官网安装包 https://nodejs.org/
```

### REA（npm 包 `rea-agents`）

```bash
npm install -g rea-agents        # 全局安装（推荐）
rea --version

# 临时使用可免安装（首次会下载，需网络）：
npx -y rea-agents@latest --version
```

### JDK 21（64-bit）

```bash
# Debian / Ubuntu
sudo apt update && sudo apt install -y openjdk-21-jdk

# macOS（Homebrew）
brew install openjdk@21

java -version     # 确认 64-bit JDK 21
```

### Ghidra

> **为什么不是 apt/npm**：Ghidra 在 apt 与 npm 均**没有官方包**（官方只以 zip 压缩包发布），因此必须手动下载解压；第三方 snap 等非官方渠道版本滞后且来源不可控，不建议用于分析环境。

```bash
# 1. 从官方 releases 下载并解压（版本号以 https://github.com/NationalSecurityAgency/ghidra/releases 最新为准）
#    示例（<版本> 与 <构建日期> 需按实际发布替换）：
curl -L -o ghidra.zip https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_<版本>_build/ghidra_<版本>_PUBLIC_<构建日期>.zip
sudo unzip ghidra.zip -d /opt/

# 2. 配置环境变量（写入 shell 配置文件持久化）
export GHIDRA_INSTALL_DIR=/opt/ghidra_<版本>_PUBLIC

# 3. 核实 REA 侧 provider 可用
rea doctor --provider ghidra --json
```

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

## 二、系统自带项（无需安装）

| 工具 | 用途 | 自检命令 |
|---|---|---|
| `file` / `sha256sum` / `du` / `ls` | triage 基础事实采集（magic、哈希、大小） | `file <目标>` |
| `readelf` | 大目标 Ghidra 建库受阻时的 ELF 轻量事实兜底（binutils 自带） | `readelf -h <目标>` |

## 三、建议项

| 依赖 | 用途 | 安装命令 |
|---|---|---|
| `jq` | REA 大输出落盘后的视图投影（evidence view JSON 构造）；缺失时可手写 view JSON 降级 | Debian/Ubuntu：`sudo apt install -y jq`；macOS：`brew install jq` |

## 四、按场景项

| 依赖 | 用途 | 说明 |
|---|---|---|
| 隔离执行环境（沙箱/VM/容器） | dynamic 深度：执行目标做动态观测或重建对拍 | 用户自备；**必须用户明确批准**才提供此选项，默认绝不执行目标 |

## 五、开工前自检清单（主会话环境预检）

对应 `skills/reverse/SKILL.md` 的「环境预检」步骤：

```bash
node --version                       # ≥ 22.19
command -v rea && rea --version      # REA 就位
java -version                        # JDK 21（64-bit）
rea doctor --provider ghidra --json  # Ghidra provider 可用（需已设 GHIDRA_INSTALL_DIR）
cc --version                         # C/C++ 工具链
tsc --version                        # TypeScript
dotnet --info                        # .NET SDK 8.0
file <目标>                           # 初步识别目标形态
# 能力核实（可选）：
rea capabilities --json && rea providers --json
```

预检结论写入工作区 `scope.md`。安装动作仍遵循插件纪律：**执行安装前先征得用户同意**；某项暂不可用时如实记录能力缺口并降级，不假装完成。

## 六、维护约定

- 本清单是插件依赖的唯一权威汇总；各 `skills/*/SKILL.md` 中的操作卡只描述用法，不重复维护安装细节。
- 新增依赖（如未来接入动态沙箱、新 provider）时同步更新本表与 README 的「安装与依赖」一节。
- REA 自身的能力演进以其上游文档为准：[REA CLI 指南](https://github.com/morluto/rea/blob/main/docs/cli.md)。
- Ghidra / JDK 版本要求以 Ghidra 官方发布说明为准（11.x 要求 JDK 21，10.x 要求 JDK 17）。
