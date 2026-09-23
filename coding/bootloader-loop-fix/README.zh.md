# 启动程序模拟：找到并修复死循环

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 模拟，环检测 | ★★☆☆☆ | 中等 | SWE | simulation, cycle-detection, graphs, parsing | 3 个部分 / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

*启动程序*（boot program）由一系列指令组成，每行一条，格式是 `<op> <value>`，其中 `op` 是 `plus`、`next`、
`jump` 三者之一，`value` 是一个带符号整数。执行过程中维护一个*程序计数器*（program counter）`pc`——即将
执行的那一行的下标，从 0 开始——以及一个全局*累加器*（accumulator）`acc`；两者初始时都是 `pc = 0`、
`acc = 0`。每条指令依次按下面的方式更新它们，然后执行在新的 `pc` 处继续：

- `plus v`——把 `v` 加到 `acc` 上，然后移到下一行（`pc += 1`）。
- `next v`——不改变 `acc`，直接移到下一行（`pc += 1`）；`v` 在执行过程中完全不起作用，也不会被读取。
- `jump v`——不改变 `acc`，相对当前行移动 `v` 行（`pc += v`）；`v` 可以是正数、负数或 0。

一个有 `n` 行的程序，下标从 `0` 到 `n - 1`。当 `pc` 恰好变成 `n`——刚好越过最后一行——的那一刻，执行*正常
结束*，此时的 `acc` 就是答案。如果某条指令让 `pc` 变成 `0 .. n` 之外的其他值（负数，或者越过终点不止一
行），程序就是*越界*（out of range）的：这种情况下没有下一行可以执行，也没有规定接下来该怎么办，必须把它
和正常结束区分开来报告，绝不能当作程序已经跑完来处理。最后，如果 `pc` 即将取一个它在本次运行中已经取过的
值——即将第二次执行自己的某一行——程序就进入了*死循环*（infinite loop）：从它第一次到达这个 `pc` 开始，
后面的走向不可能再有任何不同，所以模拟器完全可以就此停下。

### Part 1 —— 找到死循环

写出 `run_program`：从 `pc = 0`、`acc = 0` 开始，按照上面的规则执行一个启动程序。

```py
def run_program(program: list[tuple[str, int]]) -> tuple[bool, int, int | None]:
    ...
```

`program[i]` 是第 `i` 行的 `(op, value)` 二元组。返回值是 `(terminated, acc, loop_line)`：

- 正常结束时，`terminated` 为 `True`，`acc` 是此时的累加器值，`loop_line` 为 `None`；
- 陷入死循环时，`terminated` 为 `False`，`loop_line` 是即将第二次执行的那一行的下标，`acc` 恰好是上一条
  指令执行完之后的累加器值——是这次重复*之前*的值，不是之后的；
- 如果某次跳转会让 `pc` 越界，抛出 `ProgramError`，而不是返回结果。

```py
class ProgramError(Exception): ...
```

```text
第 0 行: next +0
第 1 行: plus +6
第 2 行: next +5
第 3 行: plus +100
第 4 行: jump +2
第 5 行: plus +50
第 6 行: jump -3
第 7 行: plus +7
第 8 行: next +0

pc=0  next +0    -> 忽略该数值，pc=1                          # acc=0
pc=1  plus +6    -> acc=0+6=6，pc=2                           # acc=6
pc=2  next +5    -> 忽略该数值，pc=3                          # acc=6
pc=3  plus +100  -> acc=6+100=106，pc=4                       # acc=106
pc=4  jump +2    -> pc=4+2=6                                  # acc=106
pc=6  jump -3    -> pc=6-3=3                                  # acc=106
pc=3  —— 两步之前已经执行过：即将第二次执行，就在这里报告死循环

run_program(program) == (False, 106, 3)
```

### Part 2 —— 修复被改错的指令行

这一部分的程序里，恰好有一行 `jump` 或 `next` 被改错了：它的操作符被换成了另一个——一个 `jump` 被写成了
`next`，或者一个 `next` 被写成了 `jump`——而它的数值恰好还是本该有的那个；`plus` 这一行永远不会被改错。正
因为如此，对这一部分的程序调用 `run_program` 永远不会正常结束：它总会陷入死循环。把其中恰好一行的操作符
改回正确的（数值不变），就能让整个程序正常结束；改动其他任何一行都做不到这一点。写出 `find_fix`，返回这一
行的下标以及结束时的累加器值。

```py
def find_fix(program: list[tuple[str, int]]) -> tuple[int, int]:
    ...
```

接着 Part 1 的程序往下看，第 2 行（`next +5`）就是被改错的那一行——它本该是 `jump +5`。把它改回来之后，
第 0、1 行的执行完全不受影响，然后直接跳过第 3 到 6 行这个陷阱：

```text
pc=0  next +0          -> pc=1                    # acc=0
pc=1  plus +6          -> acc=6，pc=2              # acc=6
pc=2  jump +5（已修复）-> pc=7                     # acc=6
pc=7  plus +7          -> acc=13，pc=8             # acc=13
pc=8  next +0          -> pc=9 = len(program)：正常结束

find_fix(program) == (2, 13)
```

### Part 3 —— 解析程序文件

到目前为止，`program` 都已经是解析好的列表。实际场景中它一开始是文本：一行一条指令，中间可能夹杂空行和
注释。*注释*（comment）由 `#` 引出：从 `#` 到这一行结尾的所有内容都不算程序的一部分，不论这一行整行都是
注释，还是 `#` 跟在一条真正的指令后面。一行如果是空的、只有空白字符，或者去掉注释之后变成空的，就不产生
任何指令。除此之外的每一行，在去掉注释和首尾空白之后，都必须恰好变成两个以空白分隔的记号：先是 `plus`、
`next`、`jump` 三者之一，然后是一个带显式正负号的带符号整数（`+0` 和 `-0` 都合法，都表示零；`3`、`3.0`、
`++3` 都不是合法的带符号整数，都要拒绝）。不满足这个形式的行——记号数目不对、操作符无法识别、数值写法不
对——都是格式错误的，报告时必须带上它在输入文本中的行号（1 起始，把每一行都计算在内，包括空行和注释行）。

写出 `parse_program`，把程序的完整文本转换成上面 `run_program` 和 `find_fix` 使用的 `list[tuple[str, int]]`，
顺序与指令在文本中出现的顺序一致，下标从 `0` 开始，不管前面出现过多少行注释或空行。

```py
class ParseError(ValueError):
    """line_number is 1-based and counts every line of the input text -- not the 0-based index
    the matching instruction gets in the returned program."""
    line_number: int


def parse_program(text: str) -> list[tuple[str, int]]:
    ...
```

```text
# module 7 boot sequence
next +0

plus +4      # initial credit
jump +2
plus +999    # skipped trap
plus +3

parse_program(text) == [
    ("next", 0), ("plus", 4), ("jump", 2), ("plus", 999), ("plus", 3),
]
run_program(parse_program(text)) == (True, 7, None)   # “jump +2” 跳过了解析后第 3 行的陷阱
```

```text
"plus 4"       -> 格式错误：4 没有显式正负号
"boot +1"      -> 格式错误："boot" 不是 plus/next/jump
"jump +1 +2"   -> 格式错误：三个记号，而不是两个
"jump"         -> 格式错误：缺少数值
```

## 参考解答

<details>
<summary>展开参考解答</summary>

开始写代码前有两点值得确认。第一，“修复”具体指哪个任务：下面的 Part 2 要找的就是那一行——把它翻转之后能
让*整个*运行正常结束。有些面试官用的是另一种更局部的理解：一旦某个 `jump` 即将落在一个已经访问
过的行上，就把这一个 `jump` 当作 `next` 处理，仅此一步，然后继续往下走，不去寻找一个全局的修复方案。这
两者并不是同一个任务：局部规则只能处理由 `jump` 落在已访问行上引起的重复，如果重复是由一次普通的
`pc += 1` 落到很早之前某次跳转已经访问过的行上引起的——一旦跳转可以同时往前往后跳，这种情况完全可能出
现——局部规则就无话可说了。第二，落在 `0 .. len(program)` 之外永远是非法的，不是第三种结束方式：一个越
过终点不止一行的跳转，同样没有下一行可以执行，但这不算正常结束。

### Part 1

一次遍历就够了：维护一个已访问行号的集合；一旦 `pc` 即将重复一个已经在集合里的行号，就在这里停下并报告
死循环；一旦 `pc` 等于 `n`，就停下并报告累加器的值；`0 .. n` 之外的其他 `pc` 都是非法的。因为一个程序只
有 `n` 个不同的行号，已访问集合在触发这三种结局之一以前，规模不可能超过 `n`，所以不论指令怎么排列，这个
循环总能在 `n + 1` 步以内结束。

```python
class ProgramError(Exception):
    """Raised when the program counter would move outside 0 .. len(program)."""


def run_program(program: list[tuple[str, int]]) -> tuple[bool, int, int | None]:
    n = len(program)
    pc, acc, visited = 0, 0, set()
    while True:
        if pc == n:
            return True, acc, None
        if pc < 0 or pc > n:
            raise ProgramError(f"line {pc} is out of range for a {n}-line program")
        if pc in visited:
            return False, acc, pc
        visited.add(pc)
        op, value = program[pc]
        if op == "plus":
            acc += value
            pc += 1
        elif op == "next":
            pc += 1
        elif op == "jump":
            pc += value
        else:
            raise ValueError(f"unknown op {op!r}")
```

### Part 2

最直接的做法是把每一种可能都试一遍：对每一行 `jump`/`next`，把它翻转之后用 `run_program` 把整个程序从头
跑一遍，第一个能让程序正常结束的翻转就是答案。最多有 `n` 个这样的候选，每次重新运行又是 `O(n)`，最坏情
况下总代价是 `O(n^2)`。

```python
FLIPPED = {"jump": "next", "next": "jump"}


def find_fix_brute_force(program: list[tuple[str, int]]) -> tuple[int, int]:
    for line, (op, value) in enumerate(program):
        if op not in FLIPPED:
            continue                                    # NOTE: plus is never the corrupted line
        candidate = list(program)
        candidate[line] = (FLIPPED[op], value)
        try:
            terminated, acc, _ = run_program(candidate)
        except ProgramError:
            continue                                     # this flip sends pc out of range -- not the fix
        if terminated:
            return line, acc
    raise ValueError("no single flip terminates this program")
```

一次遍历就能在线性时间里找到修复方案。把给定的、已经被改错的程序跑一遍，记录下它在即将重复某一行之前访
问过的一串互不相同的行号 $p_0 = 0, p_1, \dots, p_{k-1}$——这正是 Part 1 的模拟过程，因为其中没有哪一行
出现两次，代价是 $O(n)$。被改错的那一行必然在 $p_0, \dots, p_{k-1}$ 之中：一行如果被改错程序的这次运行
根本没有走到，那么不管之后把它翻转成什么样子，运行的行为都完全一样，翻转它绝不可能把一个死循环变成能正
常结束的运行。

另外，按程序*现在给出的样子*（也就是被改错、未翻转的样子）定义每一行的*后继*（successor）：`plus`/`next`
的 `succ(i) = i + 1`，`jump` 的 `succ(i) = i + value`，如果这个值落在 `0 .. n` 之外就没有定义。这样构成
的图里，每一行至多有一条出边，代表结束的第 `n` 行则没有出边，所以从任意一行出发反复沿 `succ` 走下去，只
有两种可能的结局：走到 `n`，或者回到某个已经走过的行、从此永远循环——既然每个节点至多只有一条出路，就不
会再有别的可能了。用一次从 `n` 出发、沿*反向边*的遍历（第 `j` 行的前驱就是那些 `succ` 等于 `j` 的行），
就能算出所有能这样走到 `n` 的行组成的集合 $R$：一行在 $R$ 中，当且仅当它能从 `n` 反向走到——因为沿着它
的正向边、再沿着下一行的正向边这样一直走下去，正是这条反向路径倒着看的样子。这一步是 $O(n)$：`n + 1` 个
节点，至多 `n` 条边。

接下来按顺序遍历 $p_0, \dots, p_{k-1}$。对每一行 `jump`/`next` 的 $p_t$，算出如果*只翻转这一行*，它会
换成哪个*另一种*后继——`jump` 翻成 `next` 得到 $p_t + 1$，`next` 翻成 `jump` 得到 $p_t + \text{value}$——
再检查这个后继是否落在 $R$ 里。第一个满足条件的 $p_t$ 就是被改错的那一行，不需要再去试任何别的行，理由有
两条，缺一不可：

- 如果这个另一种后继在*未翻转*的图里往下走的这条链，中途又回到了 $p_0, \dots, p_t$ 中的某一行，那么从那
  里开始它就会沿着原本那次运行的尾巴重新走一遍，闭合出同一个死循环——也就是说它根本不可能走到 `n`，根本
  不可能落在 $R$ 里。所以只要它落在 $R$ 里，就已经保证了这条链绕开了整个前缀，那正好就是翻转之后的程序接
  下来会做的事，一路走到 `n`，中途没有任何一行被重复执行：确实是一个有效的修复。
- 真正的修复行一旦被翻转，同样不可能再回到 $p_0, \dots, p_t$——那样也会构成第二次访问，仍然是死循环，而
  不是这次翻转本该带来的正常结束。所以它接下来的走向，不论翻不翻转都完全一样，而且确实会走到 `n`，这就
  让它的另一种后继落在 $R$ 里。

这两条缺了任何一条，单独的 $R$ 都不足以准确回答关于*真正*那次修复的问题；两条放在一起，才说明“落在 $R$
里”正是能找到它、而且不需要为每个候选都重新跑一遍程序的那个判据。

```python
def find_fix(program: list[tuple[str, int]]) -> tuple[int, int]:
    n = len(program)

    def succ(i: int) -> int:
        op, value = program[i]
        return i + value if op == "jump" else i + 1

    predecessors = {j: [] for j in range(n + 1)}
    for i in range(n):
        s = succ(i)
        if 0 <= s <= n:
            predecessors[s].append(i)

    reaches_end = [False] * (n + 1)
    reaches_end[n] = True
    stack = [n]
    while stack:
        j = stack.pop()
        for i in predecessors[j]:
            if not reaches_end[i]:
                reaches_end[i] = True
                stack.append(i)

    pc, acc, seen = 0, 0, set()
    while pc not in seen and 0 <= pc < n:
        seen.add(pc)
        op, value = program[pc]
        if op in FLIPPED:
            alternative = pc + 1 if op == "jump" else pc + value
            if 0 <= alternative <= n and reaches_end[alternative]:
                return pc, acc + _run_from(program, alternative)
        if op == "plus":
            acc += value
        pc = succ(pc)
    raise ValueError("no single flip terminates this program")


def _run_from(program: list[tuple[str, int]], start: int) -> int:
    """Sums the plus values from start to termination, following the UNFLIPPED program -- valid
    because membership in reaches_end already guarantees this chain runs to n without a repeat."""
    n, acc, pc = len(program), 0, start
    while pc != n:
        op, value = program[pc]
        if op == "plus":
            acc += value
        pc = pc + value if op == "jump" else pc + 1
    return acc
```

构建 `predecessors` 和 `reaches_end` 是 $O(n)$，遍历 $p_0, \dots, p_{k-1}$ 是 $O(n)$，`_run_from` 只在
最终胜出的那一个候选上调用一次，又是 $O(n)$——总共 $O(n)$，不会从同一个起点把同一段路径模拟两遍。

### Part 3

先用一次 `split("#", 1)` 去掉注释，再检查别的内容，所以注释既可以单独占一行，也可以跟在一条真正的指令
后面。`str.split()` 不带分隔符调用时本来就会把连续的空白字符合并、并去掉首尾空白，所以操作符和数值之间
多几个空格也不需要特殊处理。

```python
import re

_SIGNED_INT = re.compile(r"^[+-]\d+$")
_OPS = {"plus", "next", "jump"}


class ParseError(ValueError):
    def __init__(self, line_number: int, raw_line: str):
        super().__init__(f"line {line_number}: cannot parse {raw_line.strip()!r} as '<op> <signed integer>'")
        self.line_number = line_number


def parse_program(text: str) -> list[tuple[str, int]]:
    program = []
    for line_number, raw_line in enumerate(text.splitlines(), start=1):
        code = raw_line.split("#", 1)[0].strip()   # NOTE: strip the comment BEFORE checking for blank
        if not code:
            continue
        tokens = code.split()
        if len(tokens) != 2 or tokens[0] not in _OPS or not _SIGNED_INT.match(tokens[1]):
            raise ParseError(line_number, raw_line)
        program.append((tokens[0], int(tokens[1])))
    return program
```

`text.splitlines()` 对整个输入只调用一次，之后每一行的处理相对这一行的长度都是 $O(1)$，所以解析的总代价
是 $O(\text{len(text)})$。

### 追问

- **把暴力解法并行化。** `find_fix_brute_force` 里每一个候选翻转都和其他候选互不相关，把每一行
  `jump`/`next` 各自作为一个任务提交给线程池或进程池，就能把这个 `O(n^2)` 的做法并行化，不需要协调任何
  共享状态——但只有在 `n` 大到确实还打算用 `O(n^2)` 做法时才值得这样做。`find_fix` 本身只有一次线性遍
  历，没有可以这样拆分给多个候选去做的工作。
- **如果被改错的不止一行。** 如果有两行被改错而不是一行，`find_fix` 的论证就不成立了：按这个方法找到的
  第一行 $p_t$ 就算修好，也不能保证程序会正常结束，因为从它的另一种后继出发的这条路径仍然可能碰上*另
  一处*被改错的行，再次陷入死循环。要找出所有最小的修复方案，就得在每接受一个修复之后重新模拟一遍，而
  不能只靠一次预先算好的反向遍历就把答案都读出来。
- **真实硬件里的对应机制。** 固件防范这一类 bug 的办法是看门狗定时器（watchdog timer）：一块自由运行
  的硬件计时器，除非软件按固定节奏显式“喂”它，否则会把芯片复位。如果启动流程卡死或者陷入循环，就再也
  轮不到去喂这个定时器，它的计数归零，芯片随即重启——这可以看作 Part 1 中基于已访问集合的死循环检测器在硬
  件层面上更粗粒度的对应版本，只是它完全不知道是哪一行出的问题。
- **用标签代替相对偏移量。** 真正的汇编器允许跳转目标是一个符号化的标签，而不是一个原始的行偏移量。要
  让 `parse_program` 支持 `jump some_label` 这样的写法，需要两趟扫描：第一趟记录下每个标签对应的行号，
  第二趟等所有标签的位置都确定之后，再把每一处标签引用换算成相对偏移量 `target_line - current_line`。
- **“唯一”是需要检验的，不是可以假定的。** 两个版本的 `find_fix` 都是一找到第一个可行的候选就直接返
  回，从不检查是否还有第二个候选也可行；如果还要求检测出零个或者不止一个可行的翻转，`find_fix_brute_force`
  就得在第一次命中之后继续往下扫描而不是立刻返回，`find_fix` 也得在经过 $p_t$ 之后继续往下走，因为路径
  上更靠后的某一行同样可能是一个同样有效的修复。

<details>
<summary>验证代码（可运行）</summary>

```python
import random
import time

# --- Part 1: the worked example ---
demo = [
    ("next", 0), ("plus", 6), ("next", 5), ("plus", 100), ("jump", 2),
    ("plus", 50), ("jump", -3), ("plus", 7), ("next", 0),
]
assert run_program(demo) == (False, 106, 3)

# --- Part 2: the same program, fixed ---
assert find_fix_brute_force(demo) == (2, 13)
assert find_fix(demo) == (2, 13)
fixed_demo = list(demo)
fixed_demo[2] = ("jump", 5)
assert run_program(fixed_demo) == (True, 13, None)

# --- Part 3: the worked example, and the malformed-line examples ---
text = """\
# module 7 boot sequence
next +0

plus +4      # initial credit
jump +2
plus +999    # skipped trap
plus +3
"""
parsed = parse_program(text)
assert parsed == [("next", 0), ("plus", 4), ("jump", 2), ("plus", 999), ("plus", 3)]
assert run_program(parsed) == (True, 7, None)

for bad_line, why in [("plus 4", "no sign"), ("boot +1", "unknown op"),
                       ("jump +1 +2", "three tokens"), ("jump", "missing value")]:
    try:
        parse_program(f"next +0\n{bad_line}\nplus +1\n")
        raise AssertionError(f"expected ParseError for {bad_line!r} ({why})")
    except ParseError as e:
        assert e.line_number == 2, (bad_line, e.line_number)
assert parse_program("# only comments\n\n   \n# another\n") == []

# --- edge cases ---
assert run_program([]) == (True, 0, None)                         # empty program
assert run_program([("jump", 0)]) == (False, 0, 0)                # jump 0: immediate self-loop
try:
    run_program([("plus", 1), ("jump", 5)])                       # overshoots the end by 3, not exactly onto it
    raise AssertionError("expected ProgramError")
except ProgramError:
    pass
try:
    run_program([("jump", -1)])                                   # negative pc
    raise AssertionError("expected ProgramError")
except ProgramError:
    pass
assert run_program([("jump", 2), ("plus", 999), ("next", 0)]) == (True, 0, None)  # jump over a mid-range trap

print("worked examples and edge cases passed")


# --- an independent brute force, written from the statement, with a defensive step cap ---
def spec_run(program, step_cap):
    n = len(program)
    pc, acc, seen = 0, 0, []
    for _ in range(step_cap + 1):
        if pc == n:
            return True, acc, None
        if pc < 0 or pc > n:
            raise ProgramError(f"out of range: {pc}")
        if pc in seen:
            return False, acc, pc
        seen.append(pc)
        op, value = program[pc]
        if op == "plus":
            acc, pc = acc + value, pc + 1
        elif op == "next":
            pc = pc + 1
        else:
            pc = pc + value
    raise AssertionError("exceeded the step cap -- a well-formed run never needs more than n + 1 steps")


def random_terminating_program(rng, n):
    """A random n-line program that terminates normally from line 0, built by laying out a random
    path of distinct lines from 0 to the virtual exit n, then filling every unused line with junk
    that the correct run never reaches."""
    unused = set(range(n))
    unused.discard(0)
    path, program, pc = [0], [None] * n, 0
    while pc != n:
        options = []
        if pc + 1 == n or (pc + 1) in unused:
            options += [pc + 1, pc + 1]                  # weighted up, so next/plus steps are common too
        if unused:
            options.append(rng.choice(list(unused)))
        options.append(n)                                # always allowed to jump straight to the exit
        target = rng.choice(options)
        if target == pc + 1:
            program[pc] = (rng.choice(["next", "plus"]), rng.randint(-50, 50))
        else:
            program[pc] = ("jump", target - pc)
        if target != n:
            unused.discard(target)
            path.append(target)
        pc = target
    path.append(n)
    span = max(3, min(30, n))     # local filler jumps: an excursion off the path tends to loop, not fly out
    for i in range(n):
        if program[i] is None:
            op = rng.choice(["plus", "next", "jump"])
            value = rng.randint(-span, span) if op == "jump" else rng.randint(-50, 50)
            program[i] = (op, value)
    return program, path


def single_fix_instance(rng, n):
    """A random n-line program with exactly one corrupted jump/next line, verified (independently
    of run_program/find_fix) to loop as given and to have a unique terminating flip."""
    for _ in range(20):
        program, path = random_terminating_program(rng, n)
        corruptible = [i for i in path[:-1] if program[i][0] in FLIPPED]
        if not corruptible:
            continue
        line = rng.choice(corruptible)
        op, value = program[line]
        corrupted = list(program)
        corrupted[line] = (FLIPPED[op], value)
        try:
            if spec_run(corrupted, n + 5)[0]:
                continue                                  # coincidence: the flip changed nothing observable
        except ProgramError:
            continue

        fixes = []
        for i, (iop, ival) in enumerate(corrupted):
            if iop not in FLIPPED:
                continue
            trial = list(corrupted)
            trial[i] = (FLIPPED[iop], ival)
            try:
                ok, acc, _ = spec_run(trial, n + 5)
            except ProgramError:
                continue
            if ok:
                fixes.append((i, acc))
        if len(fixes) == 1:
            return corrupted, fixes[0], op
    return None


trials, by_direction = 0, {"jump": 0, "next": 0}
for seed in range(800):
    rng = random.Random(seed)
    result = single_fix_instance(rng, rng.randint(4, 30))
    if result is None:
        continue
    corrupted, (expected_line, expected_acc), original_op = result
    by_direction[original_op] += 1

    assert run_program(corrupted) == spec_run(corrupted, len(corrupted) + 5)
    assert find_fix_brute_force(corrupted) == (expected_line, expected_acc), seed
    assert find_fix(corrupted) == (expected_line, expected_acc), seed
    trials += 1

assert trials > 600 and by_direction["jump"] > 50 and by_direction["next"] > 50, (trials, by_direction)
print(f"brute-force cross-check passed on {trials} random single-fix programs "
      f"({by_direction['jump']} corrupted jumps, {by_direction['next']} corrupted nexts)")

# --- find_fix is O(n): a deliberately adversarial program where every wrong flip loops only after
# touching every line, so find_fix_brute_force pays for it and find_fix does not ---
n = 3000
worst_case = [("jump", 1)] * (n - 1) + [("jump", -(n - 1))]      # a single n-line cycle if never fixed
assert run_program(worst_case) == (False, 0, 0)
t0 = time.perf_counter()
slow = find_fix_brute_force(worst_case)
brute_seconds = time.perf_counter() - t0
t0 = time.perf_counter()
fast = find_fix(worst_case)
fast_seconds = time.perf_counter() - t0
assert slow == fast == (n - 1, 0)
assert brute_seconds > 20 * fast_seconds, (brute_seconds, fast_seconds)   # measured: roughly 500x on n=3,000

big = [("jump", 1)] * 199_999 + [("jump", -199_999)]
t0 = time.perf_counter()
assert find_fix(big) == (199_999, 0)
big_seconds = time.perf_counter() - t0
assert big_seconds < 5.0        # measured: well under a second; generous for a loaded 2-core machine

print(f"find_fix stayed linear: {brute_seconds:.3f}s (brute) vs {fast_seconds:.3f}s (fast) at n={n}, "
      f"{big_seconds:.3f}s at n=200,000")

print("all checks passed")
```

</details>

</details>
