## 项目概要 (Project Overview)

本项目是一个基于 **Rust** 构建的高性能多智能体（Multi-Agent）对抗与评测框架。项目以经典的不完全信息博弈——**“狼人杀”（Werewolf）** 作为核心演练沙盒，**旨在彻底打破传统静态测试集“易被污染、易针对性刷分（Benchmark Gaming）”的弊端**，通过多轮动态博弈对抗，客观评估不同 AI 在**心智揣摩（Theory of Mind）、欺骗与反欺骗（Deception）、社会推理与多轮说服**等高阶智能维度上的真实水平。

本项目采用**“双轮驱动”**的设计架构，包含两大核心目标：

### 1. 高通量动态评测引擎（快模式 · Headless Benchmark）
* **定位**：严肃评测与科学度量。
* **核心能力**：构建一个零 I/O 渲染、高并发的自包含无头（Headless）引擎。不仅针对大语言模型（LLM），同时支持传统规则智能体与强化学习算法接入。
* **价值主张（杜绝刷分，拒绝死记硬背）**：
  * **对抗基准刷分（Anti-Gaming by Design）**：当前各类静态问答/选择题测试集已被严重“污染”，模型极易通过数据投毒、提示词微调产生虚高的跑分，导致“榜单高分与实际智商严重脱节”。
  * **动态博弈取代静态考卷**：狼人杀具有极高的随机发牌空间、多轮信息差和不可预测的交互状态。在无先验答案的环境下，模型**无法依靠记忆题库或模板套路刷分**，唯有展现出真正的上下文自适应、身份伪装与策略推演能力才能生存。
  * **多维定性与定量基准**：结合角色分级 Elo 积分、反欺骗成功率、逻辑一致性及胜率方差等多维指标，为极客与算法社区建立一个抗污染、高鲁棒的多智能体权威天梯。

### 2. 具象化观赛与人机交互沙盒（慢模式 · Visual & Interactive）
* **定位**：视觉化表达、社交传播与人机协同。
* **核心能力**：通过事件流（Event Stream）驱动相对精致的可视化界面（如像素风动画桌游交互）。将枯燥的对局日志解构为富有戏剧张力的角色微表情、对话气泡与动态投票过程。
* **价值主张**：降低大众与桌游爱好者的理解门槛，解决学术/算法评测“酒香也怕巷子深”的传播痛点，赋能社交媒体二创；同时提供人类介入接口（Human-in-the-Loop），支持人类玩家亲自入局，与不同 AI 模型展开同台对抗。

---

## Rust 编码规范（Coding Standards）

Rust 的编码规范分为两层：**工具可自动检查的部分**（命名、格式）和**工程实践部分**（错误处理、API 设计、unsafe 纪律）。前者交给 `rustfmt` + `clippy` 解决，后者是拉开代码质量差距的关键。

### 一、工具链强制（Tool-Enforced Standards）

**规范的第一原则：别靠人肉记规范**，让工具自动检查：

```bash
cargo fmt                          # 统一格式化
cargo clippy -- -D warnings        # 静态检查，将警告视为错误
```

**CI 必须**：在持续集成中运行 `cargo fmt --check` 和 `cargo clippy`，确保团队风格一致。

---

### 二、命名规范（Naming Conventions）

遵循官方 RFC 430 的硬性约定：

| 对象 | 风格 | 示例 |
|------|------|------|
| 类型、trait、enum 变体 | `UpperCamelCase` | `struct HttpClient`、`trait Iterator` |
| 函数、方法、变量、模块 | `snake_case` | `fn fetch_user()`、`user_name` |
| 常量、静态变量 | `SCREAMING_SNAKE_CASE` | `const MAX_RETRIES: u32 = 3;` |
| 生命周期 | 短小写 | `'a`、`'de` |
| 宏 | `snake_case!` | `println!`、`vec!` |
| 类型参数 | 单个大写字母 | `T`、`K`、`V` |

**关键细节**：
- **缩写视为普通单词**：用 `HttpClient` 而非 `HTTPClient`，`Uuid` 而非 `UUID`
- **转换方法前缀有严格语义**（API 指南 C-CONV）：
  - `as_`：廉价的借用转换（`as_ref()`）
  - `to_`：开销较大的转换（`to_string()`）
  - `into_`：消耗所有权的转换（`into_iter()`）
- **避免 Java 式 getter/setter**：Rust 社区习惯直接暴露 `pub` 字段，仅在需要维护不变量时才加校验方法

---

### 三、错误处理（Error Handling）

这是 Rust 代码质量的核心分水岭：

- **可恢复错误用 `Result<T, E>`**，不可恢复（程序 bug）才 `panic!`
- **生产代码避免 `unwrap()`**：如果确定不会失败，用 `expect("说明为什么安全")` 留下调试线索
- **用 `?` 传播错误**，避免层层 `match` 嵌套
- **库定义自己的错误类型**（配合 `thiserror`），应用层可用 `anyhow` 快速兜底
- **错误信息要有上下文**：`file.read()?.context("读取配置文件失败")?`

---

### 四、所有权与借用（Ownership & Borrowing）

- **优先借用 `&T`**，能不 `clone()` 就不 `clone()`——每次 `clone()` 都该问自己设计是否合理
- **参数能收 `&str` 就别收 `String`**，能收 `&[T]` 就别收 `Vec<T>`
- **生命周期能省略就省略**，编译器推不出来时再显式标注
- **能用迭代器就用迭代器**，少写 `for i in 0..len { v[i] }` 这种索引循环（既慢又可能越界 panic）

---

### 五、API 设计（API Design）

遵循官方 API 指南的高价值原则：

- **C-BOOL**：用 `enum` 代替 `bool` 参数——`draw(Shape::Rounded)` 比 `draw(true)` 可读性强
- **C-NEWTYPE**：给基本类型包一层（如 `struct UserId(u64)`），用类型系统防止参数传错
- **C-DEFAULT**：能实现 `Default` 就实现
- **C-BUILDER**：可选参数多时用构建器模式
- **C-DEBUG**：公开类型都该 `#[derive(Debug)]`
- **C-MUST-USE**：不该被忽略的返回值加 `#[must_use]`

---

### 六、文档注释（Documentation）

- `///` 写公开 API 文档，`//!` 写模块级文档
- **公共 crate 建议启用**：`#![warn(missing_docs)]`
- **注释解释「为什么」而非「是什么」**——代码本身已说明是什么
- **Doctest 既是文档又是测试**：文档中的 ``` 代码块会被 `cargo test` 执行
- **模块顶部用 `//!`**：简述模块职责

---

### 七、测试（Testing）

- **单元测试**：写在同文件的 `#[cfg(test)] mod tests` 里
- **集成测试**：放 `tests/` 目录，测试公开 API
- **断言优先用 `assert_eq!`**：失败时会打印两个值，调试友好

---

### 八、`unsafe` 纪律（Unsafe Discipline）

- **最小化 unsafe 块**，把不安全操作封装在安全接口后面
- **每个 `unsafe` 块上方必须写 `// SAFETY:`**：说明为什么这段代码安全、维护了哪些不变式
- **能用安全抽象解决就别上 unsafe**

---

### 九、工程实践（Engineering Practices）

- **`Cargo.lock`**：二进制项目必须提交，库项目视情况而定
- **精简依赖树**：定期用 `cargo audit` / `cargo deny` 检查漏洞
- **避免全局变量**：需要共享状态用 `Arc<Mutex<T>>` 或消息传递
- **提取魔法数字为常量**
- **早返回（early return）**：减少嵌套层数

---

### 十、模块与可见性（Modules & Visibility）

- **默认私有，只在需要时 `pub`**——可见性收得越紧，重构越自由
- **2018 edition 之后**：不必强求 `mod.rs`，可用与目录同名的 `.rs` 文件
- **一个模块一个职责**：文件过长就该拆分

---

## 源码组织规范（Project Structure）

1. **单文件长度上限**：任何单一 `.rs` 源文件长度应控制在 **200 行以内**（单测极为充分时最多不超过 250 行）。一旦接近阈值，必须进行关注点分离（SoC）并拆分子模块。
2. **门面隐藏机制（Facade Pattern）**：
   - `lib.rs` 是外部唯一的深接口窗口，负责统一 `pub use` 核心 API。
   - 所有内部子模块均声明为 `pub(crate)`，严禁对外泄露内部细分阶段逻辑。
3. **分层依赖单向性**：
   - 依赖流向：`phases/*` -> `types.rs`。
   - 同级子模块（如 `night_wolf.rs` 与 `night_witch.rs`）严禁横向交叉引用，所有协同必须通过 `engine.rs` 进行调度。