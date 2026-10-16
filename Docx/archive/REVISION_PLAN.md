# FORMAL_RULES.md 修改方案

## 修改清单

基于审核反馈，需要对 `FORMAL_RULES.md` 进行以下 7 处修改：

---

## 修改 1：更新文档头部时间戳

**位置**：文档顶部元数据区域

**原文**：
```
最后更新：[当前日期]
```

**修改为**：
```
最后更新：2026/10/15 14:00
```

---

## 修改 2：角色唯一性约束

**位置**：`1.2.2 角色集合 R`

**原文**：
```
每个角色在单局游戏中至多出现一次（唯一性约束）
```

**修改为**：
```markdown
### 角色唯一性约束

角色的唯一性分为两类：

**唯一角色（Unique Roles）**：单局游戏中至多出现一次
- 治安阵营全部角色：JUDGE_KILLER, PALADIN, ARCHMAGE
- 混沌阵营全部角色：LUNATIC, ALIEN
- 邪恶阵营：TRAITOR（非常者）
- 平凡阵营信息位：WHISTLEBLOWER（吹哨人）, CORONER（验尸官）

**可重复角色（Repeatable Roles）**：单局游戏中可出现多次
- COMMONER（普通人）
- WEREWOLF（嗜血者）

形式化表示：
```
∀r ∈ {JUDGE_KILLER, PALADIN, ARCHMAGE, LUNATIC, ALIEN, TRAITOR, WHISTLEBLOWER, CORONER},
  |{p ∈ P : role(p) = r}| ≤ 1

∀r ∈ {COMMONER, WEREWOLF},
  |{p ∈ P : role(p) = r}| ≥ 0  (无上限约束)
```
```

---

## 修改 3：处决类型执行人说明

**位置**：`1.4.1 处决机制`

**原文**：
```markdown
### 1.4.1 处决机制

处决（Execution）是移除玩家的唯一合法途径，分为两种类型：

1. **公共处决（Public Execution）**：
   - 触发时机：投票时的多数票裁决、裁判时圣杀者技能
   - 可见性：全场可见被处决者身份

2. **秘密处决（Secret Execution）**：
   - 触发时机：猎杀时嗜血者的集体暗杀
   - 可见性：仅公布死亡事实，不公开凶手身份
```

**修改为**：
```markdown
### 1.4.1 处决机制

处决（Execution）是移除玩家的唯一合法途径，分为两种类型：

1. **公共处决（Public Execution）**：
   - **执行人可见性**：执行人身份向全场公开
   - **执行人类型**：
     - **全体投票**：投票时阶段，所有存活玩家的多数票裁决
     - **圣杀者（JUDGE_KILLER）**：裁判时阶段，圣杀者施放【铁腕神裁】
   - **目标可见性**：全场可见被处决者身份
   - **形式化定义**：
     ```
     PublicExecution := {
       executor: {ALL_VOTERS} ∪ {JUDGE_KILLER},
       executor_visible: true,
       target_visible: true,
       phase: VOTING | ADJUDICATION
     }
     ```

2. **秘密处决（Secret Execution）**：
   - **执行人可见性**：执行人身份对非嗜血者保密
   - **执行人类型**：
     - **嗜血者全体（All WEREWOLVES）**：猎杀时阶段，所有存活嗜血者集体投票决定
   - **目标可见性**：仅公布死亡事实，不公开凶手身份
   - **形式化定义**：
     ```
     SecretExecution := {
       executor: {p ∈ P_alive : role(p) = WEREWOLF},
       executor_visible: false,  // 对非嗜血者不可见
       target_visible: true,      // 死亡事实公开，但不公开具体角色
       phase: HUNT
     }
     ```

**执行人公开性对比表**：

| 处决类型 | 执行人 | 执行人是否公开 | 触发阶段 |
|---------|--------|--------------|---------|
| 公共处决 | 全体投票 | ✅ 公开（投票记录） | 投票时 |
| 公共处决 | 圣杀者 | ✅ 公开（技能施放时自动公开身份） | 裁判时 |
| 秘密处决 | 嗜血者全体 | ❌ 不公开（对非嗜血者保密） | 猎杀时 |
```

---

## 修改 4：开局身份抽取随机化规则

**位置**：在 `第一部分` 末尾新增 `1.7 游戏初始化配置`

**新增内容**：
```markdown
## 1.7 游戏初始化配置

### 1.7.1 身份池构建算法

游戏开始前，需要根据总玩家人数 `n` 和配置参数生成合法的身份池。

#### 输入参数

```
InitConfig := {
  n: ℕ,                          // 总玩家人数
  role_pool_flags: R → {0, 1},   // 角色是否加入池子（1=加入，0=排除）
  min_non_commoner: ℕ,           // 最低非普通人角色数量
  commoner_ratio_range: [a, b]   // 普通人占比范围，0 < a < b < 1
}
```

#### 约束条件

**C1. 普通人占比约束**：
```
n/4 < |{p : role(p) = COMMONER}| ≤ 3n/5
```
即普通人数量必须 **严格大于总人数的 1/4，且不超过总人数的 3/5**。

**C2. 嗜血者人数约束**：
```
|{p : role(p) = WEREWOLF}| ≤ n/5
```
即嗜血者数量 **不得大于总人数的 1/5**（向下取整）。

**C3. 非普通人最低数量约束**：
```
|{p : role(p) ≠ COMMONER}| ≥ min_non_commoner
```

**C4. 角色启用约束**：
```
∀p ∈ P, role_pool_flags[role(p)] = 1
```
即只有被标记为启用的角色才能出现在身份池中。

**C5. 唯一性约束**（参见 1.2.2）：
```
唯一角色至多出现一次
```

#### 初始化算法伪代码

```python
def generate_role_pool(config: InitConfig) -> List[Role]:
    n = config.n
    available_roles = [r for r in R if config.role_pool_flags[r] == 1]
    
    # 步骤 1：确定普通人数量范围
    min_commoners = ceil(n / 4) + 1  # 严格大于 n/4
    max_commoners = floor(3 * n / 5)
    
    # 步骤 2：确定嗜血者数量上限
    max_werewolves = floor(n / 5)
    
    # 步骤 3：随机选择非普通人角色
    non_commoner_roles = [r for r in available_roles if r != COMMONER]
    unique_roles = [r for r in non_commoner_roles if is_unique(r)]
    repeatable_roles = [r for r in non_commoner_roles if not is_unique(r)]
    
    # 从唯一角色中随机抽取
    selected_unique = random_sample(unique_roles, max_count=len(unique_roles))
    
    # 从可重复角色中随机抽取嗜血者
    werewolf_count = random_int(1, max_werewolves)
    selected_werewolves = [WEREWOLF] * werewolf_count
    
    # 步骤 4：计算普通人数量
    non_commoner_count = len(selected_unique) + werewolf_count
    
    # 检查非普通人最低数量
    if non_commoner_count < config.min_non_commoner:
        raise ConfigError("无法满足最低非普通人数量约束")
    
    commoner_count = n - non_commoner_count
    
    # 检查普通人占比约束
    if not (min_commoners <= commoner_count <= max_commoners):
        raise ConfigError("无法满足普通人占比约束")
    
    # 步骤 5：构建最终身份池
    role_pool = selected_unique + selected_werewolves + [COMMONER] * commoner_count
    
    # 步骤 6：随机打乱并分配
    shuffle(role_pool)
    return role_pool

def is_unique(role: Role) -> bool:
    """判断角色是否具有唯一性"""
    return role in {
        JUDGE_KILLER, PALADIN, ARCHMAGE,    # 治安阵营
        LUNATIC, ALIEN,                     # 混沌阵营
        TRAITOR,                            # 邪恶阵营
        WHISTLEBLOWER, CORONER              # 平凡阵营信息位
    }
```

#### 标准配置示例

**7 人局（基础配置）**：
```
n = 7
min_non_commoner = 3
role_pool_flags = {WEREWOLF: 1, TRAITOR: 1, WHISTLEBLOWER: 1, 其余: 0}
```
可能生成：`[WEREWOLF, TRAITOR, WHISTLEBLOWER, COMMONER×4]`

**10 人局（标准配置）**：
```
n = 10
min_non_commoner = 4
role_pool_flags = {WEREWOLF: 1, TRAITOR: 1, WHISTLEBLOWER: 1, CORONER: 1, PALADIN: 1, 其余: 0}
```
可能生成：`[WEREWOLF×2, TRAITOR, WHISTLEBLOWER, CORONER, PALADIN, COMMONER×4]`

**15 人局（完整配置）**：
```
n = 15
min_non_commoner = 6
role_pool_flags = 全部角色启用
```
可能生成：`[WEREWOLF×3, TRAITOR, WHISTLEBLOWER, CORONER, PALADIN, JUDGE_KILLER, ARCHMAGE, LUNATIC, ALIEN, COMMONER×4]`

#### 配置验证检查清单

在游戏启动前，必须验证：
- [ ] `n ≥ 5`（最低玩家数）
- [ ] 存在至少一个嗜血者
- [ ] 普通人占比在合法范围内
- [ ] 嗜血者数量不超过上限
- [ ] 非普通人数量满足最低要求
- [ ] 所有唯一角色不重复
```

---

## 修改 5：身份公开与信息报告约束

**位置**：`第三部分 3.2 阶段二：辩论时` 的发言选项部分

**原文**：
```markdown
2. 合法发言范畴：
   * 提交信息报告 / 阐述逻辑链条
   * 宣告质疑 / 宣布信任
   * 发起指控 / 举证自辩
   * 自主公开身份（禁止主动翻卡自证，允许谎报身份）
```

**修改为**：
```markdown
2. 合法发言范畴与约束：

   **A. 信息报告（Information Report）**
   - **前置条件**：必须先公开身份（Reveal Identity）
   - **执行顺序**：
     1. 玩家声明："我是 [角色名]"
     2. GM 标记该玩家身份为公开状态：`revealed(p) := true`
     3. 玩家提交信息报告内容
   - **约束**：
     ```
     ∀p ∈ P_alive, 若 p 提交信息报告，则必须满足：
       revealed(p) = true  // 身份已公开
     ```
   - **重要说明**：
     - 公开的身份 **可以是谎报的身份**（如嗜血者谎称自己是吹哨人）
     - 信息报告的内容 **未必是真实的**（如虚假的检测结果）
     - GM **不验证** 公开身份的真实性，仅记录玩家的声明

   **B. 逻辑推理（Logic Reasoning）**
   - 无前置条件，自由发言
   - 内容：分析已有信息、构建逻辑链、推测角色身份

   **C. 质疑与信任声明（Challenge / Trust Declaration）**
   - 无前置条件
   - 内容：
     - **质疑（Challenge）**：`"我质疑玩家 X 的身份/报告"`
     - **信任（Trust）**：`"我信任玩家 Y 的身份/报告"`

   **D. 指控与辩护（Accusation / Defense）**
   - 无前置条件
   - 内容：
     - **指控（Accusation）**：`"我指控玩家 Z 是 [阵营/角色]"`
     - **辩护（Defense）**：`"我澄清关于我的指控，因为..."`

   **E. 身份声明（Identity Claim）**
   - 玩家可以 **声称** 自己是某个角色（无论真假）
   - GM 记录声明但 **不验证真实性**
   - **区分两个概念**：
     - **身份声明（Claim）**：口头宣称 `"我是 X 角色"`，可以撒谎
     - **身份公开（Reveal）**：在系统中标记 `revealed(p) = true`，用于信息报告的前置检查
   - **禁止行为**：
     - ❌ **物理验证**：不允许"出示角色卡"、"截图身份"等行为
     - ❌ **GM 强制验证**：GM 不会主动证实玩家声明的真伪

**发言选项逻辑约束表**：

| 发言类型 | 是否需要公开身份 | 内容可以造假 | 身份可以造假 | GM 是否验证 |
|---------|----------------|------------|------------|-----------|
| 信息报告 | ✅ 必须先公开 | ✅ 可以 | ✅ 可以 | ❌ 不验证 |
| 逻辑推理 | ❌ 不需要 | ✅ 可以 | N/A | ❌ 不验证 |
| 质疑/信任 | ❌ 不需要 | N/A | N/A | ❌ 不验证 |
| 指控/辩护 | ❌ 不需要 | ✅ 可以 | N/A | ❌ 不验证 |
| 身份声明 | ❌ 不需要 | N/A | ✅ 可以 | ❌ 不验证 |

**实现注意事项**：
- 模型在选择"信息报告"时，必须先执行"身份公开"步骤
- 身份公开后，该玩家的 `revealed` 标记永久生效（不可撤销）
- 公开的身份和报告内容都可能是虚假的，推理责任在其他玩家
```

---

## 修改 6：发言字数约束

**位置**：`第三部分 3.2 阶段二：辩论时` 的约束条件部分

**原文**（如果存在关于发言时长的描述，替换；如果不存在，新增）

**修改为**：
```markdown
### 发言字数约束

每位玩家的单次发言必须满足动态字数限制：

**约束公式**：
```
word_limit(round) = BASE_WORDS + random_seed(round)

其中：
  BASE_WORDS = 150           // 基础字数
  random_seed(round) ∈ [-50, 50]  // 本轮随机偏移量
```

**形式化定义**：
```
∀p ∈ P_alive, ∀speech ∈ Speech(p, round),
  |speech.text| ≤ word_limit(round)
```

**随机种子生成**：
- 每轮开始时，从均匀分布 `Uniform(-50, 50)` 中抽取整数 `δ`
- 该轮所有玩家共享同一个字数限制
- 示例：
  - 第 1 轮：`δ = +30`，字数限制 = `150 + 30 = 180`
  - 第 2 轮：`δ = -20`，字数限制 = `150 - 20 = 130`

**实现伪代码**：
```python
def calculate_word_limit(round: int, prng: PRNG) -> int:
    BASE_WORDS = 150
    random_offset = prng.uniform_int(-50, 50)
    return BASE_WORDS + random_offset

# 在辩论时开始时执行
def debate_phase_start(game: GameState):
    current_limit = calculate_word_limit(game.round, game.prng)
    game.current_word_limit = current_limit
    
    for player in game.alive_players:
        enforce_word_limit(player.speech, current_limit)
```

**违规处理**：
- 如果玩家发言超过字数限制，GM 截断超出部分
- 截断规则：保留前 `word_limit(round)` 个字符，后续内容丢弃
```

---

## 修改 7：嗜血者人数约束（已在修改 4 中包含）

**位置**：`1.7.1 身份池构建算法` 的约束条件 C2

**确认内容**：
```markdown
**C2. 嗜血者人数约束**：
```
|{p : role(p) = WEREWOLF}| ≤ floor(n / 5)
```
即嗜血者数量 **不得大于总人数的 1/5**（向下取整）。

示例：
- 7 人局：最多 1 个嗜血者
- 10 人局：最多 2 个嗜血者
- 15 人局：最多 3 个嗜血者
```

---

## 执行指南

### 建议的编辑顺序

1. **修改 1**（时间戳）：简单替换，优先完成
2. **修改 2**（角色唯一性）：定位到 `1.2.2` 节，替换整个段落
3. **修改 3**（处决类型）：定位到 `1.4.1` 节，替换整个段落
4. **修改 4**（初始化配置）：在 `第一部分` 末尾新增整个 `1.7` 节
5. **修改 5**（身份公开约束）：定位到 `3.2` 节的发言范畴部分，替换整个列表
6. **修改 6**（字数约束）：在 `3.2` 节末尾新增子节
7. **修改 7**（嗜血者约束）：已包含在修改 4 中，无需单独操作

### 验证检查清单

修改完成后，请确认：
- [ ] 所有 7 处修改都已应用
- [ ] 章节编号没有错乱
- [ ] 代码块格式正确（特别是数学公式和伪代码）
- [ ] 交叉引用依然有效（如"参见 1.2.2"）
- [ ] 表格对齐正确
- [ ] 没有重复内容

---

## 预期结果

修改完成后，`FORMAL_RULES.md` 应该：
1. ✅ 时间戳更新为 2026/01/15 14:00
2. ✅ 角色唯一性规则清晰区分唯一/可重复角色
3. ✅ 处决类型明确说明执行人的公开性
4. ✅ 包含完整的游戏初始化配置算法
5. ✅ 信息报告必须先公开身份的约束被形式化
6. ✅ 发言字数限制使用动态公式
7. ✅ 嗜血者人数上限被明确约束

所有修改都保持了原文档的形式化风格和数学严谨性。
