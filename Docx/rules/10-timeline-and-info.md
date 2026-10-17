# 第三部分 游戏流程：时序状态机、初始化阶段与信息时

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`3.1`、`3.2`（导言、`3.2.1`–`3.2.2`）｜**所属部分**：第三部分（游戏流程时序定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

## 3.1 时序状态机

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

## 3.2 各阶段详细定义

### 3.2.1 初始化阶段（Initialization）

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

### 3.2.2 信息时（INFORMATION Phase）

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
