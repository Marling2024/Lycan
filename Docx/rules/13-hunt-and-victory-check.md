# 3.2.6 猎杀时 与 3.2.7 胜利检定时序

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`3.2.6`–`3.2.7`｜**所属部分**：第三部分（游戏流程时序定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

### 3.2.6 猎杀时（HUNT Phase）

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

### 3.2.7 胜利检定（Victory Check）

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
    result ← V(S, t)
    if result ≠ ONGOING:
      endGame(result)  // FACTION_WIN 或 NO_SURVIVORS 均为正式终局
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

> **空场终局**：整批处决后若 `|Alive(t)| = 0`，`V(S, t)` 返回 `NO_SURVIVORS`；
> 游戏正式结束且无获胜阵营，不继续当前轮剩余阶段或进入下一轮。

---
