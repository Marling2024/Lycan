# 1.5 游戏初始化：身份分配、示例与配置检查

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`1.5.4`–`1.5.6`｜**所属部分**：第一部分（形式化基础定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

### 1.5.4 随机分配给玩家ID

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

### 1.5.5 身份池构建示例

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

### 1.5.6 配置验证检查清单

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
