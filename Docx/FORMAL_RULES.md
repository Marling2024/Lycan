# 狼人杀变体游戏形式化规则说明书（Formal Specification）

> **版本**：alpha 0.3  
> **状态**：已落实 7 项规则变更与空场终局 `NO_SURVIVORS`（文本修订 + 人工推演）；**未经实现层验证**，且存在未决事项  
> **最后更新**：2026/10/17  
> **本版说明**：胜利条件与胜检时机、投票与平票、猎杀目标、身份公开因果、公共处决信息、初始配额下限、空场终局 `NO_SURVIVORS`  
> **依据**：基于 `Docx/RULES.md` 改进而来；本版**不宣称规则已完全闭合**，未决事项集中见 `8.5`
> **多文件版**：本文件是**索引与门面**；规则正文按语义拆分于 `Docx/rules/`（单文件 ≤200 行，见 `Agent.md`「重要约束」）

---

## 文档信息（Document Info）

| 项目 | 内容 |
|------|------|
| **文档版本** | `alpha 0.3` |
| **版本说明** | 本版规则变更（见「版本摘要」）：胜利条件、结算单元胜检、投票三轮 + 终局表决、猎杀目标限制、技能公开因果、公共处决不公开身份、初始嗜血者下限、空场终局 `NO_SURVIVORS` |
| **文档状态** | 文本修订完成，**未经实现层验证**；存在未决事项（见 `8.5`） |
| **最后更新** | 2026/10/17 |
| **依据文档** | `Docx/RULES.md`（严谨版规则说明书） |
| **定位** | 程序实现的唯一权威规则来源（Single Source of Truth）；**权威范围 = 本索引 + `Docx/rules/*` 全套文件** |
| **覆盖范围** | 4 阵营 · 10 种身份 · 5 阶段轮次循环 · 全部技能与判定规则 |
| **修订历史** | 见 `Docx/archive/REVISION_LOG.md`（过程记录，非规则来源） |
| **审核记录** | 见 `Docx/archive/VERIFICATION_LOG.md`（审核意见状态对照） |
| **未决事项** | 见「第八部分：验证状态与待办事项」`8.5`（批内豁免判定、痴呆者追加投票与终局表决、追加辩论预算、异星客免疫反馈、治安技能频次上限、"非常者独存"状态、投票自由度） |

---

## 文件索引（File Index）

> **编号全局唯一、跨文件可解析**：`1.1`–`1.5`、`2.1`–`2.5`、`3.1`–`3.2`、`4.1`–`4.3`、`5.1`–`5.3`、
> `Q1`–`Q9`、`7.1`–`7.7`、`8.1`–`8.5`、`A.1`–`A.2`、`INV1`–`INV12`、`C1`–`C7`、`B1`–`B6`、`RC1`–`RC5`
> 在拆分前后**完全一致**；正文中的跨文件引用（如「见 `3.2.7`」）一律按下表定位。

| 部分 | 文件 | 覆盖编号 | 内容 |
|------|------|---------|------|
| 第一部分 形式化基础定义 | [`rules/01-game-instance.md`](rules/01-game-instance.md) | `1.1` | 游戏实例、术语约定、表示法约定、相关文档 |
| | [`rules/02-player-state-identity.md`](rules/02-player-state-identity.md) | `1.2`（导言、`1.2.1`–`1.2.4`） | 玩家状态总表、存活状态、身份类型、阵营、种族标签 |
| | [`rules/03-player-state-flags.md`](rules/03-player-state-flags.md) | `1.2.5`–`1.2.8` | 身份宣称、身份公开、保护状态、封印状态 |
| | [`rules/04-victory-and-execution.md`](rules/04-victory-and-execution.md) | `1.3`–`1.4` | 胜利条件与 `NO_SURVIVORS`、处决类型与处决函数 |
| | [`rules/05-initialization-quota.md`](rules/05-initialization-quota.md) | `1.5`（导言、`1.5.1`–`1.5.3`） | 身份池构建算法、引擎最小接口、配额约束 `RC1`–`RC5` |
| | [`rules/06-initialization-assignment.md`](rules/06-initialization-assignment.md) | `1.5.4`–`1.5.6` | 随机分配给玩家ID、身份池示例、配置验证检查清单 |
| 第二部分 角色技能形式化定义 | [`rules/07-skills-commoner-evil.md`](rules/07-skills-commoner-evil.md) | `2.1`–`2.3` | 技能表示法；平凡（普通人 / 吹哨人 / 验尸官）、邪恶（嗜血者 / 非常者） |
| | [`rules/08-skills-law.md`](rules/08-skills-law.md) | `2.4` | 治安：大法师、圣骑士、圣杀者 |
| | [`rules/09-skills-chaos.md`](rules/09-skills-chaos.md) | `2.5` | 混沌：痴呆者、异星客 |
| 第三部分 游戏流程时序定义 | [`rules/10-timeline-and-info.md`](rules/10-timeline-and-info.md) | `3.1`、`3.2`（导言、`3.2.1`–`3.2.2`） | 时序状态机、初始化阶段、信息时 |
| | [`rules/11-debate-phase.md`](rules/11-debate-phase.md) | `3.2.3` | 辩论时：发言范畴、字数约束、追加辩论 |
| | [`rules/12-voting-and-adjudication.md`](rules/12-voting-and-adjudication.md) | `3.2.4`–`3.2.5` | 投票时（三轮 + 终局表决）、裁判时 |
| | [`rules/13-hunt-and-victory-check.md`](rules/13-hunt-and-victory-check.md) | `3.2.6`–`3.2.7` | 猎杀时、结算单元与胜利检定时序 |
| 第四部分 / 第五部分 | [`rules/14-invariants-and-priority.md`](rules/14-invariants-and-priority.md) | `4.1`–`4.3`、`5.1`–`5.3` | 不变式 `INV1`–`INV12`、技能约束 `C1`–`C7`、边界 `B1`–`B6`、豁免与胜检优先级、同阶段行动顺序 |
| 第六部分 FAQ 形式化解答 | [`rules/15-faq.md`](rules/15-faq.md) | `Q1`–`Q9` | 技能交互、公共处决、投票流程、复活与胜检等裁定 |
| 第七部分 实现检查清单 | [`rules/16-implementation-checklist.md`](rules/16-implementation-checklist.md) | `7.1`–`7.7` | 数据结构、状态机、技能、投票、豁免、随机性、测试覆盖 |
| 附录 A 术语对照表 | [`rules/17-appendix-glossary.md`](rules/17-appendix-glossary.md) | `A.1`–`A.2` | 形式化术语 ↔ 原规则术语 ↔ 英文术语；身份词条对照 |
| 版本摘要 | [`rules/18-version-summary.md`](rules/18-version-summary.md) | —（变更历史，无编号） | `v1.0-draft` → `alpha 0.3` 各版本要点与变更清单 |
| 第八部分 验证状态与待办事项 | [`rules/19-verification-status.md`](rules/19-verification-status.md) | `8.1`–`8.5` | 验证路径、已知事项、已解决问题、待办、**未决事项** |

**阅读路径建议**

- **只看规则**：`rules/01` → `rules/13`（第一至第三部分），按编号定位细节
- **准备实现**：`rules/10`–`rules/13`（流程）→ `rules/14`（不变式与优先级）→ `rules/16`（检查清单）
- **评审仲裁**：`rules/15`（FAQ）→ `rules/19`（验证状态与未决事项）
- **改规则前**：先读 `rules/14`（不变式），再改对应章节，并同步 `rules/18`（版本摘要）

**维护约定**

- 新增 / 拆分 / 移动文件后，**必须同步更新本表**与 `Agent.md`「重要约束」的现状段落。
- 单文件**尽量不超过 200 行**；超限时按语义拆分（见 `Agent.md`）。
- **编号一经发布不得改动**；跨文件引用沿用编号，不引用物理文件名（便于后续再拆分）。

**评审信息**

**审核状态**：`alpha 0.3` 文本修订完成，**未经实现层验证**，存在未决事项（见 `8.5`）  
**文档版本**：`alpha 0.3`  
**版本说明**：胜利条件与胜检时机、投票与平票、猎杀目标、身份公开因果、公共处决信息、初始配额下限、空场终局 `NO_SURVIVORS`  
**修订历史**：见 `Docx/archive/REVISION_LOG.md`

---

**审核清单（供审核者使用）**

- [ ] 所有术语定义清晰无歧义
- [ ] 状态转换规则完整
- [ ] 技能逻辑无漏洞
- [ ] 优先级规则无冲突
- [ ] 胜利条件数学完备
- [ ] 边界条件全面覆盖
- [ ] 可直接转换为代码实现
