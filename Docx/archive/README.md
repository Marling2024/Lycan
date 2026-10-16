# `Docx/archive/` 归档说明

> 本目录存放**过程文档**，不属于规则的权威来源。

| 文件 | 性质 |
|------|------|
| `RULES_ANALYSIS.md` | 原规则漏洞分析（`FORMAL_RULES.md` 的修复目标来源） |
| `REVISION_PLAN.md` | 第一轮修改方案（7 处修改的原始工单） |
| `FORMAL_RULES_AUDIT_REPORT.md` | 外部审核报告（历史快照，部分意见已被后续决策取代） |
| `REVISION_LOG.md` | 修订日志（施工历史） |
| `VERIFICATION_LOG.md` | 验证记录（审核意见状态对照） |

## 当前生效的文件位置

| 文件 | 位置 | 说明 |
|------|------|------|
| `FORMAL_RULES.md` | `Docx/` | **形式化规则说明书**（`alpha 0.1`，唯一权威规则来源） |
| `RULES.md` | `Docx/` | **原有规则说明书**（依据文档，术语尚未同步） |
| `Agent.md` / `architecture.md` / `ToDo.md` | 仓库根 | 项目上下文 |

## 注意

`FORMAL_RULES_AUDIT_REPORT.md` 的 P1 / P2 / P3 三条意见针对的是更早版本，
**已被后续用户决策取代**（时间戳、嗜血者上限改为 `⌊n/3⌋`、取消普通人占比约束）。
采纳任何审核意见前，请先查 `VERIFICATION_LOG.md` 第 1 节的状态对照。
