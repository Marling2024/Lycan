# 1.5 游戏初始化：输入参数与配额约束

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`1.5`（导言、`1.5.1`–`1.5.3`）｜**所属部分**：第一部分（形式化基础定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

## 1.5 游戏初始化配置（Game Initialization）

> 本节定义"如何从 `n` 与配置参数生成一个合法的身份池，并把身份随机落到玩家ID上"。
> 引擎接口边界见 1.5.2。

### 1.5.1 身份池构建算法

游戏开始前，需要根据总玩家人数 `n` 和配置参数生成合法的身份池。算法分两步：
**先确定配额 `k`（见 1.5.2 的输入参数与 1.5.3 的 `RC1–RC5` 约束），再把身份随机分配给玩家ID（见 1.5.4）**。

### 1.5.2 输入参数（引擎最小接口）
> **接口设计原则**：游戏引擎**保持少接口**。人数配置、各身份数量策略等"实现层"逻辑
> 交由**测试脚本层**与**用户交互层**负责；引擎只暴露"**定义某个身份是否进入身份池**"这一个开关，
> 以及在池子超限时的默认兜底行为（均匀随机抽取）。

```
InitConfig := {
  n: ℕ,                                    // 总玩家人数，n ∈ [6, 18]（由外部给定）
  role_pool_flags: IdentityType → {0, 1},  // 唯一接口：身份是否进入身份池（1=进入，0=排除）
  k_civilian: ℕ,                           // 普通人数量（由外部最终确定，引擎不推导）
  k_bloodthirsty: ℕ                        // 嗜血者数量（由外部最终确定，引擎不推导）
                                           // 须满足 1 ≤ k_bloodthirsty ≤ ⌊n/3⌋（RC1，引擎强制校验）
}
```

- **引擎不提供**"按人数自动推导各身份数量"的接口；`k_civilian` 与 `k_bloodthirsty` 由外部传值
- **引擎不提供**优先级、权重、裁剪顺序等参数；超限时只有一种默认行为：**均匀随机抽取**
- 外部若只想说"这个身份进池子"，就只给 `role_pool_flags`；数量策略完全留在引擎之外

### 1.5.3 配额约束（RC1–RC5；其中 RC4 为构造规则）

**RC1. 嗜血者约束（引擎强制校验）**：
```
1 ≤ k(BLOODTHIRSTY) ≤ ⌊n/3⌋
```
即每局 **至少 1 个**嗜血者，**不得超过总人数的 1/3**（向下取整）。

> **强制校验要求**：配置验证必须**拒绝** `k_bloodthirsty = 0`（以及 `k_bloodthirsty > ⌊n/3⌋`），
> 并抛出 `ConfigError`；引擎**不得**放行零嗜血者配置。"每局至少 1 个嗜血者"不再是仅由外层负责的
> 成局约束，而是**引擎侧的初始化约束**（同步见 `1.5.6`、`4.1 INV6`、`4.3 B5`）。
>
> **作用范围**：该约束针对**初始身份配置**，**不表示**游戏过程中始终至少有一名嗜血者存活
> （嗜血者可在其后的处决中被全部消灭）。

示例：
- 6 人局：1–2 个
- 7 人局：1–2 个
- 9 人局：1–3 个
- 15 人局：1–5 个

**RC2. 唯一身份约束**（参见 1.2.2）：
```
∀r ∈ UniqueIdentity = {EXECUTIONER, PALADIN, ARCHMAGE, LUNATIC, ALIEN, TRAITOR, WHISTLEBLOWER, CORONER}:
  |{p : R(p) = r}| ∈ {0, 1}
```
即唯一身份 **要么不进池子、要么恰好 1 个**。

**RC3. 身份启用约束**：
```
∀p ∈ P: role_pool_flags[R(p)] = 1
```
即只有被标记为"进入身份池"的身份才能出现在最终身份池中。

**RC4. 容量约束（构造规则：超限即随机抽取）**：
```
|{p : R(p) ∈ UniqueIdentity}| = min( |{r ∈ UniqueIdentity : role_pool_flags[r] = 1}|,
                                     n − k(BLOODTHIRSTY) − k(CIVILIAN) )
```
即 **进入池子的唯一身份数量，取"启用数量"与"余量"的较小值**。余量为
"总人数 − 外部设定的嗜血者数量 − 外部设定的普通人数量"。

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
# 步骤 0：RC1 强制校验（拒绝零嗜血者，拒绝超过 ⌊n/3⌋）
#   0a. 若 k_bloodthirsty = 0 或 k_bloodthirsty > ⌊n/3⌋ → ConfigError

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

    # 步骤 0：RC1 强制校验（下限 1，上限 ⌊n/3⌋）
    if not (1 <= config.k_bloodthirsty <= n // 3):
        raise ConfigError("嗜血者数量必须满足 1 <= k_bloodthirsty <= floor(n/3)")

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
                                          // RC1 含下限：k_bloodthirsty = 0 判为非法

若 ¬VALID:
  1. 超限（启用的唯一身份多于余量）⟹ 均匀随机抽取至余量（正常路径，不是错误）
  2. `k_bloodthirsty = 0` 或 `k_bloodthirsty > ⌊n/3⌋` ⟹ 抛出 ConfigError（RC1 强制校验）
  3. 配额本身不成立（如 k_bloodthirsty + k_civilian > n）⟹ 抛出 ConfigError
```

> **设计意图**：合规判定对**每次实际生成的配额 `k`** 做校验，而非针对预先写死的推荐配置表
> （原附录 B 已删除）。引擎侧只有"身份是否进池"一个接口，加"超限随机抽取"一个默认行为；
> 人数与配比策略完全留在测试脚本层 / 用户交互层。
