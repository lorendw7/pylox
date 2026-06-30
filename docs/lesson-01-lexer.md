# Lesson 1 — The Lexer / 词法分析器

> **Output files / 产出文件:** `pylox/tokens.py`, `pylox/lexer.py`
> **You write every line of code.** This handout gives you the *concepts, design, API
> specification, pseudocode, and test cases* — the implementation is yours.
> 本讲义只给**概念、设计、API 规格、伪代码、验收用例**；**每一行 Python 由你自己写**。

---

## 1. Goal of This Lesson / 本课目标

Turn a raw **source string** into a flat **list of tokens** — the first of the three
stages from the README's big-picture diagram.

```
"let x = 10"   ─►   [ LET "let" ] [ IDENT "x" ] [ ASSIGN "=" ] [ NUMBER "10" ] [ EOF ]
```

> 🇨🇳 **本课要做的事：** 把一串源代码字符，切成一个个有意义的“词”（Token），输出一个
> Token 列表。这就是 README 全局图里的**第一阶段：词法分析**。计算机一开始只看到字符
> `l`,`e`,`t`,` `,`x`...；词法分析器的工作就是把它们归类成 `LET`、`IDENT(x)`、`ASSIGN`
> 这些有类型、有含义的单元，交给下一阶段（语法分析）使用。

---

## 2. The Core Idea / 核心思想

A lexer (also called a **scanner**) reads the source **left to right, one character at a
time**, and groups characters into tokens. It never looks at *grammar* (it doesn't know
that `if` must be followed by a condition) — it only recognizes **shapes of words**.

> 🇨🇳 **类比：** 就像你读一句没有空格的英文 `"thecatsat"`，你的大脑会自动切成
> `the` / `cat` / `sat`。词法分析器做的就是这件事——它**不关心语法对不对**（“猫坐”
> 是否合理由后面的语法分析管），只负责把字符流切成一个个“单词”，并贴上类型标签。
>
> 🇨🇳 **它的工作方式：** 维护一个“当前位置”指针，从左往右扫。每一步看当前字符，
> 决定它是哪类词的开头（数字？标识符？运算符？），然后一直吃到这个词的结尾，吐出一个
> Token，再继续。

### Theory frame / 理论背景 — regex ↔ finite automata

The rules for "what a number looks like" or "what an identifier looks like" are exactly
**regular expressions**, and any regex can be recognized by a **finite automaton (DFA/NFA)**
— a tiny machine with states and transitions. Your hand-written scanning loop *is* such a
state machine: "I'm in the middle of reading a number" is a state; seeing a non-digit is a
transition out of it.

> 🇨🇳 **理论框：** “数字长什么样”“标识符长什么样”这些规则，本质就是**正则表达式**；
> 而任何正则表达式都能用一个**有限自动机（DFA/NFA）**——一个由“状态 + 转移”组成的
> 小机器——来识别。你手写的扫描循环*本身就是*这样一台状态机：“我正在读一个数字”是一个
> 状态，遇到非数字字符就是一次“转移出去”。这就是词法分析背后的理论根基，记住这一点，
> 你写循环时就知道自己在做什么。

---

## 3. Output #1 — The `Token` (in `tokens.py`) / 第一个产出：Token

A **token** is a small data object carrying four things. Design it as a class (a
`dataclass` or `class` with `__init__` — your choice). It is *just data*; no logic.

| Field | 中文 | Purpose |
| --- | --- | --- |
| `type` | 类型 | Which kind of token (e.g. `NUMBER`, `LET`, `PLUS`). Use an `Enum` or string constants. |
| `lexeme` | 词素 | The exact source text this token came from, e.g. `"10"`, `"let"`. |
| `literal` | 字面值 | The *runtime value* for literals: the number `10`, the string `big`. `None` for non-literals. |
| `line` | 行号 | Which source line it started on — you'll need this for error messages in Lesson 9. |

> 🇨🇳 **为什么 `lexeme` 和 `literal` 要分开？** `lexeme` 是源码里**原封不动的那段文本**
> （字符串 `"10"`）；`literal` 是它对应的**真实值**（整数 `10`）。对 `NUMBER`、`STRING`
> 这类字面量，两者不同；对关键字、运算符，`literal` 一般是 `None`。现在就分开，后面
> 求值阶段会很省事。
>
> 🇨🇳 **`line` 现在就要带上：** 即使本课还用不到，养成“每个 Token 都记住自己来自第几行”
> 的习惯，第 9 课做友好报错时你会感谢现在的自己。

**Your task / 你的任务:** define the set of token types, and a `Token` class with those four
fields and a readable `__repr__` (so you can print tokens while debugging).

---

## 4. Token Types pylox Needs / pylox 需要的 Token 类型

Define these (names are a suggestion; pick a convention and stay consistent):

**Literals / 字面量**
- `NUMBER` — `10`, `3`, `42`
- `STRING` — `"big"`, `"hello"`
- `IDENTIFIER` — `x`, `sum`, `print`

**Keywords / 关键字** (reserved words — see §6.4)
- `LET`, `IF`, `ELSE`, `WHILE`

**Single-character tokens / 单字符**
- `PLUS (+)`, `MINUS (-)`, `STAR (*)`, `SLASH (/)`
- `LPAREN (()`, `RPAREN ())`, `LBRACE ({)`, `RBRACE (})`
- `ASSIGN (=)`, `GREATER (>)`, `LESS (<)`

**Two-character tokens / 双字符** (these teach *lookahead* — see §6.1)
- `EQUAL_EQUAL (==)`, `BANG_EQUAL (!=)`, `GREATER_EQUAL (>=)`, `LESS_EQUAL (<=)`

**Special / 特殊**
- `NEWLINE` — pylox separates statements by line breaks, so `\n` is *meaningful*, not
  just whitespace. Emit a `NEWLINE` token (the parser in Lesson 3 will use it).
- `EOF` — a single sentinel token at the very end.

> 🇨🇳 **为什么 `print` 不是关键字？** README 里 `print(sum)` 是一次**函数调用**，`print`
> 只是一个内置函数的名字——所以它是普通 `IDENTIFIER`，不是关键字。把 `print` 留成标识符，
> 以后用户甚至能重新定义它。**关键字是“被语言保留、不能当变量名”的词**（如 `let`/`if`）。
>
> 🇨🇳 **为什么要 `EOF`？** 在 Token 列表末尾放一个“结束哨兵”，下一阶段的语法分析器就能
> 统一地问“到结尾了吗？”，而不用每次都判断列表越界。这是个经典又好用的小技巧。
>
> 🇨🇳 **`NEWLINE` 的取舍：** pylox 没有分号，靠换行分隔语句，所以换行**有意义**，要作为
> Token 吐出来。但空格、制表符（` `、`\t`）是无意义的，直接跳过。

---

## 5. Output #2 — The `Lexer` (in `lexer.py`) / 第二个产出：Lexer

### 5.1 Recommended internal state / 建议的内部状态

Your `Lexer` holds the source and three cursors:

| State | Meaning |
| --- | --- |
| `source` | the full input string |
| `start` | index where the *current* token began |
| `current` | index of the *next* character to look at |
| `line` | current line number (starts at 1) |
| `tokens` | the list you append finished tokens to |

> 🇨🇳 **两个指针的分工：** `start` 标记“我正在读的这个词从哪开始”，`current` 是“我正
> 扫到哪了”。一个 Token 的文本就是 `source[start:current]`。每开始读一个新 Token 前，
> 先把 `start` 拉到 `current`。

### 5.2 Recommended helper methods (API spec — you write the bodies) / 建议的辅助方法

These are *signatures and contracts*. Implement each body yourself.

```
is_at_end() -> bool
    True when current has reached the end of source.

advance() -> str
    Return source[current], THEN move current forward by one. (consumes a char)

peek() -> str
    Return the current char WITHOUT consuming it. Return '\0' if at end.

peek_next() -> str
    Return the char after current without consuming. '\0' if out of range.

match(expected: str) -> bool
    If the current char equals `expected`, consume it and return True.
    Otherwise return False. (This is your one-character lookahead.)

add_token(type, literal=None) -> None
    Build a Token using source[start:current] as the lexeme and append it.

scan_token() -> None
    Scan exactly ONE token starting at current, and add it.

scan_tokens() -> list
    The main loop. Repeatedly: set start = current, scan_token(),
    until is_at_end(). Finally append the EOF token. Return tokens.
```

> 🇨🇳 **`advance` vs `peek` 是初学者最容易搞混的：** `advance` 会**吃掉**字符（指针前进），
> `peek` 只**看一眼**不前进。规则：当你确定要这个字符时用 `advance`；当你需要“先看看是
> 什么再决定”时用 `peek`。`match` 是两者的结合——“如果下一个是我想要的，就吃掉”。

### 5.3 The scanning algorithm / 扫描算法 (pseudocode — translate to Python yourself)

```
scan_tokens():
    while not at end:
        start = current          # mark the beginning of a new token
        scan_token()
    append Token(EOF, "", line)
    return tokens

scan_token():
    c = advance()
    switch on c:
        '(' -> add_token(LPAREN)
        ')' -> add_token(RPAREN)
        '{' -> add_token(LBRACE)
        '}' -> add_token(RBRACE)
        '+' -> add_token(PLUS)
        '-' -> add_token(MINUS)
        '*' -> add_token(STAR)

        '/' -> ??? careful: '/' could start a comment? (pylox uses '#', so '/' is just SLASH)
        '#' -> a comment: consume chars with peek()/advance() until '\n' or end. Add NO token.

        '=' -> if match('=') add_token(EQUAL_EQUAL) else add_token(ASSIGN)
        '!' -> if match('=') add_token(BANG_EQUAL) else ERROR (lone '!' is invalid for now)
        '>' -> if match('=') add_token(GREATER_EQUAL) else add_token(GREATER)
        '<' -> if match('=') add_token(LESS_EQUAL) else add_token(LESS)

        '"' -> read a string literal (see §6.2)

        ' ', '\t', '\r' -> ignore (whitespace)
        '\n' -> add_token(NEWLINE); line += 1

        otherwise:
            if c is a digit      -> read a number  (see §6.3)
            elif c starts a name -> read identifier/keyword (see §6.4)
            else                 -> ERROR: "Unexpected character: c" (with line number)
```

---

## 6. The Four Tricky Parts / 四个难点

### 6.1 Multi-character operators & "maximal munch" / 多字符运算符与“最长匹配”

When you see `=`, you can't yet tell if it's `=` (assign) or `==` (equality). Peek one more
character: if the next is `=`, consume it and emit `EQUAL_EQUAL`; otherwise emit `ASSIGN`.

> 🇨🇳 **最长匹配原则（maximal munch）：** 词法分析器永远尽量匹配**最长**的合法 Token。
> 看到 `>=` 时绝不能先吐一个 `>` 再吐一个 `=`，而要合成一个 `GREATER_EQUAL`。实现方法
> 就是 `match('=')`：看下一个字符是不是 `=`，是就吃掉、合成双字符 Token。

### 6.2 Strings / 字符串

After the opening `"`, keep consuming characters until the closing `"`. The `literal` value
is the text *between* the quotes (quotes excluded). If you hit end-of-input before a closing
quote, that's an **unterminated string** error — report it with the line number.

> 🇨🇳 **要点：** (1) `lexeme` 含引号、`literal` 不含；(2) 字符串里可以跨行（遇到 `\n` 时
> 记得 `line += 1`）——是否允许跨行是你的语言设计选择，本课先允许；(3) 没有结束引号就到了
> 文件末尾 → 报“未闭合字符串”错误。转义字符（`\n`、`\"`）本课可以先不做，留作 TODO。

### 6.3 Numbers / 数字

Once you see a digit, keep consuming digits with `peek()`. The `literal` is the parsed
numeric value. Decide: integers only this lesson, or also decimals like `3.14`? (For `3.14`
you must peek past the `.` only if a digit follows it — use `peek_next()`.)

> 🇨🇳 **设计决策：** 本课可以只支持整数，把小数留到以后；如果想做小数，注意 `3.14` 里的
> `.` 只有在**后面紧跟数字**时才算数字的一部分（用 `peek_next()` 判断），否则 `3.` 后面的
> `.` 可能是别的东西。把 `literal` 存成真正的 `int`/`float`，不是字符串。

### 6.4 Identifiers vs keywords / 标识符与关键字

An identifier starts with a letter (or `_`) and continues with letters/digits/`_`. Read the
whole word first, **then** check: is this word in your keyword set? If yes, emit the keyword
token type (`LET`/`IF`/...); if no, emit `IDENTIFIER`.

> 🇨🇳 **关键技巧——先当标识符读，再查表：** 不要一看到 `l` 就猜是不是 `let`。正确做法是
> 把整个词 `letx` 完整读出来，再去“关键字表”里查。`let` 在表里 → 关键字；`letx` 不在 →
> 普通标识符。这样 `letx` 才不会被错误地拆成关键字 `let` + `x`。维护一个
> `{"let": LET, "if": IF, "else": ELSE, "while": WHILE}` 的字典即可。（`true`/`false`/`and`/
> `or`/`not`/`fn`/`for`/`return` 等以后再往表里加。`fn` 用来定义函数和 lambda（Tier B/F）。）

---

## 7. Acceptance Criteria / 验收标准

Implement, then verify your lexer produces these token sequences. (`EOF` is implicit at the
end of each.) 写完后，用这些用例自测，看输出是否一致（每个末尾都隐含一个 `EOF`）。

**Test 1 — basics**
```
input:  let x = 10
tokens: LET("let")  IDENTIFIER("x")  ASSIGN("=")  NUMBER("10", literal=10)
```

**Test 2 — multi-char operators (maximal munch)**
```
input:  x >= 5 == y
tokens: IDENTIFIER("x")  GREATER_EQUAL(">=")  NUMBER("5", literal=5)
        EQUAL_EQUAL("==")  IDENTIFIER("y")
```

**Test 3 — strings & a call**
```
input:  print("big")
tokens: IDENTIFIER("print")  LPAREN("(")  STRING("\"big\"", literal=big)  RPAREN(")")
```

**Test 4 — comments & newlines are handled**
```
input:  let i = 0   # counter
        i = i + 1
tokens: LET IDENTIFIER("i") ASSIGN NUMBER(0) NEWLINE
        IDENTIFIER("i") ASSIGN IDENTIFIER("i") PLUS NUMBER(1)
        (the comment produces NO tokens; the line break produces NEWLINE)
```

**Test 5 — errors**
```
input:  let x = "oops      (no closing quote)
expect: an error reporting "unterminated string" with the correct line number
        (it should NOT crash with a raw Python exception/traceback)

input:  let x = @
expect: an error reporting an unexpected character '@' with its line number
```

> 🇨🇳 **自测建议：** 写一个临时脚本，读入一段源码，调用 `Lexer(src).scan_tokens()`，把
> 每个 Token 用你的 `__repr__` 打印出来，肉眼比对上面的预期。错误用例要确认你的程序是
> **优雅报错**（带行号的友好信息），而不是抛出 Python 原始异常崩掉。

---

## 8. Common Pitfalls / 常见坑

- **Forgetting to advance** — if a branch doesn't consume any character, your `while` loop
  spins forever. 每条分支都要确保推进了 `current`，否则死循环。
- **Off-by-one in `source[start:current]`** — remember `advance()` returns the char *then*
  increments, so after reading a one-char token, `current` already points *past* it. 切片
  边界要想清楚：`advance` 是“先取后进”。
- **Treating `print` as a keyword** — it's an identifier. 见 §4。
- **Splitting `>=` into `>` and `=`** — apply maximal munch. 见 §6.1。
- **Eating the `#` comment but forgetting the `\n` still needs a `NEWLINE`** — stop the
  comment *at* the newline, don't consume the newline inside the comment branch. 注释吃到
  换行**为止**，换行本身留给 `\n` 分支去产生 `NEWLINE`。
- **Reading numbers/strings as Python `str` for the `literal`** — convert to real `int`/
  `float`/`str` value. `literal` 要存真实值，不是文本。

---

## 9. Reflection Questions / 思考题

Answer these to yourself — they confirm you understand *why*, not just *how*. 自问自答，
检验你是否真的理解了：

1. Why does the lexer not care whether `if 5 5 5` is *grammatically* valid? Whose job is
   that? / 为什么词法分析器不管 `if 5 5 5` 语法上对不对？那是谁的活？
2. Why emit a single `EOF` token instead of just stopping? / 为什么要专门吐一个 `EOF`？
3. What state in your scanning loop corresponds to "I'm in the middle of reading a number"?
   How is that a finite automaton? / 你循环里“正在读数字”对应自动机的哪个状态？
4. If pylox later adds `>>` (e.g. a shift operator), what one place changes? / 以后要加
   `>>`，你只需改哪一处？（体会词法设计的可扩展性）

---

## 10. Done? / 完成了吗？

You're done with Lesson 1 when:
- `tokens.py` defines your token types and a `Token` class with a clear `__repr__`.
- `lexer.py` defines `Lexer` with `scan_tokens()` and the helpers from §5.2.
- All five acceptance tests in §7 pass, including graceful errors.

Then update the status in the main [README](../README.md): change Lesson 1 from ⏳ to ✅ in
both the Tier A table and the Progress checklist. 然后回到 README 把 Lesson 1 标成 ✅。

> 🇨🇳 **下一课预告（Lesson 2 — AST）：** 你现在有了一串“词”。下一步是定义一组**节点类**，
> 用来把这些词组装成一棵**树**——这棵树才能表达 `2 + 3 * 4` 里“先乘后加”的结构。先把
> 词法器写扎实，AST 会顺理成章。
