# 3.2.4 投票时 与 3.2.5 裁判时

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`3.2.4`–`3.2.5`｜**所属部分**：第三部分（游戏流程时序定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

### 3.2.4 投票时（VOTING Phase）

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

### 3.2.5 裁判时（ADJUDICATION Phase）

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
