# 类型代数

> Type Algebra
>
> 积类型、和类型、函数类型构成完备的域建模原语。

---

## §0 核心思想

类型**即**集合。每个领域实体建模为其合法值的集合。类型的势（cardinality, $|T|$）紧密关联其信息量。

$$\text{Bool} = \lbrace\text{true}, \text{false}\rbrace,\quad \lvert\text{Bool}\rvert = 2$$

$$\text{TrafficLight} = \lbrace\text{Red}, \text{Yellow}, \text{Green}\rbrace,\quad \lvert\text{TrafficLight}\rvert = 3$$

$\lvert T\rvert = 1$ 的类型无信息量（Unit）。$\lvert T\rvert = 0$ 的类型不可能被构造（Bottom/Never）。

---

## §1 基本构造

### §1.1 积类型（$\times$ — 同时持有）

$$A \times B:\text{持有 A 且持有 B}$$

$$\lvert A \times B\rvert = \lvert A\rvert \cdot \lvert B\rvert$$

对应 `struct { a: A, b: B }` 或元组 `(A, B)`。积类型编码"并列属性"。

### §1.2 和类型（$+$ — 择一持有）

$$A + B:\text{持有 A 或持有 B（绝不同时）}$$

$$\lvert A + B\rvert = \lvert A\rvert + \lvert B\rvert$$

对应标签联合 `enum { A(A), B(B) }`。

**和类型是建模"可能性空间"的精确工具**。穷举匹配保证每种可能性都被处理，遗漏任何变体是编译错误。

### §1.3 函数类型

$$f : A \to B \quad \text{（全函数：每个输入必产生输出）}$$

$$f : A \rightharpoonup B \quad \text{（偏函数：部分输入无确定输出）}$$

$$\lvert A \to B\rvert = \lvert B\rvert^{\lvert A\rvert}$$

### §1.4 递归类型

$$\text{List}(A) = \mu L.\ \mathbf{1} + (A \times L)$$

一个列表要么为空（$\mathbf{1}$），要么是一个元素与另一个列表的积（$A \times L$）。

递归类型建模所有自引用结构：树、图、表达式、协议消息等。

### §1.5 特殊类型

| 记法 | 名称 | 势 | 含义 |
|------|------|-----|------|
| $\mathbf{1}$ | 单位类型 | 1 | 无信息 |
| $\mathbf{0}$ | 空类型 | 0 | 不可能 |

### §1.6 汇总

| 构造 | 记法 | 语义 | 典型映射 |
|------|------|------|----------|
| 积类型 | $A \times B$ | 同时持有 | `struct { a: A, b: B }` |
| 和类型 | $A + B$ | 择一持有 | `enum { A(A), B(B) }` |
| 函数类型 | $A \to B$ | 确定性映射 | `fn(A) -> B` |
| 偏函数 | $A \rightharpoonup B$ | 可失败映射 | `fn(A) -> Result<B, E>` |
| 递归类型 | $\mu T.\ F(T)$ | 自引用结构 | 递归 enum |
| 单位类型 | $\mathbf{1}$ | 无信息 | `()` |
| 空类型 | $\mathbf{0}$ | 不可能 | `!` (never) |

---

## §2 代数恒等式

类型服从的代数恒等式直接指导编程决策：

| 恒等式 | 编程含义 |
|--------|----------|
| $A \times \mathbf{1} \cong A$ | 持有 Unit 的字段无信息量——删除之 |
| $A + \mathbf{0} \cong A$ | Never 变体不可构造——删除之 |
| $(A + B) \to C \cong (A \to C) \times (B \to C)$ | 处理和类型 = 为每个变体提供处理器 = 穷举 match |
| $A \to (B \times C) \cong (A \to B) \times (A \to C)$ | 返回积类型 = 分别计算每个分量 |
| $A \times (B + C) \cong (A \times B) + (A \times C)$ | 分配律：积套和可展平为和套积 |

这些不是"技巧"，是数学事实。违反它们会在代码中制造结构性冗余。

---

## §3 全函数优先

### 公理

$$f: A \to B \succ f: A \rightharpoonup B$$

每个函数都应是全函数。偏函数意味着：
- 和类型缺少变体
- 可选字段本应必选
- 意图表达不完备

### 全函数化方法

使偏函数全函数化的规范方法是扩展值域：

$$f : A \rightharpoonup B \implies f' : A \to B + E$$

即把"可能失败"显式编码到返回类型中。这不是"加了错误处理"，而是承认了域的完整可能性空间。

### 路由/分发函数的全性约束

若系统存在路由函数 $f: \text{Intent} \to \text{Target}$，则 $f$ 必须是全函数——每一个合法的 $\text{Intent}$ 值都必须产生合法的 $\text{Target}$。

禁止 `Option`、`Wildcard`、`fallback` 替用户的意图缺失买单。若用户不知道某个维度的值，这是**编译期错误**或 **API 设计缺陷**，不是运行时 fallback 场景。

---

## §4 势分析

### 原则

$$\lvert\text{Type}\rvert = \lvert\text{Domain}\rvert$$

当一个类型能表达的值的数量远大于域中合法值的数量时，类型设计过于宽松——存在非法状态可被构造。

### 诊断方法

计算类型的势，与域中合法值对比：

$$\lvert\text{Type}\rvert \gg \lvert\text{Domain}\rvert \implies \text{收窄类型}$$

$$\lvert\text{Type}\rvert < \lvert\text{Domain}\rvert \implies \text{扩展类型（缺少合法状态）}$$

### 示例

**过于宽松**：用 `(Option<A>, Option<B>)` 表示"A 或 B 必有其一"

$$\lvert\text{Option}(A) \times \text{Option}(B)\rvert = (\lvert A\rvert + 1)(\lvert B\rvert + 1)$$

包含 `(None, None)` 这一非法状态。应使用和类型：

$$\text{Either}(A, B) = A + B,\quad \lvert A + B\rvert = \lvert A\rvert + \lvert B\rvert$$

精确匹配域语义，无非法状态。

---

## §5 代数恒等式指导重构

当感觉代码冗余时，检查是否违反了代数恒等式。

### DRY 的代数定义

$$f \cong g \wedge f \ne g \implies \text{合并为参数化的 } h(T)$$

若 N 个函数仅在类型参数上不同，它们是同一个泛型函数的 N 次实例化。

- 差异在类型 → 用泛型参数
- 差异在值 → 用函数参数
- 差异在行为 → 用接口/trait 方法
- 不允许"文本级复制"代替"代数级抽象"

### 分配律指导数据结构设计

$$A \times (B + C) \cong (A \times B) + (A \times C)$$

左侧是"一个结构体里放了个枚举"，右侧是"枚举的每个变体各自携带数据"。两者同构——选择更能反映域语义的形式。
