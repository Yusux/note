---
date: true
comment: true
totalwd: true
---

# 词法分析

## 概述

程序是以**字符串**的形式传递给编译器的，词法分析的作用在于：

- 将输入的字符串识别为**有意义的字串**
- 过滤注释、空格
- ...

基于此，有两个基本概念：

- Token
  - 类似于英语中的 noun, verb, adjective, ...
  - 在编程语言中 keyword, identifier, operator, ...
- Lexeme
  - 是 Token 集合中的一个元素，类似于 `if` `else`
  - 可以称为 Token 的一个实例

??? example "示例：Token 与 Lexeme"
    对于如下的程序：
    ```c
    if (i == j) {
        printf("i equals j");
    } else {
        num = 5;
    }
    ```
    我们从中看出
    
    |Token|Lexeme|Token 的非形式化定义|
    |:-:|:-:|:-:|
    |if|if|字符 i, f|
    |else|else|字符 e, l, s, e|
    |relation|<, <=, =, ...|< 或 <= 或 = 或 ...|
    |id|sum, count, D5|由字母开头的字母数字串|
    |number|5, 3.1, 2.8e12|任何数值常熟|

??? example "示例：字符流 -> Token 流"
    对于如下的程序：
    ```c
    float match0(char *s) /* find a zero */
    {
        if (!strncmp(s, "0.0", 3))
        return 0.;
    }
    ```
    其中的 Token 流为：
    ```
    FLOAT       ID(match0) LPAREN CHAR   STAR ID(s)
    RPAREN      LBRACE     IF     LPAREN BANG
    ID(strncmp) LPAREN     ID(s)  COMMA  STRING(0.0)
    COMMA       NUM(3)     RPAREN RPAREN RETURN
    REAL(0.0)   SEMI       RBRACE EOF
    ```

对于词法分析器的构造，可以通过定义声明式的规范，然后使用工具自动生成。

![词法分析器构造](../../assets/img/docs/CS/Compilers/ch2/image.png)

可以看作：
```
Program = Specification + Implementation
              "What"          "How"
```

## 正则表达式

??? question "如何形式化地描述词法分析器的规范？"
    这就引入了正则表达式的概念。

### 前置知识

#### 字母表和串

字母表（alphabet）是**符号**的**有限集合**

- 包括字母、数字、标点符号等

串（string, word）是字母表中**符号**的**有限序列**

- 串 $s$ 的**长度**，通常记作 $|s|$，指代 $s$ 中符号的个数
- **空串**是长度为 0 的串，通常记作 $\epsilon$

有如下的运算定义：

- 连接（concatenation）
    - $y$ 附加到 $x$ 后形成的串 $xy$
    - 例如，$x = abc$, $y = def$，则 $xy = abcdef$
    - 空串是连接运算的单位元，即对于任何串 $s$ 都有 $εs = sε = s$
- 幂（power）
    - $s^n$ 是 $n$ 个 $s$ 的连接
    - $s^n=\left\{\begin{matrix} &ε &, n=0 \\ &s^{n-1}s &, n \ge 1 \end{matrix}\right.$
    - 例如，$s = abc$，则 $s^3 = abcabcabc$

#### 语言

语言（language）：字母表 $\Sigma$ 上的一个串集

- 例如，$\{\epsilon,0,00,000,...\}$, $\{s\}$, $\emptyset$
- 属于语言的串，可以称为句子（sentence）

语言的运算：

- 并（union）
    - $L_1 \cup L_2 = \{s | s \in L_1 \text{ or } s \in L_2\}$
- 连接（concatenation）
    - $L_1L_2 = \{xy | x \in L_1 \text{ and } y \in L_2\}$
- 幂（power）
    - $L^n = \left\{\begin{matrix} &\{ε\} &, n=0 \\ &L^{n-1}L &, n \ge 1 \end{matrix}\right.$
- Kleene 闭包（Kleene closure）
    - $L^* = \bigcup_{i=0}^{\infty}L^i$
- 正闭包（positive closure）
    - $L^+ = \bigcup_{i=1}^{\infty}L^i$

### 正则表达式的定义
正则表达式（Regular Expression, RE）是用来描述、匹配文中全部匹配指定格式串的表达式，如 $a$ 匹配 $a$，$a|b$ 匹配 $a$ 或 $b$

正则表达式 $r$ 定义正则语言，记为 $L(r)$，有如下性质：

- $\epsilon$ 是一个 RE，$L(\epsilon) = {\epsilon}$
- 如果 $a \in \Sigma$，则 $a$ 是一个 RE，$L(a) = \{a\}$
- 假设 $x$ 和 $y$ 都是 RE，分别表示语言 $L(x)$ 和 $L(y)$
    - $x|y$ 是一个 RE，$L(x|y) = L(x) \cup L(y)$
    - $xy$ 是一个 RE，$L(xy) = L(x)L(y)$
    - $x^*$ 是一个 RE，$L(x^*) = (L(x))^*$
    - $(x)$ 是一个 RE，$L((x)) = L(x)$
- 优先级：$() > * > xy > |$

### 正则表达式的定律

一些正则表达式的定律如下：

|定律|描述|
|:-:|:-:|
|$r\text{\textbar}s = s\text{\textbar}r$|$\text{\textbar}$ 是可交换的|
|$(r\text{\textbar}s)\text{\textbar}t = r\text{\textbar}(s\text{\textbar}t)$|$\text{\textbar}$ 是可结合的|
|$r(st) = (rs)t$|连接是可结合的|
|$r(s\text{\textbar}t) = rs\text{\textbar}rt$<br>$(s\text{\textbar}t)r = sr\text{\textbar}tr$|连接对 $\text{\textbar}$ 是可分配的|
|$\epsilon r = r\epsilon = r$|$\epsilon$ 是连接的单位元|
|$r^* = (r \text{\textbar} \epsilon)^*$|闭包中一定包含 $\epsilon$|
|$(r^*)^* = r^*$|闭包的闭包等于闭包|

### 正则定义

!!! warning "区分于正则表达式的定义"

对于比较复杂的语言，为了构造简洁的正则式，可先构造简单的正则式，再将这些正则式组合起来，形成一个与该语言匹配的正则序列

正则定义是具有如下形式的定义序列：

$$
\begin{gather*}
d_1 \rightarrow r_1 \\
d_2 \rightarrow r_2 \\
\vdots \\
d_n \rightarrow r_n
\end{gather*}
$$

其中：

- 每个 $d_i$ 的名字都不相同
- 每个 $r_i$ 都是 $\Sigma = \cup \{d_1, d_2, ..., d_{i-1}\}$ 上的正则式

??? exmaple "示例：C 语言的标识符"
    对于如下的正则定义：

    $$
    \begin{align*}
    &digit &\rightarrow \quad &0|1|2|\ldots|9 \\
    &letter\_ &\rightarrow \quad &a|b|c|\ldots|z|A|B|C|\ldots|Z|\_ \\
    &id &\rightarrow \quad &letter\_(letter\_|digit)^*
    \end{align*}
    $$
    
    其中，$digit$ 表示数字，$letter\_$ 表示字母和下划线，$id$ 表示标识符。也可以简写为：

    $$
    \begin{align*}
    &digit &\rightarrow \quad &[0\text{-}9] \\
    &letter\_ &\rightarrow \quad &[a\text{-}zA\text{-}Z\_] \\
    &id &\rightarrow \quad &letter\_(letter\_|digit)^*
    \end{align*}
    $$

### 词法分析的规约

正则表达式是词法分析的规约（Specification），是字符流到 Token-lexeme 对的过程，大体上可以分为

1. 选择一系列 tokens

   - 如 Number, Identifier, Keyword, ...

2. 为每个 token 的 lexemes 定义一个正则表达式

   - Number: $digit^+$
   - Keyword: $'if'|'else'|\ldots$
   - Identifier: $letter\_(letter\_|digit)^*$
   - LeftPar: $'('$

### 正则规则的二义性

!!! question "给定 if8，这是单个 Indentifier 还是两个 Token：if 和 8？"

对于这种情况，可以通过如下的规则解决：

- 最长匹配（Longest match）
    - 输入可以匹配任何正则表达式的中，最长初始子字符串将被视为下一个标记
- 规则优先（Rule priority）
    - 对于特定的最长初始子串，第一个可以匹配的正则表达式确定其标记类型
    - 这意味着书写正则表达式的顺序很重要

## 有穷自动机

!!! tips "计算理论"
    很多内容和计算理论很像，但是因为~~懒~~不想整理了，可以参考 Tony 老师的[相关笔记](https://note.tonycrane.cc/cs/tcs/toc/topic1/#_4)

### 定义

!!! quote "Wikipedia"
    A finite-state machine (FSM) or finite-state automaton (FSA, plural: automata), finite automaton, or simply a state machine, is a mathematical model of computation. It is an abstract machine that can be in exactly one of a finite number of states at any given time.

对于一个有穷自动机：$M = (S, \Sigma, move, s_0, F)$

- $S$：有穷状态集
- $\Sigma$：输入字母表（符号集合）
- $move(s, a)$：状态转移函数，$s \in S, a \in \Sigma$，表示从状态 $s$ 读入输入 $a$ 后的下一个状态
- $s_0$：初始状态（开始状态），$s_0 \in S$
- $F$：接受状态（终止状态）集合，$F \subseteq S$

此外，还有一种特殊的状态转换方式 $\epsilon$-moves，指 FA 不读入任何输入，而从一个状态转移到另一个状态

### 表示方式

#### 转换图

在转换图中，有如下的基本元素：


|元素|示例|
|:-:|:-:|
|状态|\tikzcd-automata
    \node[state] (s) {s};|
|初始状态（开始状态）|\tikzcd-automata
    \node[state, initial] (s) {s};|
|接受状态（终止状态）|\tikzcd-automata
    \node[state, accepting] (s) {s};|
|状态转移|\tikzcd-automata
    \node[state] (s0) {s_0};
    \node[state, right of=s0] (s1) {s_1};
    \draw (s0) edge[above] node{a} (s1);|


对于 $\epsilon$-moves，可以用如下的方式表示：

\tikzcd-automata
    \node[state] (0) {0};
    \node[state, right of=0] (1) {1};
    \draw (0) edge[above] node{\epsilon} (1);

#### 转换表

以如下的 FA 为例：

\tikzcd-automata
    \node[state, initial] (0) {0};
    \node[state, right of=0] (1) {1};
    \node[state, right of=1] (2) {2};
    \node[state, accepting, right of=2] (3) {3};
    \draw (0) edge[loop above] node{a} (0)
          (0) edge[loop below] node{b} (0)
          (0) edge[above] node{a} (1)
          (1) edge[above] node{b} (2)
          (2) edge[above] node{b} (3);

对应的转换表为：

|状态\输入|$a$|$b$|
|:-:|:-:|:-:|
|$0$|$\{0, 1\}$|$\{0\}$|
|$1$|$\emptyset$|$\{2\}$|
|$2$|$\emptyset$|$\{3\}$|
|$3$|$\emptyset$|$\emptyset$|

### 接收

#### 有穷自动机接收的串

给定输入串 $x$，如果存在一个对应于串 $x$ 的从**初始状态**到**某个终止状态**的转换序列，则称**串 $x$ 被该 FA 接收**

例如对于上面的 FA，串 $ababb$ 被接收，因为存在如下的转换序列：

$$
0 \xrightarrow{a} 0 \xrightarrow{b} 0 \xrightarrow{a} 1 \xrightarrow{b} 2 \xrightarrow{b} 3
$$

#### 有穷自动机接收的语言

由一个有穷自动机 $M$ 接收的所有串构成的集合，称为**该 FA 接收（或定义）的语言**，记为 $L(M)$

同样以上面的 FA 为例，其接收的语言为：

$$
L(M) = \text{所有以 } abb \text{ 结尾的字母表 } {a, b} \text{ 上的串}
$$

### 分类

!!! danger "TODO"

## 词法分析器自动生成

!!! danger "TODO"

## Lex 工具

!!! danger "TODO"