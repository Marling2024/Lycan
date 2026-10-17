# 狼人杀变体游戏形式化规则说明书（Formal Specification）

> **版本**：alpha 0.3  
> **状态**：已落实 7 项规则变更（文本修订 + 人工推演）；**未经实现层验证**，且存在未决事项  
> **最后更新**：2026/10/17  
> **本版说明**：胜利条件与胜检时机、投票与平票、猎杀目标、身份公开因果、公共处决信息、初始配额下限  
> **依据**：基于 `Docx/RULES.md` 改进而来；本版**不宣称规则已完全闭合**，未决事项集中见 `8.5`

---

## 文档信息（Document Info）

| 项目 | 内容 |
|------|------|
| **文档版本** | `alpha 0.3` |
| **版本说明** | 7 项规则变更（见「版本摘要」）：胜利条件、结算单元胜检、投票三轮 + 终局表决、猎杀目标限制、技能公开因果、公共处决不公开身份、初始嗜血者下限 |
| **文档状态** | 文本修订完成，**未经实现层验证**；存在未决事项（见 `8.5`） |
| **最后更新** | 2026/10/17 |
| **依据文档** | `Docx/RULES.md`（严谨版规则说明书） |
| **定位** | 程序实现的唯一权威规则来源（Single Source of Truth） |
| **覆盖范围** | 4 阵营 · 10 种身份 · 5 阶段轮次循环 · 全部技能与判定规则 |
| **修订历史** | 见 `Docx/archive/REVISION_LOG.md`（过程记录，非规则来源） |
| **审核记录** | 见 `Docx/archive/VERIFICATION_LOG.md`（审核意见状态对照） |
| **未决事项** | 见「第八部分：验证状态与待办事项」`8.5`（空场结果、批内豁免判定、痴呆者追加投票与终局表决、追加辩论预算、异星客免疫反馈） |

**结构总览**

| 部分 | 内容 | 覆盖小节 |
|------|------|---------|
| 第一部分 | 形式化基础定义 | `1.1`–`1.5`（游戏实例与术语约定、玩家状态、胜利条件、处决机制、初始化配置） |
| 第二部分 | 角色技能形式化定义 | `2.1`–`2.5`（平凡 / 邪恶 / 治安 / 混沌四阵营共 10 种身份） |
| 第三部分 | 游戏流程时序定义 | `3.1`–`3.2`（五阶段轮次循环、结算单元与胜利检定时序） |
| 第四部分 | 约束与不变式 | `4.1`–`4.3`（全局不变式、技能约束、边界条件） |
| 第五部分 | 优先级与冲突解决 | `5.1`–`5.3`（豁免优先级、胜利检定优先级、同阶段行动顺序） |
| 第六部分 | FAQ 形式化解答 | `Q1`–`Q9`（含与原规则的裁定差异说明） |
| 第七部分 | 实现检查清单 | `7.1`–`7.7`（数据结构、状态机、技能、投票、豁免、随机性、测试） |
| 第八部分 | 验证状态与待办事项 | `8.1`–`8.5`（验证路径、已知事项、已解决问题、待办、**未决事项**） |
| 附录 A | 术语对照表 | 形式化术语 ↔ 原规则术语 ↔ 英文术语 |

**阅读路径建议**

- **只看规则**：第一部分 → 第五部分
- **准备实现**：第三部分 → 第四部分 → 第七部分
- **评审仲裁**：第六部分 → 第八部分

---

## 第一部分：形式化基础定义

### 1.1 游戏实例与术语约定（Game Instance & Terminology）

#### 1.1.1 游戏实例

一个游戏实例 `G` 定义为八元组：

```
G = (P, Id, Seat, R, F, S, T, C)
```

其中：
- `P = {p₁, p₂, ..., pₙ}` 是玩家集合，`n ∈ [6, 18]`（合法玩家数范围）
- `Id` 是玩家标识函数：`P → PlayerId`，为每个玩家分配唯一索引（见 1.1.2"术语约定"）
- `Seat` 是席位分配函数：`P → SeatId`，`SeatId = {1, 2, ..., n}`，用于发言与裁定排序
- `R` 是身份分配函数：`P → IdentityType`（即身份池到玩家的落地映射，见 1.5）
- `F` 是阵营分配函数：`P → Faction`（由身份唯一确定）
- `S` 是游戏状态，包含所有玩家的当前状态
- `T` 是当前轮次计数器：`T ∈ ℕ₀`（非负整数）
- `C` 是配置参数集（身份配比、随机种子等）

#### 1.1.2 术语约定（本文档唯一口径）

| 术语 | 英文 | 含义 |
|------|------|------|
| **身份** | Identity | 玩家获得的、带有技能与阵营（及种族）属性的属性集合。本文档中的"身份"仅此一义 |
| **身份池** | Identity Pool | 待分配给玩家的身份多重集（初始化时构建，见 1.5） |
| **身份牌** | Identity Card | 身份的物理载体（牌面），用于实体流程描述 |
| **玩家ID** | `player_id` | 唯一标识玩家的索引。**与"身份"无关，不是身份** |
| **席位** | `seat_id` | 排序键，用于发言顺序与猎杀分歧裁定。**不是身份** |
| **身份宣称** | `identity_claimed` | 玩家**主动陈述**的身份，可造假（见 1.2.5） |
| **身份公开** | `identity_revealed` | 真实身份被**系统强制**公开，不可造假（见 1.2.6） |
| **身份声明** | Identity Claim (speech) | 辩论时的**一类发言行为**（见 3.2.3 E 类） |

**术语辨析（易混三组）**

- **身份 vs 玩家ID / 席位**：身份是属性，玩家ID 与席位是索引；后两者不是身份
- **身份宣称 vs 身份声明**：前者是状态字段，后者是发言行为；宣称必然发生，声明是可选发言
- **身份宣称 vs 身份公开**：前者可造假且可改口，后者由系统强制且不可逆

**不变性**

- **玩家ID 不变性**：`∀p ∈ P, ∀t: Id(p)` 恒定
- **席位不变性**：`∀p ∈ P, ∀t: Seat(p)` 恒定（即使玩家阵亡或复活）
- **身份不可变性**：`∀p ∈ P, ∀t: R(p)` 恒定（除复活的时序影响外不改变身份本身）

#### 1.1.3 表示法约定

- `[关键词]`：技术术语或状态标识
- `{集合}`：数学集合表示
- `→`：状态转换或映射关系
- `∈`：属于关系
- `∧`：逻辑与
- `∨`：逻辑或
- `¬`：逻辑非
- `∀`：全称量词（对所有）
- `∃`：存在量词（存在）

#### 1.1.4 相关文档

| 文档 | 关系 |
|------|------|
| `Docx/RULES.md` | **依据文档**：本文档由它改进而来；两者术语尚存在差异，见第八部分 |
| `Docx/archive/REVISION_LOG.md` | **修订日志**：本文档的完整施工历史（过程记录，非规则来源） |
| `Docx/archive/VERIFICATION_LOG.md` | **验证记录**：审核意见与验证结论的状态对照（判断某条意见是否仍有效的依据） |
| `Docx/archive/FORMAL_RULES_AUDIT_REPORT.md` | **审核报告**：外部审核意见（历史快照，需对照验证记录使用） |
| `Docx/archive/RULES_ANALYSIS.md` | **漏洞分析**：本文档修复目标的来源 |
| `Agent.md` / `architecture.md` / `ToDo.md` | **项目上下文**：工程约束与推进路线 |

### 1.2 玩家状态（Player State）

每个玩家 `p ∈ P` 在时刻 `t` 的状态定义为（九元组）：

```
State(p, t) = (alive, role, faction, species, identity_claimed, identity_revealed, protected, sealed, last_used_round)
```

| 字段 | 含义 | 取值域 | 定义处 |
|------|------|--------|--------|
| `alive` | 存活状态 | `{ALIVE, DEAD}` | 1.2.1 |
| `role` | 真实身份 | `RoleType` | 1.2.2 |
| `faction` | 阵营 | `Faction` | 1.2.3 |
| `species` | 种族标签 | `Species` | 1.2.4 |
| `identity_claimed` | 玩家**宣称**的身份（可造假、可改口） | `{None} ∪ RoleType` | 1.2.5 |
| `identity_revealed` | 真实身份是否已被**系统公开** | `{false, true}` | 1.2.6 |
| `protected` | 圣骑士庇护状态 | `{NONE, PROTECTED}` | 1.2.7 |
| `sealed` | 圣骑士封印状态 | `{NONE, SEALED}` | 1.2.8 |
| `last_used_round` | 最近一次施放技能（【回魂秘术】）的轮次；`⊥` 表示尚未使用 | `{⊥} ∪ ℕ` | 2.4.1 / 4.2 C2 |

> 说明：`role` 由 1.1.1 的 `R` 确定且不可变；`identity_claimed` 与 `identity_revealed` 是
> 两个**独立**字段，切勿合并理解（辨析见 1.1.2）。

#### 1.2.1 存活状态（Alive Status）

```
alive(p, t) ∈ {ALIVE, DEAD}
```

- **初始状态**：`∀p ∈ P: alive(p, 0) = ALIVE`
- **不可逆性（带复活例外）**：`alive(p, t) = DEAD → ∀t' > t: alive(p, t') = DEAD ∨ (∃ 复活事件发生于 (t, t'])`
- **权利约束**：
  - `alive(p, t) = DEAD → ¬canVote(p, t) ∧ ¬canSpeak(p, t) ∧ ¬canUseSkill(p, t)`
  - **例外**：复活技能可使 `alive(p, t') = ALIVE`（其中 `t' > t`）

#### 1.2.2 身份类型（Identity Type）

```
RoleType = {
  CIVILIAN,        // 普通人
  WHISTLEBLOWER,   // 吹哨人
  CORONER,         // 验尸官
  BLOODTHIRSTY,    // 嗜血者
  TRAITOR,         // 非常者
  ARCHMAGE,        // 大法师
  PALADIN,         // 圣骑士
  EXECUTIONER,     // 圣杀者
  LUNATIC,         // 痴呆者
  ALIEN            // 异星客
}
```

> **命名约定**：枚举名 `RoleType` 沿用原文档变量名以保持实现契约稳定，**语义上它就是"身份类型"**。
> 本文档中 `IdentityType ≡ RoleType`，仅在讨论"身份的取值域"时使用 `IdentityType` 这一写法。

- **不可变性**：身份分配在游戏初始化后永不改变

**身份唯一性约束（Identity Uniqueness Constraint）**

身份的唯一性分为两类：

- **唯一身份（Unique Identities）**：单局游戏中**至多**出现一次；是否进池由外部配置的 `role_pool_flags` 决定，**可以完全不进池（0 个）**（见 1.5.1）
  - 治安阵营全部身份：`EXECUTIONER`（圣杀者）、`PALADIN`（圣骑士）、`ARCHMAGE`（大法师）
  - 混沌阵营全部身份：`LUNATIC`（痴呆者）、`ALIEN`（异星客）
  - 邪恶阵营：`TRAITOR`（非常者）
  - 平凡阵营信息位：`WHISTLEBLOWER`（吹哨人）、`CORONER`（验尸官）
- **可重复身份（Repeatable Identities）**：单局游戏中可出现多次（数量由初始化配置决定，见 1.5）
  - `CIVILIAN`（普通人）—— 主要作用是"凑数"，用于稀释特殊身份密度、强化推理测试
  - `BLOODTHIRSTY`（嗜血者）—— 每局**至少 1 个**（下限约束），上限 `⌊n/3⌋`

形式化表示（完整约束见 1.5.3 的 RC1–RC5）：

```
∀r ∈ UniqueIdentity = {EXECUTIONER, PALADIN, ARCHMAGE, LUNATIC, ALIEN, TRAITOR, WHISTLEBLOWER, CORONER},

  |{p ∈ P : R(p) = r}| ∈ {0, 1}        // 可被配置裁掉，但一旦分配至多 1 个

1 ≤ |{p ∈ P : R(p) = BLOODTHIRSTY}| ≤ ⌊n/3⌋

|{p ∈ P : R(p) = CIVILIAN}| ≥ 0        // 吃余量，不设下限


```
**注**：本条为身份唯一性的定义性表述；配额计算、超限随机抽取与合规判定统一在 1.5.3 中给出。

#### 1.2.3 阵营（Faction）

```
Faction = {COMMONER, EVIL, LAW, CHAOS}
```

阵营映射函数 `F: RoleType → Faction` 定义为：

```
F(CIVILIAN)        = COMMONER
F(WHISTLEBLOWER)   = COMMONER
F(CORONER)         = COMMONER
F(BLOODTHIRSTY)    = EVIL
F(TRAITOR)         = EVIL
F(ARCHMAGE)        = LAW
F(PALADIN)         = LAW
F(EXECUTIONER)     = LAW
F(LUNATIC)         = CHAOS
F(ALIEN)           = CHAOS
```

#### 1.2.4 种族标签（Species）

```
Species = {HUMAN, NONHUMAN, TRANSCENDENT}
```

种族映射函数 `Sp: RoleType → Species` 定义为：

```
Sp(CIVILIAN)        = HUMAN
Sp(WHISTLEBLOWER)   = HUMAN
Sp(CORONER)         = HUMAN
Sp(TRAITOR)         = HUMAN
Sp(LUNATIC)         = HUMAN
Sp(BLOODTHIRSTY)    = NONHUMAN
Sp(ALIEN)           = NONHUMAN
Sp(ARCHMAGE)        = TRANSCENDENT
Sp(PALADIN)         = TRANSCENDENT
Sp(EXECUTIONER)     = TRANSCENDENT
```

- **不可变性**：种族由角色唯一确定，游戏过程中不变

#### 1.2.5 身份宣称状态（Identity Claim Status）

```
identity_claimed(p, t) ∈ {None} ∪ RoleType
```

- **初始状态**：`∀p ∈ P: identity_claimed(p, 0) = None`
- **可变性**：玩家可以在辩论时多次修改宣称的身份（允许"改口"）
- **语义**：记录玩家**宣称**的身份，可以与真实身份 `R(p)` 不一致
- **约束**：
```
  identity_claimed(p, t) = r → r ∈ RoleType ∨ r = None
```

#### 1.2.6 身份公开状态（Identity Reveal Status）

```
identity_revealed(p, t) ∈ {false, true}
```

- **初始状态**：`∀p ∈ P: identity_revealed(p, 0) = false`
- **不可逆性**：`identity_revealed(p, t) = true → ∀t' > t: identity_revealed(p, t') = true`
- **唯一触发条件（治安阵营技能）**：
  - `usesSkill(p, t) ∧ F(R(p)) = LAW → identity_revealed(p, t+1) = true`
  - **因果方向**：满足技能自身条件并施放技能 **→** 施技者真实身份公开；
    公开状态 **不是** 施放技能的前置条件（技能可用性只见 2.4 与 3.2.5）
  - 技能触发时，系统强制设置：`identity_claimed(p, t+1) := R(p)`（覆盖之前的宣称）
  - 该公开**立即生效**：不因技能效果被处决豁免而落空，也不因对局随即结束而省略
- **不触发身份公开的事件**：
  - **公共处决**（投票处决、终局表决的整批处决、圣杀者【铁腕神裁】）——不公开**被处决者**的真实身份
  - **秘密处决**（夜间猎杀）——只公布死亡事实
  - 玩家**死亡**与玩家**复活**
- **语义边界**：`identity_revealed` **只表示"真实身份是否已被系统公开"这一事实**，
  **不得**兼任技能使用次数标志，也 **不得** 作为技能可用性判据
- **语义**：记录玩家的**真实身份**是否已被系统强制公开

**术语澄清与状态关系**：

| 术语 | 对应状态字段 | 语义 | 可造假 | 触发方式 |
|------|------------|------|--------|---------|
| **身份宣称（Claim）** | `identity_claimed` | 玩家主观陈述的身份 | ✅ 可以 | 玩家主动宣称 |
| **身份公开（Reveal）** | `identity_revealed` | 真实身份被系统公开 | ❌ 不可以 | 系统强制（治安技能触发） |

> 注意区分：**身份宣称**是状态字段（本表），**身份声明**是辩论时的一类发言行为（见 3.2.3 E 类）。

**状态组合示例**：

1. **诚实的信息位**：
   - `R(p) = WHISTLEBLOWER`
   - `identity_claimed(p, t) = WHISTLEBLOWER`
   - `identity_revealed(p, t) = false`
   - 解读：玩家宣称自己是吹哨人（真实），但系统未强制验证

2. **伪装的嗜血者**：
   - `R(p) = BLOODTHIRSTY`
   - `identity_claimed(p, t) = WHISTLEBLOWER`
   - `identity_revealed(p, t) = false`
   - 解读：嗜血者谎称自己是吹哨人（欺骗策略）

3. **已公开的圣骑士**：
   - `R(p) = PALADIN`
   - `identity_claimed(p, t) = PALADIN`
   - `identity_revealed(p, t) = true`
   - 解读：圣骑士使用技能后，系统强制公开真实身份

#### 1.2.7 保护状态（Protected Status）

```
protected(p, t) ∈ {NONE, PROTECTED}
```

- 仅在圣骑士对非嗜血者目标施放守护后生效
- **持续时长**：从施放时刻 `t₀` **立即生效**，直到**以下两个事件中先发生者**：
  ① 圣骑士的**下一次行动**（即下一轮裁判时中圣骑士的判定窗口被开启——该窗口无论是否再次施放，
  旧庇护均在此失效）；② 圣骑士被处决。覆盖阶段为：本轮猎杀时 + 下一轮信息时、辩论时、投票时
  （时效定义与图示见 `2.4.2`）
- **与裁判时窗口顺序的关系**：裁判时窗口顺序为 圣杀者 → 圣骑士，故下一轮**圣杀者窗口**开启时
  庇护**仍然有效**（与 FAQ `Q2` 一致）。本版以 `2.4.2` 的口径为准；
  `alpha 0.2` 中"到下一个裁判时阶段开始前"的表述**已废弃**

#### 1.2.8 封印状态（Sealed Status）

```
sealed(p, t) ∈ {NONE, SEALED}
```

- 仅在圣骑士对嗜血者目标施放守护后生效
- **持续时长**：仅在紧接的猎杀时阶段生效

---

### 1.3 胜利条件形式化定义

#### 1.3.1 存活玩家集合

定义时刻 `t` 的存活玩家集合：

```
Alive(t) = {p ∈ P | alive(p, t) = ALIVE}
```

#### 1.3.2 阵营存活集合

定义阵营 `f` 在时刻 `t` 的存活成员集合：

```
Alive_f(t) = {p ∈ Alive(t) | F(R(p)) = f}
```

#### 1.3.3 嗜血者存活集合（特殊定义）

```
Bloodthirsty_Alive(t) = {p ∈ Alive(t) | R(p) = BLOODTHIRSTY}
```

#### 1.3.4 胜利检定函数（Victory Check）

记时刻 `t` 的五个人数口径：

```
N(t) = |Alive(t)|                    // 存活玩家总数
B(t) = |Bloodthirsty_Alive(t)|       // 存活嗜血者人数
C(t) = |Alive_CHAOS(t)|              // 存活混沌阵营人数
L(t) = |Alive_LAW(t)|                // 存活治安阵营人数
P(t) = |Alive_COMMONER(t)|           // 存活平凡阵营人数
```

对 `N(t) > 0`，胜利检定函数 `V: State → Faction ∪ {ONGOING}` 定义为：

```
V(S, t) =
  if N ≤ 3 ∧ C > 0:                 CHAOS
  else if 2 × B ≥ N:                EVIL
  else if B = 0 ∧ C = 0 ∧ L > 0:    LAW
  else if B = 0 ∧ C = 0 ∧ P > 0:    COMMONER
  else:                             ONGOING
```

**优先级顺序**：`CHAOS > EVIL > LAW > COMMONER`（按上列判定次序实现）

**关键澄清**：
- **平凡 / 治安不再要求"邪恶阵营整体不存在"**：两者的共同前提是 `B = 0 ∧ C = 0`，
  即**不存在存活嗜血者**且**不存在存活混沌阵营玩家**；
  **非常者（TRAITOR）是否存活不影响**这两个分支——非常者仍属邪恶阵营，但**不计入 `B`**
- **治安优先于平凡**：`B = 0 ∧ C = 0` 且 `L > 0 ∧ P > 0` 时 `LAW` 分支先命中，治安获胜；
  两个分支不再是"互斥关系"，而是**优先级关系**
- **邪恶条件改用整数运算**：`2 × B ≥ N` 与旧的实数写法 `B / N ≥ 0.5` 等价，但避免浮点除法
- **`N = 0` 尚未裁定**：函数仅对 `N > 0` 定义。批量处决可能导致空场，
  其胜负结果**未获批准**——既不得让 `2 × B ≥ N`（即 `0 ≥ 0`）意外触发 `EVIL`，
  也不得由实现自行定义为平局或某阵营获胜；应作为未裁定状态上报（见 `8.5`）
- **不新增未经确认的规则**：如"非常者独存即邪恶获胜"**不在本版规则之内**
- **判定时机**：不再表述为"每次处决结算后"；改为**每个改变存活人数的结算单元完成后**检定
  （结算单元清单、批量处决与不检定情形见 `3.2.7`）
- **复活计入**：大法师复活的玩家在复活后立即进入 `Alive(t)`；复活本身是一个结算单元，同样触发检定

---

### 1.4 处决机制（Execution Mechanism）

#### 1.4.1 处决类型

```
ExecutionType = {PUBLIC, SECRET}
```

- **公共处决（PUBLIC）**：白昼投票或治安阵营特权处决
- **秘密处决（SECRET）**：嗜血者夜间猎杀

**执行人可见性**

1. **公共处决（Public Execution）**
   - **执行人可见性**：执行人向全场公开
   - **执行人类型**：
     - **全体投票**：投票时阶段，所有存活玩家的多数票裁决（含终局表决的整批处决）
     - **圣杀者（EXECUTIONER）**：裁判时阶段，圣杀者施放【铁腕神裁】
   - **公开内容**：被处决的玩家、处决与死亡结果、按既有规则公开的投票记录或施技者信息
   - **不公开内容**：**死者的真实身份牌**。公共处决 **不设置**死者的 `identity_revealed`，
     也 **不改写** 死者的 `identity_claimed`；若死者身份在处决前已因其他合法事件公开，死亡不撤销该事实
   - **形式化定义**：
```
     PublicExecution := {
       executor: {ALL_VOTERS} ∪ {EXECUTIONER},
       executor_visible: true,
       target_visible: true,             // 被处决者与其死亡结果公开
       target_identity_visible: false,   // 不公开死者真实身份
       phase: VOTING | ADJUDICATION
     }
```

2. **秘密处决（Secret Execution）**
   - **执行人可见性**：执行人对非嗜血者保密
   - **执行人类型**：
     - **嗜血者全体（All BLOODTHIRSTY）**：猎杀时阶段，所有存活且未被封印的嗜血者集体投票决定
   - **目标可见性**：仅公布死亡事实，不公开**实施者**
   - **形式化定义**：
```
     SecretExecution := {
       executor: {p ∈ P_alive : R(p) = BLOODTHIRSTY ∧ sealed(p, t) = NONE},
       executor_visible: false,  // 对非嗜血者不可见
       target_visible: true,     // 死亡事实公开，但不公开实施者
       phase: HUNT
     }
```

**执行人公开性对比表**：

| 处决类型 | 执行人 | 执行人是否公开 | 触发阶段 |
|---------|--------|--------------|---------|
| 公共处决 | 全体投票 | ✅ 公开（投票记录） | 投票时 |
| 公共处决 | 圣杀者（EXECUTIONER） | ✅ 公开（技能施放时自动公开） | 裁判时 |
| 秘密处决 | 嗜血者全体（BLOODTHIRSTY） | ❌ 不公开（对非嗜血者保密） | 猎杀时 |

#### 1.4.2 处决函数

```
execute(p, t, type) → State
```

**前置条件**：
- `alive(p, t) = ALIVE`
- `protected(p, t) = NONE` （若为 PROTECTED，处决失效）

**效果**：
- `alive(p, t+1) = DEAD`
- 若 `type = PUBLIC`：全场知晓**被处决者与其死亡结果**；**不公开其真实身份**
  （`identity_revealed` 与 `identity_claimed` 均不因公共处决改变）
- 若 `type = SECRET`：仅 GM 与嗜血者知晓实施过程，白天公布死亡事实
- 无论何种类型：**死亡本身不触发身份公开**（身份公开的唯一触发条件见 1.2.6）

**豁免规则**：
- 若 `protected(p, t) = PROTECTED`：处决无效，`alive(p, t+1) = ALIVE`
- 若 `R(p) = ALIEN ∧ type = SECRET`：处决无效（相位躯壳特性）
- **与圣杀者的交互**（对应 FAQ Q2）：受保护目标面对 `EXECUTIONER` 的【铁腕神裁】同样免疫，
  该结论在 `2.4.3` 的技能前置条件中实现；本条豁免适用于**所有** `ExecutionType`，故优先于圣杀者处决

---

### 1.5 游戏初始化配置（Game Initialization）

> 本节定义"如何从 `n` 与配置参数生成一个合法的身份池，并把身份随机落到玩家ID上"。
> 引擎接口边界见 1.5.2。

#### 1.5.1 身份池构建算法

游戏开始前，需要根据总玩家人数 `n` 和配置参数生成合法的身份池。算法分两步：
**先确定配额 `k`（见 1.5.2 的输入参数与 1.5.3 的 `RC1–RC5` 约束），再把身份随机分配给玩家ID（见 1.5.4）**。

#### 1.5.2 输入参数（引擎最小接口）
> **接口设计原则**：游戏引擎**保持少接口**。人数配置、各身份数量策略等"实现层"逻辑
> 交由**测试脚本层**与**用户交互层**负责；引擎只暴露"**定义某个身份是否进入身份池**"这一个开关，
> 以及在池子超限时的默认兜底行为（均匀随机抽取）。

```
InitConfig := {
  n: ℕ,                                    // 总玩家人数，n ∈ [6, 18]（由外部给定）
  role_pool_flags: IdentityType → {0, 1},  // 唯一接口：身份是否进入身份池（1=进入，0=排除）
  k_civilian: ℕ,                           // 普通人数量（由外部最终确定，引擎不推导）
  k_bloodthirsty: ℕ                        // 嗜血者数量（由外部最终确定，引擎不推导）
                                           // 须满足 1 ≤ k_bloodthirsty ≤ ⌊n/3⌋（RC1，引擎强制校验）
}
```

- **引擎不提供**"按人数自动推导各身份数量"的接口；`k_civilian` 与 `k_bloodthirsty` 由外部传值
- **引擎不提供**优先级、权重、裁剪顺序等参数；超限时只有一种默认行为：**均匀随机抽取**
- 外部若只想说"这个身份进池子"，就只给 `role_pool_flags`；数量策略完全留在引擎之外

#### 1.5.3 配额约束（RC1–RC5；其中 RC4 为构造规则）

**RC1. 嗜血者约束（引擎强制校验）**：
```
1 ≤ k(BLOODTHIRSTY) ≤ ⌊n/3⌋
```
即每局 **至少 1 个**嗜血者，**不得超过总人数的 1/3**（向下取整）。

> **强制校验要求**：配置验证必须**拒绝** `k_bloodthirsty = 0`（以及 `k_bloodthirsty > ⌊n/3⌋`），
> 并抛出 `ConfigError`；引擎**不得**放行零嗜血者配置。"每局至少 1 个嗜血者"不再是仅由外层负责的
> 成局约束，而是**引擎侧的初始化约束**（同步见 `1.5.6`、`4.1 INV6`、`4.3 B5`）。
>
> **作用范围**：该约束针对**初始身份配置**，**不表示**游戏过程中始终至少有一名嗜血者存活
> （嗜血者可在其后的处决中被全部消灭）。

示例：
- 6 人局：1–2 个
- 7 人局：1–2 个
- 9 人局：1–3 个
- 15 人局：1–5 个

**RC2. 唯一身份约束**（参见 1.2.2）：
```
∀r ∈ UniqueIdentity = {EXECUTIONER, PALADIN, ARCHMAGE, LUNATIC, ALIEN, TRAITOR, WHISTLEBLOWER, CORONER}:
  |{p : R(p) = r}| ∈ {0, 1}
```
即唯一身份 **要么不进池子、要么恰好 1 个**。

**RC3. 身份启用约束**：
```
∀p ∈ P: role_pool_flags[R(p)] = 1
```
即只有被标记为"进入身份池"的身份才能出现在最终身份池中。

**RC4. 容量约束（构造规则：超限即随机抽取）**：
```
|{p : R(p) ∈ UniqueIdentity}| = min( |{r ∈ UniqueIdentity : role_pool_flags[r] = 1}|,
                                     n − k(BLOODTHIRSTY) − k(CIVILIAN) )
```
即 **进入池子的唯一身份数量，取"启用数量"与"余量"的较小值**。余量为
"总人数 − 外部设定的嗜血者数量 − 外部设定的普通人数量"。

- 若启用的唯一身份数量 **≤** 余量：全部进入池子
- 若 **>** 余量：**默认从启用集合中均匀随机抽取**，抽满余量为止；**不引入任何优先级或权重**
- 随机抽取由 `prng` 驱动，因此同一 seed 结果可复现
- 因此该式**恒成立**，它是构造规则而非待校验条件（`VALID` 中不重复列出）

**RC5. 总量约束**：
```
k_bloodthirsty + k_civilian ≤ n                        // 外部配置自洽性
k(CIVILIAN) = n − k(BLOODTHIRSTY) − |{p : R(p) ∈ UniqueIdentity}|   // 余量闭合
Σ_{r ∈ IdentityType} k(r) = n
```
- 第一步为**外部配置的自洽性检查**：若 `k_bloodthirsty + k_civilian > n`，配置本身不成立
- 第二步说明：`k_civilian` 是外部给定值，其余量部分由超限随机抽取独占，二者必须闭合；
  若外部给定的 `k_civilian` 与闭合值不等，则判为非法配置（抛出 `ConfigError`）
- 第三步：所有身份数量之和恰等于玩家人数

**配额算法（两步）**

> **实现注记**：本节伪代码采用与 `IdentityType` 定义完全一致的标识符（`CIVILIAN` / `BLOODTHIRSTY`），
> 以避免与 `Faction` 中的阵营名 `COMMONER` 混淆。

```python
# 步骤 0：RC1 强制校验（拒绝零嗜血者，拒绝超过 ⌊n/3⌋）
#   0a. 若 k_bloodthirsty = 0 或 k_bloodthirsty > ⌊n/3⌋ → ConfigError

# 步骤 1：唯一身份进池，超限则均匀随机抽取
#   1a. 按 role_pool_flags 取出启用的唯一身份（引擎唯一接口）
#   1b. 计算余量 capacity = n - k_bloodthirsty - k_civilian
#   1c. 若 |candidates| > capacity：从 candidates 中均匀随机抽取 capacity 个
#       否则：candidates 全部进池
#   1d. 被抽中的每个唯一身份恰好 1 个

# 步骤 2：普通人数量由外部给定
#   2a. k(CIVILIAN) 由外部传值，引擎不推导
#   2b. 若 k_bloodthirsty + k_civilian > n，则配额不成立（ConfigError）
```

```python
UniqueIdentity = {
    EXECUTIONER, PALADIN, ARCHMAGE,
    LUNATIC, ALIEN,
    TRAITOR,
    WHISTLEBLOWER, CORONER,
}


def build_identity_pool(config: InitConfig, prng: PRNG) -> List[IdentityType]:
    n = config.n
    candidates = [r for r in UniqueIdentity if config.role_pool_flags[r] == 1]

    # 步骤 0：RC1 强制校验（下限 1，上限 ⌊n/3⌋）
    if not (1 <= config.k_bloodthirsty <= n // 3):
        raise ConfigError("嗜血者数量必须满足 1 <= k_bloodthirsty <= floor(n/3)")

    # 步骤 1：唯一身份进池；超限则均匀随机抽取（RC4）
    capacity = n - config.k_bloodthirsty - config.k_civilian
    if capacity < 0:
        raise ConfigError("嗜血者与普通人数量之和已超过总人数")
    kept_unique = random_sample(candidates,
                                k=min(len(candidates), capacity),
                                prng=prng)

    # 步骤 2：普通人数量由外部给定，引擎不推导
    k_civilian = config.k_civilian

    pool = (kept_unique
            + [BLOODTHIRSTY] * config.k_bloodthirsty
            + [CIVILIAN] * k_civilian)
    assert len(pool) == n
    return pool


def is_unique(identity: IdentityType) -> bool:
    """判断身份是否具有唯一性（与 1.2.2 的 UniqueIdentity 集合一致）"""
    return identity in UniqueIdentity
```

**合规化判定（判定点：配额构建后、玩家分配前）**

```
VALID(k, n) ⟺ RC1 ∧ RC2 ∧ RC3 ∧ RC5      // RC4 是构造规则，恒成立，不重复校验
                                          // RC1 含下限：k_bloodthirsty = 0 判为非法

若 ¬VALID:
  1. 超限（启用的唯一身份多于余量）⟹ 均匀随机抽取至余量（正常路径，不是错误）
  2. `k_bloodthirsty = 0` 或 `k_bloodthirsty > ⌊n/3⌋` ⟹ 抛出 ConfigError（RC1 强制校验）
  3. 配额本身不成立（如 k_bloodthirsty + k_civilian > n）⟹ 抛出 ConfigError
```

> **设计意图**：合规判定对**每次实际生成的配额 `k`** 做校验，而非针对预先写死的推荐配置表
> （原附录 B 已删除）。引擎侧只有"身份是否进池"一个接口，加"超限随机抽取"一个默认行为；
> 人数与配比策略完全留在测试脚本层 / 用户交互层。

#### 1.5.4 随机分配给玩家ID

配额确定后，需要把身份池中的每个身份落到具体玩家上。

```
σ : IdentityType → 2^P           // 身份 → 拥有该身份的玩家集合

要求：
  ∀r: |σ(r)| = k(r)              // 数量匹配配额
  ⨆_{r ∈ IdentityType} σ(r) = P  // 每个玩家恰好持有一个身份（双射）
  ∀r ≠ r': σ(r) ∩ σ(r') = ∅      // 身份集合互不相交
```

**算法（随机抽取尚未被分配的玩家ID）**：

```python
def assign_identities(pool: List[IdentityType], players: List[PlayerId],
                      prng: PRNG) -> Dict[IdentityType, List[PlayerId]]:
    unassigned = list(players)          # 尚未被分配的玩家ID集合
    groups = group_by_identity(pool)    # 按身份聚合，得到每个身份的 k(r)

    sigma = {}
    for identity in prng.shuffle(list(groups.keys())):   # 分配顺序本身也随机化
        count = groups[identity]
        picks = prng.sample(unassigned, count)           # 均匀随机抽取 count 个不同玩家
        sigma[identity] = picks
        unassigned = [p for p in unassigned if p not in picks]
    assert not unassigned               # 分配完毕，无剩余玩家
    return sigma
```

**性质**：
- **均匀性**：对任意玩家 `p` 与任意身份 `r`，`Pr[p ∈ σ(r)] = k(r)/n`
- **可复现性**：分配完全由 `prng`（初始化时固定 seed）驱动，同一 seed 得到同一分配
- **顺序无关推导**：上述"逐个身份抽取"与"随机洗牌后按序切片"在分布上等价

#### 1.5.5 身份池构建示例

> 下列由 `1.5.1` 的两步算法直接推出；所有示例均满足 RC1–RC5，且**不预设任何固定配置表**。
> 示例中 `k_bloodthirsty` 与 `k_civilian` 均由外部给定，引擎不做推导。

**6 人局（全开，超限 → 随机抽取）**

```
n = 6
role_pool_flags: 全部唯一身份置 1（8 个候选）
k_bloodthirsty = 2
k_civilian    = 0
```
- 余量 `capacity = 6 − 2 − 0 = 4`；候选 8 个 > 4，**均匀随机抽取 4 个**
- 结果：`随机唯一身份×4 + BLOODTHIRSTY×2`（`CIVILIAN×0`）
- 注：抽中哪 4 个取决于 seed，不再由任何优先级决定

**7 人局（超限 → 随机抽取）**

```
n = 7
role_pool_flags: 全部唯一身份置 1
k_bloodthirsty = 2
k_civilian    = 0
```
- 余量 `capacity = 7 − 2 − 0 = 5`；候选 8 个 > 5，**均匀随机抽取 5 个**
- 结果：`随机唯一身份×5 + BLOODTHIRSTY×2`

**9 人局（超限 → 随机抽取）**

```
n = 9
role_pool_flags: 全部唯一身份置 1
k_bloodthirsty = 3
k_civilian    = 0
```
- 余量 `capacity = 9 − 3 − 0 = 6`；候选 8 个 > 6，**均匀随机抽取 6 个**
- 结果：`随机唯一身份×6 + BLOODTHIRSTY×3`

**10 人局（部分启用 + 随机抽样，普通人由外部给定）**

```
n = 10
role_pool_flags = {TRAITOR: 1, WHISTLEBLOWER: 1, CORONER: 1, PALADIN: 1, 其余唯一身份: 0}
k_bloodthirsty = 2
k_civilian    = 4
```
- 余量 `capacity = 10 − 2 − 4 = 4`；候选 4 个 = 4，**全部进池**
- 结果：`[BLOODTHIRSTY×2, TRAITOR, WHISTLEBLOWER, CORONER, PALADIN, CIVILIAN×4]`

**15 人局（全开，不超限）**

```
n = 15
role_pool_flags: 全部唯一身份置 1
k_bloodthirsty = 5
k_civilian    = 2
```

- 余量 `capacity = 15 − 5 − 2 = 8`；候选 8 个 = 8，**全部进池**
- 结果：`BLOODTHIRSTY×5 + 唯一身份×8 + CIVILIAN×2`

**唯一身份全部排除的极端配置（纯推理局）**

```
n = 7
role_pool_flags: 全部唯一身份置 0   // 8 个唯一身份一个都不进池
k_bloodthirsty = 1
k_civilian    = 6
```

- 余量 `capacity = 7 − 1 − 6 = 0`；候选 0 个（全部排除），**抽取 0 个**
- 结果：`[BLOODTHIRSTY, CIVILIAN×6]`，唯一身份 0 个
- 说明：外部只用了"是否进池"这一个开关，就把唯一身份全部关掉；普通人的"凑数 + 推理压力"作用被放到最大

> **关于小局普通人数量**：上列 6/7/9 人局之所以是 `CIVILIAN×0`，是因为示例里外部把 `k_civilian` 设成了 0。
> 引擎不推导普通人数，所以**想要多少普通人在外部直接给 `k_civilian` 即可**，无需修改任何引擎规则。

#### 1.5.6 配置验证检查清单

在游戏启动前，必须验证 `VALID(k, n)`：

- [ ] `n ∈ [6, 18]`（玩家数范围，与 1.1 的定义统一）
- [ ] 身份池规模恰好等于玩家人数（RC5）：`Σ_r k(r) = n`
- [ ] 至少且不超过 1/3 的嗜血者（RC1）：`1 ≤ k(BLOODTHIRSTY) ≤ ⌊n/3⌋`；
      **`k_bloodthirsty = 0` 必须被拒绝并抛 `ConfigError`**（不得放行零嗜血者配置）
- [ ] 唯一身份要么 0 个、要么恰好 1 个（RC2）
- [ ] 未启用的身份数量为 0（RC3）
- [ ] 唯一身份数量等于"启用数量"与"余量"的较小值（RC4，构造规则，恒成立；此处仅作核对）
- [ ] 普通人数量等于外部给定值，且总量自洽（RC5）：`k(BLOODTHIRSTY) + k(CIVILIAN) ≤ n`
- [ ] 若候选唯一身份超出余量，已执行**均匀随机抽取**（而非优先级裁剪）

---

## 第二部分：角色技能形式化定义

### 2.1 技能表示法

每个技能定义为五元组：

```
Skill = (name, phase, trigger, precondition, effect)
```

其中：
- `name`：技能名称
- `phase`：可触发的游戏阶段
- `trigger`：触发类型（主动/被动/强制）
- `precondition`：前置条件（谓词逻辑表达式）
- `effect`：效果函数（状态转换）

---

### 2.2 平凡阵营技能

#### 2.2.1 普通人（CIVILIAN）

```
Skill_CIVILIAN = (
  name: "凡夫之身",
  phase: NONE,
  trigger: PASSIVE,
  precondition: TRUE,
  effect: NONE
)
```

**说明**：无特殊能力，仅依赖投票权

---

#### 2.2.2 吹哨人（WHISTLEBLOWER）

```
Skill_WHISTLEBLOWER = (
  name: "警觉哨音",
  phase: INFORMATION,
  trigger: AUTOMATIC,
  precondition: alive(p_whistleblower, t) = ALIVE,
  effect: receiveInfo(p_whistleblower, randomTarget(), speciesCheck)
)

其中：
randomTarget() → p' ∈ (Alive(t) \ {p_whistleblower})  // 随机选择非自身存活玩家
speciesCheck(p') → {
  "人类" if Sp(R(p')) = HUMAN
  "非人类" otherwise
}
```

**关键澄清**：
- **随机性来源**：由 GM 使用伪随机数生成器（PRNG）选择，种子在游戏初始化时固定（保证同一 seed 可复现，见 7.6）
- **信息准确性**：100% 准确，基于种族映射函数 `Sp`
- **每轮触发**：只要吹哨人存活，每轮信息时自动触发一次

---

#### 2.2.3 验尸官（CORONER）

```
Skill_CORONER = (
  name: "死者遗言",
  phase: INFORMATION,
  trigger: ACTIVE,
  precondition: 
    alive(p_coroner, t) = ALIVE ∧
    target ∈ DeadHistory(t) ∧
    target ∉ InspectedByCoroner(t),
  effect: receiveInfo(p_coroner, target, R(target))
)

其中：
DeadHistory(t) = {p ∈ P | ∃t' < t: alive(p, t') = DEAD}  // 历史上所有阵亡玩家
InspectedByCoroner(t) = {p ∈ P | 已被验尸官查验过}  // 查验记录集合（全局唯一）
```

**关键澄清**：
- **"历史上被处决"修正为"历史上阵亡"**：包含所有死亡原因（投票、猎杀、圣杀者处决）
- **不可重复查验**：每个阵亡者只能被查验一次（跨轮次全局限制）
- **查验时机**：必须在目标阵亡后的任意一轮信息时查验

---

### 2.3 邪恶阵营技能

#### 2.3.1 嗜血者（BLOODTHIRSTY）

```
Skill_BLOODTHIRSTY = (
  name: "夜幕捕食",
  phase: HUNT,
  trigger: TEAM_ACTIVE,
  precondition: ∃p ∈ Bloodthirsty_Alive(t): sealed(p, t) = NONE,
  effect: teamVote({p ∈ Bloodthirsty_Alive(t) | sealed(p, t) = NONE}, target)
          → execute(target, t, SECRET)
)
```

合法猎杀目标（全流程统一口径）：
```
  合法猎杀目标(t) = Alive(t) \ Bloodthirsty_Alive(t)
```
即猎杀目标**必须排除全部存活嗜血者**，而不是仅排除投票者本人。

团队投票机制 teamVote:
1. 所有存活且未被封印的嗜血者进行投票（目标限于 `合法猎杀目标(t)`）
2. 若意见一致，执行处决
3. 若存在分歧：
```
   - 按席位升序排列嗜血者：{b₁, b₂, ..., bₘ}（席位 `Seat(p)` 见 1.1）
   - 取 min{i | bᵢ ∈ Bloodthirsty_Alive(t) ∧ sealed(bᵢ, t) = NONE} 的投票为最终决策
4. 若所有嗜血者均被封印，本轮猎杀自动失败（平安夜）
```

**关键澄清**：
- **席位规则**：`Seat(p) = rank_by_Id(p) ∈ {1, 2, ..., n}`，即按 `Id(p)` 升序确定排序键，与"身份"无关（见 1.1）
- **"席位靠前"定义**：`seat_id` 较小的玩家
- **封印影响**：被封印的嗜血者无投票权，但不影响其他嗜血者行动；
  **封印不改变真实身份**——被封印者仍是嗜血者，仍落在猎杀目标排除集合内
- **团队通讯**：嗜血者之间共享私密频道，可自由沟通

**初始信息同步**：
```
∀p, p' ∈ P: R(p) = BLOODTHIRSTY ∧ R(p') = BLOODTHIRSTY → knows(p, R(p'))
```
嗜血者在游戏开始时互相知晓身份

---

#### 2.3.2 非常者（TRAITOR）

```
Skill_TRAITOR = (
  name: "伪善之面",
  phase: NONE,
  trigger: PASSIVE,
  precondition: TRUE,
  effect: NONE
)

特性约束：
1. ¬canAccessChannel(p_traitor, BloodthirstyChat)  // 无法访问嗜血者频道
2. ∀p ∈ Bloodthirsty_Alive(t): ¬knows(p, R(p_traitor))  // 嗜血者不知非常者身份
3. Sp(TRAITOR) = HUMAN  // 种族检测为人类
```

**关键澄清**：
- **单向隐藏机制**：非常者不知道嗜血者身份，嗜血者也不知道非常者身份
- **完全孤立**：非常者无法参与夜间猎杀决策
- **种族优势**：在吹哨人检测中显示为"人类"

---

### 2.4 治安阵营技能

> **全局约束（本版明确因果方向）**：
> 1. **因果**：满足技能自身条件并施放技能 **→** 施技者真实身份**立即公开**
>    （`identity_revealed := true`、`identity_claimed := R(p)`）。
>    **公开状态不是施放技能的前置条件**：身份已经公开的治安角色**不因此被禁止**使用技能，
>    只需检查该技能自身的条件。
> 2. **不得遗漏公开**：技能效果因处决豁免而失效、或技能结算后对局立即结束时，
>    本次公开**依然发生并已被记录**。
> 3. **`identity_revealed` 不是使用次数标志**：不得用"身份尚未公开"实现"一局一次"之类的频次限制。
> 4. **技能频次**：现有明确规则为——每个裁判时窗口至多施放一次（见 `3.2.5`）；
>    是否存在全局次数上限**尚未裁定**（见 `8.5`）。本版**不新增**"一局一次"等限制。

#### 2.4.1 大法师（ARCHMAGE）

```
Skill_ARCHMAGE = (
  name: "回魂秘术",
  phase: ADJUDICATION,
  trigger: ACTIVE,
  precondition:
    alive(p_archmage, t) = ALIVE ∧
    canUseCooldown(p_archmage, t) ∧
    target ∈ DeadHistory(t),
  effect: 
    resurrect(target, t) → alive(target, t+1) = ALIVE,
    identity_revealed(p_archmage, t+1) = true,
    identity_claimed(p_archmage, t+1) = R(p_archmage),  // 强制覆盖为真实身份
    last_used_round(p_archmage) := t                    // 记录当轮轮次（冷却判定见下）
)
```

冷却时间机制（只需记录"最近使用轮次"）：

```
last_used_round(p, 0) = ⊥                          // ⊥ 表示本局尚未使用过
使用后：last_used_round(p_archmage) := t           // 记录当轮轮次
可用条件：canUseCooldown(p, t) ↔
          last_used_round(p) = ⊥ ∨ (t − last_used_round(p)) ≥ 2
```

**关键澄清**：

- **"每 2 轮仅可使用 1 次"的精确定义**：`当前轮次 − 最近使用轮次 ≥ 2`
  - 设第 1 轮使用 ⟹ `last_used_round = 1`
  - 第 2 轮：`2 − 1 = 1 < 2` → 不可用
  - 第 3 轮起：`3 − 1 = 2 ≥ 2` → 可用（此后同理）
- **无需单独定义冷却的生命周期**：`last_used_round` **只在施放技能时被写入**，
  因此玩家阵亡期间该值自然保持不变（等效于"死亡瞬间冻结"），被复活后沿用同一判定。
  - 注：大法师是**唯一身份**且死后无行动权（无法施放复活），故"复活大法师"当前无触发途径；
    本条为未来扩展保持语义完整
- **复活玩家行动权**：
  - 复活发生在裁判时，复活后的玩家 **立即恢复所有权利**
  - 在本轮后续阶段（猎杀时）可正常参与
  - 在下一轮所有阶段（信息时、辩论时、投票时等）均可正常行动
- **复活治安角色限制**：复活的治安角色 **不能在本轮再次使用技能**（避免连锁触发）

---

#### 2.4.2 圣骑士（PALADIN）

```
Skill_PALADIN = (
  name: "誓约裁决",
  phase: ADJUDICATION,
  trigger: ACTIVE,
  precondition: alive(p_paladin, t) = ALIVE ∧ target ∈ Alive(t),
  effect: 
    if R(target) = BLOODTHIRSTY:
      sealed(target, t_hunt) = SEALED  // 仅在紧接的猎杀时生效
    else:
      protected(target, t_protected) = PROTECTED,  // 直至下一次圣骑士行动或圣骑士被处决
    identity_revealed(p_paladin, t+1) = true,
    identity_claimed(p_paladin, t+1) = R(p_paladin)  // 强制覆盖为真实身份
)
```

守护时效精确定义：
```
- 施放时刻：t₀（第 N 轮裁判时）
- 保护生效时段：
  - 起点：t₀（立即生效）
  - 终点：**以下两个事件中先发生者**
    1. 圣骑士的下一次行动（即第 N+1 轮裁判时：该窗口无论是否再次施放，旧庇护均在此失效）
    2. 圣骑士被处决（庇护随施放者出局而立即终止）
  - 覆盖阶段：第 N 轮猎杀时 + 第 N+1 轮信息时、辩论时、投票时
```

**关键澄清**：
- **时效窗口图示**：
```
  第N轮：  [信息时] → [辩论时] → [投票时] → [裁判时★施放庇护] → [猎杀时✓保护中]
  第N+1轮：[信息时✓保护中] → [辩论时✓保护中] → [投票时✓保护中] → [裁判时★圣骑士行动/庇护失效]
```
- **"下一次行动"的口径**：指圣骑士在**下一轮裁判时**的判定窗口被开启（无论其选择是否再次施放）。
  因此第 N+1 轮的**投票处决同样被庇护**（与 `RULES.md`、FAQ Q2 一致）
- **提前失效**：若圣骑士在第 N 轮猎杀时至第 N+1 轮裁判时之间被处决，庇护立即终止
- **封印效果**：仅影响被封印嗜血者的投票权，不影响其他嗜血者执行猎杀
- **豁免优先级**：
  - 保护状态免疫所有形式处决（公共处决、秘密处决、圣杀者处决）
  - 异星客的相位躯壳仅免疫秘密处决

---

#### 2.4.3 圣杀者（EXECUTIONER）

```
Skill_EXECUTIONER = (
  name: "铁腕神裁",
  phase: ADJUDICATION,
  trigger: ACTIVE,
  precondition: alive(p_executioner, t) = ALIVE ∧ target ∈ Alive(t),
  effect:
    if protected(target, t) = PROTECTED:
      // 处决失效，目标存活
      NONE
    else:
      execute(target, t, PUBLIC),
    identity_revealed(p_executioner, t+1) = true,
    identity_claimed(p_executioner, t+1) = R(p_executioner)  // 强制覆盖为真实身份
)
```

**关键澄清**：
- **优先级规则**：圣骑士的保护优先级 **高于** 圣杀者的处决
  - 若目标受圣骑士守护，圣杀者处决无效
- **即时处决**：效果立即生效，不进入投票流程
- **公开的是施技者本人**：本技能公开的是**圣杀者自己的身份**（`identity_revealed(p_executioner)`），
  **不是被处决者的身份**——被处决者的身份不因本次处决而公开（见 1.2.6、1.4.1）
- **豁免不构成例外**：即使目标受庇护导致处决失效，圣杀者的身份公开**仍然发生**
- **结算单元**：本技能属"技能相关结算"，完成后若存活人数发生变化则进行胜负检定（见 `3.2.7`）

---

### 2.5 混沌阵营技能

#### 2.5.1 痴呆者（LUNATIC）

```
Skill_LUNATIC = (
  name: "混乱思绪",
  phase: VOTING,
  trigger: FORCED,
  precondition: alive(p_lunatic, t) = ALIVE,
  effect: vote(p_lunatic, randomTarget(Alive(t)))
)
```

随机投票机制：
```
- 第 1 轮投票：使用 PRNG 从 `Alive(t)` 中等概率随机选择一名玩家（包含痴呆者自身）
- 第 2、3 轮投票：随机目标 **必须落在当前候选集合内**（候选集合定义见 `3.2.4`）；
  在候选集合内是否**均匀**随机选择 **尚未裁定**（见 `8.5`）
- 终局二选一表决（`EXECUTE_ALL` / `PARDON_ALL`）中痴呆者如何投票 **尚未裁定**（见 `8.5`）
- 痴呆者 **无法自主控制** 投票目标
- 该机制 **优先于** 玩家主观意愿
```

---

#### 2.5.2 异星客（ALIEN）

```
Skill_ALIEN = (
  name: "相位躯壳",
  phase: HUNT,
  trigger: PASSIVE,
  precondition: TRUE,
  effect: immuneToSecretExecution(p_alien)
)
```

豁免规则：
```
∀t: 若嗜血者在猎杀时选择异星客作为目标 →
  execute(p_alien, t, SECRET) 无效化 →
  alive(p_alien, t+1) = ALIVE →
  宣告"平安夜"（无人死亡）
```

**关键澄清**：
- **仅免疫秘密处决**：不免疫公共处决、圣杀者处决
- **与圣骑士守护叠加**：若异星客同时受守护，优先级为 圣骑士守护 = 相位躯壳（双重保护，冗余）
- **豁免反馈口径未裁定**：嗜血者遭遇免疫时看到的是"猎杀失败"这一事实，还是明确的身份级反馈；
  如何区分"目标受庇护"与"目标为异星客"；两种保护同时成立时如何反馈——**均尚未裁定**（见 `8.5`）。
  本版**不规定**系统向嗜血者无条件告知目标的真实身份；"免疫可能招致试探或针对"属于策略推断，
  不作为已成立的规则写入

---

## 第三部分：游戏流程时序定义

### 3.1 时序状态机

游戏的时序按 **轮（Round）** 组织，每轮包含五个顺序阶段：

```
Round(t) = [INFORMATION(t), DEBATE(t), VOTING(t), ADJUDICATION(t), HUNT(t)]
```

状态转移图：

```
   [初始化] → ┌─ 每轮循环 ──────────────────────────────────────────────────┐
              │ [信息时] → [辩论时] → [投票时] → [裁判时] → [猎杀时]        │
              │      ↑                  │           │          │            │
              │      │                  ▼           ▼          ▼            │
              │      │             处决结算后   处决/复活后  处决结算后       │
              │      │      [结算单元完成 → 存活人数变化? → 胜利检定]         │
              │      │      若某阵营获胜 ⇒ 游戏结束（不再进入后续阶段）        │
              └──────┴────────────────── V(S,t) = ONGOING ──────────────────┘
```

> **判定点说明（本版修正）**：胜利检定挂在**改变存活人数的结算单元**之后（见 `3.2.7`），
> 而不是"每次单人处决之后"：
> - 投票时：单目标处决结算后检定；**终局表决的整批处决在整批完成后只检定一次，中途不检定**
> - 裁判时：圣杀者处决结算后、大法师复活结算后各检定一次
> - 猎杀时：秘密处决结算后检定一次
> - 免除死亡的结算单元（处决豁免、全部赦免、猎杀失败）不改变存活人数，不额外触发检定

### 3.2 各阶段详细定义

#### 3.2.1 初始化阶段（Initialization）

> **顺序约束（本版修正）**：先取得初始化所需的全部输入，再使用这些输入。
> 不得**先使用随机源、后生成 `seed`**；不得**先分配身份、后分配玩家ID与席位**。

```
INITIALIZE(G):
  1. 取得配置并验证：
     n ← C.n；role_pool_flags ← C.role_pool_flags
     k_bloodthirsty ← C.k_bloodthirsty；k_civilian ← C.k_civilian
     校验 VALID(k, n)：含 RC1 下限，k_bloodthirsty = 0 ⟹ ConfigError（见 1.5.3）
  2. 确定玩家ID与席位：∀p ∈ P: Id(p) ← assignPlayerId(p), Seat(p) ← rank_by_Id(p)
  3. 确定随机种子并初始化 PRNG：seed ← generateSeed(); prng ← initPRNG(seed)
     // 具体 PRNG 算法尚未选定，属实现待定（见 7.6、8.2）
  4. 构建身份池：pool ← build_identity_pool(C, prng)     // 见 1.5.1，含超限随机抽取与合规化判定
  5. 分配身份：σ ← assign_identities(pool, P, prng)      // 随机抽取尚未分配的玩家ID，见 1.5.4
     落地：∀p ∈ P: R(p) ← identityOf(p, σ)   // 反查 p 落在哪个身份组中
  6. 初始化游戏状态：∀p ∈ P:
     alive(p, 0) ← ALIVE
     identity_claimed(p, 0) ← None
     identity_revealed(p, 0) ← false
     protected(p, 0) ← NONE
     sealed(p, 0) ← NONE
     last_used_round(p, 0) ← ⊥   // 尚未使用过技能
     T ← 1                       // 轮次计数器
  7. 向各玩家发送获准知道的初始信息：
     ∀p, p' ∈ {x | R(x) = BLOODTHIRSTY}: notify(p, R(p'))    // 嗜血者互相知晓身份
     其余玩家只收到其权限内的初始信息（不泄露他人身份）
  8. 开始第一轮：进入 INFORMATION(1)
```

---

#### 3.2.2 信息时（INFORMATION Phase）

```
INFORMATION(t):
  1. 闭合全场视野（除GM外所有玩家无法观察）
  2. 触发吹哨人技能：
     if ∃p ∈ Alive(t): R(p) = WHISTLEBLOWER:
       target ← PRNG.choice(Alive(t) \ {p})
       info ← speciesCheck(target)
       sendPrivateMessage(p, info)
  3. 触发验尸官技能：
     if ∃p ∈ Alive(t): R(p) = CORONER:
       允许验尸官选择目标 target ∈ (DeadHistory(t) \ InspectedByCoroner(t))
       if target 已选择:
         sendPrivateMessage(p, R(target))
         InspectedByCoroner(t) ← InspectedByCoroner(t) ∪ {target}
```

**时长**：固定时间窗口（如30秒），超时则跳过

---

#### 3.2.3 辩论时（DEBATE Phase）

```
DEBATE(t):
  1. 开放全场视野
  2. 按席位顺序依次发言：
     for p in sortedByPosition(Alive(t)):
       if alive(p, t) = ALIVE:
         allowSpeech(p, maxDuration)
  3. 校验发言范畴合法性（见下文 A–E 类）
  4. 校验发言字数约束（见下文「发言字数约束」）
```

**合法发言范畴与约束**

**A. 信息报告（Information Report）**
- **前置条件**：必须先宣称身份（Identity Claim）
- **执行顺序**：
  1. 玩家发言："我是 [角色名]"
  2. GM 记录宣称的身份：`identity_claimed(p, t) := [角色名]`
  3. 玩家提交信息报告内容
- **约束**：
```
  ∀p ∈ Alive(t), 若 p 提交信息报告，则必须满足：
    identity_claimed(p, t) ≠ None   // 必须先宣称某个身份
```
- **重要说明**：
  - 宣称的身份 **可以与真实身份不符**（如嗜血者谎称自己是吹哨人）
  - 信息报告的内容 **未必是真实的**（如虚假的检测结果）
  - GM **不验证** 宣称身份的真实性，仅记录玩家的发言
  - 其他玩家看到的是：`(玩家p, 宣称的身份, 报告内容)`，需自行推理真伪

**B. 逻辑推理（Logic Reasoning）**
- 无前置条件，自由发言
- 内容：分析已有信息、构建逻辑链、推测角色身份

**C. 质疑与信任声明（Challenge / Trust Declaration）**
- 无前置条件
- 内容：
  - **质疑（Challenge）**：`"我质疑玩家 X 的身份/报告"`
  - **信任（Trust）**：`"我信任玩家 Y 的身份/报告"`

**D. 指控与辩护（Accusation / Defense）**
- 无前置条件
- 内容：
  - **指控（Accusation）**：`"我指控玩家 Z 是 [阵营/角色]"`
  - **辩护（Defense）**：`"我澄清关于我的指控，因为..."`

**E. 身份声明（Identity Claim）**

> 本节是**发言行为**的定义；对应的**状态字段**定义见 1.2.5（`identity_claimed`）。

- 玩家可以 **宣称** 自己是某个角色（无论真假）
- GM 记录宣称但 **不验证真实性**
- **状态更新**：
```
  当玩家 p 宣称自己是角色 r 时：
    identity_claimed(p, t) := r
```
- **区分两个概念**（术语定义见 1.2.5 / 1.2.6）：
  - **身份宣称（Claim）**：玩家主动宣称 `"我是 X 角色"`，设置 `identity_claimed` 字段，可以撒谎
  - **身份公开（Reveal）**：系统强制公开真实身份，设置 `identity_revealed = true` 并覆盖 `identity_claimed := R(p)`，仅在治安角色使用技能时触发
- **禁止行为**：
  - ❌ **物理验证**：不允许"出示身份牌"、"截图身份"等行为
  - ❌ **GM 强制验证**：GM 不会主动证实玩家发言的真伪（除非治安技能触发）

**发言选项逻辑约束表**：

| 发言类型 | 前置条件 | 状态更新 | 内容可造假 | 身份可造假 | GM验证 |
|---------|---------|---------|----------|----------|-------|
| 信息报告 | `identity_claimed ≠ None` | 无 | ✅ 可以 | ✅ 可以（通过宣称） | ❌ 不验证 |
| 逻辑推理 | 无 | 无 | ✅ 可以 | N/A | ❌ 不验证 |
| 质疑/信任 | 无 | 无 | N/A | N/A | ❌ 不验证 |
| 指控/辩护 | 无 | 无 | ✅ 可以 | N/A | ❌ 不验证 |
| 身份声明 | 无 | `identity_claimed := r` | N/A | ✅ 可以 | ❌ 不验证 |

**实现注意事项**：
- 模型在选择"信息报告"时，必须先执行"身份声明"（如果尚未宣称）
- 玩家可以多次宣称不同身份，`identity_claimed` 字段会被最新的宣称覆盖
- 宣称的身份和报告内容都可能是虚假的，推理责任在其他玩家
- 只有治安角色使用技能时，系统才会强制公开真实身份（设置 `identity_revealed = true`）

**时长**：每人发言时长上限（如60秒/人）

**发言字数约束**

每位玩家的单次发言必须满足动态字数限制：

约束公式：
```
token_limit(round) = BASE_TOKENS + random_seed(round)

其中：
  BASE_TOKENS = 150                // 基础字数（计量口径：token，与 7.6 的 tokenizer 统计一致）
  random_seed(round) ∈ [-50, 50]   // 本轮随机偏移量
```

形式化定义：
```
∀p ∈ Alive(t), ∀speech ∈ Speech(p, t):
  tokens(speech.text) ≤ token_limit(t)
```

随机种子生成：
- 每轮辩论时开始时，从均匀分布 `Uniform(-50, 50)` 中抽取整数 `δ`
- 该轮所有玩家共享同一个字数限制
- 示例：
  - 第 1 轮：`δ = +30`，字数限制 = `150 + 30 = 180`（token）
  - 第 2 轮：`δ = -20`，字数限制 = `150 - 20 = 130`（token）

实现伪代码：
```python
def calculate_token_limit(round: int, prng: PRNG) -> int:
    BASE_TOKENS = 150
    random_offset = prng.uniform_int(-50, 50)
    return BASE_TOKENS + random_offset

# 在辩论时开始时执行
def debate_phase_start(game: GameState):
    game.current_token_limit = calculate_token_limit(game.round, game.prng)
    for player in game.alive_players:
        enforce_token_limit(player.speech, game.current_token_limit)
```

违规处理：
- 如果玩家发言超过字数限制，GM 截断超出部分
- 截断规则：保留前 `token_limit(t)` 个 token，后续内容丢弃
- 截断必须按 token 边界对齐，禁止在 token 内部截断

> **计量口径**：本条约束**统一以 token 为唯一计量单位**，由实现层使用的分词器（tokenizer）统计。
> 不再另行定义"字符数"口径，以免两种口径并存导致不同模型间评测不可比。
> 若以汉字近似换算，150 token 约相当于 100–120 个汉字，仅供理解参考，不作为实现依据。

> **评测一致性要求**：`random_offset` 必须由初始化时固定的随机种子 `seed`（见 3.2.1）派生，
> 以保证在 Headless 评测模式下同一 seed 可完全复现，避免因字数限制随机化而影响不同模型间的可比性。

---

**追加辩论（投票时阶段插入的辩论）**

- **触发**：投票时阶段的第 2、3 轮玩家目标投票之前，各插入一次辩论（见 `3.2.4`）
- **参与人**：所有存活玩家按席位顺序发言，**不限于候选人**
- **字数上限**：是否沿用本轮辩论时已抽取的 `token_limit(t)`，还是重新抽取，**尚未裁定**（见 `8.5`）。
  **建议**（未采纳为规则）：沿用本轮已抽取的上限，以保持同一轮内的可比性
- **不影响上限**：追加辩论不改变"玩家目标投票最多三轮"的上限（见 `4.1 INV9`）

---

#### 3.2.4 投票时（VOTING Phase）

```
VOTING(t):
  1. candidate_set ← Alive(t)        // 第 1 轮：投票目标为全体存活玩家
  2. round_no ← 0
  3. while round_no < 3:             // 含第 1 轮，玩家目标投票最多三轮
       round_no ← round_no + 1
       if round_no > 1:
         DEBATE_INSERT(t)            // 追加辩论：全体存活玩家按席位发言，不限候选人
       votes ← collectVotes(Alive(t), candidate_set)   // 所有存活玩家重新投票
                                                       // 痴呆者的选票由 2.5.1 的强制随机机制产生，
                                                       // 其目标须落在 candidate_set 内（见 8.5）
       tally ← countVotes(votes)                       // 每轮独立计票，不累加前轮票数
       maxVotes ← max(tally.values())
       top ← {p ∈ candidate_set | tally[p] = maxVotes} // 本轮并列最高票
       if |top| = 1:
         EXECUTE_SINGLE(top[0], t)   // 单目标处决 → 存活变化则胜检 → 结束投票阶段
         return
       candidate_set ← top           // 候选集合只可缩小或保持不变
  4. FINAL_BINARY_VOTE(candidate_set, t)   // 三轮仍平票 → 终局二选一表决

EXECUTE_SINGLE(target, t):
  execute(target, t, PUBLIC)        // 受处决豁免约束
  AFTER_SETTLEMENT_UNIT(t)          // 存活人数变化 → 检定（见 3.2.7）

FINAL_BINARY_VOTE(candidates, t):
  options ← {EXECUTE_ALL, PARDON_ALL}
  votes2 ← collectVotes(Alive(t), options)   // 痴呆者在此处的投票行为尚未裁定（见 8.5）
  if count(PARDON_ALL) > count(EXECUTE_ALL):
    // 全部赦免：无人死亡 → 不触发额外胜检
    NONE
  else:
    // 处决票更多，或两选项票数相同 → 整批处决（默认全部处决）
    for p in candidates:
      execute(p, t, PUBLIC)         // 仍受处决豁免约束
    AFTER_SETTLEMENT_UNIT(t)        // 整批完成后只检定一次，批次内不检定
  // 不递归，不再发起新的赦免表决
```

**关键澄清**：
- **最多三轮**：包含第 1 轮在内，**玩家目标投票最多三轮**；三轮上限与终局表决共同保证
  **投票阶段有限结束**
- **追加轮的两条硬规则**：① 每次追加投票前插入一次辩论，所有存活玩家（**不限于候选人**）按席位发言；
  ② 追加轮投票目标**仅限上一轮并列最高票的候选集合**，**每轮独立计票、不累加前轮票数**
- **候选集合单调不增**：可缩小或保持不变，**不能扩大**，也**不能重新加入已被排除者**
- **唯一最高票立即收敛**：任一轮出现唯一最高票者，即进入单目标处决结算并结束投票阶段，
  不再进行后续轮次，也不进行终局表决
- **终局表决（第 3 轮仍平票时）**：所有存活玩家在 `EXECUTE_ALL` / `PARDON_ALL` 中二选一；
  **赦免票严格多于处决票才全部赦免**，否则（处决票更多）**以及两票相同**时→**全部处决**
- **平票口径变更（相对 `alpha 0.2`）**：旧版"二次表决平票 → 默认全部赦免"**已废除**，
  现行为 **默认全部处决**
- **豁免仍然有效**："全部处决"**不表示**无视保护强制杀死——整批处决逐人受处决豁免约束
  （庇护、以及秘密处决下的异星客规则同 1.4.2）。**同批内庇护随圣骑士死亡而失效的判定口径尚未裁定**（见 `8.5`）
- **不递归**：终局表决结束后不再发起新的赦免表决，也不重新进入投票轮次
- **终止性口径（不夸大）**：三轮上限与终局表决**保证一次投票阶段有限结束**，
  **不保证整局游戏必然终止**——多轮全部赦免、处决豁免等仍可能造成长局
- **痴呆者**：第 2、3 轮的随机目标必须落在候选集合内；是否均匀随机、以及在终局表决中如何投票，
  **尚未裁定**（见 `2.5.1`、`8.5`）

---

#### 3.2.5 裁判时（ADJUDICATION Phase）

```
ADJUDICATION(t):
  1. 圣杀者判定窗口：
     if ∃p ∈ Alive(t): R(p) = EXECUTIONER:
       允许圣杀者选择是否使用技能
       if 使用:
         target ← playerChoice(p, Alive(t))
         execute(target, t, PUBLIC)  // 受保护状态豁免
         identity_revealed(p, t+1) ← true
         identity_claimed(p, t+1) ← EXECUTIONER  // 强制公开施技者本人
        AFTER_SETTLEMENT_UNIT(t)    // 存活人数变化 → 检定（见 3.2.7）
  
  2. 圣骑士判定窗口：
     if ∃p ∈ Alive(t): R(p) = PALADIN:
       允许圣骑士选择是否使用技能
       if 使用:
         target ← playerChoice(p, Alive(t))
         if R(target) = BLOODTHIRSTY:
           sealed(target, t_hunt) ← SEALED
         else:
           protected(target, t_protected) ← PROTECTED
         identity_revealed(p, t+1) ← true
         identity_claimed(p, t+1) ← PALADIN  // 强制公开施技者本人
        // 本技能不改变存活人数 → 不触发胜检
  
  3. 大法师判定窗口：
     if ∃p ∈ Alive(t): R(p) = ARCHMAGE ∧ canUseCooldown(p, t):
       允许大法师选择是否使用技能
       if 使用:
         target ← playerChoice(p, DeadHistory(t))
         alive(target, t+1) ← ALIVE
         // 恢复所有权利，但本轮不能再使用治安技能
         identity_revealed(p, t+1) ← true
         identity_claimed(p, t+1) ← ARCHMAGE  // 强制公开真实身份
         last_used_round(p) ← t      // 见 2.4.1：记录使用轮次，供轮次差判定
        AFTER_SETTLEMENT_UNIT(t)    // 复活改变存活人数 → 检定（见 3.2.7）
```

**执行顺序**：圣杀者 → 圣骑士 → 大法师（按优先级排序）

**关键澄清（本版）**：
- **删除公开门槛**：三个窗口原先都要求 `identity_revealed(p, t) = false`，本版**全部删除**。
  身份已公开的治安角色**仍可使用技能**，只需满足该技能自身的条件（大法师还需满足轮次差冷却）
- **公开照常发生**：即使处决被豁免、或技能结算后对局立即结束，
  `identity_revealed := true` 与 `identity_claimed := R(p)` **都照常写入**，不因豁免或结束而省略
- **频次**：每个裁判时窗口至多施放一次；是否存在全局次数上限**尚未裁定**（见 `8.5`），
  本版不新增"一局一次"等限制
- **胜检挂载**：圣杀者处决与复活各自是结算单元（完成后按 3.2.7 检定）；
  圣骑士的裁决不改变存活人数，故不触发胜检

---

#### 3.2.6 猎杀时（HUNT Phase）

```
HUNT(t):
  1. 建立嗜血者通讯频道：
     channel ← openChannel({p ∈ Alive(t) | R(p) = BLOODTHIRSTY})
  
  2. 收集投票：
     activeBloodthirsty ← {p ∈ channel | sealed(p, t) = NONE}
     if |activeBloodthirsty| = 0:
       // 所有嗜血者均被封印，平安夜
       return NONE
     legalTargets ← Alive(t) \ Bloodthirsty_Alive(t)   // 合法猎杀目标，见 2.3.1
     votes ← collectVotes(activeBloodthirsty, legalTargets)
  
  3. 决策裁定：
     if allAgree(votes):
       target ← votes[0].target
     else:
       // 按席位编号选择最小编号嗜血者的投票
       target ← votes[minPosition(activeBloodthirsty)].target
  
  4. 豁免检查：
     if protected(target, t) = PROTECTED:
       // 守护免疫，平安夜
       return NONE
     if R(target) = ALIEN:
       // 相位躯壳免疫，平安夜
       return NONE
  
  5. 执行处决：
     execute(target, t, SECRET)
     AFTER_SETTLEMENT_UNIT(t)        // 存活人数变化 → 检定（见 3.2.7）
```

**关键澄清**：
- **嗜血者不能猎杀彼此**：投票目标必须 ∈ `合法猎杀目标(t) = Alive(t) \ Bloodthirsty_Alive(t)`。
  该集合排除**全部存活嗜血者**，而非仅排除投票者本人；**被封印的嗜血者同样被排除**（封印不改变身份）
- **平安夜条件**：所有嗜血者被封印 ∨ 目标受守护 ∨ 目标为异星客
- **豁免反馈口径未裁定**：见 `2.5.2` 与 `8.5`（不得自行写成"系统告知目标身份"）
- **猎杀失败不触发胜检**：无死亡 ⟹ 存活人数未变化 ⟹ 不因此额外检定（见 `3.2.7`）

---

#### 3.2.7 胜利检定（Victory Check）

> **本版重写判定点**：胜利检定挂在**"改变存活人数的结算单元"**之后，
> 而不是"每次单人处决之后"，也不是"轮末统一判定"。

**统一规则**

> 每次**结算单元**完成后，若**存活人数发生变化**，立即进行胜利检定。

**结算单元与检定时机**

| 结算单元 | 检定时机 |
|---|---|
| 单目标投票处决（第 1–3 轮中任一轮出现唯一最高票） | 完成处决结算后 |
| 圣杀者【铁腕神裁】 | 完成技能相关结算后（含施技者的身份公开） |
| 夜间秘密处决（猎杀时） | 完成处决结算后 |
| 大法师【回魂秘术】复活 | 完成复活相关结算后 |
| 终局表决的全部处决（`EXECUTE_ALL`） | **整批处决结算完成后（批次内逐人不检定）** |

**规则条文**

- **批量处决不逐人检定**：整批处决期间不逐人检定，**不因中途短暂满足胜利条件而提前停止**
- **检定使用该结算单元完成后的状态**：不使用预计轮末状态，也不使用批次中途状态
- **若判定游戏结束**：立即 `endGame`，**不再执行后续阶段或其他决策窗口**（见 `4.1 INV10`）
- **未改变存活人数者不触发**：处决豁免（庇护）、全部赦免、猎杀失败等不改变存活人数的情况，
  不因本条额外触发检定
- **不得合并到轮末**：同一轮的多项独立操作各自检定，不得合并到轮末才检定
- **不得笼统表述为"每次处决后"**：单人处决与批量处决的检定粒度不同（见上表）

```
AFTER_SETTLEMENT_UNIT(t):
  if alive_count_after(unit) ≠ alive_count_before(unit):
    if |Alive(t)| = 0:
      // 空场：V(S, t) 无定义 → 标记未裁定并停止推进（见 8.5）
      markUndecided(t); suspendGame(); return UNDECIDED
    result ← V(S, t)
    if result ≠ ONGOING:
      endGame(result)
      return result
  continue current phase's remaining steps
```

```
BATCH_EXECUTE(candidates, t):
  // 逐人执行，仍受处决豁免约束；批次内不调用 V
  // batch_order：批次内逐人处决的顺序。若 8.5 裁定"以批次开始时的保护状态统一判断"，
  //   则结果与顺序无关，该参数可省略；若裁定"逐人动态"（庇护随圣骑士出局立即终止），
  //   则必须同时给出顺序口径（如按席位升序）。见 8.5「批内豁免判定」与 4.3 B6
  for p in batch_order(candidates):
    execute(p, t, PUBLIC)
  AFTER_SETTLEMENT_UNIT(t)        // 整批完成后检定一次
```

```
ROUND_END(t):
  // 轮末不重复判定：本轮的结算单元已在各自完成时判定过
  if game_ended: return
  T ← T + 1
  continue to INFORMATION(T)
```

> **未裁定情形**：整批处决后 `|Alive(t)| = 0`（空场）时 `V(S, t)` **无定义**——
> 不得以 `0 ≥ 0` 判 `EVIL`，也不得自行定义为平局或某阵营获胜；
> 实现应将其标记为**未裁定状态**并**停止推进**（不继续当前轮剩余阶段，也不进入下一轮），
> 由上层按 `8.5` 的裁定处理（伪代码出口见上方 `AFTER_SETTLEMENT_UNIT` 的 `UNDECIDED` 分支）。

---

## 第四部分：约束与不变式（Invariants）

### 4.1 全局不变式

```
INV1: ∀t ∀p ∈ P: R(p) ∈ IdentityType  // 身份类型有效
INV2: ∀t ∀p ∈ P: F(R(p)) ∈ Faction  // 阵营映射有效
INV3: ∀r ∈ UniqueIdentity: |{p ∈ P | R(p) = r}| ∈ {0, 1}  // 唯一身份要么不出现、要么恰好 1 个；可重复身份不受此约束（见 1.2.2）
INV4: ∀t ∀p ∈ P: alive(p, t) = DEAD → ∀t' > t: alive(p, t') = DEAD ∨ (∃复活事件)  // 死亡不可逆（除非复活）
INV5: 每个**改变存活人数的结算单元**完成后立即执行一次 V(S, t)（见 3.2.7）
      改变存活人数的结算单元 = {单目标投票处决, 圣杀者处决, 夜间秘密处决, 大法师复活, 终局表决整批处决}
INV6: 1 ≤ |{p ∈ P | R(p) = BLOODTHIRSTY}| ≤ ⌊n/3⌋（**初始身份配置**约束，见 1.5.3 RC1）；
      该式**不表示**游戏过程中始终至少一名嗜血者存活
INV7: |{p ∈ P | R(p) = CIVILIAN}| = n − |{p ∈ P | R(p) ≠ CIVILIAN}| ≥ 0  // 普通人吃满余量（见 1.5.3 RC5）
INV8: 批量处决（终局表决 `EXECUTE_ALL`）**中途不检定**，仅在该批处决结算完成后检定一次（见 3.2.7）
INV9: 玩家目标投票最多三轮；追加轮（第 2、3 轮）的投票目标 ⊆ 上一轮并列最高票候选集合；候选集合不得扩大（见 3.2.4）
INV10: 对局结束后不再接受任何新的游戏行动，也不再进入后续阶段或决策窗口（见 3.2.7）
INV11: 公共处决不新增死者的身份公开事实：不设置 `identity_revealed`、不改写 `identity_claimed`
       （此前已因合法事件公开者，死亡不撤销；见 1.2.6、1.4.1）
```

> **注意**：`|Alive(t)| = 0`（空场）**不是**已成立的不变式，也不是"不可能状态"——
> 批量处决下它**可达**，且其结果**尚未裁定**（见 `8.5` 与 FAQ `Q5`）。
> `alpha 0.2` 的 `INV5`（"无人生还必然已判定获胜阵营"）**已作废**。

### 4.2 技能约束

```
C1: ∀t ∀p: R(p) ∈ {ARCHMAGE, PALADIN, EXECUTIONER} ∧ usesSkill(p, t) → 
    identity_revealed(p, t+1) = true ∧ identity_claimed(p, t+1) = R(p)
    // 治安角色使用技能后，系统强制公开**其本人**的真实身份
    // 因果方向：使用技能 → 公开；公开状态不是技能可用性判据（见 C7）
    // 该公开不因技能效果被处决豁免、或对局随即结束而省略

C2: ∀t ∀p: R(p) = ARCHMAGE ∧ usesSkill(p, t) → last_used_round(p) := t
    // 只需记录最近使用轮次；可用性由 canUseCooldown(p, t) ↔ last_used_round(p) = ⊥ ∨ (t − last_used_round(p)) ≥ 2 判定
    // 该字段仅在施放时写入，故玩家阵亡期间自然保持不变（等效"死亡冻结"），无需额外规则

C3: ∀t ∀p: sealed(p, t) = SEALED → R(p) = BLOODTHIRSTY ∧ phase(t) = HUNT

C4: ∀t ∀p: protected(p, t) = PROTECTED → immuneToExecution(p, t)

C5: ∀t ∀p: identity_claimed(p, t) ≠ None → identity_claimed(p, t) ∈ RoleType
    // 宣称的身份必须是合法的身份类型

C6: ∀t ∀p: identity_revealed(p, t) = true → identity_claimed(p, t) = R(p)
    // 系统强制公开时，宣称身份必须与真实身份一致（一致性约束）

C7: ∀t ∀p: identity_revealed(p, t) = true ↛ ¬canUseSkill(p, t)
    // 身份已公开 ⇏ 不能施放技能：公开状态只表示"真实身份已被公开"，
    // 不得作为技能可用性判据，也不得兼任技能使用次数标志（见 1.2.6、2.4、3.2.5）
```

### 4.3 边界条件

```
B1: 游戏初始化时 n ∈ [6, 18]
B2: 每轮至少有一名玩家存活才能进入下一轮；
    `|Alive(t)| = 0`（空场）时**立即标记未裁定并停止推进**——不继续当前轮剩余阶段，也不进入下一轮；
    其结果**尚未裁定**，不得由实现自行补判定（见 3.2.7、8.5）
B3: 验尸官已查验集合 InspectedByCoroner 全局唯一，跨轮次累积；
    目标死于公共处决**不影响**其查验资格（公共处决不公开死者身份，见 1.4.1）
B4: 大法师的 `last_used_round` 在整局中持续有效（仅在施放技能时写入，死亡期间不变），跨轮次参与冷却判定
B5: 初始配置必须满足 `k_bloodthirsty ≥ 1`；`k_bloodthirsty = 0` 的配置在初始化阶段被拒绝（见 1.5.3 RC1）
B6: 终局表决的整批处决逐人受处决豁免约束；**同批内庇护因圣骑士死亡而失效的判定口径尚未裁定**（见 8.5）
```

---

## 第五部分：优先级与冲突解决

### 5.1 豁免优先级（从高到低）

```
1. 圣骑士守护（免疫所有处决）
2. 异星客相位躯壳（仅免疫秘密处决）
3. 正常处决生效
```

> 整批处决（终局表决 `EXECUTE_ALL`）**逐人**适用上述优先级；同批内庇护因圣骑士死亡而失效的
> 判定口径尚未裁定（见 `8.5`）。

### 5.2 胜利检定优先级

```
CHAOS > EVIL > LAW > COMMONER
```

- `LAW` 优先于 `COMMONER`：两者条件同时成立（`B = 0 ∧ C = 0 ∧ L > 0 ∧ P > 0`）时，**治安获胜**
- 非常者（`TRAITOR`）存活**不阻止** `LAW` / `COMMONER` 分支命中
- **非直觉后果（依既定优先级，非缺陷）**：`N ≤ 3` 且存在存活混沌时，`CHAOS` 会压过嗜血者数量优势——
  例如 `{BLOODTHIRSTY×2, ALIEN}`（`2 × 2 = 4 ≥ 3` 本已满足 `EVIL`）仍判 **`CHAOS`**；
  该局面在 `k_bloodthirsty` 可取 2 的 6–8 人局中可达
- `N = 0`（空场）时 `V(S, t)` 无定义，其结果**尚未裁定**（见 `8.5`）

### 5.3 同阶段行动顺序

- **裁判时**：圣杀者 → 圣骑士 → 大法师（各窗口至多施放一次技能）
- **猎杀时**：按席位编号升序决策；目标限于 `合法猎杀目标(t)`（排除全部存活嗜血者）
- **投票时**：最多三轮玩家目标投票（第 2、3 轮前各插入一次追加辩论），每轮**同时收集、
  独立计票、统一结算**；三轮仍平票时进入终局二选一表决（平票默认全部处决）

---

## 第六部分：FAQ 形式化解答

### Q1：圣骑士守护异星客后，嗜血者袭击会怎样？

```
前提：protected(p_alien, t) = PROTECTED ∧ 嗜血者选择 p_alien
推导：
  Step 1: 检查保护状态 → protected = PROTECTED → 处决无效
  Step 2: 检查相位躯壳 → R(p_alien) = ALIEN → 处决无效（冗余）
结论：袭击失败，平安夜（双重保护，优先级1 > 优先级2）
```

---

### Q2：圣杀者处决圣骑士守护的目标会怎样？

```
前提：protected(p_target, t) = PROTECTED ∧ 圣杀者使用铁腕神裁(p_target)
推导：
  圣杀者技能前置条件检查：
    if protected(p_target, t) = PROTECTED:
      处决无效
结论：目标存活（保护优先级高于圣杀者）
```

---

### Q3：验尸官能否查验上轮刚被击杀的角色？

```
前提：p_victim 在轮次 t-1 被处决 → alive(p_victim, t-1) = DEAD
推导：
  在轮次 t 的信息时：
    p_victim ∈ DeadHistory(t)  // 符合历史阵亡条件
    若 p_victim ∉ InspectedByCoroner(t) → 可以查验
结论：可以（只要未被查验过）
```

---

### Q4：3 人局嗜血者占比 1/3，混沌存活，谁获胜？

```
场景：Alive(t) = {p₁: BLOODTHIRSTY, p₂: TRAITOR, p₃: ALIEN}
人数口径：N = 3，B = 1，C = 1，L = 0，P = 0
胜利检定（按 CHAOS > EVIL > LAW > COMMONER 顺序）：
  ① CHAOS：N ≤ 3 ∧ C > 0  →  3 ≤ 3 ∧ 1 > 0  →  TRUE
结论：混沌阵营获胜
```

**说明**：
- `EVIL` 本身也不成立：`2 × B ≥ N` 即 `2 ≥ 3` 为假（旧实数写法同为 `1/3 < 0.5`）
- `TRAITOR` 属于邪恶阵营，但**不计入 `B`**，故不改变本判定
- 结论与旧版一致；变化在于胜利条件的表述（`B = 0` 口径、整数运算）与优先级链
- **历史修正记录**：原 Q4 中"1 嗜血者、1 非常者、1 异星客"的嗜血者比例为 1/3，不满足 `≥ 50%` 条件

---

### Q5：会不会出现"处决后全场无人生还"？此时谁获胜？

```
情形：整批处决可能一次移除多名玩家，因此不能沿用"每次至多移除 1 人"的论证

可构造序列（全部为合法行动；初始配置 n = 6，身份 {BLOODTHIRSTY, TRAITOR, CIVILIAN×4}）：
  起始：N = 6，B = 1 → 2 × 1 ≥ 6 为假；C = 0 不命中 CHAOS；B ≠ 0 不命中 LAW / COMMONER → ONGOING
  第 1 轮投票时（候选集合 = 全体 6 人）：6 名存活者循环投票（C1→C2→…→C6→C1）→ 每人恰得 1 票
    → 六方并列最高票
  → 追加辩论后第 2、3 轮投票仍为六方并列（目标限于全体 6 人，各轮独立计票）
  → 终局表决：处决票不低于赦免票（含两票相同）→ 整批处决 6 人 → N = 0
  注：本序列在第 1 轮的投票阶段内即终止，未进入猎杀时。若该轮进入猎杀时，本配置下
      每轮还会额外死亡 1 人（嗜血者未被封印，且无庇护 / 异星客目标），存活数下降更快。
      **批量处决是产生空场的唯一路径**；单目标处决路径不会产生空场。

变体：N = 4 的 {TRAITOR, ALIEN, CIVILIAN, CIVILIAN}，B = 0、C = 1
  → 不命中 CHAOS（N > 3）、EVIL（2 × 0 ≥ 4 为假）、LAW（L = 0）、COMMONER（C ≠ 0）→ ONGOING
  → 四人循环投票、三轮平票、终局表决全部处决 → N = 0
  （该局面可达：唯一嗜血者在更早轮次被投票处决；此后无存活嗜血者，
    猎杀时 `|activeBloodthirsty| = 0` 自动平安夜，故后续只可能由投票处决改变存活数）

结论：
  · **空场（N = 0）是可达状态**，其胜负结果**尚未裁定**（见 8.5）
  · 不得以 `2 × B ≥ N`（即 `0 ≥ 0`）意外判为 EVIL
  · 不得自行定义为平局，也不得把空场一概归咎于引擎漏检
  · 单目标处决路径下，新胜利条件确实使多数残局在处决时即被判出胜负（见 1.3.4）
```

> **实现建议（已修正）**：引擎遇到 `|Alive| = 0` 时应**标记为未裁定状态并上报**，
> 交由上层按后续裁定处理；**不得**用 `assert` 将其一律当作"胜利检定漏接"的实现缺陷。
> `alpha 0.2` 的"无人生还不可达"证明（依赖"每次处决至多移除 1 人"）**已作废**。

---

### Q6：公共处决会不会公开死者的真实身份？

```
前提：p_target 在投票时被公共处决（单目标处决或终局表决整批处决）
推导：
  1. 公共处决公开：被处决者、处决与死亡结果、按既有规则公开的投票记录或施技者信息
  2. 公共处决不公开：死者真实身份牌
  3. 公共处决不设置死者的 identity_revealed，也不覆盖其 identity_claimed
  4. 若死者身份在处决前已因其他合法事件公开，该公开事实不因死亡而撤销
  5. 圣杀者施技公开的是圣杀者本人的身份，不是被处决者的身份
结论：不公开；且验尸官此后仍可按规则查验该尸体（公共处决不取消查验资格）
```

---

### Q7：三轮投票与终局表决如何运作？

```
流程（某一轮投票时阶段连续平票）：
  第 1 轮玩家目标投票（目标 = 全体存活玩家）→ 并列最高票 {A, B}
  → 追加辩论（全体存活玩家按席位发言，不限候选人）
  → 第 2 轮投票（目标限 {A, B}，独立计票）→ 仍并列 {A, B}
  → 追加辩论
  → 第 3 轮投票（目标限 {A, B}，独立计票）→ 仍并列 {A, B}
  → 终局二选一表决（全体存活玩家；选项 EXECUTE_ALL / PARDON_ALL）
      · 赦免票 > 处决票 → 全部赦免（无人死亡，不额外触发胜检）
      · 否则（处决票更多，或两票相同）→ 全部处决（整批结算完成后检定一次）
结论：
  · 玩家目标投票最多三轮；终局表决不计入三轮上限
  · 终局表决平票默认全部处决（旧版"默认全部赦免"已废除）
  · 任一轮出现唯一最高票者，立即单目标处决并结束投票阶段
  · 三轮上限 + 终局表决保证投票阶段有限结束，不保证整局必然终止
```

---

### Q8：大法师复活是否触发胜利检定？

```
前提：大法师在某轮裁判时成功复活 p_target
推导：
  1. 复活是"改变存活人数的结算单元"之一
  2. 完成复活相关结算（含施技者身份公开、last_used_round 写入）后，存活人数增加
  3. 存活人数发生变化 → 立即执行 V(S, t)，使用复活后的存活集合
  4. 若判定某阵营获胜 → 立即 endGame，不再执行后续阶段（当轮猎杀时不再进行）
结论：会触发；旧版"复活不需要检定"的表述已作废（见 3.2.7）
```

---

### Q9：治安角色的身份已经公开后，还能使用技能吗？

```
前提：identity_revealed(p, t) = true，且 p 为治安阵营角色（alive）
推导：
  1. identity_revealed 只表示"真实身份已被系统公开"这一事实
  2. 本版已删除三个裁判时窗口中的 identity_revealed = false 前置条件（见 3.2.5）
  3. 仍需检查的只是该技能自身的条件：
     · 圣杀者 / 圣骑士：alive ∧ target ∈ Alive(t)，且本窗口尚未施放
     · 大法师：alive ∧ target ∈ DeadHistory(t) ∧ canUseCooldown(p, t)
  4. 施放后照常写入 identity_revealed := true、identity_claimed := R(p)（幂等）
结论：可以使用；身份公开不是技能可用性判据，也不得兼任使用次数标志（见 4.2 C7）
```

---

## 第七部分：实现检查清单

在实现该规则时，请确保满足以下检查点：

### 7.1 数据结构
- [ ] 定义玩家状态结构体（包含所有状态字段）
- [ ] 定义角色枚举类型
- [ ] 定义阵营枚举类型
- [ ] 定义种族枚举类型
- [ ] 定义技能触发器枚举类型
- [ ] 定义结算单元类型（单目标处决 / 圣杀者处决 / 秘密处决 / 复活 / 整批处决）
- [ ] 定义投票候选集合与终局表决选项（`EXECUTE_ALL` / `PARDON_ALL`）

### 7.2 状态机
- [ ] 实现游戏阶段状态机（五阶段循环）
- [ ] 实现状态转换函数（满足不可逆性约束）
- [ ] 实现胜利检定函数，并在**每个改变存活人数的结算单元完成后**调用（见 3.2.7）
- [ ] 实现批量处决"**整批完成后只检定一次**"语义（批次内不检定）
- [ ] 实现"对局结束后拒绝新的游戏行动"（见 4.1 INV10）
- [ ] 实现空场（`|Alive| = 0`）的**未裁定状态上报**，不得自动补判定（见 8.5）

### 7.3 技能系统
- [ ] 实现所有 10 个角色的技能逻辑
- [ ] 实现技能前置条件检查（**不得**把 `identity_revealed` 用作前置条件，见 4.2 C7）
- [ ] 实现技能效果应用
- [ ] 实现身份池构建（配额 RC1–RC5、超限均匀随机抽取、合规判定 `VALID(k, n)`）
- [ ] **配置验证拒绝 `k_bloodthirsty = 0`**（RC1 下限，见 1.5.3）
- [ ] 实现身份到玩家ID的随机分配（`assign_identities`，见 1.5.4）
- [ ] 实现初始化顺序：配置验证 → ID/席位 → seed/PRNG → 身份池 → 分配 → 状态 → 初始信息（见 3.2.1）
- [ ] 实现治安阵营身份公开机制：**施技即公开**，豁免或对局结束均不省略（见 1.2.6、2.4）
- [ ] 实现大法师轮次差冷却（`⊥ ∨ (t − last_used_round) ≥ 2`），不新增"一局一次"限制

### 7.4 投票系统
- [ ] 实现三轮玩家目标投票（含第 2、3 轮前的追加辩论），每轮独立计票
- [ ] 实现候选集合单调不增约束（追加轮目标 ⊆ 上一轮并列最高票）
- [ ] 实现终局二选一表决，且**平票默认全部处决**（`EXECUTE_ALL`）
- [ ] 实现整批处决（豁免逐人判定、整批完成后一次胜检）
- [ ] 实现痴呆者随机投票（追加轮目标须落在候选集合内；均匀性与终局表决行为待裁定）
- [ ] 实现公共处决**不写入死者身份公开**（见 1.2.6、4.1 INV11）

### 7.5 豁免系统
- [ ] 实现保护状态检查
- [ ] 实现封印状态检查（封印不改变身份；被封印嗜血者仍在猎杀目标排除集合内）
- [ ] 实现异星客免疫
- [ ] 实现优先级判定（圣骑士守护 > 异星客相位躯壳 > 正常处决生效）
- [ ] 实现"豁免反馈"的待裁定占位（区分庇护与异星客免疫的口径见 8.5）

### 7.6 随机性
- [ ] 实现PRNG（可复现的随机数生成）
- [ ] 实现种子管理
- [ ] 实现吹哨人随机目标选择
- [ ] 实现痴呆者随机投票
- [ ] PRNG 算法选定并在文档中定稿（当前为**实现待定**，见 3.2.1）

### 7.7 测试覆盖
- [ ] 单元测试：每个技能的前置条件和效果
- [ ] 集成测试：完整轮次流程
- [ ] 边界测试：极端人数配置（6 人、18 人）；**`k_bloodthirsty = 0` 必须被拒**
- [ ] 冲突测试：所有 FAQ 场景（`Q1`–`Q9`）
- [ ] 投票测试：三轮上限、候选集合单调不增、终局表决平票默认全部处决
- [ ] 胜检测试：批量处决整批只检定一次；复活触发检定；豁免与全赦不触发
- [ ] 信息测试：公共处决不新增身份公开事实；身份已公开的治安角色仍可施技
- [ ] 空场测试：整批处决致 `|Alive| = 0` 时上报未裁定状态（不判 EVIL、不判平局）
- [ ] 压力测试：1000+ 轮次的长局稳定性（注意：三轮上限**不保证**整局终止）
- [ ] 不变式测试：`INV1–INV11`、`C1–C7`、`B1–B6` 全程不被破坏（含空场与终局结算路径）

---

## 附录 A：术语对照表

### A.1 机制与状态术语

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

### A.2 身份词条对照

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

## 版本摘要（Version Summary）

| 版本 | 阶段 | 要点 |
|------|------|------|
| `v1.0-draft` | 初版 | 基于 `RULES.md` 建立完整形式化体系，修复 11 个已知漏洞 |
| `v1.0-draft` 修订 | 第一轮 | 落地 `Docx/archive/REVISION_PLAN.md` 的 7 处修改；坚持沿用原变量名 |
| `v1.0-draft` 修订 | 第二轮 | 不变式与人数口径一致性修正；字数约束统一为 token |
| `v1.0-draft` 修订 | 第三轮 | 术语定稿（身份 / 玩家ID / 席位）、配额约束 `RC1–RC5`、删除附录 B |
| `v1.0-draft` 修订 | 第四轮 | 引擎最小接口；超限唯一身份改为均匀随机抽取 |
| `v1.0-draft` 修订 | 第五轮 | 双字段身份机制（`identity_claimed` / `identity_revealed`） |
| **`alpha 0.3`** | **当前** | **7 项规则变更 + 初始化顺序与 FAQ Q5 重写；未决事项集中至 `8.5`** |
| `alpha 0.2` | 已归档 | 进一步的结构整理与部分缺陷补足 |
| `alpha 0.1` | 已归档 | 结构整理与术语对齐，形成可送审版本 |

**`alpha 0.3` 相对 `alpha 0.2` 的变化**

**一、7 项已确认规则变更**

1. **胜利条件改为"不存在存活嗜血者"口径**：平凡 / 治安不再要求整个邪恶阵营不存在，
   共同前提为 `B = 0 ∧ C = 0`；非常者存活不阻止二者获胜；`LAW` 优先于 `COMMONER`；
   邪恶条件改写为整数式 `2 × B ≥ N`；新增 `N / B / C / L / P` 五个人数口径（见 `1.3.4`）
2. **胜检挂载到"改变存活人数的结算单元"**：单目标处决、圣杀者处决、秘密处决、复活、
   终局表决整批处决；批量处决**整批完成后只检定一次**，中途不检定；豁免与全赦不触发（见 `3.2.7`）
3. **重写投票与平票**：含首轮**最多三轮**玩家目标投票；追加轮前插入辩论（全员可发言，不限候选人）；
   追加轮目标**仅限上一轮并列最高票**、每轮独立计票；三轮仍平票 → 全体**二选一终局表决**，
   **平票默认全部处决**（旧版"默认全部赦免"废除）（见 `3.2.4`、`3.2.3`）
4. **嗜血者不能猎杀嗜血者**：统一 `合法猎杀目标(t) = Alive(t) \ Bloodthirsty_Alive(t)`，
   排除**全部**存活嗜血者，而非仅排除投票者本人（见 `2.3.1`、`3.2.6`）
5. **技能使用与身份公开的因果**：满足技能自身条件并施放 → 施技者身份立即公开；
   删除裁判时三窗口的 `identity_revealed = false` 前置门槛；`identity_revealed`
   **只表示公开状态**，不得兼任使用次数标志或可用性判据（见 `2.4`、`3.2.5`、`4.2 C7`）
6. **公共处决不公开死者身份**：公开被处决者与死亡结果，不展示身份牌、不设置死者的
   `identity_revealed`、不改写 `identity_claimed`；圣杀者公开的是**自己**的身份（见 `1.4.1`、`1.4.2`）
7. **初始至少一名嗜血者**：`1 ≤ k_bloodthirsty ≤ ⌊n/3⌋` 由**引擎强制校验**，
   `k_bloodthirsty = 0` 必须抛 `ConfigError`（见 `1.5.3`、`1.5.6`、`4.1 INV6`、`4.3 B5`）

**二、同步修正的流程问题**

- **初始化顺序**：改为"配置验证 → ID/席位 → seed/PRNG → 身份池 → 分配身份 → 状态 →
  初始信息 → 第一轮"，消除"先用随机源后生成 seed"与"先分配身份后分配 ID"（见 `3.2.1`）
- **FAQ Q5 重写**：删除"无人生还不可达"的证明与保证；明确批量处决下**空场可达**、
  结果待裁定，不得归咎于引擎漏检（见 `Q5`）
- **不变式与验证标准**：新增结算单元胜检、批内不检定、结束后拒绝新行动、三轮上限、
  候选集合约束、公共处决不新增公开事实、初始下限等条目，并**移除**旧的"空场必已判定"不变式（见 `4.1`）

**三、清除的旧表述**

- `1.3.4` 旧公式（`B / N ≥ 0.5` 实数写法、`COMMONER` 与 `LAW` 互斥、`Alive_EVIL = 0` 前提）
- `1.3.4`、`4.1 INV5`、`Q5` 的"无人生还不可达"论证及其 `assert` 实现建议
- `1.4.1`、`1.4.2` 的"公共处决公开身份牌""目标真实身份全场公开"
- `1.5.3` 的"引擎豁免说明"（引擎允许 `k_bloodthirsty = 0`）与 `8.2` 对应条目
- `3.2.4` 的"每轮最多一次决选""二次平票默认全部赦免"
- `3.2.5` 三个治安技能窗口的 `identity_revealed = false` 前置条件
- `3.2.6` 的 `Alive(t) \ {嗜血者自身}`（仅排除投票者本人）写法
- `3.2.7` 的"每次处决后立即检定"与 `4.1` 旧 `INV8`
- `1.2.7` 的"庇护持续到**下一个裁判时阶段开始前**"（与 `2.4.2` 冲突；本版统一为"到圣骑士的下一次行动窗口"）

**四、未决事项**：空场结果、批内豁免判定、痴呆者追加投票、痴呆者终局表决、
追加辩论预算、异星客免疫反馈、治安技能频次上限、"非常者独存"状态、投票自由度
——集中列于 `8.5`；**本版不宣称规则已完全闭合**

**五、独立复核后的修正（同一 `alpha 0.3` 版本内）**

- `3.2.6` 猎杀伪代码补 `AFTER_SETTLEMENT_UNIT(t)`（原缺口会导致夜间杀人后永不检定）
- `3.2.7` 的 `AFTER_SETTLEMENT_UNIT` 增加 `N = 0` 的 `UNDECIDED` 出口，`4.3 B2` 写明停止推进
- `1.2.7` 的庇护终点改为与 `2.4.2` 一致（消除"阶段开始前 vs 圣骑士窗口"的分歧）
- `2.3.1` 五元组的投票者集合排除被封印者；`batch_order` 的定义与 8.5 裁定项绑定
- 附录标题、`8.1` 判据（补 `B1–B6`）、`7.7`（补不变式测试项）等编号与引用对齐
- `Q5` 的空场构造序列重写为**可执行**版本（原序列漏掉猎杀时，按原文不可执行）；
  `8.5` 第 3 条的可达性论证改写为可审计形式，并把"非常者独存""治安技能频次上限""投票自由度"并入未决表

**`alpha 0.2` 相对 `alpha 0.1` 的变化**

- 版本号升为 `alpha 0.2`；本版说明记为"经过进一步的结构整理与部分缺陷补足"
- **修复 3 处逻辑矛盾**：
  - `1.3.4` 胜利检定 `LAW` 分支与 `COMMONER` 分支重叠 → 补 `∧ |Alive_LAW(t)| > 0`
  - `1.2.1` 「死亡不可逆」的绝对式与同节「复活例外」互相排斥 → 改为条件式
  - `2.4.2` 守护时效的"终点"与时间轴图示冲突 → 改为"直至下一次圣骑士行动或圣骑士被处决"
- **补足 6 处不完备**：删除 `4.3 B2` 冗余表述；删除 `3.2.5` 治安技能窗口的公开门槛（该门槛会使冷却机制永不可达）；
  明确圣杀者可处决自己；修正 `1.5.5` 示例自指措辞；胜利检定改为**每次处决结算后**判定；改写 `4.1 INV5`
- **新增 FAQ Q5**：证明"处决后无人生还"不可达（反证 + 建议断言）
- **修复排版结构**：重建代码围栏，消除 20+ 处"标题被吞入代码块"与 20 处"散文开头错配块"（正文逐行未变）
- 连锁同步：`1.3.4` 判定时机注释、`3.1` 时序图、`3.2.7`、`4.1` 新增 `INV8`、`7.2` 检查清单

**`alpha 0.1` 相对 `v1.0-draft` 的变化**

- 版本号定为 `alpha 0.1`；状态改为"通过初审，等待后续项目推进验证"
- 头部新增「文档信息」表与「结构总览」（含阅读路径建议）
- 新增 `1.1.1 术语约定`、`1.1.2 术语辨析`、`1.1.3 表示法约定`、`1.1.4 相关文档`
- `1.2` 补充状态字段总表，明确 8 个字段的取值域与定义处
- 补齐 `1.5.2 输入参数`，消除 `1.5.1 → 1.5.3` 的编号空洞（`1.5.1`–`1.5.6` 连续）
- 术语对齐：**身份声明**（发言类型）与**身份宣称**（状态字段）彻底区分
- 施工式变更日志移出正文，改存 `Docx/archive/REVISION_LOG.md`
- 「遗留不一致清单」改为「第八部分：验证状态与待办事项」
- 附录 A 拆为 `A.1 机制与状态术语` 与 `A.2 身份词条对照`
- 补充 `2.4.1` 冷却判定：改为"轮次差"式——记录 `last_used_round`，可用条件 `⊥ ∨ t − last_used_round ≥ 2`（见 4.2 `C2`）

---

## 第八部分：验证状态与待办事项

### 8.1 验证路径

本版本为 `alpha 0.3`，**仅完成文本修订与人工推演，尚未经过实现层验证**。建议按下列顺序推进：

| 阶段 | 验证内容 | 通过判据 |
|------|---------|---------|
| 1. 类型系统 | 第一部分的状态元组能否直接映射为 Rust 类型 | `types.rs` 无需引入文档外的字段或枚举 |
| 2. 状态机 | 第三部分五阶段循环与 `3.2.7` 结算单元胜检 | 单局可从初始化跑至某阵营获胜；批量处决整批只检定一次 |
| 3. 初始化 | `1.5` 的配额 `RC1–RC5`、零嗜血者拒绝、随机分配 | 对 `n ∈ [6, 18]` 全枚举，`VALID(k, n)` 恒成立；`k_bloodthirsty = 0` 被拒 |
| 4. 技能判定 | 第二部分 10 个身份的技能前置条件与效果 | 第六部分 FAQ 场景（`Q1`–`Q9`）全部通过 |
| 5. 投票判定 | `3.2.4` 三轮上限、候选集合约束、终局表决默认全部处决 | 连续平票场景按三轮 + 终局表决收敛，不出现第四轮 |
| 6. 信息隔离 | `1.2.6` / `1.4.1` 的身份公开范围与公共处决信息 | 公共处决不新增死者公开事实；身份已公开的治安角色仍可施技 |
| 7. 长局稳定性 | 第四部分不变式与边界条件在整局中不被破坏 | `INV1–INV11`、`C1–C7`、`B1–B6` 全程无断言失败 |
| 8. 随机可复现 | 同一 seed 复现同一局 | 身份分配、吹哨人目标、痴呆者投票、字数上限均可复现 |

> 说明：上表 8 项**均未执行**——仓库内尚无引擎实现，本版无法进行运行时测试。
> 本文档的**文本一致性核对**（代码围栏配对、编号连续性、交叉引用、旧表述清除）已完成，
> 但**规则语义层面的验证仍未执行**；这是 `alpha` 阶段的主要缺口。

**外部审核意见的状态对照见 `Docx/archive/VERIFICATION_LOG.md`**：

| 审核意见 | 状态 |
|---------|------|
| P1 时间戳、P2 嗜血者上限 `⌊n/5⌋` | **已被后续用户决策取代**（本版本为 `⌊n/3⌋`，时间为 `2026/10/15`） |
| P3 普通人占比约束 | **已被取代**（普通人不设上下限） |
| P4 信息报告歧义 | 已按双字段机制修复 |
| P5 大法师复活限制 | 转入 8.4 待办 |
| P6 / P7 / P9 | 已满足 |
| P8 PRNG 种子接口 | 有意不加（见 8.2） |

### 8.2 已知事项（不阻塞初审）

下列事项已识别并裁定，**本版本有意保持现状**：

| # | 事项 | 位置 | 处置与理由 |
|---|------|------|-----------|
| 1 | `seed` 未规定落盘与回放方式 | `3.2.1` | **有意不在引擎实现**：seed 落盘属测试脚本层 / 交互层职责，引擎不为此增加接口 |
| 2 | 与 `RULES.md` 的术语差异（角色卡 / 翻卡 / 暴露） | 跨文档 | **待授权**后才能同步修改 `RULES.md`；本版本不动该文件（该文件现存状态需用户确认） |
| 3 | `RC4` 为恒等构造规则，不进 `VALID` | `1.5.3` | **有意如此**：`min` 式保证其恒成立，列入校验集等于空转 |
| 4 | PRNG 具体算法未选定 | `3.2.1` | **实现待定**：本版只要求"seed 固定 + 可复现"，不擅自指定算法（见 `7.6`） |

> `alpha 0.2` 的已知事项"引擎允许 `k_bloodthirsty = 0`"**已删除**：
> 该配置自本版起必须被引擎拒绝（见 `1.5.3` RC1、`4.3 B5`）。

### 8.3 已解决的评审问题（保留记录备查）

| # | 原问题 | 位置 | 处置结果 |
|---|--------|------|---------|
| 1 | `INV3` 要求全体身份互异，与可重复身份矛盾 | `4.1` | ✅ 改为"仅唯一身份互异" |
| 2 | 原附录 B 的固定配置与初始化约束冲突 | 附录 B | ✅ 附录整体删除，改为对实际配额做 `VALID(k, n)` 判定 |
| 3 | 玩家数范围不一致（`[6,18]` vs `n ≥ 5`） | `1.1` / `1.5.6` | ✅ 统一为 `n ∈ [6, 18]` |
| 4 | 原唯一性公式量词绑定错误 | `1.2.2` | ✅ 由新的形式化表示取代 |
| 5 | `1.4.2` 豁免规则未体现 FAQ Q2 结论 | `1.4.2` | ✅ 补"与圣杀者的交互"，指向 `2.4.3` 与 Q2 |
| 6 | 字数约束"字数"命名与"字符"截断不一致 | `3.2.3` | ✅ 统一为 token 口径，截断按 token 边界对齐 |
| 7 | 随机源术语未定稿（PRNG vs CSPRNG） | `2.2.2` | ✅ 定稿为"PRNG + 初始化固定 seed" |
| 8 | 引擎自动推导人数，与"少接口"原则冲突 | `1.5.2` | ✅ 收敛为最小接口（`role_pool_flags` + 外部传值） |
| 9 | `revealed` 字段语义歧义（"可谎报又公开"） | `1.2.5`/`1.2.6` | ✅ 拆为 `identity_claimed` / `identity_revealed` 双字段 |
| 10 | `1.5.1 → 1.5.3` 编号空洞 | `1.5` | ✅ 补齐 `1.5.2`，现为 `1.5.1`–`1.5.6` 连续 |
| 11 | 引擎放行 `k_bloodthirsty = 0`（原 `8.2 #3`） | `1.5.3` RC1 | ✅ 改为引擎强制校验并抛 `ConfigError` |
| 12 | `3.2.5` 三个技能窗口残留 `identity_revealed = false` 门槛（`alpha 0.2` 声称已删但正文仍在） | `3.2.5` | ✅ 本版全部删除：公开改为"施技的结果"，而非"施技的前提" |
| 13 | 公共处决"展示身份牌 / 公开真实身份" | `1.4.1`、`1.4.2` | ✅ 改为不公开死者身份，仅公开被处决者与死亡结果 |
| 14 | `3.2.6` 猎杀伪代码在 `execute(SECRET)` 后缺少胜检调用（照此实现则夜间杀人后永不检定） | `3.2.6` | ✅ 补 `AFTER_SETTLEMENT_UNIT(t)`（见 `3.2.7`） |
| 15 | `AFTER_SETTLEMENT_UNIT` 对 `N = 0` 无控制流出口 | `3.2.7`、`4.3 B2` | ✅ 增 `UNDECIDED` 分支（`markUndecided` / `suspendGame`），并在 `B2` 写明停止推进 |
| 16 | `1.2.7` 与 `2.4.2` 对庇护终点的表述冲突（影响下一轮圣杀者窗口） | `1.2.7`、`2.4.2` | ✅ 统一以 `2.4.2` 为准：终点＝圣骑士下一次行动窗口；下一轮圣杀者窗口仍在保护期内 |
| 17 | 文档信息称"附录 A"，实际标题无编号 | 附录 | ✅ 标题改为「附录 A：术语对照表」 |
| 18 | `2.3.1` 五元组的投票者集合未排除被封印者 | `2.3.1` | ✅ 改为 `{p ∈ Bloodthirsty_Alive(t) \| sealed(p, t) = NONE}` |

### 8.4 待办（进入实现阶段后处理）

- [ ] 建立自动化校验脚本，替代 `8.1` 表中的人工核算
- [x] `2.4.1` 冷却判定（**已定义**）：记录 `last_used_round`，可用条件 `last_used_round = ⊥ ∨ (t − last_used_round) ≥ 2`；该字段只在施放时写入，故死亡期间自然不变；见 2.4.1 与 4.2 `C2`
- [ ] 由测试脚本层实现 `seed` 持久化与回放
- [ ] 授权后同步 `RULES.md` 术语（该文件现存状态需用户确认）
- [ ] 裁定 `8.5` 的全部未决事项，并将结论回填正文
- [ ] 实现 `3.2.4` 的三轮投票与终局表决、`3.2.7` 的结算单元胜检（`ToDo.md` P2）
- [ ] 实现空场（`N = 0`）的未裁定上报路径（依赖 `8.5` 裁定）
- [ ] 实现 `types.rs` 核心数据类型与最小公开 API（`ToDo.md` P1）

---

### 8.5 未决事项（待裁定，不得自行落实为规则）

下列事项**尚未获得批准**。本版仅登记现状与影响面，**不得**在实现或后续文档中自行补成确定规则。

| 事项 | 尚需明确的内容 | 涉及位置 |
|---|---|---|
| **空场结果** | 整批处决后 `|Alive| = 0` 时，是平局还是其他结果 | `1.3.4`、`3.2.7`、`Q5` |
| **批内豁免判定** | 同批候选人包含圣骑士及其保护目标时，以批次开始时的保护状态统一判断，还是允许圣骑士死亡立即影响同批目标 | `3.2.4`、`5.1`、`4.3 B6` |
| **痴呆者追加投票** | 第 2、3 轮是否在当前候选集合中**均匀**随机选择 | `2.5.1`、`3.2.4` |
| **痴呆者终局表决** | 在"全部处决 / 全部赦免"中如何投票 | `2.5.1`、`3.2.4` |
| **追加辩论预算** | 是否沿用本轮原有 token 上限，还是重新抽取 | `3.2.3`、`3.2.4` |
| **异星客免疫反馈** | 嗜血者看到的是失败事实还是明确身份反馈；如何区分保护与异星客免疫；同时受保护时如何处理 | `2.5.2`、`3.2.6` |
| **治安技能频次上限** | 除"每个裁判时窗口至多施放一次"外，是否存在跨轮或全局次数上限（本版**不新增**"一局一次"等限制） | `2.4`、`3.2.5` |
| **"非常者独存"状态** | `N = 1` 且唯一存活者为 `TRAITOR` 时 `V` 无分支命中（得 `ONGOING`）；本版未找到可达序列，结果**未裁定**，不得自行补判定 | `1.3.4`、本小节第 3 条 |
| **投票自由度** | 非痴呆者是否可**投自己**、是否允许**弃权**（影响候选集合与本小节第 3 条不可达论证的前提） | `2.5.1`、`3.2.4`、本小节第 3 条 |

**补充说明**

1. **空场（`N = 0`）可达**：批量处决下确实可能出现全场无人生还（构造见 `Q5`）。
   本版不判定其结果，也不把该情形当作引擎漏检。
2. **异星客的设计意图不构成规则**："免疫猎杀可能使其被嗜血者识别并针对"属于**策略推断**；
   本版**不**将其写成"系统无条件告知嗜血者目标的真实身份"。反馈口径留待裁定。
3. **"非常者独存"（`N = 1` 且唯一存活者为 `TRAITOR`）未找到可达序列，但状态未被函数覆盖**：
   该状态下 `V(S, t)` 四个分支全部不成立（`C = 0` 不命中 `CHAOS`；`B = 0` 使 `2 × B ≥ N` 为假；
   `B = 0 ∧ C = 0` 下 `L = P = 0`），结果为 `ONGOING`。按本版规则独立推导，它**不可达**：
   - **`N = 2` 的局面不会开启投票阶段**：除本状态自身外，`N ≤ 2` 的局面必然命中某个分支——
     `B ≥ 1 → EVIL`（`2 × 1 ≥ 2`）；`B = 0 ∧ C > 0 → CHAOS`（`N ≤ 3`）；
     `B = 0 ∧ C = 0` 且 `L > 0 → LAW`；其余 `B = 0 ∧ C = 0 ∧ L = 0` 的情形必 `P > 0 → COMMONER`
     （`N = 2` 不可能由两名 `TRAITOR` 构成，`TRAITOR` 为唯一身份，见 `1.2.2` / `INV3`）。
     故不存在"从 `N = 2` 再处决一人"而抵达该状态的路径；
   - **批量处决无法恰好留下唯一幸存者**：设第 3 轮候选集合大小为 `k`（即并列最高票人数）。
     所有 `N` 名存活者各投 1 票，且追加轮选票只能落在候选集合内（见 `3.2.4`），
     故 `N` 票全部落在 `k` 人上；又集合内每人票数都等于最大值 `m`，于是 `k · m ≤ N`。
     若要恰好留下 1 人，需 `k = N − 1`；代入得 `(N − 1) · m ≤ N`，对整数只能 `m = 1`，
     此时前 `N − 1` 人已吃掉 `N − 1` 票，最后 1 票无论投给谁都会使某人票数达到 `1 = m`
     而进入候选集合，于是 `k = N`，与 `k = N − 1` 矛盾。故 `N ≥ 3` 时不可能只留 1 人，
     批量处决的结果只有"全体处决（空场）"或"留下 ≥ 2 人"两种。
   **本版未找到合法可达序列**，因此**不新增**"非常者独存即邪恶获胜"等规则；
   若后续发现可达序列，应连同完整行动序列一并提交裁定。

   > 该推导依赖三条当前文本成立的前提：存活者**人人投票**（弃权未定义）、
   > 追加轮选票**不越出候选集合**、候选集合 = **并列最高票**集合。
   > 若将来允许弃权，本结论需重新论证（相关未定义口径见本小节表格「投票自由度」）。
4. **本版不宣称规则完全闭合**：除上表外，`8.2` 的已知事项与 `8.4` 的待办同样尚未消解。

---

**评审信息**

**审核状态**：`alpha 0.3` 文本修订完成，**未经实现层验证**，存在未决事项（见 `8.5`）  
**文档版本**：`alpha 0.3`  
**版本说明**：胜利条件与胜检时机、投票与平票、猎杀目标、身份公开因果、公共处决信息、初始配额下限  
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
