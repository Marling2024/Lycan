# 2.5 角色技能：混沌阵营（疯狂与未知）

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`2.5`｜**所属部分**：第二部分（角色技能形式化定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

## 2.5 混沌阵营技能

### 2.5.1 痴呆者（LUNATIC）

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

### 2.5.2 异星客（ALIEN）

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
