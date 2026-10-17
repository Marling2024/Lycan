# 1.3 胜利条件 与 1.4 处决机制

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`1.3`–`1.4`｜**所属部分**：第一部分（形式化基础定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

## 1.3 胜利条件形式化定义

### 1.3.1 存活玩家集合

定义时刻 `t` 的存活玩家集合：

```
Alive(t) = {p ∈ P | alive(p, t) = ALIVE}
```

### 1.3.2 阵营存活集合

定义阵营 `f` 在时刻 `t` 的存活成员集合：

```
Alive_f(t) = {p ∈ Alive(t) | F(R(p)) = f}
```

### 1.3.3 嗜血者存活集合（特殊定义）

```
Bloodthirsty_Alive(t) = {p ∈ Alive(t) | R(p) = BLOODTHIRSTY}
```

### 1.3.4 胜利检定函数（Victory Check）

记时刻 `t` 的五个人数口径：

```
N(t) = |Alive(t)|                    // 存活玩家总数
B(t) = |Bloodthirsty_Alive(t)|       // 存活嗜血者人数
C(t) = |Alive_CHAOS(t)|              // 存活混沌阵营人数
L(t) = |Alive_LAW(t)|                // 存活治安阵营人数
P(t) = |Alive_COMMONER(t)|           // 存活平凡阵营人数
```

区分阵营与对局结果；`Faction` 仍只有四个阵营，“无人生还”不是第五个阵营：

```
GameResult = ONGOING | FACTION_WIN(Faction) | NO_SURVIVORS
```

- `ONGOING`：游戏继续。
- `FACTION_WIN(f)`：阵营 `f` 获胜，游戏结束。
- `NO_SURVIVORS`：**无人生还**，游戏结束，**无获胜阵营**。这是正式终局，不是异常或未裁定状态。
- 本说明书中的“CHAOS / EVIL / LAW / COMMONER 获胜”分别表示对应的 `FACTION_WIN(...)`。
- 无人生还是否在外部评测中记作平局、如何计分，属于评测规则，本次不作规定。

对所有合法存活人数 `N(t) ≥ 0`，终局检定函数（沿用名称 `V`）`V: State → GameResult` 定义为：

```
V(S, t) =
  if N = 0:                        NO_SURVIVORS
  else if N ≤ 3 ∧ C > 0:           FACTION_WIN(CHAOS)
  else if 2 × B ≥ N:               FACTION_WIN(EVIL)
  else if B = 0 ∧ C = 0 ∧ L > 0:   FACTION_WIN(LAW)
  else if B = 0 ∧ C = 0 ∧ P > 0:   FACTION_WIN(COMMONER)
  else:                            ONGOING
```

**检定顺序**：先检查空场；非空场的阵营胜利优先级仍为 `CHAOS > EVIL > LAW > COMMONER`。不得让空场通过 `0 ≥ 0` 意外命中邪恶胜利。

**关键澄清**：
- **平凡 / 治安不再要求"邪恶阵营整体不存在"**：两者的共同前提是 `B = 0 ∧ C = 0`，
  即**不存在存活嗜血者**且**不存在存活混沌阵营玩家**；
  **非常者（TRAITOR）是否存活不影响**这两个分支——非常者仍属邪恶阵营，但**不计入 `B`**
- **治安优先于平凡**：`B = 0 ∧ C = 0` 且 `L > 0 ∧ P > 0` 时 `LAW` 分支先命中，治安获胜；
  两个分支不再是"互斥关系"，而是**优先级关系**
- **邪恶条件改用整数运算**：`2 × B ≥ N` 与旧的实数写法 `B / N ≥ 0.5` 等价，但避免浮点除法
- **空场终局**：当 `N = 0` 时，`V` 返回 `NO_SURVIVORS`，游戏结束且无获胜阵营；此分支先于阵营公式，
  不得让 `2 × B ≥ N`（即 `0 ≥ 0`）意外触发 `EVIL`。该结局不等同于评测层的平局计分规则
- **不新增未经确认的规则**：如"非常者独存即邪恶获胜"**不在本版规则之内**
- **判定时机**：不再表述为"每次处决结算后"；改为**每个改变存活人数的结算单元完成后**检定
  （结算单元清单、批量处决与不检定情形见 `3.2.7`）
- **复活计入**：大法师复活的玩家在复活后立即进入 `Alive(t)`；复活本身是一个结算单元，同样触发检定

---

## 1.4 处决机制（Execution Mechanism）

### 1.4.1 处决类型

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

### 1.4.2 处决函数

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
