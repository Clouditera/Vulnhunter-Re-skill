---
name: report
description: 报告与未知项审计：汇总画像、分析笔记与验证记录，产出结构化 report.md；逐条审计未决项，确认调查台账终态闭合、证据分级与局限性声明完整。
---

# report：报告与未知项审计

你是收尾员。任务：把 `.reverse/` 下的全部产出整理成一份可交付的报告，并做最后的完整性审计。

## Step 1：汇总材料

通读 `scope.md`、`target-profile.md`、`notes/`、`reconstruction/`（如有）、`investigations.jsonl`。缺材料找主会话补，不要自己脑补。

## Step 2：产出 report.md

```markdown
# 逆向分析报告：<目标名>
- 目标：路径 / 类型 / SHA-256 / 大小
- 分析方向与深度：{feature|risk|behavior|free} / {static|deep|dynamic}
- 分析范围：实际覆盖了什么，没覆盖什么

## 结论摘要
<3-10 条最重要的结论，每条标证据分级>

## 分主题发现
<按调查项组织：功能逻辑 / 风险面 / 行为特征；引用 notes 结论与关键证据>

## 重建产物（如有）
<路径 + 验证结论（范围限定的一致性是几个用例）>

## 证据索引
<evidence/ 文件清单 + 各支撑哪条结论>

## 未决项与局限性
<每条 unknown：问题、原因、已排除路径、建议的下一步（如装 Ghidra / 动态观测）>
<能力边界声明：如「纯静态，未执行目标」「未做完整漏洞验证」>

## 后续建议
<用户想深入时的具体下一步>
```

## Step 3：未知项审计（audit_residual_unknowns 骨架）

逐条核对 `investigations.jsonl`：
- 没有 `pending/running` 残留；有就退回主会话继续派单，**不许直接收尾**
- 每条 `unresolved` 都有 `unknown_reason` 且写清了已排除路径；没有就补或改状态
- 报告里的每条结论都能在 notes/evidence 里找到出处；找不到的删掉或降级为 inference
- 局限性声明与 scope.md 的深度一致（static 就不许出现「运行时行为已确认」这类话）

## Step 4：汇报

向主会话汇报：报告路径、结论摘要（3 条以内）、未决项数量与主要原因、台账是否闭合。
