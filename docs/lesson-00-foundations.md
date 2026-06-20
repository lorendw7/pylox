# Lesson 0 — Foundations, From Zero / 从零开始的基础

> **No code this lesson.** This is the mental foundation everything else stands on. Read it
> slowly; if the rest of the project ever feels confusing, come back here.
> 本课**不写代码**，是整个项目的思想地基。建议慢读；以后哪里发懵，就回到这一课。
>
> Assumes only basic Python. By the end you'll understand *what* an interpreter is, *why*
> it has stages, and you'll have traced a real program through all of them by hand.
> 只需要基础 Python。读完你会明白解释器*是什么*、*为什么*要分阶段，并亲手把一个真实
> 程序在脑子里跑一遍。

---

## 1. What Is a Program, Really? / 程序到底是什么？

Open any source file. It is **just text** — a long string of characters saved in a file.
`let x = 10` is not "an instruction" to the computer; it is the eight characters
`l e t   x   =   1 0`. The computer has **no idea** what they mean. It can't add, branch, or
print based on that text — not yet.

> 🇨🇳 **第一个、也是最重要的觉悟：** 源代码*只是文本*。`let x = 10` 在硬盘上就是一串字符，
> 计算机一开始**完全看不懂**。它不会因为你写了 `print` 就去打印——`print` 此刻只是 5 个字母。
> 我们这个项目的全部工作，就是搭一座桥，把这串“人能读的文本”变成“机器能做的动作”。

---

## 2. What Does "Run a Program" Even Mean? / “运行程序”到底意味着什么？

A CPU only understands **machine code** — tiny numeric instructions like "add these two
numbers," "jump to address N." Humans almost never write that directly; we write
*high-level* languages (Python, our pylox) that are far from machine code. So there is
always a **gap** to bridge between the text you wrote and the actions a machine performs.

Two classic ways to bridge that gap:

| Strategy | What it does | Real examples / 真实例子 |
| --- | --- | --- |
| **Compile** / 编译 | Translate the whole program *ahead of time* into a lower-level form (machine code or bytecode), then run that. | C, C++, Rust, Go |
| **Interpret** / 解释 | Walk through the program and *perform each action directly*, right now. | classic Python, Ruby, JavaScript |

> 🇨🇳 **类比：** 你有一本中文书想给只懂英文的朋友看。
> - **编译**＝先把整本书译成英文，再交给他读。前期慢，但他读得飞快。
> - **解释**＝你坐在他旁边，他指一句你译一句。立刻能开始，但每句都现译，慢一些。
>
> 🇨🇳 **关键事实：** 这两条路的**前半段是完全一样的**——不管最后是编译还是解释，你都得先
> “读懂”这段文本：把它切成词、理出结构。差别只在最后一步：是“翻译成更底层的指令存起来
> 以后跑”（编译），还是“现在就照着做”（解释）。pylox 的 Tier A–C 走解释这条路，Tier D
> 改走编译+虚拟机，Tier E 再加 JIT——但前半段始终共用。

---

## 3. The Pipeline, Stage by Stage / 流水线：一阶段一阶段来

Bridging the gap in one giant leap is too hard, so we split it into small, well-defined
stages. Each stage takes one data shape and produces the next:

```
   "let total = 2 + 3 * 4"        ← (1) raw text / 原始文本
            │
            │   LEXER / 词法分析器
            ▼
   [LET][IDENT total][=][2][+][3][*][4][EOF]   ← (2) a flat list of TOKENS / 词流
            │
            │   PARSER / 语法分析器
            ▼
        Assign(total, Binary(+, 2, Binary(*, 3, 4)))   ← (3) a TREE (AST) / 语法树
            │
            │   (optional) RESOLVER / 语义分析（可选）
            ▼   same tree, now checked: is `total` declared? etc.
            │
            │   INTERPRETER / 解释器
            ▼
        environment: { total: 14 }   ← (4) actions & results / 动作与结果
```

**Why each stage exists / 每一阶段为什么不能省：**

1. **Lexer (text → tokens).** Working character-by-character is painful. Grouping
   `1`,`0` into one `NUMBER(10)` token gives the next stage clean "words" to work with.
   字符太碎，先把它们归并成有类型的“词”。
2. **Parser (tokens → tree).** A flat list can't express structure. Does `2 + 3 * 4` mean
   `(2+3)*4` or `2+(3*4)`? A **tree** answers that; a list can't. 列表没有结构，树才能表达
   “谁先算、谁修饰谁”。
3. **Resolver (tree → checked tree).** *Optional, Tier C.* Catch mistakes (using an
   undeclared variable) **without running** the program. 不运行就能查出的错误，提前查。
4. **Interpreter (tree → result).** Walk the tree and actually compute, branch, print,
   change variables. 真正干活的一步。

> 🇨🇳 **一个心智模型：** 把它想成一条工厂流水线。每个工位只做一件简单的事，把半成品交给
> 下一个工位。没有哪个工位试图“一口气全做完”——复杂的大问题，被拆成了一串简单的小问题。
> 这正是软件工程里最常见、最强大的思路，你会在整个项目里反复用到它。

---

## 4. Syntax vs. Semantics — Two Kinds of "Correct" / 语法 vs. 语义：两种“对”

These two words appear everywhere; nail them now.

- **Syntax / 语法** = the *form/shape*. Are the words arranged legally? `let = 10 x` is
  syntactically wrong — the pieces are in an illegal order. The **parser** catches this.
- **Semantics / 语义** = the *meaning*. `let x = y + 1` is syntactically fine, but if `y`
  was never defined, it's *meaningless* — a semantic error. The **resolver/interpreter**
  catches this.

> 🇨🇳 **类比英文句子：**
> - `"Cat the sat mat on"` —— 单词都对，但**排列违法** → **语法**错误（parser 管）。
> - `"The cat sat on the colorless green idea"` —— 语法完全合法，但**意思不通** → **语义**
>   错误（resolver/interpreter 管）。
>
> 记住：**语法管“摆得对不对”，语义管“说得通不通”**。它们由不同阶段负责，这也是为什么要
> 分阶段。

---

## 5. What Is a "Grammar"? / 什么是“文法”？

A grammar is a set of **rules describing which token sequences are legal**, written as
*rules that refer to each other*. Here is a tiny grammar for arithmetic, in a common
notation called **BNF/EBNF**:

```
expression → term   ( ("+" | "-") term )*
term       → factor ( ("*" | "/") factor )*
factor     → NUMBER | "(" expression ")"
```

Read each arrow as "is defined as." `*` means "zero or more times." Notice `expression` is
defined using `term`, `term` using `factor`, and `factor` can loop back to `expression`
inside parentheses — the rules are **recursive**.

**This is where precedence comes from.** Because `term` (which handles `*` `/`) sits
*below* `expression` (which handles `+` `-`), multiplication is forced to group more
tightly than addition — automatically, just from how the rules nest.

> 🇨🇳 **文法就是“合法句子的拼装说明书”。** 上面三条规则规定了：一个 `expression` 是若干
> 个 `term` 用 `+`/`-` 连起来；一个 `term` 是若干个 `factor` 用 `*`/`/` 连起来；一个
> `factor` 要么是个数字，要么是括号里再套一个 `expression`。
>
> 🇨🇳 **优先级是“免费”得到的：** 因为 `term`（管乘除）被嵌在 `expression`（管加减）的
> *更里层*，所以乘除天然比加减“抓得更紧”。你不用写任何“先乘后加”的特判——它直接从规则的
> **嵌套结构**里冒出来。第 3 课写 parser 时，你会发现**每条文法规则正好对应一个函数**，
> 这种技术叫“递归下降”。现在不用全懂，留个印象就好。

---

## 6. The Data You'll Build / 你会亲手造的三种数据

Three structures carry the program through the pipeline. Meet them now; build them later.

| Structure | 中文 | What it is | Built in |
| --- | --- | --- | --- |
| **Token** | 词法单元 | A small object: a type + the source text + a value + a line number. | Lesson 1 |
| **AST node** | 语法树节点 | An object representing one piece of structure (e.g. `Binary(+, left, right)`). Nodes point to child nodes → a tree. | Lesson 2 |
| **Environment** | 环境 | A dictionary mapping variable names to their current values, e.g. `{ total: 14 }`. | Lesson 4 |

> 🇨🇳 **一句话记住数据流：** **字符 → Token（词）→ AST（树）→ 用 Environment（变量表）边走树边求值 → 结果**。整条流水线就是这串数据在不停变形。

---

## 7. Trace One Program by Hand / 亲手追踪一个程序

This is the payoff. Let's run `let total = 2 + 3 * 4` through the whole pipeline **on paper**,
the way your code eventually will. The answer should be `14` (because `*` binds tighter
than `+`): `3 * 4 = 12`, then `2 + 12 = 14`.

**Stage 1 — Lexer: text → tokens.** Scan left to right, group characters into words:
```
LET  IDENTIFIER("total")  ASSIGN  NUMBER(2)  PLUS  NUMBER(3)  STAR  NUMBER(4)  EOF
```

**Stage 2 — Parser: tokens → AST.** Apply the grammar from §5. `*` groups tighter, so the
tree looks like this (read it as "assign to `total` the value of (2 + (3*4))"):
```
        Assign
        ├── name:  total
        └── value: Binary( + )
                   ├── left:  Number(2)
                   └── right: Binary( * )
                              ├── left:  Number(3)
                              └── right: Number(4)
```

**Stage 3 — Interpreter: walk the tree, bottom-up.** To get a node's value, first get its
children's values:
```
Number(3) → 3        Number(4) → 4
Binary(*) → 3 * 4 = 12
Number(2) → 2
Binary(+) → 2 + 12 = 14
Assign    → store total = 14 in the environment
```
Final state: `environment = { total: 14 }`. ✅

> 🇨🇳 **盯着第 2、3 阶段看，这是全项目的灵魂：**
> - **结构决定计算顺序。** 因为 `*` 在树里更靠下（更深），所以求值时它**先被算到**。树的
>   形状*就是*运算顺序——这就是为什么我们费力气建树，而不是用一个扁平列表。
> - **“边走树边算”就是解释器。** 求一个节点的值，先求它孩子的值，再合并——这是一个**递归**
>   过程。整个 Tier A 的解释器，本质就是这个“后序遍历树并求值”的递归。
>
> 🇨🇳 **建议你现在就做：** 拿张纸，把 `(2 + 3) * 4` 也画一遍树，看看树的形状怎么变、答案
> 怎么从 14 变成 20。亲手画两遍，你对“AST 为什么重要”的理解会立刻扎实。

---

## 8. The One Skill You Must Be Comfortable With: Recursion / 必须吃透的一项技能：递归

Notice §5 (grammar refers to itself) and §7 (value of a node needs values of its children)
both **loop back on themselves**. That pattern is **recursion**, and it is the heartbeat of
this whole project — the parser, the tree, and the interpreter are all recursive.

A function is recursive when it calls itself on a smaller piece of the problem, with a
**base case** that stops the recursion. Walking a tree is the classic example: "to process
a node, process each child (recursion), then combine."

> 🇨🇳 **为什么强调递归：** 树是递归结构（节点里装着节点），文法是递归定义（规则引用规则），
> 所以处理它们最自然的工具就是**递归函数**——“处理一个节点 = 先递归处理它的孩子，再合并
> 结果”，遇到叶子（数字、字面量）就是**递归出口**。如果你对递归还不太有把握，先用纯 Python
> 写几个小练习找找感觉：
> - 计算列表所有数字之和（递归版）；
> - 计算阶乘 `n!`；
> - 给一棵用嵌套字典/列表表示的树，递归求所有叶子之和。
>
> 这三个练习的“形状”，和你后面写 parser、interpreter 的形状**一模一样**。先在简单问题上
> 把递归练顺，正式写解释器时会轻松非常多。

---

## 9. How This Maps to pylox / 这些概念对应到 pylox 的文件

| Concept (this lesson) | File you'll build | Lesson |
| --- | --- | --- |
| text → tokens | `pylox/tokens.py`, `pylox/lexer.py` | 1 |
| tokens → AST | `pylox/ast_nodes.py`, `pylox/parser.py` | 2–3 |
| static checks | `pylox/resolver.py` | 10 |
| walk tree → result | `pylox/interpreter.py` | 4 |
| run files / REPL | `main.py`, `pylox/repl.py` | 5 |

---

## 10. Mental Models to Carry / 随身携带的几个心智模型

1. **Source code is just text; the computer starts out understanding none of it.** 源码只是
   文本，计算机起初什么都不懂。
2. **One hard translation is split into a pipeline of easy stages.** 一次困难的翻译，被拆成
   一串简单的流水线工位。
3. **Compile vs. interpret differ only at the last step; the front-end is shared.** 编译与
   解释只在最后一步分道扬镳，前端共用。
4. **Structure (the tree), not order-of-text, decides what computes first.** 决定运算顺序的
   是树的结构，不是文本的先后。
5. **Recursion is the tool that matches the shape of grammars and trees.** 递归是与文法、树
   的形状天然契合的工具。

---

## 11. Self-Check / 自测

You're ready for Lesson 1 if you can answer these in your own words. 能用自己的话答出来，就可
以开始第 1 课了：

1. Why is `let x = 10` not directly runnable by the CPU? / 为什么 CPU 不能直接跑 `let x = 10`？
2. Name the pipeline stages and what data each consumes and produces. / 说出每个阶段的输入和输出。
3. Give one syntax error and one semantic error in pylox. / 各举一个语法错误和语义错误的例子。
4. Why must `2 + 3 * 4` become a *tree*, and why does that tree compute `*` first? / 为什么
   `2 + 3 * 4` 要变成树？为什么这棵树会先算 `*`？
5. Where does recursion show up in this project, and why is it the natural tool? / 递归在本项目
   哪里出现？为什么它是自然的工具？

> 🇨🇳 **下一课（Lesson 1 — Lexer）：** 你现在懂了“为什么”和“整条流水线长什么样”。第 1 课
> 就动手造流水线的第一个工位——把字符切成 Token。去读 [Lesson 1 讲义](lesson-01-lexer.md)。
