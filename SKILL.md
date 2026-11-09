---
name: vulnhunter-reverse
description: VulnHunter 逆向分析套件。当用户要求对二进制、安装包、JS/Electron 应用、.NET 程序集等非源码目标做逆向分析、功能逻辑提取、漏洞风险梳理或行为画像时使用。内含 5 个按分析流程组织的子技能（调度→画像→分析→重建→报告），按需加载。
---

# VulnHunter 逆向分析套件

对非源码目标做逆向分析的完整工具箱，按分析流程分为 5 个子技能，各自独立目录、独立 `SKILL.md`。分析工具层为 REA CLI。

## 使用方式

入口为调度技能 `skills/reverse/SKILL.md`（主会话调度规则：交互确认、依赖补全、派发与台账）；其余子技能由主会话按角色派发给 subagent，子技能内部会再按需引用自己的参考文档与操作卡。

## 分析主流程（建议顺序）

1. **reverse** — 主会话调度（交互确认、派发、investigations.jsonl 台账、完成条件把关）：`skills/reverse/SKILL.md`
2. **triage** — 目标画像与路由：`skills/triage/SKILL.md`
3. **investigate** — 静态分析与功能追踪（函数级证据）：`skills/investigate/SKILL.md`
4. **reconstruct** — 可选：最小重建与对照验证（仅用户要求可运行复刻时）：`skills/reconstruct/SKILL.md`
5. **report** — 报告与未知项审计：`skills/report/SKILL.md`

## 纪律

- 目标文件只读，默认不执行目标；产物写入目标旁的 `.reverse/` 目录
- 任何依赖安装动作先征得用户同意；深度能力不可用时如实降级，不假装完成
- 结论必须带证据分级（observation / derivation / inference / unknown），不许虚构反编译与重建结论
- 每个子技能的 `SKILL.md` 是该阶段的唯一权威指令，以其为准
