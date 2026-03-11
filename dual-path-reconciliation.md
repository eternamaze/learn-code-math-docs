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

### 双路

到达范式（NormalForm）有两条路径：

1. **直接看见**：主端报告的 Effect 本身就是范式
2. **推理看见**：从 Cause 经 derive 归约出范式

对账 = `reconcile(推理出的范式, 直接看见的范式)`。

### derive

$$\text{derive} : \text{State} \to \text{Cause} \to \text{Option}\ (\text{List Effect})$$

$$E = \text{Cause'} \mid \text{NormalForm}$$

- derive 是唯一的实质性操作
- 一个 Cause 可产出多个独立 Effect——**因果分叉**
- 每个 Effect：Cause' → 继续归约，NormalForm → 终止

### 因果树

归约结构是**树**，线性链是退化特例：

```
         Cause₀
        /      \
    Cause₁    NF₁
    /    \
  NF₂    NF₃
```

根 = 初始 Cause，内部节点 = 中间 Cause，叶子 = NormalForm。
每个节点双路 reconcile。

### 线性消费

**Cause 被 derive 独占消费（线性的）**。

若两个 derive 似乎需要同一个 Cause，则必然存在一个先行 derive 将该 Cause 分叉为正交的 Effect 列表——每个后续 derive 消费的是分叉后的独立 Cause，而非原始 Cause。

"两个 derive 消费同一个 Cause"是幻觉。因果树中每条边恰好被走一次。

### State = 最大范式，递归容器

State 本身就是范式——覆盖最大记忆的范式。

因果分叉直接解释了片段的存在：树有多个叶子，每个叶子写入 State 的不同区域。**分片是分叉的本体论后果**。

State 是递归容器：

- State 的任意投影是 sub-State，且是 NormalForm
- NormalForm 在不相交并上封闭——子片段合并仍是范式
- overlay = 叶子范式写入最大范式的对应位置——静态数据的唯一可能操作

overlay 不是独立公理。

---

## 定理

**引理（线性消费）**：每个 Cause 恰好被一个 derive 消费一次。共享 Cause 的需求由先行分叉消解。

**引理（分叉）**：derive 产出 `List Effect`，每个 Effect 独立进入自己的子因果树。

**定理（片段覆盖）**：因果树的叶子集恰好覆盖 State 中受根 Cause 影响的子区域。

**定理（范式封闭）**：若 $\text{NF}_1, \text{NF}_2$ 作用于不相交区域，则 $\text{NF}_1 \cup \text{NF}_2$ 仍是 NormalForm。

**推论（递归容器）**：State 是 NormalForm；State 的任意投影是 sub-State 且是 NormalForm。

---

## 性质

| 性质 | 说明 |
|------|------|
| 分离性 | 法则一独立；法则二引用法则一，反之不成立 |
| 降级性 | derive 返回 none → 法则一自然回退到直接路径 |
| 线性性 | Cause 被独占消费，共享需求由先行分叉消解 |
| 分叉性 | derive 产出 List Effect，因果结构是树 |
| 片段性 | 叶子 = NormalForm 片段，分片是分叉的本体论后果 |
| 递归性 | State 是 NormalForm；投影是 sub-State 且是 NormalForm |
| 封闭性 | NormalForm 在不相交并上封闭 |

## 适用条件

1. 主端消息可分解为 (因, 果) 对
2. 从端能对因实现至少部分的本地 derive
3. 从端接受"主端是权威"的信任前提

不适用于对等架构——不存在单一权威路径。
