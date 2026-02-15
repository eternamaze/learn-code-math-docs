# 形式化代码诊断

> Formal Code Diagnostics
>
> 用形式化诊断谓词替代直觉——让审计可复现。

---

## §0 动机

代码审查通常依赖个人直觉和经验。问题在于：直觉不可复现，不可传授，不可自动化。

形式化诊断将审计转化为**对谓词的求值**——每个诊断有精确的数学定义，对同一代码的审计结论一致。

---

## §1 诊断谓词

### 定义

诊断谓词是对代码元素（类型或函数）的布尔判定函数。

| 谓词 | 定义 | 含义 |
|------|------|------|
| $\text{WrongTier}(x)$ | $\text{tier}(x) \ne \text{natural\_tier}(x)$ | 类型在错误的抽象层级 |
| $\text{Leak}(x)$ | $\sigma(x) = ◆ \wedge \text{visible}(x)$ | 内部实现泄露到公开表面 |
| $\text{DRY}(f, g)$ | $f \cong g \wedge f \ne g$ | 同构代码重复 |
| $\text{Partial}(f)$ | $f: A \rightharpoonup B \wedge \exists f': A \to B$ | 可全函数化的偏函数 |
| $\text{VerticalBloat}(T)$ | $\exists m \in T : m\ \text{默认返回 NotSupported}$ | 接口包含非普适方法 |
| $\text{DeadCode}(x)$ | $\nexists \text{path}(\text{root}, x)$ | 无可达路径的死代码 |
| $\text{SemanticAmbiguity}(x)$ | $\lvert\text{interpretations}(x)\rvert > 1$ | 同一表示有多种互不兼容的语义 |

### 谓词间的蕴含关系

- $\text{Leak}(x) \implies \text{WrongTier}(x)$——泄露一定意味着层级错误
- $\text{DeadCode}(x) \implies \neg\text{Leak}(x)$——不可达的代码不可能泄露

---

## §2 严重性分级

| 级别 | 触发条件 | 行动 |
|------|----------|------|
| **SEVERE** | $\text{WrongTier} \vee \text{Leak}$ | 必须立即修复——结构性缺陷 |
| **MEDIUM** | $\text{DRY} \vee \text{Partial}$ | 应当修复——代码质量问题 |
| **LOW** | $\text{VerticalBloat} \vee \text{SemanticAmbiguity}$ | 记录并计划——设计改进空间 |
| **INFO** | 无违规但可优化 | 知悉即可 |

---

## §3 系统化分析流程

对任意代码库，依次执行：

### Step 1: 类型清单

对每个类型 $T$：写出代数形式 $T = \ldots$，标注层级 $\text{Tier}_n$，标注特异性 $\sigma(T)$。

**示例**：

$$\text{Config} = \text{Name} \times \text{Timeout} \times \text{RetryCount}$$

$$\text{tier}(\text{Config}) = \text{Tier}_1,\quad \sigma(\text{Config}) = ★$$

### Step 2: 态射清单

对每个函数/方法 $f$：写出签名 $f : A \to B$，验证全性，验证层级。

**示例**：

$$\text{resolve} : \text{Intent} \to \text{Result}(\text{Target}, \text{Error})$$

$\text{resolve}$ 是全函数 ✓，$\text{tier} = \text{Tier}_3$ ✓

### Step 3: 应用诊断谓词

对每个 $(T, f)$ 执行 §1 的全部诊断谓词，记录所有命中。

### Step 4: 物理路径一致性

$$\text{物理路径}(x) \cong \text{逻辑路径}(x)$$

文件目录结构必须与类型代数层级一致。路径即文档。

---

## §4 常见诊断模式

### 模式 1: 和类型缺少变体

**触发谓词**：$\text{Partial}(f)$

**信号**：函数体中出现 `panic!`、`unreachable!`、或 `todo!` 处理"不应该发生"的情况。

**修正**：扩展输入的和类型以覆盖所有情况，或收窄函数的定义域以排除非法输入。

### 模式 2: 接口膨胀

**触发谓词**：$\text{VerticalBloat}(T)$

**信号**：接口中某些方法的默认实现返回"不支持"。

**修正**：将非普适方法提取为独立接口。

$$\text{Fat Interface} \to \text{Core Interface} + \text{Extension Interface}$$

只有真正需要扩展能力的实现者才实现扩展接口。

### 模式 3: 封装泄露

**触发谓词**：$\text{Leak}(x)$

**信号**：内部实现类型出现在公开 API 签名中，或用户代码直接构造内部类型。

**修正**：
- 降低类型可见性
- 引入工厂方法或 Builder 模式
- 使用不透明类型（`impl Trait` 等）

### 模式 4: 代数同构重复

**触发谓词**：$\text{DRY}(f, g)$

**信号**：N 个函数骨架相同，仅在类型参数或少量值上不同。

**修正**：
- 差异在类型 → 泛型参数
- 差异在值 → 函数参数
- 差异在行为 → 接口方法
- 禁止"文本级复制"代替"代数级抽象"

---

## §5 诊断报告格式

每次审计产出标准化报告：

```
| # | 严重性 | 发现 | 谓词 | 修正方向 |
|---|--------|------|------|----------|
| 1 | SEVERE | ... | WrongTier | ... |
| 2 | MEDIUM | ... | DRY(f, g) | ... |
```

### 审计指标

$$\text{健康度} = 1 - \frac{|\text{SEVERE}| \cdot 4 + |\text{MEDIUM}| \cdot 2 + |\text{LOW}|}{|\text{总元素}| \cdot 4}$$

### 框架吸收率验证

$$|★| \leq |●| + |○| \implies \text{框架吸收了足够的复杂性} \checkmark$$

---

## §6 诊断与重构的关系

诊断是重构的**前置分析**。重构不应该从"感觉哪里不对"开始，而应该从诊断报告的 SEVERE 项开始。

$$\text{审计} \xrightarrow{\text{诊断谓词}} \text{诊断报告} \xrightarrow{\text{按严重性}} \text{重构优先级} \xrightarrow{\text{手术}} \text{修复}$$

这确保了重构是**有目标的手术**，而非**无方向的重写**。
