# pylox — Build Your Own Programming Language in Pure Python

> **pylox** is a tiny scripting language *and* a step-by-step tutorial for building
> its interpreter from scratch — in pure Python, with **zero third-party dependencies**.
>
> This is not a library you install. It is a course you *write*. By the end you will
> have hand-built every stage of a real language, and you will understand how the
> code you type every day is actually read and run by a computer.

---

## 0. Who This Is For / 适合谁

This tutorial assumes **no compiler-theory background**. If you can write basic Python
(functions, classes, `if`/`while`, lists, dictionaries), you have everything you need.

> 🇨🇳 **中文：** 你不需要学过《编译原理》。只要你会写基础的 Python——函数、类、
> `if`/`while`、列表、字典——就足够了。这门教程会从“一个程序到底是怎么被计算机
> 读懂并执行的”这个最根本的问题讲起，带你亲手把每一层都实现一遍。
>
> 🇬🇧 **English:** You don't need a CS degree. We start from the most fundamental
> question — *how does a computer actually read and run a program?* — and you build
> every layer by hand.

**The one rule of this project:** *you* write every line of code. The documentation
explains the **concepts, vocabulary, and design**; the keyboard is yours. Understanding
comes from typing it, breaking it, and fixing it yourself.

> 🇨🇳 **本项目唯一的规则：** 每一行代码都由你自己写。文档负责讲清楚**概念、术语和
> 设计思路**，但键盘是你的。真正的理解来自亲手敲、亲手写崩、再亲手修好。

---

## 1. The Big Picture / 全局图景

Every language tool — Python, Java, your browser's JavaScript engine — solves the same
problem: turning a **string of text** that humans can read into **actions** a machine can
perform. pylox solves it in the three classic stages used by virtually every real
interpreter and compiler:

```
   "let x = 10 + 2"          ← source code (just a string of characters)
          │
          ▼   ① LEXING / 词法分析
   [let] [x] [=] [10] [+] [2]   ← a flat list of TOKENS ("words")
          │
          ▼   ② PARSING / 语法分析
        Assign(                  ← a TREE that captures structure & precedence
          name = x,                (the Abstract Syntax Tree, "AST")
          value = Binary(+, 10, 2))
          │
          ▼   ③ INTERPRETING / 求值执行
   environment: { x: 12 }        ← walk the tree, compute, change program state
```

> 🇨🇳 **核心直觉：** 源代码只是一串字符，计算机一开始“看不懂”。我们分三步把它
> “消化”掉：
> 1. **词法分析（Lexing）**——把字符流切成一个个有意义的“单词”（Token）。就像
>    把 `"letx=10"` 这堆字符识别成 `let`、`x`、`=`、`10`。
> 2. **语法分析（Parsing）**——按照语法规则，把这串 Token 组装成一棵**树**（AST），
>    这棵树记录了“谁修饰谁”“先算哪个”等结构信息。
> 3. **求值执行（Interpreting）**——遍历这棵树，真正地做加法、改变量、控制流程。
>
> 🇬🇧 **Why a tree, not a list?** A flat list of tokens has no notion of grouping or
> precedence. `2 + 3 * 4` and `(2 + 3) * 4` are different *trees* even though they share
> tokens. The tree is where meaning lives. / 树才能表达“先算谁”这种结构，列表不行。

> 🇨🇳 **进阶补充（第 10 课）：** 很多真实编译器在“语法分析 ②”和“执行 ③”之间还会
> 插入**第四步——语义分析（Semantic Analysis）**：在*不运行程序*的前提下做静态检查，
> 比如“这个变量用之前声明过吗？”“`return` 是不是写在了函数外面？”。我们会在 Tier C
> 用一个 `resolver.py` 亲手实现这一步。
>
> 🇬🇧 Real compilers often insert a **4th pass — semantic analysis** — between parsing and
> execution: static checks done *without running* the program. We build one in Tier C.

### Interpreter vs. Compiler / 解释器 vs. 编译器

Both share stages ① and ②. They differ at stage ③:

| | What stage ③ does | Analogy / 类比 |
| --- | --- | --- |
| **Interpreter** (Tiers A–C) | *Walks* the AST and performs each action immediately | A live translator speaking as you talk / 同声传译 |
| **Compiler** (Tier D) | *Translates* the AST into lower-level instructions (bytecode) to be run later by a virtual machine | A translator who writes a full translated book first / 先把整本书译好 |

> 🇨🇳 **区别：** 解释器一边走树一边干活，写起来直观但慢；编译器先把树翻译成一种
> 更接近机器、更紧凑的“字节码”，再交给虚拟机（VM）高速执行。Tier D 我们会亲手
> 完成这次“从解释器进化成编译器+虚拟机”的飞跃，并对比两者的性能。

> 🇨🇳 **JIT 是两者的融合（Tier E，收尾巅峰）：** 即时编译器先像解释器一样执行，
> 同时统计哪段代码是“热点”，再像编译器一样把热点**在运行时**编译成更快的形式。
> 这正是 V8、PyPy、JVM HotSpot 又快又灵活的秘诀。
>
> 🇬🇧 **JIT blends both (Tier E):** it interprets first, watches for *hot* code, then
> compiles those paths *at runtime* — the trick behind V8, PyPy, and the JVM.

---

## 2. What pylox Looks Like / pylox 长什么样

```lox
# This is a comment
let x = 10
let y = 20
let sum = x + y
print(sum)            # prints 30

if sum > 25 {
    print("big")
} else {
    print("small")
}

let i = 0
while i < 5 {
    print(i)
    i = i + 1
}
```

By the final tier this same language will also support functions, arrays, dictionaries,
closures, and friendly error messages — all of which **you** will have implemented.

---

## 3. Core Vocabulary / 术语表

These words appear constantly in compiler texts. Learn them once here and the rest of the
tutorial reads smoothly. / 这些术语在编译原理里反复出现，先在这里搞懂，后面就顺了。

| Term | 中文 | Meaning in one line |
| --- | --- | --- |
| **Token** | 词法单元 | The smallest meaningful unit: a keyword, name, number, or symbol. 最小的有意义单位。 |
| **Lexeme** | 词素 | The exact source text a token came from (e.g. the characters `"10"`). Token 对应的原始文本。 |
| **Lexer / Scanner** | 词法分析器 | The component that turns characters into tokens. 把字符变成 Token 的部件。 |
| **Grammar** | 文法 | The rules describing which token sequences are legal. 描述“怎样的 Token 序列合法”的规则。 |
| **Parser** | 语法分析器 | Turns a token stream into an AST using the grammar. 按文法把 Token 流变成 AST。 |
| **AST** | 抽象语法树 | A tree of node objects representing program structure. 表示程序结构的树。 |
| **Recursive descent** | 递归下降 | A parsing technique: one function per grammar rule, calling each other recursively. 每条文法规则写一个函数。 |
| **Precedence** | 优先级 | Which operator binds tighter (`*` before `+`). 谁先算（`*` 比 `+` 紧）。 |
| **Evaluate** | 求值 | Compute the value/effect of an AST node. 算出某个 AST 节点的值或效果。 |
| **Environment / Scope** | 环境 / 作用域 | Where variable names map to their current values. 变量名到值的映射表。 |
| **Closure** | 闭包 | A function that "remembers" the environment it was created in. 记住自己出生环境的函数。 |
| **Bytecode** | 字节码 | Compact low-level instructions a VM executes. 虚拟机执行的紧凑底层指令。 |
| **VM (Virtual Machine)** | 虚拟机 | A program that executes bytecode, usually via a stack. 执行字节码的程序（通常基于栈）。 |
| **Finite automaton** | 有限自动机 | The formal machine (DFA/NFA) behind lexing; equal in power to regular expressions. 词法分析背后的理论模型，与正则表达式等价。 |
| **Context-free grammar (CFG)** | 上下文无关文法 | The grammar class parsers recognize, usually written in BNF/EBNF. 语法分析器能识别的文法类别，常用 BNF/EBNF 书写。 |
| **Semantic analysis** | 语义分析 | A static pass after parsing that checks meaning before execution. 语法分析之后、执行之前的静态检查。 |
| **Garbage collection (GC)** | 垃圾回收 | Automatic reclamation of memory the program can no longer reach. 自动回收程序不再能访问到的内存。 |
| **JIT (Just-In-Time)** | 即时编译 | Compiling hot code to a faster form *while the program runs*. 程序运行中把热点代码动态编译成更快的形式。 |
| **Hot path** | 热点路径 | Code run often enough to be worth compiling/optimizing. 执行足够频繁、值得编译优化的代码。 |

---

## 4. Learning Roadmap / 学习路线

The project is organized into **five tiers**, each a meaningful milestone. Tier D is the
leap from a tree-walking interpreter into a real **bytecode compiler + virtual machine**
(the architecture of CPython and the JVM); the final goal, **Tier E**, adds a **JIT** —
compiling hot code at runtime, the way V8, PyPy, and HotSpot do.

> 🇨🇳 **怎么用这张表：** 一节课只学一个核心概念。先读“What you'll learn”想清楚
> *为什么*要做这件事，再自己写出对应文件的代码，跑通后把状态从 ⏳ 改成 ✅。
> 不要跳课——每一课都依赖前一课的产物。

### Tier A — Core: a language that runs / 能跑起来的语言

| Lesson | Topic | What you'll learn | Output files | Status |
| --- | --- | --- | --- | --- |
| 0 | Overview & setup | The three-stage architecture; project layout | directory skeleton | ⏳ |
| 1 | [Lexer](docs/lesson-01-lexer.md) | Tokens, character scanning; the theory behind it — regular expressions ↔ finite automata (DFA/NFA) | `tokens.py`, `lexer.py` | ⏳ |
| 2 | AST | Representing syntax as classes; why a tree | `ast_nodes.py` | ⏳ |
| 3 | Parser | Recursive descent, operator precedence; context-free grammars & BNF/EBNF notation | `parser.py` | ⏳ |
| 4 | Interpreter | Tree walking, environments, evaluation | `interpreter.py` | ⏳ |
| 5 | REPL & files | An interactive shell; running script files | `repl.py`, `main.py` | ⏳ |

### Tier B — Usable: a practical language / 实用的语言

| Lesson | Topic | What you'll learn | Status |
| --- | --- | --- | --- |
| 6 | Functions | Definitions, calls, return values, the call stack | ⏳ |
| 7 | Data structures | Arrays, strings, indexing | ⏳ |
| 8 | Logic & loops | `and`/`or`/`not`, `for` loops, built-in functions | ⏳ |

### Tier C — Engineered: quality & robustness / 工程化与健壮性

| Lesson | Topic | What you'll learn | Status |
| --- | --- | --- | --- |
| 9 | Error reporting | Line/column tracking, friendly error messages | ⏳ |
| 10 | Resolver / semantic analysis | A static pass over the AST (`resolver.py`): variable resolution, scope binding, compile-time checks like "used before declared" | ⏳ |
| 11 | Testing & exceptions | Unit tests, a clean exception hierarchy | ⏳ |

### Tier D — Advanced: real-world techniques / 工业级技术

| Lesson | Topic | What you'll learn | Status |
| --- | --- | --- | --- |
| 12 | Closures & scope chains | First-class functions, captured environments | ⏳ |
| 13 | Dictionaries | Hash maps as a first-class language type | ⏳ |
| 14 | Bytecode compiler | Compiling the AST into bytecode instructions | ⏳ |
| 15 | Virtual machine | A stack-based VM that executes bytecode | ⏳ |
| 16 | Garbage collection | Automatic memory management: a mark-and-sweep collector for the VM's objects | ⏳ |
| 17 | Optimization | Benchmark tree-walking vs. the VM; basic optimizations | ⏳ |

### Tier E — JIT: from interpreter to native speed / 即时编译（收尾巅峰）

| Lesson | Topic | What you'll learn | Status |
| --- | --- | --- | --- |
| 18 | Profiling & hot-path detection | Execution counters; deciding when code is "hot" enough to compile | ⏳ |
| 19 | Baseline JIT (pure Python) | Specialize hot bytecode into generated Python closures, killing dispatch overhead — a real, measurable speedup with zero dependencies | ⏳ |
| 20 | Native JIT *(optional, advanced)* | Emit raw x86-64 machine code, mark it executable via `mmap`, and jump into it with `ctypes`. Platform-specific; steps outside the zero-dependency rule | ⏳ |

> ✅ = done　⏳ = not yet started. Update this table as each lesson is completed.
>
> 🇨🇳 **Lesson 20 是可选挑战**：它会发射真实机器码，因此**平台相关**且**走出“纯
> Python、零依赖”**的边界。核心的 JIT 思想在 18–19 已经学到，20 只是想冲顶的人的加餐。

---

## 5. Project Structure / 项目结构

```
pylox/
├── README.md
├── docs/
│   └── lesson-01-lexer.md  # per-lesson teaching handouts (concepts, API specs, tests)
├── main.py                 # Lesson 5: entry point — run a script file or start the REPL
├── pylox/
│   ├── __init__.py
│   ├── tokens.py           # Lesson 1: Token type definitions
│   ├── lexer.py            # Lesson 1: characters  -> tokens
│   ├── ast_nodes.py        # Lesson 2: AST node classes
│   ├── parser.py           # Lesson 3: tokens      -> AST
│   ├── resolver.py         # Lesson 10: static semantic analysis pass over the AST
│   ├── interpreter.py      # Lesson 4: AST         -> results
│   └── repl.py             # Lesson 5: interactive shell
└── examples/
    └── hello.lox           # an example pylox script
```

> 🇨🇳 **注意文件的依赖顺序**：`lexer` 产出喂给 `parser`，`parser` 产出喂给
> `interpreter`。这正是第 1 节那张三阶段流水线图在文件层面的映射。读代码时，按
> `tokens → lexer → ast_nodes → parser → interpreter → main` 的顺序，体会数据是
> 怎样一层层流过去的。

---

## 6. How to Run / 如何运行

```bash
# Run an example script
python main.py examples/hello.lox

# Start the interactive REPL
python main.py
```

> Requires Python 3.8+. No third-party dependencies.
> 需要 Python 3.8 及以上版本，无需任何第三方库。

---

## 7. Progress / 进度

**Tier A — Core**
- [ ] Lesson 0: project skeleton
- [ ] Lesson 1: Lexer
- [ ] Lesson 2: Abstract Syntax Tree (AST)
- [ ] Lesson 3: Parser
- [ ] Lesson 4: Interpreter
- [ ] Lesson 5: REPL & file execution

**Tier B — Usable**
- [ ] Lesson 6: Functions
- [ ] Lesson 7: Arrays & strings
- [ ] Lesson 8: Logic operators & `for` loops

**Tier C — Engineered**
- [ ] Lesson 9: Friendly error reporting
- [ ] Lesson 10: Resolver / semantic analysis
- [ ] Lesson 11: Tests & exception hierarchy

**Tier D — Advanced**
- [ ] Lesson 12: Closures & scope chains
- [ ] Lesson 13: Dictionaries
- [ ] Lesson 14: Bytecode compiler
- [ ] Lesson 15: Virtual machine
- [ ] Lesson 16: Garbage collection (mark-and-sweep)
- [ ] Lesson 17: Optimization & benchmarks

**Tier E — JIT**
- [ ] Lesson 18: Profiling & hot-path detection
- [ ] Lesson 19: Baseline JIT (pure Python closures)
- [ ] Lesson 20: Native JIT — emit x86-64 machine code *(optional)*

---

## 8. Going Deeper / 延伸阅读

pylox is inspired by **Lox**, the teaching language from Robert Nystrom's free book
*Crafting Interpreters*. After (or alongside) this project, these are the classic next steps:

- **[Crafting Interpreters](https://craftinginterpreters.com/)** — the best modern,
  free introduction. Its Part II (a tree-walker) and Part III (a bytecode VM) mirror our
  Tiers A–C and Tier D almost exactly — including the resolver and a garbage collector.
  强烈推荐，免费在线阅读。
- **The "Dragon Book"** — *Compilers: Principles, Techniques, and Tools* (Aho, Lam,
  Sethi, Ullman). The canonical, theory-heavy reference. 经典教材，理论权威。
- **CPython internals** — once Tier D works, read how Python compiles to and runs its own
  bytecode (`dis` module, `python -m dis`). 学完 Tier D 再看 CPython 自己的字节码会很亲切。

> 🇨🇳 **学习建议：** 本教程偏“动手先于理论”——先把东西做出来、跑起来，建立直觉，
> 再回头去读上面这些更系统的资料，效果最好。遇到看不懂的概念，回到第 1 节的流水线
> 图和第 3 节的术语表，多半能对上号。
