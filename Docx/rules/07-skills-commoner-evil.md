# 第二部分 角色技能：平凡阵营与邪恶阵营

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`2.1`–`2.3`｜**所属部分**：第二部分（角色技能形式化定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

## 2.1 技能表示法

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

## 2.2 平凡阵营技能

### 2.2.1 普通人（CIVILIAN）

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

### 2.2.2 吹哨人（WHISTLEBLOWER）

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

### 2.2.3 验尸官（CORONER）

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

## 2.3 邪恶阵营技能

### 2.3.1 嗜血者（BLOODTHIRSTY）

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

### 2.3.2 非常者（TRAITOR）

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
