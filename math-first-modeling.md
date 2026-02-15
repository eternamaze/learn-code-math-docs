# 数学优先建模

> Math-First Modeling
>
> 程序是数学模型的可执行翻译——绕过建模的代码是有损映射。

---

## §0 根公理

$$\text{Reality} \cong \text{Mathematics}$$

自然世界服从数学规律。网络拓扑是图。状态机是自动机。并发模型是 Petri 网。业务规则是逻辑谓词。

**推论**：任何领域都可以先被数学建模，再被编码为程序。数学模型是程序的规格说明（specification），程序是数学模型的可执行实现。

---

## §1 建模链

正确的建模链是三阶同构映射：

$$\boxed{\text{Reality} \xrightarrow[\text{建模}]{\phi} \text{Math Model} \xrightarrow[\text{编码}]{\psi} \text{Type System} \xrightarrow[\text{编译}]{\gamma} \text{Executable}}$$

| 阶段 | 输入 | 输出 | 性质 |
|------|------|------|------|
| $\phi$ | 现实世界的观察 | 集合、函数、关系、公理 | 应当同构；不可避免时保序 |
| $\psi$ | 数学结构 | 类型、接口、枚举、函数签名 | 必须同构——类型系统是数学模型的字面翻译 |
| $\gamma$ | 类型约束 | 机器码 | 由编译器保证正确性 |

**关键洞见**：编程中的大部分缺陷发生在 $\phi$（错误的数学模型）和 $\psi$（代码偏离数学模型）。$\gamma$ 由编译器保证，几乎不会出错。

$$\text{投入精力的正确分布}:\ \phi > \psi \gg \gamma$$

把人的精力从调试运行时行为转移到建模和类型设计——这是"在类型系统上投入更多"的根本理由。

---

## §2 为什么不能从自然语言直接跳到代码

绕过数学建模的映射链是有损的：

$$\text{Reality} \xrightarrow[\text{有损}]{\alpha} \text{自然语言需求} \xrightarrow[\text{歧义}]{\beta} \text{代码}$$

- $\alpha$ 有损——自然语言无法精确表达量化关系、边界条件和不变量
- $\beta$ 歧义——相同自然语言被不同开发者解读为不同代码

| 维度 | 自然语言需求文档 | 数学建模 |
|------|----------------|----------|
| 精确性 | "用户可以处于活跃或冻结状态" | $\text{State} = \lbrace\text{Active}, \text{Frozen}\rbrace$, $\lvert\text{State}\rvert = 2$, 穷举可验证 |
| 完备性 | 隐含假设遍布 | 公理显式声明，推论可验证 |
| 边界条件 | 常被遗漏 | 函数定义域/值域强制覆盖 |
| 可组合性 | 段落无法组合 | 类型可代数组合：$A \times B$, $A + B$, $A \to B$ |
| 可验证性 | 人工审查 | 编译器 + 穷举检查 |
| 可重用性 | 复制粘贴 | 参数化多态（泛型） |

**数学公式的信噪比远高于文本描述。**

$$\text{Request} = \text{Target} \times \text{Action} \times \text{Quantity} \times \text{Price}?$$

这一行公式精确编码了请求的完整结构。等价的自然语言需要一个段落，且仍然可能产生歧义。

---

## §3 同构原则

### 定义

$$\text{Good Code} \iff \text{Code} \cong \text{Math Model} \cong \text{Reality}$$

"好代码"不是"看起来整洁的代码"，而是"与其数学模型同构的代码"。同构保证：

- **正确性**：代码精确反映域语义
- **完备性**：所有域状态都可在代码中表达
- **无冗余**：域中不存在的状态在代码中也不可构造

### 违反同构的信号

| 信号 | 含义 |
|------|------|
| 字符串匹配决定行为 | 类型系统未编码域语义 |
| `Option` 用于"有时候不需要" | 和类型未区分不同种类的"没有" |
| Boolean 参数切换行为 | 本应是两个独立函数或两个枚举变体 |
| 注释解释代码意图 | 代码本身不自解释——类型系统不够精确 |
| 能构造出域中非法的值 | 构造权失控——需要智能构造器 |

### 恢复同构的方法

1. **提升到类型**：将值层判断提升为类型层区分
2. **收窄构造**：用智能构造器保证不可能构造出非法状态
3. **封闭枚举**：域的有限可能性用穷举枚举编码
4. **参数化类型**：域的无限模式用泛型编码
5. **幽灵类型**：编译期状态标记，零运行时开销

---

## §4 域建模流程

给定任何业务领域，依次执行以下数学分析：

### Step 1: 实体 → 类型

识别域中的原子实体。每个实体是一个类型。

$$\text{Customer},\ \text{Account},\ \text{Transaction},\ \text{Product}$$

### Step 2: 属性 → 积类型

每个实体的属性构成积类型。

$$\text{Customer} = \text{Name} \times \text{Email} \times \text{RegistrationDate}$$

### Step 3: 状态 → 和类型

实体的可能状态构成和类型。

$$\text{TaskState} = \text{Pending} + \text{Running} + \text{Completed} + \text{Failed}(\text{Error})$$

### Step 4: 关系 → 函数

实体之间的关系是函数或多值映射。

$$\text{owner} : \text{Account} \to \text{Customer}$$

$$\text{tasks} : \text{Customer} \to \mathcal{P}(\text{Task})$$

### Step 5: 约束 → 类型限制

业务规则是类型约束或智能构造器。

$$\text{Quantity} \subseteq \mathbb{Q}^+ \quad \text{（正有理数，排除零和负数）}$$

### Step 6: 操作 → 函数签名

业务操作是纯函数签名。

$$\text{process} : \text{State} \times \text{Request} \to \text{State'} \times (\text{Response} + \text{Error})$$

注意纯函数返回新状态——状态转换显式编码，旧状态不被修改。

### Step 7: 不变量 → 编译器验证

将尽可能多的约束编码进类型系统，使违反约束的代码不能编译。

$$\text{NonEmptyList}(T) \ne \text{List}(T)$$

类型级别保证"非空"——不需要运行时检查。

---

## §5 验证的数学结构

### 类型即定理

类型签名是定理陈述。编译通过是定理证明。

$$f : \text{NonEmptyList}(A) \to A$$

这个签名断言："对于任何非空列表，总能取出一个元素。"编译通过意味着该定理成立。

### 测试即抽样验证

当类型系统无法完全编码某个性质时，测试是最后的手段：

$$\forall x \in A.\ P(f(x)) \quad \text{（全称命题——理想中由类型保证）}$$

$$\exists S \subseteq A.\ \forall x \in S.\ P(f(x)) \quad \text{（抽样命题——由测试验证有限子集）}$$

**目标**：最大化类型系统能证明的命题范围，最小化需要测试的命题范围。

### 测试的代数结构

$$\text{Test} = \text{Precondition} \times \text{Action} \times \text{Postcondition}$$

$$\text{Verdict} = \text{Pass}(\text{Evidence}) + \text{Fail}(\text{Evidence}) + \text{Skip}(\text{Reason})$$

$$\text{Suite} = [\text{Test}] \times \text{ExecutionPolicy}$$

$$\text{run} : \text{Suite} \to \text{Report}$$

其中 $\text{Report} = \text{Stats} \times [\text{Verdict}]$。测试本身也有代数结构，不是随意编写的脚本。

---

## §6 实践原则

### 原则 1: 先写数学模型，再写代码

$$\text{Model} \succ \text{Code}$$

在 IDE 中敲第一行之前，先在纸上或文档中写出类型的代数定义。代码是数学模型的翻译结果，翻译是机械的。

### 原则 2: 数学公式先于自然语言注释

当需要解释一段代码时，先尝试用数学公式表述。如果数学公式简洁清晰，用它替代自然语言注释。

$$\text{resolve} : \text{Intent} \to \text{Result}(\text{Target}, \text{Error})$$

这一行比"该函数接收一个意图对象，解析后返回目标或错误"更精确、更紧凑。

### 原则 3: 编译错误优于运行时错误

$$\text{compile-time error} \succ \text{runtime error} \succ \text{silent wrong behavior}$$

每一个运行时错误都意味着类型系统有未覆盖的空间。第一反应不是"加个 catch"，而是"类型系统为什么没拦截这个"。

### 原则 4: 穷举优于条件分支

穷举匹配比条件链更安全——编译器保证不遗漏变体。每当写 `if-else` 时，先问：**这是否应该是和类型上的穷举匹配？**

### 原则 5: 代数恒等式指导重构

当感觉代码冗余时，检查是否违反了代数恒等式。恒等式违反 = 结构性冗余。

$$f \cong g \wedge f \ne g \implies \text{DRY 违规——合并为参数化的 } h(T)$$

### 原则 6: 势分析指导设计

当一个类型能表达的值的数量远大于域中合法值的数量时，类型设计过于宽松：

$$\lvert\text{Type}\rvert \gg \lvert\text{Domain}\rvert \implies \text{存在非法状态可被构造}$$

目标是：

$$\lvert\text{Type}\rvert = \lvert\text{Domain}\rvert$$

---

## 参考脉络

- **类型论** (Martin-Löf, 1984)：类型即命题，程序即证明
- **代数数据类型** (ADT)：积类型 + 和类型作为域建模原语
- **Curry-Howard 对应**：逻辑与类型系统的同构
- **领域驱动设计** (Evans, 2003)：将"战术模式"替换为数学等价物
