# 2.4 角色技能：治安阵营（神圣的铁腕）

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`2.4`｜**所属部分**：第二部分（角色技能形式化定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

## 2.4 治安阵营技能

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

### 2.4.1 大法师（ARCHMAGE）

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

### 2.4.2 圣骑士（PALADIN）

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

### 2.4.3 圣杀者（EXECUTIONER）

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
