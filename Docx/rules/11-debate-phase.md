# 3.2.3 辩论时（DEBATE Phase）

> **文档集**：《狼人杀变体游戏形式化规则说明书（Formal Specification）》`alpha 0.3`（多文件版）
> **本文覆盖编号**：`3.2.3`｜**所属部分**：第三部分（游戏流程时序定义）
> **索引 / 编号与引用约定**：[`Docx/FORMAL_RULES.md`](../FORMAL_RULES.md)

### 3.2.3 辩论时（DEBATE Phase）

```
DEBATE(t):
  1. 开放全场视野
  2. 按席位顺序依次发言：
     for p in sortedByPosition(Alive(t)):
       if alive(p, t) = ALIVE:
         allowSpeech(p, maxDuration)
  3. 校验发言范畴合法性（见下文 A–E 类）
  4. 校验发言字数约束（见下文「发言字数约束」）
```

**合法发言范畴与约束**

**A. 信息报告（Information Report）**
- **前置条件**：必须先宣称身份（Identity Claim）
- **执行顺序**：
  1. 玩家发言："我是 [角色名]"
  2. GM 记录宣称的身份：`identity_claimed(p, t) := [角色名]`
  3. 玩家提交信息报告内容
- **约束**：
```
  ∀p ∈ Alive(t), 若 p 提交信息报告，则必须满足：
    identity_claimed(p, t) ≠ None   // 必须先宣称某个身份
```
- **重要说明**：
  - 宣称的身份 **可以与真实身份不符**（如嗜血者谎称自己是吹哨人）
  - 信息报告的内容 **未必是真实的**（如虚假的检测结果）
  - GM **不验证** 宣称身份的真实性，仅记录玩家的发言
  - 其他玩家看到的是：`(玩家p, 宣称的身份, 报告内容)`，需自行推理真伪

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

> 本节是**发言行为**的定义；对应的**状态字段**定义见 1.2.5（`identity_claimed`）。

- 玩家可以 **宣称** 自己是某个角色（无论真假）
- GM 记录宣称但 **不验证真实性**
- **状态更新**：
```
  当玩家 p 宣称自己是角色 r 时：
    identity_claimed(p, t) := r
```
- **区分两个概念**（术语定义见 1.2.5 / 1.2.6）：
  - **身份宣称（Claim）**：玩家主动宣称 `"我是 X 角色"`，设置 `identity_claimed` 字段，可以撒谎
  - **身份公开（Reveal）**：系统强制公开真实身份，设置 `identity_revealed = true` 并覆盖 `identity_claimed := R(p)`，仅在治安角色使用技能时触发
- **禁止行为**：
  - ❌ **物理验证**：不允许"出示身份牌"、"截图身份"等行为
  - ❌ **GM 强制验证**：GM 不会主动证实玩家发言的真伪（除非治安技能触发）

**发言选项逻辑约束表**：

| 发言类型 | 前置条件 | 状态更新 | 内容可造假 | 身份可造假 | GM验证 |
|---------|---------|---------|----------|----------|-------|
| 信息报告 | `identity_claimed ≠ None` | 无 | ✅ 可以 | ✅ 可以（通过宣称） | ❌ 不验证 |
| 逻辑推理 | 无 | 无 | ✅ 可以 | N/A | ❌ 不验证 |
| 质疑/信任 | 无 | 无 | N/A | N/A | ❌ 不验证 |
| 指控/辩护 | 无 | 无 | ✅ 可以 | N/A | ❌ 不验证 |
| 身份声明 | 无 | `identity_claimed := r` | N/A | ✅ 可以 | ❌ 不验证 |

**实现注意事项**：
- 模型在选择"信息报告"时，必须先执行"身份声明"（如果尚未宣称）
- 玩家可以多次宣称不同身份，`identity_claimed` 字段会被最新的宣称覆盖
- 宣称的身份和报告内容都可能是虚假的，推理责任在其他玩家
- 只有治安角色使用技能时，系统才会强制公开真实身份（设置 `identity_revealed = true`）

**时长**：每人发言时长上限（如60秒/人）

**发言字数约束**

每位玩家的单次发言必须满足动态字数限制：

约束公式：
```
token_limit(round) = BASE_TOKENS + random_seed(round)

其中：
  BASE_TOKENS = 150                // 基础字数（计量口径：token，与 7.6 的 tokenizer 统计一致）
  random_seed(round) ∈ [-50, 50]   // 本轮随机偏移量
```

形式化定义：
```
∀p ∈ Alive(t), ∀speech ∈ Speech(p, t):
  tokens(speech.text) ≤ token_limit(t)
```

随机种子生成：
- 每轮辩论时开始时，从均匀分布 `Uniform(-50, 50)` 中抽取整数 `δ`
- 该轮所有玩家共享同一个字数限制
- 示例：
  - 第 1 轮：`δ = +30`，字数限制 = `150 + 30 = 180`（token）
  - 第 2 轮：`δ = -20`，字数限制 = `150 - 20 = 130`（token）

实现伪代码：
```python
def calculate_token_limit(round: int, prng: PRNG) -> int:
    BASE_TOKENS = 150
    random_offset = prng.uniform_int(-50, 50)
    return BASE_TOKENS + random_offset

# 在辩论时开始时执行
def debate_phase_start(game: GameState):
    game.current_token_limit = calculate_token_limit(game.round, game.prng)
    for player in game.alive_players:
        enforce_token_limit(player.speech, game.current_token_limit)
```

违规处理：
- 如果玩家发言超过字数限制，GM 截断超出部分
- 截断规则：保留前 `token_limit(t)` 个 token，后续内容丢弃
- 截断必须按 token 边界对齐，禁止在 token 内部截断

> **计量口径**：本条约束**统一以 token 为唯一计量单位**，由实现层使用的分词器（tokenizer）统计。
> 不再另行定义"字符数"口径，以免两种口径并存导致不同模型间评测不可比。
> 若以汉字近似换算，150 token 约相当于 100–120 个汉字，仅供理解参考，不作为实现依据。

> **评测一致性要求**：`random_offset` 必须由初始化时固定的随机种子 `seed`（见 3.2.1）派生，
> 以保证在 Headless 评测模式下同一 seed 可完全复现，避免因字数限制随机化而影响不同模型间的可比性。

---

**追加辩论（投票时阶段插入的辩论）**

- **触发**：投票时阶段的第 2、3 轮玩家目标投票之前，各插入一次辩论（见 `3.2.4`）
- **参与人**：所有存活玩家按席位顺序发言，**不限于候选人**
- **字数上限**：是否沿用本轮辩论时已抽取的 `token_limit(t)`，还是重新抽取，**尚未裁定**（见 `8.5`）。
  **建议**（未采纳为规则）：沿用本轮已抽取的上限，以保持同一轮内的可比性
- **不影响上限**：追加辩论不改变"玩家目标投票最多三轮"的上限（见 `4.1 INV9`）

---
