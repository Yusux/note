---
date: true
comment: true
totalwd: true
---

# 语法分析

## 上下文无关法

### CFG 简介

基本作用

!!! warning "Parse Tree 不是 AST"

手写原因如语法太过复杂，手写反而方便一些（GCC 3.4 later）

!!! info "CFG 定义的方式和计算理论所讲的形式略不同，符号需要注意一下"

### 推导和规约

??? question "给定 CFG，如何判定输入串属于文法规定的语言？"

- 直接推导/规约
- 多步推导/规约
- 最左推导/规约

句型、句子、语言

??? question "上下文无关是什么意思？"
    $\alpha A \beta \Rightarrow \alpha \gamma \beta$
    在文法推导的每一步，符号串 $\gamma$ 仅根据 $A$ 的产生式推导，无需依赖 $A$ 的上下文 $\alpha$ $\beta$

### RE 和 CFG

正则文法（Regular Grammar）

??? lab "形式文法的分类"

??? lab "CFL-Reachability"