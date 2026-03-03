# 编程范式文档

> Programming Paradigm Documents
>
> 从工程实践中提炼的通用编程思想——与领域无关，与语言无关，与项目无关。

---

## 阅读指南

每篇文档是一个**自包含闭包**：覆盖一个完整主题，可独立阅读，无需依赖其他文档。

文档之间是**正交**的：每篇回答一个独立的核心问题，不重叠。

## 文档索引

| 文件 | 核心问题 | 一句话 |
|------|----------|--------|
| [math-first-modeling.md](math-first-modeling.md) | 为什么先建模再编码？ | 程序是数学模型的可执行翻译，绕过建模的代码是有损映射 |
| [type-algebra.md](type-algebra.md) | 用什么数学工具建模？ | 积类型、和类型、函数类型构成完备的域建模原语 |
| [three-tier-specificity.md](three-tier-specificity.md) | 怎么分层？ | ∀ 通用结构 / ∃! 静态特异 / λ 动态输入——复杂性守恒 |
| [hollywood-principle.md](hollywood-principle.md) | 框架与用户的边界怎么画？ | 框架引导用户，而非用户学习框架——编译器就是文档 |
| [refactoring-philosophy.md](refactoring-philosophy.md) | 什么时候打破并重建？ | 重构是手术不是修补——数学拓扑优先于编译器妥协 |
| [formal-diagnostics.md](formal-diagnostics.md) | 怎么系统地审计代码质量？ | 用形式化诊断谓词替代直觉——让审计可复现 |
| [category-theory-intuition.md](category-theory-intuition.md) | 更高阶的抽象推理工具？ | 范畴论提供"最优抽象"的判定标准——不多不少 || [capability-oriented-programming.md](capability-oriented-programming.md) | 能力和身份是什么关系？ | 能力的组合决定身份——名字只是能力集合的简称 |
| [programming-as-algebra.md](programming-as-algebra.md) | 编程的尽头是什么？ | 程序建模是发现领域中已存在的代数结构——知识地图与搜索关键词 |
## 设计原则

- **领域无关**：不涉及任何具体业务领域（金融、游戏、Web……）
- **语言无关**：原理适用于任何具备类型系统的语言，示例偶尔使用伪代码或 Rust 语法
- **正交覆盖**：每一篇恰好覆盖一个主题，全部文档的并集覆盖"数学优先编程"的核心知识空间
- **数学严格**：使用数学符号精确表达，消除自然语言歧义

## 许可

本仓库内容以 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 许可发布。
