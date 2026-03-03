# 基于能力的编程

> Capability-Oriented Programming
>
> 能力的组合决定身份，而非用身份推理能力。

---

## §0 核心命题

**实体不是先有名字再有属性，而是先有能力再有名字。**

名字是能力集合的简称。两个具有不同能力集合的东西就是两种东西——即使人类习惯用同一个词称呼它们。

$$\text{Identity}(x) \cong \text{Capabilities}(x)$$

推论：如果两个实体的能力集合不同，它们就是不同的类型，无论词汇上多么相似。类型系统的职责是忠实表达这一事实。

---

## §1 反范式：从身份到能力

传统设计的思维路径是**身份优先**：

```
"这是一个金融工具"
  → 它有种类（Spot / Perpetual / ...）
    → 根据种类，它可能有某些属性
      → 用 Option 表达"可能有"
```

这条路径产生 **Option 地狱**——积类型中充满了"有时存在，有时不存在"的字段：

```rust
// 反范式：身份优先
struct Instrument {
    pair: AssetPair,
    kind: InstrumentKind,
}
struct InstrumentSpec {
    maintenance_margin_rate: Option<Decimal>,  // Spot 永远是 None
    max_leverage: Option<Decimal>,             // Spot 永远是 None
    funding_interval: Option<Duration>,        // Spot + Future 永远是 None
    expiry: Option<Timestamp>,                 // Spot + Perpetual 永远是 None
    strike_price: Option<Decimal>,             // 只有 Option 才有
    // ...
}
```

`Option::None` 在这里表达的不是"值未知"，而是"概念不存在"。这是一个**范畴错误**——用值层面的缺席表达类型层面的不存在。

---

## §2 正范式：从能力到身份

基于能力的思维路径是**能力优先**：

```
先定义原子能力（Margined, FundingBearing, Expiring, ...）
  → 每种实体是一组能力的交集
    → 名字只是这个交集的简称
      → 每个实体恰好只携带它需要的数据
```

```rust
// 正范式：能力优先
trait Tradeable    { fn pair(&self) -> &AssetPair; }
trait Margined     : Tradeable {}
trait FundingBearing: Tradeable {}
trait Expiring     : Tradeable { fn expiry(&self) -> Timestamp; }
trait Strikeable   : Expiring  { fn strike_price(&self) -> Decimal; fn option_kind(&self) -> OptionKind; }

// 杠杆 l 是通用量——l=1（现货全额）是平凡值而非"空"，不是类型判别式
// 保证金才是真正的类型判别式：纯现货不存在抵押品/强平等概念

// "Spot" 只是 {Tradeable} 这个能力集合的名字
struct Spot { pair: AssetPair }
impl Tradeable for Spot { ... }
// Spot 不实现 Margined —— 纯现货没有保证金机制

// "Perpetual" 只是 {Tradeable, Margined, FundingBearing} 的名字
struct Perpetual { pair: AssetPair }
impl Tradeable for Perpetual { ... }
impl Margined for Perpetual {}
impl FundingBearing for Perpetual {}
```

此时函数签名可以**精确表达它需要什么能力**：

```rust
fn subscribe_funding_rates(instrument: &impl FundingBearing) -> Stream<FundingRate>;
// Spot 在编译期被拒绝 —— 它没有 FundingBearing 能力
// 不再需要运行时 match + NotSupported 错误
```

---

## §3 形式化：Yoneda 引理的直觉

范畴论中的 Yoneda 引理给出了这一范式的数学基础：

$$\text{一个对象完全由它接受的态射（能力）决定。}$$

不必知道对象的"内部结构"——只需知道它能参与哪些操作，就能完全确定它是什么。这正是**能力即身份**的形式化表述：

$$X \cong \text{Hom}(-, X)$$

一个金融工具**是什么**，完全等价于**其他东西能对它做什么**。现货就是那些"只能进行可交易操作"的东西；永续合约就是那些"能进行可交易、有保证金、有资金费率操作"的东西。

---

## §4 公理：可选不等于不存在

$$\text{Option}\langle A \rangle = 1 + A \quad \neq \quad \text{概念 } A \text{ 不适用于此类型}$$

`Option<A>` 表达的是：**值 $A$ 存在但当前未知或未填充**。它编码的是信息的暂时缺失。

但当我们把 Spot 的 `maintenance_margin_rate` 设为 `None` 时，我们表达的不是"维护保证金率未知"，而是"维护保证金率这个概念对 Spot 不存在"。这是一个范畴性的区别：

| 情况 | 语义 | 正确编码 |
|------|------|----------|
| 交易所未返回维护保证金率 | 值暂时未知 | `Option<Decimal>` |
| 现货根本没有维护保证金率 | 概念不存在 | 类型中不含此字段 |

如果两种截然不同的语义被编码为同一个 `None`，**调用者无法区分"交易所没告诉我"和"这个东西根本没有"**。这导致防御性编程的退化——每次使用 `Option` 时都需要额外的上下文信息（比如检查 `InstrumentKind`）来决定 `None` 的含义。

### §4.1 升维谬误

一种常见的反模式是将较简单的实体视为较复杂实体的退化情形：

> "现货就是杠杆=1、无仓位、无资金费率的永续合约"

这在值层面看似合理，但在类型层面是一个**升维嵌入谬误**——把低维对象嵌入高维空间只为了在高维空间中统一处理，然后在高维空间中对某些维度取零或 None。

$$\text{Spot} \hookrightarrow \text{Perpetual}|_{\text{leverage}=1,\ \text{funding}=0,\ \text{margin}=\varnothing}$$

问题在于：嵌入之后，类型系统看到的是 `Perpetual`，它具备所有永续合约的能力。你可以调用 `set_leverage(spot_as_perp, 10)`，编译通过，运行时炸裂。

**正确的做法是**：不嵌入。Spot 就是 Spot，它的能力集合就是 $\lbrace\text{Tradeable}\rbrace$。它不是退化的 Perpetual，因为它的能力集合是 Perpetual 能力集合的真子集——在范畴论中，这两个对象不同构。

---

## §5 接口隔离原理的重新推导

经典 SOLID 的接口隔离原则（Interface Segregation Principle, ISP）声明：

> 不应强迫客户端依赖它不使用的方法。

从能力编程的视角，ISP 不是一条设计规则，而是**自然推论**：

1. 每个函数声明它需要的能力（trait bound）
2. 调用者只需要提供满足该能力的类型
3. 如果一个 trait 包含了调用者不需要的方法，那这个 trait 混合了不同的能力——它不再是原子能力

因此 ISP 的本质是**能力的原子性**：每个 trait 应恰好编码一个原子能力。多个原子能力的组合通过 trait bound 的 `+` 实现，而非在单个 trait 中聚合。

$$\text{fn } f(x: \&\text{impl } A + B + C) \quad \text{而非} \quad \text{fn } f(x: \&\text{impl } \text{GodTrait})$$

---

## §6 设计工作流

### §6.1 领域分析

1. **列举所有原子能力**：从领域的所有操作出发，识别最小的不可再分的能力单元
2. **为每个能力定义 trait**：一个能力 = 一个 trait，携带该能力必需的关联数据
3. **识别能力组合**：列举领域中实际存在的能力组合（不是所有排列组合——只有领域中真实存在的交集）
4. **命名**：每个真实存在的能力组合获得一个 struct 名称——这个名称是简称，不是本体

### §6.2 实现

1. 每个能力组合实现其对应的所有 trait（也仅实现这些 trait）
2. 如果需要统一处理所有组合，使用 ADT（enum）作为和类型包装
3. 函数签名使用 `&impl Trait` 或泛型 bound 声明能力需求
4. 需要动态分发时，使用 `&dyn Trait` 或 enum + match

### §6.3 验证

- 每个函数是否只声明了它真正使用的能力？
- 是否存在某个 struct 实现了它不需要的 trait？（不需要 = 该能力的关联数据在此类型上不具有领域语义）
- 是否存在某个函数接受了过宽的类型？（例如接受 `&Instrument` 却只服务 Perpetual）
- 任何编译时可检查的约束是否被推迟到了运行时？

---

## §7 学术渊源

本文描述的范式并非发明，而是多个成熟理论的汇聚应用：

| 领域 | 理论 | 与本文的关系 |
|------|------|-------------|
| 范畴论 | Yoneda 引理 | 对象由其态射完全确定 → 能力即身份 |
| 类型论 | 代数数据类型 (ADT) | 和类型精确编码"N 选 1" → 消除 Option 地狱 |
| 面向对象设计 | 接口隔离原则 (ISP) | 不依赖不使用的方法 → 能力原子化 |
| 面向对象设计 | Liskov 替换原则 (LSP) | 子类型必须满足父类型契约 → 禁止升维谬误 |
| 能力安全 | Object-Capability Model | 对象的权力由其持有的能力决定 → 最小权限 |
| 协议理论 | Session Types | 通道的类型编码其交互能力 → 编译期协议正确性 |
| 效果系统 | Algebraic Effects | 函数声明其副作用能力 → 类型安全的效果组合 |

这些理论的共同核心是：**不要把信息藏在值里，把信息放在类型里。编译器能检查类型，不能检查值。**

---

## §8 反模式速查

| 反模式 | 症状 | 修复 |
|--------|------|------|
| Option 地狱 | struct 中多个 `Option` 字段在某些变体下永远是 `None` | 拆成 ADT，每个变体只含有意义的字段 |
| 升维嵌入 | 简单实体被视为复杂实体的退化；`leverage = 1` 表示"无杠杆" | 保持类型独立，不嵌入 |
| 上帝 trait | 单个 trait 包含所有操作，部分实现返回 `NotSupported` | 拆成原子能力 trait |
| 运行时类型检查 | `match instrument.kind { Spot => Err(NotSupported) }` | 用 trait bound 在编译期拒绝 |
| 标签分发 | 根据 `kind` 字段决定执行路径 | 用 enum match 或 trait 动态分发 |
| 语义重叠的 None | `Option<T>` 既表示"未知"又表示"不适用" | 这两种语义必须有不同的类型表示 |
