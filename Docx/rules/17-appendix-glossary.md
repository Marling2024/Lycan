# 附录 A：术语对照表

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`A.1`–`A.2`｜**所属部分**：附录 A
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

## A.1 机制与状态术语

| 形式化术语 | 原规则术语 | 英文术语 |
|-----------|-----------|---------|
| 身份（Identity） | 角色、角色卡 | Identity |
| 身份池 | 角色池 | Identity Pool |
| 身份牌 | 角色卡、身份卡 | Identity Card |
| **身份宣称**（Claim） | 报身份、自证 | Identity Claim |
| **身份声明**（Claim，发言类型） | 报身份、自证 | Identity Claim (speech act) |
| **身份公开**（Reveal） | 翻卡、暴露 | Identity Revelation |
| 唯一身份 | — | Unique Identity |
| 可重复身份 | — | Repeatable Identity |
| 公共处决 | 放逐、票出 | Public Execution |
| 秘密处决 | 刀杀、夜刀 | Secret Execution |
| 平安夜 | 空刀、守卫成功 | Peaceful Night |
| 席位编号（`seat_id`） | 座位号、顺序 | Seat Index |
| 玩家ID（`player_id`） | 玩家编号、座位号 | Player ID |
| 追加投票 | 平票PK、加时投票 | Runoff Vote |
| 终局表决 | 平票后的终局选择 | Final Binary Vote |

## A.2 身份词条对照

| 中文 | 形式化标识符 | 阵营 |
|------|-------------|------|
| 普通人 | `CIVILIAN` | 平凡 |
| 吹哨人 | `WHISTLEBLOWER` | 平凡 |
| 验尸官 | `CORONER` | 平凡 |
| 嗜血者 | `BLOODTHIRSTY` | 邪恶 |
| 非常者 | `TRAITOR` | 邪恶 |
| 大法师 | `ARCHMAGE` | 治安 |
| 圣骑士 | `PALADIN` | 治安 |
| 圣杀者 | `EXECUTIONER` | 治安 |
| 痴呆者 | `LUNATIC` | 混沌 |
| 异星客 | `ALIEN` | 混沌 |

> 说明：标识符沿用项目原有变量名（`BLOODTHIRSTY` / `CIVILIAN` / `EXECUTIONER` 等），
> 不采用 `Docx/archive/REVISION_PLAN.md` 中提出的替代命名，以保证实现层契约稳定。

---
