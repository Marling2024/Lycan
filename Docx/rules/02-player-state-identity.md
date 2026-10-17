# 1.2 玩家状态（Player State）：身份、阵营与种族

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`1.2`（导言、`1.2.1`–`1.2.4`）｜**所属部分**：第一部分（形式化基础定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

## 1.2 玩家状态（Player State）

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

### 1.2.1 存活状态（Alive Status）

```
alive(p, t) ∈ {ALIVE, DEAD}
```

- **初始状态**：`∀p ∈ P: alive(p, 0) = ALIVE`
- **不可逆性（带复活例外）**：`alive(p, t) = DEAD → ∀t' > t: alive(p, t') = DEAD ∨ (∃ 复活事件发生于 (t, t'])`
- **权利约束**：
  - `alive(p, t) = DEAD → ¬canVote(p, t) ∧ ¬canSpeak(p, t) ∧ ¬canUseSkill(p, t)`
  - **例外**：复活技能可使 `alive(p, t') = ALIVE`（其中 `t' > t`）

### 1.2.2 身份类型（Identity Type）

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

### 1.2.3 阵营（Faction）

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

### 1.2.4 种族标签（Species）

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
