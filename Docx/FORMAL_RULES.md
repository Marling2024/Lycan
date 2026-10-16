# 狼人杀变体游戏形式化规则说明书（Formal Specification）

> **版本**：alpha 0.2  
> **状态**：通过初审，等待后续项目推进验证  
> **最后更新**：2026/10/15  
> **本版说明**：经过进一步的结构整理与部分缺陷补足  
> **依据**：基于 `Docx/RULES.md` 改进，修复所有已知漏洞

---

## 文档信息（Document Info）

| 项目 | 内容 |
|------|------|
| **文档版本** | `alpha 0.2` |
| **版本说明** | 经过进一步的结构整理与部分缺陷补足 |
| **文档状态** | 通过初审，等待后续项目推进验证 |
| **最后更新** | 2026/10/15 |
| **依据文档** | `Docx/RULES.md`（严谨版规则说明书） |
| **定位** | 程序实现的唯一权威规则来源（Single Source of Truth） |
| **覆盖范围** | 4 阵营 · 10 种身份 · 5 阶段轮次循环 · 全部技能与判定规则 |
| **修订历史** | 见 `Docx/archive/REVISION_LOG.md`（过程记录，非规则来源） |
| **审核记录** | 见 `Docx/archive/VERIFICATION_LOG.md`（审核意见状态对照） |
| **未决事项** | 见「第八部分：验证状态与待办事项」 |

**结构总览**

| 部分 | 内容 | 覆盖小节 |
|------|------|---------|
| 第一部分 | 形式化基础定义 | `1.1`–`1.6`（游戏实例与术语约定、玩家状态、胜利条件、处决机制、初始化配置） |
| 第二部分 | 角色技能形式化定义 | `2.1`–`2.5`（平凡 / 邪恶 / 治安 / 混沌四阵营共 10 种身份） |
| 第三部分 | 游戏流程时序定义 | `3.1`–`3.2`（五阶段轮次循环、胜利检定时序） |
| 第四部分 | 约束与不变式 | `4.1`–`4.3`（全局不变式、技能约束、边界条件） |
| 第五部分 | 优先级与冲突解决 | `5.1`–`5.3`（豁免优先级、胜利检定优先级、同阶段行动顺序） |
| 第六部分 | FAQ 形式化解答 | `Q1`–`Q4`（含与原规则的裁定差异说明） |
| 第七部分 | 实现检查清单 | `7.1`–`7.7`（数据结构、状态机、技能、投票、豁免、随机性、测试） |
| 第八部分 | 验证状态与待办事项 | `8.1`–`8.4`（验证路径、已知事项、已解决问题、待办） |
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
- **触发条件**：
  - **治安阵营技能触发**：`usesSkill(p, t) ∧ F(R(p)) = LAW → identity_revealed(p, t+1) = true`
  - 技能触发时，系统强制设置：`identity_claimed(p, t+1) := R(p)`（覆盖之前的宣称）
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
- **持续时长**：从施放时刻 `t₀` 到下一个裁判时阶段开始前（详见第三部分时序定义）

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

在**每次处决结算后**执行胜利检定函数 `V: State → Faction ∪ {ONGOING}`：

```
V(S, t) = 
  if |Alive(t)| ≤ 3 ∧ |Alive_CHAOS(t)| ≥ 1:
    return CHAOS
  else if |Bloodthirsty_Alive(t)| ≥ |Alive(t)| / 2:
    return EVIL
  else if |Alive(t)| = |Alive_COMMONER(t)|:              // 仅余平凡阵营存活
    return COMMONER
  else if |Alive_EVIL(t)| = 0 ∧ |Alive_CHAOS(t)| = 0 ∧ |Alive_LAW(t)| > 0:
    return LAW
  else:
    return ONGOING
```

**优先级顺序**：CHAOS > EVIL > LAW = COMMONER（相等优先级按上列判定次序确定结果）

**关键澄清**：
- **判定时机**：每次处决结算后（投票时处决、决选处决、裁判时圣杀者处决、猎杀时秘密处决）立即执行
- **存活玩家总数计算时机**：在所有当轮结算完成后（包括复活、处决）
- **复活玩家立即计入**：大法师复活的玩家在复活后立即进入 `Alive(t)` 集合
- **比例计算**：`|Bloodthirsty_Alive(t)| / |Alive(t)|` 使用实数除法，`≥ 0.5` 即满足条件
- **`COMMONER` 与 `LAW` 分支互斥**：`|Alive(t)| = |Alive_COMMONER(t)|` 蕴含 `|Alive_LAW(t)| = 0`，
  故先判 `COMMONER`、后判 `LAW`，两者不会同时命中；`LAW` 分支额外要求 `|Alive_LAW(t)| > 0`
- **"无人生还"不可达**：由于判定挂在每次处决后，任何局都会在存活数降至 1 之前被判出胜负，
  不可能出现"处决后全场无人生还"的局面（论证见 FAQ Q5）

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
     - **全体投票**：投票时阶段，所有存活玩家的多数票裁决
     - **圣杀者（EXECUTIONER）**：裁判时阶段，圣杀者施放【铁腕神裁】
   - **目标可见性**：全场可见被处决者的**身份牌**（公开其身份）
   - **形式化定义**：
```
     PublicExecution := {
       executor: {ALL_VOTERS} ∪ {EXECUTIONER},
       executor_visible: true,
       target_visible: true,
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
- 若 `type = PUBLIC`：全场知晓被处决者
- 若 `type = SECRET`：仅 GM 知晓，白天公布结果

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
}
```

- **引擎不提供**"按人数自动推导各身份数量"的接口；`k_civilian` 与 `k_bloodthirsty` 由外部传值
- **引擎不提供**优先级、权重、裁剪顺序等参数；超限时只有一种默认行为：**均匀随机抽取**
- 外部若只想说"这个身份进池子"，就只给 `role_pool_flags`；数量策略完全留在引擎之外

#### 1.5.3 配额约束（RC1–RC5；其中 RC4 为构造规则）

**RC1. 嗜血者约束**：
```
1 ≤ k(BLOODTHIRSTY) ≤ ⌊n/3⌋
```
即每局 **至少 1 个**嗜血者，**不得超过总人数的 1/3**（向下取整）。

> **引擎豁免说明**：引擎按最小接口实现，因此当外部传入 `k_bloodthirsty = 0` 时**不会报错**
> （`0 > ⌊n/3⌋` 之外无其他拦截）。"每局至少 1 个嗜血者"是**规则书层面的成局约束**，
> 由调用方（测试脚本层 / 交互层）负责满足；引擎只负责在 `k_bloodthirsty > ⌊n/3⌋` 时判为非法配置。

示例：
- 6 人局：1–2 个
- 7 人局：1–2 个
- 9 人局：1–3 个
- 15 人局：1–5 个

**RC2. 唯一身份约束**（参见 1.2.2）：
```
∀r ∈ UniqueIdentity = {EXECUTIONER, PALADIN, ARCHMAGE, LUNATIC, ALIEN, TRAITOR, WHISTLEBLOWER, CORONER}:
```
  |{p : R(p) = r}| ∈ {0, 1}
```
即唯一身份 **要么不进池子、要么恰好 1 个**。

```
**RC3. 身份启用约束**：
```
∀p ∈ P: role_pool_flags[R(p)] = 1
```
即只有被标记为"进入身份池"的身份才能出现在最终身份池中。

**RC4. 容量约束（构造规则：超限即随机抽取）**：
```
```
|{p : R(p) ∈ UniqueIdentity}| = min( |{r ∈ UniqueIdentity : role_pool_flags[r] = 1}|,
                                     n − k(BLOODTHIRSTY) − k(CIVILIAN) )
```
即 **进入池子的唯一身份数量，取"启用数量"与"余量"的较小值**。余量为
"总人数 − 外部设定的嗜血者数量 − 外部设定的普通人数量"。

```
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

若 ¬VALID:
  1. 超限（启用的唯一身份多于余量）⟹ 均匀随机抽取至余量（正常路径，不是错误）
  2. 配额本身不成立（如 k_bloodthirsty + k_civilian > n）⟹ 抛出 ConfigError
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
- [ ] 至少且不超过 1/3 的嗜血者（RC1）：`1 ≤ k(BLOODTHIRSTY) ≤ ⌊n/3⌋`
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
  effect: teamVote(Bloodthirsty_Alive(t), target) → execute(target, t, SECRET)
)

团队投票机制 teamVote:
1. 所有存活且未被封印的嗜血者进行投票
2. 若意见一致，执行处决
3. 若存在分歧：
```
   - 按席位升序排列嗜血者：{b₁, b₂, ..., bₘ}（席位 `Seat(p)` 见 1.1）
   - 取 min{i | bᵢ ∈ Bloodthirsty_Alive(t) ∧ sealed(bᵢ, t) = NONE} 的投票为最终决策
4. 若所有嗜血者均被封印，本轮猎杀自动失败（平安夜）
```

```
**关键澄清**：
- **席位规则**：`Seat(p) = rank_by_Id(p) ∈ {1, 2, ..., n}`，即按 `Id(p)` 升序确定排序键，与"身份"无关（见 1.1）
- **"席位靠前"定义**：`seat_id` 较小的玩家
- **封印影响**：被封印的嗜血者无投票权，但不影响其他嗜血者行动
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

> **全局约束**：任何治安阵营成员使用主动技能后，立即触发身份公开

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

冷却时间机制（只需记录"最近使用轮次"）：

```
last_used_round(p, 0) = ⊥                          // ⊥ 表示本局尚未使用过
使用后：last_used_round(p_archmage) := t           // 记录当轮轮次
可用条件：canUseCooldown(p, t) ↔
          last_used_round(p) = ⊥ ∨ (t − last_used_round(p)) ≥ 2
```

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

随机投票机制：
```
- 使用 PRNG 从 Alive(t) 中等概率随机选择一名玩家（包含痴呆者自身）
- 痴呆者 **无法自主控制** 投票目标
- 该机制 **优先于** 玩家主观意愿
```

---

```
#### 2.5.2 异星客（ALIEN）

```
Skill_ALIEN = (
  name: "相位躯壳",
  phase: HUNT,
  trigger: PASSIVE,
  precondition: TRUE,
  effect: immuneToSecretExecution(p_alien)
)

豁免规则：
∀t: 若嗜血者在猎杀时选择异星客作为目标 →
  execute(p_alien, t, SECRET) 无效化 →
  alive(p_alien, t+1) = ALIVE →
  宣告"平安夜"（无人死亡）
```

**关键澄清**：
- **仅免疫秘密处决**：不免疫公共处决、圣杀者处决
- **与圣骑士守护叠加**：若异星客同时受守护，优先级为 圣骑士守护 = 相位躯壳（双重保护，冗余）

---

## 第三部分：游戏流程时序定义

### 3.1 时序状态机

游戏的时序按 **轮（Round）** 组织，每轮包含五个顺序阶段：

```
Round(t) = [INFORMATION(t), DEBATE(t), VOTING(t), ADJUDICATION(t), HUNT(t)]
```

状态转移图：

```
   [初始化] → ┌─ 每轮循环 ────────────────────────────────────────────────┐
              │ [信息时] → [辩论时] → [投票时] → [裁判时] → [猎杀时]      │
              │      ↑                  │           │          │          │
              │      │            处决后▼判定   处决后▼判定  处决后▼判定   │
              │      │        [胜利检定] → 若某阵营获胜 ⇒ 游戏结束          │
              └──────┴──────────────── V(S,t) = ONGOING ──────────────────┘
```

### 3.2 各阶段详细定义

#### 3.2.1 初始化阶段（Initialization）

```
INITIALIZE(G):
  1. 构建身份池：pool ← build_identity_pool(C, prng)     // 见 1.5.1，含超限随机抽取与合规化判定
  2. 分配身份：σ ← assign_identities(pool, P, prng)      // 随机抽取尚未分配的玩家ID，见 1.5.4
     落地：∀p ∈ P: R(p) ← identityOf(p, σ)   // 反查 p 落在哪个身份组中
  3. 初始化状态：∀p ∈ P: 
     alive(p, 0) ← ALIVE
     identity_claimed(p, 0) ← None
     identity_revealed(p, 0) ← false
     last_used_round(p, 0) ← ⊥   // 尚未使用过技能
  4. 分配玩家ID与席位：∀p ∈ P: Id(p) ← assignPlayerId(p), Seat(p) ← rank_by_Id(p)
  5. 同步嗜血者信息：∀p, p' ∈ {x | R(x) = BLOODTHIRSTY}: notify(p, R(p'))
  6. 设置轮次计数器：T ← 1
  7. 生成随机种子：seed ← generateSeed()
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
```
- 如果玩家发言超过字数限制，GM 截断超出部分
- 截断规则：保留前 `token_limit(t)` 个 token，后续内容丢弃
- 截断必须按 token 边界对齐，禁止在 token 内部截断

> **计量口径**：本条约束**统一以 token 为唯一计量单位**，由实现层使用的分词器（tokenizer）统计。
> 不再另行定义"字符数"口径，以免两种口径并存导致不同模型间评测不可比。
> 若以汉字近似换算，150 token 约相当于 100–120 个汉字，仅供理解参考，不作为实现依据。

> **评测一致性要求**：`random_offset` 必须由初始化时固定的随机种子 `seed`（见 3.2.1）派生，
> 以保证在 Headless 评测模式下同一 seed 可完全复现，避免因字数限制随机化而影响不同模型间的可比性。

---

#### 3.2.4 投票时（VOTING Phase）

```
VOTING(t):
  1. 初始化投票集合：votes ← {}
  2. 收集投票：
     for p in Alive(t):
       if R(p) = LUNATIC:
         target ← PRNG.choice(Alive(t))
       else:
         target ← playerChoice(p, Alive(t))
       votes[p] ← target
  3. 统计票数：
     tally ← countVotes(votes)
     maxVotes ← max(tally.values())
     candidates ← {p | tally[p] = maxVotes}
  4. 平票处理：
     if |candidates| = 1:
       execute(candidates[0], t, PUBLIC)
     else:  // 平票决选
       runTieBreaker(candidates, t)

平票决选机制 runTieBreaker(candidates, t):
  1. 发起二次表决：
     options ← {EXECUTE_ALL, PARDON_ALL}
     votes2 ← collectVotes(Alive(t), options)
  2. 统计结果：
     if count(EXECUTE_ALL) > count(PARDON_ALL):
       for p in candidates:
         execute(p, t, PUBLIC)
     else if count(PARDON_ALL) > count(EXECUTE_ALL):
       // 全部赦免，无人出局
       NONE
     else:  // 二次表决再次平票
       // 默认全部赦免
       NONE
```

**关键澄清**：
- **平票决选限制**：每轮最多一次决选，决选结果必须执行（避免无限循环）
- **二次平票默认行为**：自动赦免所有候选人

---

#### 3.2.5 裁判时（ADJUDICATION Phase）

```
ADJUDICATION(t):
  1. 圣杀者判定窗口：
     if ∃p ∈ Alive(t): R(p) = EXECUTIONER ∧ identity_revealed(p, t) = false:
       允许圣杀者选择是否使用技能
       if 使用:
         target ← playerChoice(p, Alive(t))
         execute(target, t, PUBLIC)  // 受保护状态豁免
         identity_revealed(p, t+1) ← true
         identity_claimed(p, t+1) ← EXECUTIONER  // 强制公开真实身份
  
  2. 圣骑士判定窗口：
     if ∃p ∈ Alive(t): R(p) = PALADIN ∧ identity_revealed(p, t) = false:
       允许圣骑士选择是否使用技能
       if 使用:
         target ← playerChoice(p, Alive(t))
         if R(target) = BLOODTHIRSTY:
           sealed(target, t_hunt) ← SEALED
         else:
           protected(target, t_protected) ← PROTECTED
         identity_revealed(p, t+1) ← true
         identity_claimed(p, t+1) ← PALADIN  // 强制公开真实身份
  
  3. 大法师判定窗口：
     if ∃p ∈ Alive(t): R(p) = ARCHMAGE ∧ identity_revealed(p, t) = false ∧ canUseCooldown(p, t):
       允许大法师选择是否使用技能
       if 使用:
         target ← playerChoice(p, DeadHistory(t))
         alive(target, t+1) ← ALIVE
         // 恢复所有权利，但本轮不能再使用治安技能
         identity_revealed(p, t+1) ← true
         identity_claimed(p, t+1) ← ARCHMAGE  // 强制公开真实身份
         last_used_round(p) ← t      // 见 2.4.1：记录使用轮次，供轮次差判定
```

**执行顺序**：圣杀者 → 圣骑士 → 大法师（按优先级排序）

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
     votes ← collectVotes(activeBloodthirsty, Alive(t) \ {嗜血者自身})
  
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
```

**关键澄清**：
- **嗜血者不能猎杀彼此**：投票目标必须 ∈ `Alive(t) \ Bloodthirsty_Alive(t)`
- **平安夜条件**：所有嗜血者被封印 ∨ 目标受守护 ∨ 目标为异星客

---

#### 3.2.7 胜利检定（Victory Check）

> **判定点变更**：胜利检定在**每次处决之后立即执行**（而不只在轮末执行）。
> 投票时处决、决选处决、裁判时圣杀者处决、猎杀时秘密处决，各自结算后均需调用一次 `VICTORY_CHECK`。

```
AFTER_EACH_EXECUTION(t):
  result ← V(S, t)
  if result ≠ ONGOING:
    endGame(result)
    return result
  else:
    continue current phase's remaining steps
```

```
ROUND_END(t):
  // 轮末不需要重复判定；处决点已判定完毕。
  // 若本轮无任何处决（如平安夜或全员赦免），局面未变，继续下一轮。
  T ← T + 1
  continue to INFORMATION(T)
```

---

## 第四部分：约束与不变式（Invariants）

### 4.1 全局不变式

```
INV1: ∀t ∀p ∈ P: R(p) ∈ IdentityType  // 身份类型有效
INV2: ∀t ∀p ∈ P: F(R(p)) ∈ Faction  // 阵营映射有效
INV3: ∀r ∈ UniqueIdentity: |{p ∈ P | R(p) = r}| ∈ {0, 1}  // 唯一身份要么不出现、要么恰好 1 个；可重复身份不受此约束（见 1.2.2）
INV4: ∀t ∀p ∈ P: alive(p, t) = DEAD → ∀t' > t: alive(p, t') = DEAD ∨ (∃复活事件)  // 死亡不可逆（除非复活）
INV5: ∀t: |Alive(t)| = 0 → V(S, t) ≠ ONGOING  // 无人生还时胜利检定必须已判定出获胜阵营（由 1.3.4 的判定时机保证）
INV8: ∀ 处决事件 e: 紧随 e 结算后执行一次 V(S, t)  // 胜利检定挂在每次处决之后（见 3.2.7）
INV6: 1 ≤ |{p ∈ P | R(p) = BLOODTHIRSTY}| ≤ ⌊n/3⌋  // 嗜血者数量恒在合法区间（见 1.5.3 RC1）
INV7: |{p ∈ P | R(p) = CIVILIAN}| = n − |{p ∈ P | R(p) ≠ CIVILIAN}| ≥ 0  // 普通人吃满余量（见 1.5.3 RC5）
```

### 4.2 技能约束

```
C1: ∀t ∀p: R(p) ∈ {ARCHMAGE, PALADIN, EXECUTIONER} ∧ usesSkill(p, t) → 
    identity_revealed(p, t+1) = true ∧ identity_claimed(p, t+1) = R(p)
    // 治安角色使用技能后，系统强制公开真实身份

C2: ∀t ∀p: R(p) = ARCHMAGE ∧ usesSkill(p, t) → last_used_round(p) := t
    // 只需记录最近使用轮次；可用性由 canUseCooldown(p, t) ↔ last_used_round(p) = ⊥ ∨ (t − last_used_round(p)) ≥ 2 判定
    // 该字段仅在施放时写入，故玩家阵亡期间自然保持不变（等效"死亡冻结"），无需额外规则

C3: ∀t ∀p: sealed(p, t) = SEALED → R(p) = BLOODTHIRSTY ∧ phase(t) = HUNT

C4: ∀t ∀p: protected(p, t) = PROTECTED → immuneToExecution(p, t)

C5: ∀t ∀p: identity_claimed(p, t) ≠ None → identity_claimed(p, t) ∈ RoleType
    // 宣称的身份必须是合法的身份类型

C6: ∀t ∀p: identity_revealed(p, t) = true → identity_claimed(p, t) = R(p)
    // 系统强制公开时，宣称身份必须与真实身份一致（一致性约束）
```

### 4.3 边界条件

```
B1: 游戏初始化时 n ∈ [6, 18]
B2: 每轮至少有一名玩家存活才能进入下一轮
B3: 验尸官已查验集合 InspectedByCoroner 全局唯一，跨轮次累积
B4: 大法师的 `last_used_round` 在整局中持续有效（仅在施放技能时写入，死亡期间不变），跨轮次参与冷却判定
```

---

## 第五部分：优先级与冲突解决

### 5.1 豁免优先级（从高到低）

```
1. 圣骑士守护（免疫所有处决）
2. 异星客相位躯壳（仅免疫秘密处决）
3. 正常处决生效
```

### 5.2 胜利检定优先级

```
CHAOS > EVIL > LAW = COMMONER
```

### 5.3 同阶段行动顺序

- **裁判时**：圣杀者 → 圣骑士 → 大法师
- **猎杀时**：按席位编号升序决策
- **投票时**：同时收集，统一结算

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

### Q4：3人局嗜血者占比50%，混沌存活，谁获胜？

```
场景：Alive(t) = {p₁:BLOODTHIRSTY, p₂:TRAITOR, p₃:ALIEN}
计算：
```
  |Alive(t)| = 3
  |Bloodthirsty_Alive(t)| = 1
  比例 = 1/3 ≈ 0.33 < 0.5  // 邪恶条件不满足
  |Alive_CHAOS(t)| = 1 ≥ 1  // 混沌条件满足
胜利检定：
  V(S, t) 按优先级检查：
    CHAOS: |Alive(t)| ≤ 3 ∧ |Alive_CHAOS(t)| ≥ 1 → TRUE
结论：混沌阵营获胜
```

```
**修正原规则错误**：原 Q4 中"1嗜血者、1非常者、1异星客"的嗜血者比例为 1/3，不满足 ≥ 50% 条件

---

### Q5：会不会出现"处决后全场无人生还"？此时谁获胜？

```
论证（反证）：
  假设某个处决事件 e 使得结算后 |Alive| = 0
  ⟹ e 之前 场上恰有 1 人存活，记为 p_last（处决每次至多移除 1 人）
  ⟹ 该状态在 e 之前已被胜利检定过（1.3.4：判定时机为每次处决后）
     · 若 p_last ∈ CHAOS：|Alive| = 1 ≤ 3 ∧ |Alive_CHAOS| ≥ 1 → CHAOS 获胜，游戏早已结束
     · 若 p_last ∈ EVIL 且 R(p_last) = BLOODTHIRSTY：1/1 = 1 ≥ 0.5 → EVIL 获胜，游戏早已结束
     · 若 p_last ∈ LAW 且场上已无 EVIL / CHAOS：LAW 获胜，游戏早已结束
     · 若 p_last ∈ COMMONER 且场上已无其他阵营：COMMONER 获胜，游戏早已结束
  矛盾 ⟹ 不存在"处决后无人生还"的可达局面
结论：该情形不可达；任一局在存活数降至 1 之前，胜利检定必已判定出获胜阵营
```

> **实现建议**：引擎可保留一个 `assert |Alive| > 0 or game_ended` 断言；
> 若触发，说明胜利检定的调用点被漏接，属实现缺陷而非规则歧义。

---

## 第七部分：实现检查清单

在实现该规则时，请确保满足以下检查点：

### 7.1 数据结构
- [ ] 定义玩家状态结构体（包含所有状态字段）
- [ ] 定义角色枚举类型
- [ ] 定义阵营枚举类型
- [ ] 定义种族枚举类型
- [ ] 定义技能触发器枚举类型

### 7.2 状态机
- [ ] 实现游戏阶段状态机（五阶段循环）
- [ ] 实现状态转换函数（满足不可逆性约束）
- [ ] 实现胜利检定函数，并在**每次处决后**调用（见 3.2.7）

### 7.3 技能系统
- [ ] 实现所有10个角色的技能逻辑
- [ ] 实现技能前置条件检查
- [ ] 实现技能效果应用
- [ ] 实现身份池构建（配额 RC1–RC5、超限均匀随机抽取、合规判定 `VALID(k, n)`）
- [ ] 实现身份到玩家ID的随机分配（`assign_identities`，见 1.5.4）
- [ ] 实现治安阵营身份公开机制

### 7.4 投票系统
- [ ] 实现标准投票流程
- [ ] 实现平票决选机制
- [ ] 实现痴呆者随机投票

### 7.5 豁免系统
- [ ] 实现保护状态检查
- [ ] 实现封印状态检查
- [ ] 实现异星客免疫
- [ ] 实现优先级判定

### 7.6 随机性
- [ ] 实现PRNG（可复现的随机数生成）
- [ ] 实现种子管理
- [ ] 实现吹哨人随机目标选择
- [ ] 实现痴呆者随机投票

### 7.7 测试覆盖
- [ ] 单元测试：每个技能的前置条件和效果
- [ ] 集成测试：完整轮次流程
- [ ] 边界测试：极端人数配置（6人、18人）
- [ ] 冲突测试：所有FAQ场景
- [ ] 压力测试：1000+轮次的长局稳定性

---

## 附录：术语对照表

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
| 决选 | 平票PK | Tie-Breaker |

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
| **`alpha 0.2`** | **当前** | **进一步的结构整理与部分缺陷补足** |
| `alpha 0.1` | 已归档 | 结构整理与术语对齐，形成可送审版本 |

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

本版本为 `alpha 0.2`，**已通过人工初审，尚未经过实现层验证**。建议按下列顺序推进：

| 阶段 | 验证内容 | 通过判据 |
|------|---------|---------|
| 1. 类型系统 | 第一部分的状态元组能否直接映射为 Rust 类型 | `types.rs` 无需引入文档外的字段或枚举 |
| 2. 状态机 | 第三部分五阶段循环与 `3.2.7` 胜利检定 | 单局可从初始化跑至某阵营获胜，不出现死锁 |
| 3. 初始化 | `1.5` 的配额 `RC1–RC5` 与随机分配 | 对 `n ∈ [6, 18]` 全枚举，`VALID(k, n)` 恒成立 |
| 4. 技能判定 | 第二部分 10 个身份的技能前置条件与效果 | 第六部分 FAQ 场景全部通过 |
| 5. 长局稳定性 | 第四部分不变式在整局中不被破坏 | `INV1–INV7` 与 `C1–C6` 全程无断言失败 |
| 6. 随机可复现 | 同一 seed 复现同一局 | 身份分配、吹哨人目标、痴呆者投票、字数上限均可复现 |

> 说明：起草阶段工作区的 shell 曾被沙箱阻断（`SetNamedSecurityInfoW ... grantWrite` 失败），
> 因此上文 6 项验证**均为人工作业**；`alpha 0.2` 已改用脚本完成结构性核对（围栏配对、编号连续性、
> 交叉引用），但**规则语义层面的验证仍未执行**。这是 `alpha` 阶段的主要缺口。

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
| 2 | 与 `RULES.md` 的术语差异（角色卡 / 翻卡 / 暴露） | 跨文档 | **待授权**后才能同步修改 `RULES.md`；本版本不动该文件 |
| 3 | 引擎允许 `k_bloodthirsty = 0` | `1.5.3` RC1 | **有意保留**："每局至少 1 个嗜血者"是规则书层面的成局约束，由调用方保证 |
| 4 | `RC4` 为恒等构造规则，不进 `VALID` | `1.5.3` | **有意如此**：`min` 式保证其恒成立，列入校验集等于空转 |

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

### 8.4 待办（进入实现阶段后处理）

- [ ] 建立自动化校验脚本，替代 `8.1` 表中的人工核算
- [x] `2.4.1` 冷却判定（**已定义**）：记录 `last_used_round`，可用条件 `last_used_round = ⊥ ∨ (t − last_used_round) ≥ 2`；该字段只在施放时写入，故死亡期间自然不变；见 2.4.1 与 4.2 `C2`
- [ ] 由测试脚本层实现 `seed` 持久化与回放
- [ ] 授权后同步 `RULES.md` 术语
- [ ] 依据本说明书生成小说叙事型 Prompt（`ToDo.md` P1-A）
- [ ] 实现 `types.rs` 核心数据类型（`ToDo.md` P1-B）

---

**评审信息**

**审核状态**：通过初审（`alpha 0.2`），等待后续项目推进验证  
**文档版本**：`alpha 0.2`  
**版本说明**：经过进一步的结构整理与部分缺陷补足  
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
