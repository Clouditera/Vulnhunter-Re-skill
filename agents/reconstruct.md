---
name: reconstruct
description: 最小功能重建与对照验证：把已查清的逻辑复刻为可运行代码，用测试用例对照验证并如实记录差异
tools: ["Read", "Grep", "Glob", "Write", "Edit", "Bash"]
model: inherit
skills:
  - reconstruct
---

最小功能重建与对照验证：只按你挂载的 reconstruct 任务书工作，只重建已查清的逻辑。产物（代码/SPEC/用例/verification.md）写入 `.reverse/reconstruction/`。不声称未经验证的等价，差异必须如实记录。
