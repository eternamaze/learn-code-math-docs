# 双路因果对账

> Dual-Path Causal Reconciliation

## 问题

主从状态机（leader-follower state machine）中，从端如何维护一份**可验证**的本地状态？

盲目信任→失去验证能力。盲目计算→漂移不可检测。定期全量同步→间隔内不可信。

共同缺陷：没有在每一步同时利用两条信息通路。

## 观察

主端消息天然可分解为 **(因, 果)** 对：

- **因**（Cause）：导致变更的事件
- **果**（Effect）：变更后的权威状态片段

因 = 推理路径的输入，果 = 直接路径的输出。

---

## 法则一：对账公理

独立于状态机的纯比较判定：

$$\text{reconcile} : \text{Option}\ T \to \text{Option}\ T \to \text{Reconciled}\ T$$

| 推理路径 | 直接路径 | 行为 |
|----------|----------|------|
| some A | some B | A = B → 接受；否则冲突 |
| some A | none | 接受 A |
| none | some B | 接受 B |
| none | none | 不动 |

无状态，仅需 $T$ 上的相等性。降级是四格矩阵的自然分支。

---

## 法则二：因果律

法则一只做比较。产出被比较的值是法则二的职责。

### 动与静

因果律的基底是二元分类——**动**（Motion）与**静**（Stasis）：

- **动**（Cause）：内禀携带位移描述的信息。动是因——它驱动变化。
- **静**（State）：不内禀携带位移的信息。静是范式——它是变化的结果。

动与静是**内禀**属性，不取决于观察者或消费者的存在。一个 Flow（资金流动）即便没有钱包去接收它，它仍然是动的——因为它描述了"从 A 到 B 转移 x 单位"这一位移。一个 Good（持有物）即便可以被未来的 derive 消费，它仍然是静的——因为它描述的是"现在有 x 单位"这一存在状态。

### Effect = 静 | 动

$$\text{Effect}\ S\ C = \text{stasis}(S) \mid \text{motion}(C)$$

Effect 是因果律的核心类型——derive 的输出。每个效应声明自己的性质：

- $\text{stasis}(s)$：到达范式——静信息，由框架写入 State
- $\text{motion}(c)$：尚未归约——动信息，由框架递归 derive

### 双路

到达范式有两条路径：

1. **直接看见**：主端报告的范式片段直接写入 State
2. **推理看见**：从 Cause 经 derive 归约，直到所有分支到达 stasis

对账 = `reconcile(推理出的范式, 直接看见的范式)`。

### derive

$$\text{derive} : \text{Cause} \to \text{State} \to \text{Option}\ (\text{List Effect})$$

因→状态→效应。因是主角（第一参数），状态是被作用的对象。

- derive 是唯一的实质性操作
- 一个 Cause 可产出多个独立 Effect——**因果分叉**
- 每个 Effect：motion(Cause') → 继续归约，stasis(State') → 终止

### State = 范式存储空间

$$\text{State} + \text{State} = \text{State}$$

State 是范式——覆盖最大记忆的静信息空间。State 在组合下封闭：状态 + 状态 = 状态，只不过更大了。

State 的子空间投影（Patch）也是范式——概念上 Patch 就是 State，只是粒度不同。overlay 是将子范式写入最大范式：

$$\text{overlay} : \text{State} \to \text{Patch} \to \text{State}$$

overlay 是静态数据的唯一可能操作——把范式片段合并到范式中。

### 因果树

归约结构是**树**，线性链是退化特例：

```
         Cause₀
        /      \
   motion₁   stasis₁
    /    \
stasis₂  stasis₃
```

根 = 初始 Cause，内部节点 = 中间 Cause（motion），叶子 = 范式片段（stasis）。

框架递归归约：从根出发，对每个 motion 调用 derive 展开子树，对每个 stasis 调用 overlay 写入 State。直到所有分支到达 stasis——即因果树的全部叶子写入 State。

### 编译期强制

$$\text{derive} : \text{Cause} \to \text{State} \to \text{Option}\ (\text{List}\ (\text{Effect}\ \text{Patch}\ \text{Cause}))$$

derive 的类型签名不含 State 输出——实现者**不能直接产出 State**。

所有状态变化必须经过 Effect.stasis(Patch) → 由框架 overlay。
所有子因必须经过 Effect.motion(Cause) → 由框架递归 derive。

框架拥有因果树的完整处理权。实现者只能声明"发生了什么分支"，不能绕过框架偷偷归约。试图在 derive 内部隐藏归约步骤（如将资金流动直接作用于钱包而不声明为 motion）的代码无法通过编译——类型签名禁止。

### 线性消费

**Cause 被 derive 独占消费（线性的）**。

若两个 derive 似乎需要同一个 Cause，则必然存在一个先行 derive 将该 Cause 分叉为正交的 Effect 列表——每个后续 derive 消费的是分叉后的独立 Cause，而非原始 Cause。

"两个 derive 消费同一个 Cause"是幻觉。因果树中每条边恰好被走一次。

---

## 定理

**引理（线性消费）**：每个 Cause 恰好被一个 derive 消费一次。共享 Cause 的需求由先行分叉消解。

**引理（分叉）**：derive 产出 `List Effect`，每个 Effect 独立进入自己的子因果树。

**定理（片段覆盖）**：因果树的叶子集（全部 stasis）恰好覆盖 State 中受根 Cause 影响的子区域。

**定理（范式封闭）**：若 $\text{NF}_1, \text{NF}_2$ 作用于不相交区域，则 $\text{NF}_1 \cup \text{NF}_2$ 仍是 NormalForm。State + State = State。

**推论（递归容器）**：State 是 NormalForm；State 的任意投影是 sub-State 且是 NormalForm。

**定理（因果显性化）**：Space 协议中，所有状态变化必须且只能经过 Effect → overlay 路径。derive 的类型签名是该定理的编译期证明。

---

## 性质

| 性质 | 说明 |
|------|------|
| 二元性 | 动（Cause）与静（State）是内禀属性，不取决于消费者 |
| 分离性 | 法则一独立；法则二引用法则一，反之不成立 |
| 降级性 | derive 返回 none → 法则一自然回退到直接路径 |
| 线性性 | Cause 被独占消费，共享需求由先行分叉消解 |
| 分叉性 | derive 产出 List Effect，因果结构是树 |
| 片段性 | 叶子 = stasis(Patch)，分片是分叉的本体论后果 |
| 封闭性 | State + State = State；NormalForm 在不相交并上封闭 |
| 强制性 | derive 的类型签名禁止产出 State，框架拥有归约控制权 |

## 适用条件

1. 主端消息可分解为 (因, 果) 对
2. 从端能对因实现至少部分的本地 derive
3. 从端接受"主端是权威"的信任前提

不适用于对等架构——不存在单一权威路径。
