# Take-home：在模拟向量机上优化计算内核

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| Take-home · 性能优化 | ★★☆☆☆ | 困难 | Perf Eng | vectorization, loop-unrolling, instruction-scheduling, software-pipelining, simulation | 2 小时 | Take-home 作业 |
<!-- meta:end -->

## 题目

你会拿到一个小型向量处理器的模拟器，以及一段在其上运行结果正确但很慢的内核。任务是重写这段内核，
使其在更少的模拟周期内算出相同的结果，处理器本身不可改动。

### 机器

处理器有 8 个标量寄存器 `s0`–`s7`，每个存一个数；还有 8 个向量寄存器 `v0`–`v7`，每个存放
`VLEN = 8` 条浮点数通道（lane）。内存是一段按字寻址（word-addressed）的浮点数平坦列表：地址 `i`
指的是一个浮点数，不是一个字节。机器没有分支指令。一个*程序*（program）是预先构造好的一串
*指令束*（bundle）——就像编译器在提前知道循环次数时把循环展开成固定代码，而不是像硬件的程序计
数器那样一条条走、每轮迭代结束时再做一次判断。

一个*指令束*是在同一个周期内一起发射（issue）的一组指令。每条指令都恰好属于一个*发射单元*
（issue unit），一个指令束在每个单元上最多能同时携带这么多条指令：2 条标量 ALU、1 条向量
ALU、1 次加载（标量或向量）、1 次存储（标量或向量）。指令集如下：

| 助记符 | 操作数 | 效果 | 单元 |
| --- | --- | --- | --- |
| `li` | `rd, imm` | `s[rd] = imm` | scalar |
| `mov` | `rd, rs` | `s[rd] = s[rs]` | scalar |
| `add` | `rd, rs1, rs2` | `s[rd] = s[rs1] + s[rs2]` | scalar |
| `addi` | `rd, rs, imm` | `s[rd] = s[rs] + imm` | scalar |
| `mul` | `rd, rs1, rs2` | `s[rd] = s[rs1] * s[rs2]` | scalar |
| `sload` | `rd, raddr` | `s[rd] = mem[s[raddr]]` | load |
| `vload` | `vd, raddr` | `v[vd] = mem[s[raddr] : s[raddr] + VLEN]` | load |
| `sstore` | `raddr, rs` | `mem[s[raddr]] = s[rs]` | store |
| `vstore` | `raddr, vs` | `mem[s[raddr] : s[raddr] + VLEN] = v[vs]` | store |
| `vadd` | `vd, vs1, vs2` | `v[vd] = v[vs1] + v[vs2]`，逐通道 | vector |
| `vmul` | `vd, vs1, vs2` | `v[vd] = v[vs1] * v[vs2]`，逐通道 | vector |
| `vmadd` | `vd, vs1, vs2, vs3` | `v[vd] = v[vs1] * v[vs2] + v[vs3]`，逐通道 | vector |
| `vbcast` | `vd, rs` | 把 `v[vd]` 的每条通道都设为 `s[rs]` | vector |
| `vrsum` | `rd, vs` | `s[rd]` = `v[vs]` 这 `VLEN` 条通道之和 | vector |

一个指令束里的每条指令读到的都是这个指令束*开始之前*的寄存器堆状态，等所有指令都读取完毕，指令
束里的写入才会一并生效。因此，即使一条指令要写的寄存器正好是同一指令束里另一条指令要读的，这两
条独立的指令也可以共处一束：读的那一条看到的是指令束生效前的旧值。同一个指令束里，两条指令永远
不能写同一个寄存器。

写入的寄存器不一定在紧接着的下一个指令束里就能读到。`li`、`mov`、`add`、`addi`、`mul`、`vadd`、
`vmul`、`vmadd`、`vbcast`、`vrsum` 都是在产生它们的指令束发射后过一个周期，其结果才可读；
`sload` 和 `vload` 则要等 `LOAD_LATENCY = 4` 个周期。当一个指令束要读的寄存器还没全部就绪时就会
*阻塞*（stall）：时间流逝、什么都不发射，直到这个指令束要读的每个寄存器都就绪为止，这才发射。
程序的*周期数*（cycle count）比最后一个指令束发射时所处的周期多 1（空程序耗费 0 个周期，此后无
论发射还是阻塞，每个指令束都至少把周期计数推进 1）。内存不做单独的冒险检测：下面出现的程序都不
会在某个地址被自己此前对它的写入真正完成之前去读它。

例如，下面这串指令束：

```text
li(s0, 0)         # 无输入；在第 0 周期发射；s0 从第 1 周期起可读
vload(v0, s0)     # 读 s0（第 1 周期就绪，而这个指令束最早也要到第 1 周期）；
                  #   在第 1 周期发射；v0 是加载结果，要到 1 + 4 = 5 才可读
vadd(v1, v0, v0)  # 读 v0；本该在第 2 周期发射，但 v0 要到第 5 周期才就绪——
                  #   在第 2、3、4 周期阻塞，到第 5 周期才发射
```

耗费 6 个周期：指令束分别在第 0、1、5 周期发射，计数为 `5 + 1`。

下面的模拟器实现的正是这台机器。

```python
VLEN = 8
NUM_SREG = 8
NUM_VREG = 8
LOAD_LATENCY = 4
UNIT_WIDTH = {"scalar": 2, "vector": 1, "load": 1, "store": 1}

# Every opcode belongs to exactly one issue unit.
OP_UNIT = {
    "li": "scalar", "mov": "scalar", "add": "scalar", "addi": "scalar", "mul": "scalar",
    "sload": "load", "vload": "load",
    "sstore": "store", "vstore": "store",
    "vadd": "vector", "vmul": "vector", "vmadd": "vector", "vbcast": "vector", "vrsum": "vector",
}


def _operand_regs(instr):
    """The (reads, writes) of one instruction, each a list of ("s" | "v", index)."""
    op, *args = instr
    if op == "li": return [], [("s", args[0])]
    if op == "mov": return [("s", args[1])], [("s", args[0])]
    if op in ("add", "mul"): return [("s", args[1]), ("s", args[2])], [("s", args[0])]
    if op == "addi": return [("s", args[1])], [("s", args[0])]
    if op == "sload": return [("s", args[1])], [("s", args[0])]
    if op == "vload": return [("s", args[1])], [("v", args[0])]
    if op == "sstore": return [("s", args[0]), ("s", args[1])], []
    if op == "vstore": return [("s", args[0]), ("v", args[1])], []
    if op in ("vadd", "vmul"): return [("v", args[1]), ("v", args[2])], [("v", args[0])]
    if op == "vmadd": return [("v", args[1]), ("v", args[2]), ("v", args[3])], [("v", args[0])]
    if op == "vbcast": return [("s", args[1])], [("v", args[0])]
    if op == "vrsum": return [("v", args[1])], [("s", args[0])]
    raise ValueError(f"unknown opcode {op!r}")


class VectorMachine:
    """An in-order vector processor. A program is a list of bundles; a bundle is a list of
    instructions issued in the same cycle, at most UNIT_WIDTH[unit] per unit. See the machine
    description above for the read/write and stall rules."""

    def __init__(self, mem):
        self.s = [0] * NUM_SREG
        self.v = [[0.0] * VLEN for _ in range(NUM_VREG)]
        self.mem = mem
        self.cycles = 0

    def _eval(self, instr):
        """The writes this instruction produces, computed from the register file as it stood
        before the enclosing bundle. Returns a list of (file, index, value); a store's effect on
        memory is applied immediately, since nothing here re-reads a location before its own
        prior write to it has retired."""
        op, *args = instr
        s, v, mem = self.s, self.v, self.mem
        if op == "li": return [("s", args[0], args[1])]
        if op == "mov": return [("s", args[0], s[args[1]])]
        if op == "add": return [("s", args[0], s[args[1]] + s[args[2]])]
        if op == "mul": return [("s", args[0], s[args[1]] * s[args[2]])]
        if op == "addi": return [("s", args[0], s[args[1]] + args[2])]
        if op == "sload":
            addr = s[args[1]]
            if not 0 <= addr < len(mem):
                raise IndexError(f"sload: address {addr} out of range")
            return [("s", args[0], mem[addr])]
        if op == "vload":
            base = s[args[1]]
            if base < 0 or base + VLEN > len(mem):
                raise IndexError(f"vload: range [{base}, {base + VLEN}) out of range")
            return [("v", args[0], mem[base:base + VLEN])]
        if op == "sstore":
            addr = s[args[0]]
            if not 0 <= addr < len(mem):
                raise IndexError(f"sstore: address {addr} out of range")
            mem[addr] = s[args[1]]
            return []
        if op == "vstore":
            base = s[args[0]]
            if base < 0 or base + VLEN > len(mem):
                raise IndexError(f"vstore: range [{base}, {base + VLEN}) out of range")
            mem[base:base + VLEN] = v[args[1]]
            return []
        if op == "vadd": return [("v", args[0], [p + q for p, q in zip(v[args[1]], v[args[2]])])]
        if op == "vmul": return [("v", args[0], [p * q for p, q in zip(v[args[1]], v[args[2]])])]
        if op == "vmadd":
            return [("v", args[0], [p * q + r for p, q, r in zip(v[args[1]], v[args[2]], v[args[3]])])]
        if op == "vbcast": return [("v", args[0], [s[args[1]]] * VLEN)]
        if op == "vrsum": return [("s", args[0], sum(v[args[1]]))]
        raise ValueError(f"unknown opcode {op!r}")

    def run(self, program):
        """Executes `program` (a list of bundles) and returns its cycle count."""
        ready = {("s", i): 0 for i in range(NUM_SREG)}
        ready.update({("v", i): 0 for i in range(NUM_VREG)})
        cycle = 0
        for bnum, bundle in enumerate(program):
            used = {"scalar": 0, "vector": 0, "load": 0, "store": 0}
            reads = []
            for instr in bundle:
                unit = OP_UNIT[instr[0]]
                used[unit] += 1
                if used[unit] > UNIT_WIDTH[unit]:
                    raise ValueError(f"bundle {bnum}: more than {UNIT_WIDTH[unit]} {unit} op(s): {bundle}")
                r, _w = _operand_regs(instr)
                reads += r
            issue = max([cycle] + [ready[reg] for reg in reads])   # stall until every source is ready
            pending, seen = [], set()
            for instr in bundle:
                unit = OP_UNIT[instr[0]]
                for file, idx, val in self._eval(instr):
                    if (file, idx) in seen:
                        raise ValueError(f"bundle {bnum}: two writes to {file}{idx}")
                    seen.add((file, idx))
                    latency = LOAD_LATENCY if unit == "load" else 1
                    pending.append((file, idx, val, issue + latency))
            for file, idx, val, ready_at in pending:
                (self.s if file == "s" else self.v)[idx] = val
                ready[(file, idx)] = ready_at
            cycle = issue + 1
        self.cycles = cycle
        return cycle
```

### 待优化的内核

给定长度相同的数组 `x`、`y`（长度都是 `n`）和标量 `a`、`b`，内核要计算

```math
S = \sum_{i=0}^{n-1} (a \cdot x_i + b) \cdot y_i .
```

```python
def reference(x, y, a, b):
    """The exact quantity the kernel must produce."""
    return sum((a * xi + b) * yi for xi, yi in zip(x, y))
```

程序的输入通过内存传入，输出也必须写回内存，布局如下：`mem[0:n]` 存 `x`，`mem[n:2n]` 存
`y`，`mem[2n]` 存 `a`，`mem[2n + 1]` 存 `b`，程序必须把 `S` 写到 `mem[2n + 2]`。`n` 可以是任意
非负整数，包括不是 `VLEN` 倍数的情形。

```python
def mem_layout(n):
    """Where each input and the output live, for arrays of length n."""
    return dict(base_x=0, base_y=n, addr_a=2 * n, addr_b=2 * n + 1, addr_out=2 * n + 2)


def make_memory(x, y, a, b):
    n = len(x)
    assert len(y) == n
    return list(x) + list(y) + [float(a), float(b), 0.0]


def run_program(build, n, x, y, a, b):
    """Builds a program for this n, runs it on a fresh VectorMachine, and returns (S, cycles)."""
    machine = VectorMachine(make_memory(x, y, a, b))
    cycles = machine.run(build(n))
    return machine.mem[mem_layout(n)["addr_out"]], cycles
```

下面这段基线内核结果正确但很朴素：每个指令束只有一条指令，一次只处理 `x`、`y` 的一个元素，并且
每次迭代都重新从内存里读一遍 `a` 和 `b`，而不是把它们存进寄存器。

```python
def build_baseline(n):
    S_ADDR_A, S_ADDR_B, S_PTR_X, S_PTR_Y, S_ACC, S_TMP_AB, S_TMP_XY, S_PROD = range(8)
    L = mem_layout(n)
    prog = [
        [("li", S_ADDR_A, L["addr_a"])],
        [("li", S_ADDR_B, L["addr_b"])],
        [("li", S_PTR_X, L["base_x"])],
        [("li", S_PTR_Y, L["base_y"])],
        [("li", S_ACC, 0.0)],
    ]
    for _ in range(n):
        prog += [
            [("sload", S_TMP_AB, S_ADDR_A)],        # re-read a
            [("sload", S_TMP_XY, S_PTR_X)],          # x[i]
            [("mul", S_PROD, S_TMP_AB, S_TMP_XY)],    # a * x[i]
            [("sload", S_TMP_AB, S_ADDR_B)],         # re-read b
            [("add", S_PROD, S_PROD, S_TMP_AB)],      # a * x[i] + b
            [("sload", S_TMP_XY, S_PTR_Y)],           # y[i]
            [("mul", S_PROD, S_PROD, S_TMP_XY)],       # (a * x[i] + b) * y[i]
            [("add", S_ACC, S_ACC, S_PROD)],          # accumulate
            [("addi", S_PTR_X, S_PTR_X, 1)],
            [("addi", S_PTR_Y, S_PTR_Y, 1)],
        ]
    prog += [[("li", S_TMP_AB, L["addr_out"])], [("sstore", S_TMP_AB, S_ACC)]]
    return prog
```

运行 `run_program(build_baseline, n, x, y, a, b)`，对任意 `n` 都能重现 `reference(x, y, a, b)`
的结果，只是周期开销随 `n` 增长。

### 交付物与评分

你有 2 小时。全程可以使用 AI 助手——Claude、ChatGPT、Copilot 或其他任何工具。

写一个函数 `build_optimised(n)`，与上面的 `build_baseline` 遵循相同的约定：给定 `n`，它返回
一个程序，通过 `run_program` 运行后，要对任意 `n`（包括不是 `VLEN` 倍数的大小）都把
`reference(x, y, a, b)` 写到 `mem[2n + 2]`。它的得分是*加速比*（speedup）：

```math
\text{speedup} = \frac{\text{cycles of build\_baseline}(n)}{\text{cycles of build\_optimised}(n)} ,
```

两个周期数都由 `run_program` 在相同的 `n` 上测得。提交 `build_optimised`，并附一份不超过一页
的简短说明，对每一处改动都写清楚：改动前后在某个测试规模下测得的周期数；当时认为瓶颈在哪个发射
单元、依据是什么；以及内核向量化之后，最终的周期数与根据机器自身单元吞吐量推出的下界相比如何。

## 参考解答

<details>
<summary>展开参考解答</summary>

内核全程维护一个*向量*形式的累加器，只在最后才把它归约成一个标量——如果每处理完一个分块就归约
一次，归约本身也要占用一条向量 ALU 指令，与计算指令争抢机器上唯一的向量发射槽位。Step 1 到
Step 3 都假设 `n` 是 `VLEN` 的倍数，这样每种技巧都能单独讲清楚，不必一开始就处理多出来的尾部
元素；Step 4 去掉了这个假设，是应该提交为 `build_optimised` 的版本。下面全部五个版本都在同一
个算例上测量：64 个元素，用固定随机种子生成一次。

### Step 1 —— 打包独立操作，提取公共常量

基线版本有两件事各自多花了一整个指令束，其实都不必如此：两个指针的自增本是相互独立的，却一
次只做一个；而 `a`、`b` 从头到尾都不变，却每轮迭代都要重新从内存读一次。把
`addi(S_PTR_X, ...)` 和 `addi(S_PTR_Y, ...)` 打包进同一个指令束，每个元素就省下一个指令束。把
`a`、`b` 的加载挪到循环之前只做一次，而不是每个元素都做一次，每个元素又省下两个指令束——而且
加载本来就是这里面更贵的一类重复操作，因为不管标量 ALU 还空着多少槽位，每个周期都只能发射一
次加载。

```python
def build_step1(n):
    S_A, S_B, S_PTR_X, S_PTR_Y, S_ACC, S_TMP, S_PROD, S_SPARE = range(8)
    L = mem_layout(n)
    prog = [
        [("li", S_PTR_X, L["base_x"]), ("li", S_PTR_Y, L["base_y"])],
        [("li", S_PROD, L["addr_a"]), ("li", S_TMP, L["addr_b"])],   # S_PROD, S_TMP as scratch addresses
        [("sload", S_A, S_PROD)],
        [("sload", S_B, S_TMP), ("li", S_ACC, 0.0)],
    ]
    for _ in range(n):
        prog += [
            [("sload", S_TMP, S_PTR_X)],
            [("mul", S_PROD, S_A, S_TMP)],
            [("add", S_PROD, S_PROD, S_B)],
            [("sload", S_TMP, S_PTR_Y)],
            [("mul", S_PROD, S_PROD, S_TMP)],
            [("add", S_ACC, S_ACC, S_PROD)],
            [("addi", S_PTR_X, S_PTR_X, 1), ("addi", S_PTR_Y, S_PTR_Y, 1)],
        ]
    prog += [[("li", S_SPARE, L["addr_out"])], [("sstore", S_SPARE, S_ACC)]]
    return prog
```

`run_program(build_step1, 64, x, y, a, b)` 耗费 838 个周期，相比基线的 1223 个周期，是 1.46
倍加速。剩下的指令束里，除了一个，全都挤在同一条依赖链上：加载 `x[i]`、乘、加、乘、加，每一步
都要等前一步的结果。唯一的例外是 `y[i]` 的加载，它只依赖 `S_PTR_Y`——但在这个调度里它同样挪不
到更早的位置，因为它复用的是 `S_TMP`，正是 `x[i]` 加载所用的那个寄存器，提前执行就会在第一次
乘法读到 `x[i]` 之前把它覆盖掉。给独立的操作数配上独立的寄存器，让一次独立的加载真能跑到还在
等别的数据的运算前面，才是这个想法更彻底的版本，Step 3 和 Step 4 用的就是它。不过这里的依赖链
无论怎么调度都是串行的，向量化——把每条指令处理一路数据变成同时处理八路——才是大得多的收益，
也是下一步要做的事。

### Step 2 —— 向量化

把每次处理一个元素，换成每次处理一个 `VLEN` 宽的分块（chunk）。`a`、`b` 在循环之前各广播进一
个向量寄存器一次——和 Step 1 一样的提取手法，只是这次提取的是广播操作而不是标量值本身。累加和
以向量 `V_ACC` 的形式维护，只在存储之前用一次 `vrsum` 归约成标量。

每个分块恰好需要 2 次加载（`x` 的分块一次，`y` 的分块一次）；又因为 `vmadd` 把一次乘法和一次
加法融合进一条指令，恰好也只需要 2 条向量 ALU 指令：先是 `V_X <- a * x + b`，再是
`V_ACC <- (a * x + b) * y + V_ACC`，两步都用 `vmadd` 写成。如果不做融合，改用 `vmul`/`vadd`
分开写同样的算式，每个分块就要 4 条向量 ALU 指令，而加载仍然只有 2 次，向量单元自己就会成为
瓶颈；融合之后两个单元又回到同样的速率——每个分块各 2 条指令，这已经是两者各自能做到的下限，
因为这个内核本来就是一次乘加的结果直接喂给第二次乘加。

```python
V_A, V_B, V_ACC = 0, 1, 2      # persistent vector registers: broadcast a, broadcast b, running sum
P_X, P_Y = 0, 1                # persistent scalar registers: pointers into x and y


def _setup(L):
    a_val, b_val, zero = 4, 5, 6     # scratch, free again once broadcast
    return [
        [("li", P_X, L["base_x"]), ("li", P_Y, L["base_y"])],
        [("li", 2, L["addr_a"]), ("li", 3, L["addr_b"])],
        [("sload", a_val, 2), ("li", zero, 0.0)],
        [("sload", b_val, 3)],
        [("vbcast", V_A, a_val)],
        [("vbcast", V_B, b_val)],
        [("vbcast", V_ACC, zero)],
    ]


def _finish(L, scalar_acc_reg):
    out_addr = 6
    return [[("li", out_addr, L["addr_out"])], [("sstore", out_addr, scalar_acc_reg)]]


def build_step2(n):
    assert n % VLEN == 0
    L = mem_layout(n)
    V_X, V_Y = 3, 4
    prog = _setup(L)
    for _ in range(n // VLEN):
        prog += [
            [("vload", V_X, P_X)],
            [("vload", V_Y, P_Y)],
            [("vmadd", V_X, V_A, V_X, V_B)],          # V_X <- a * x + b
            [("vmadd", V_ACC, V_X, V_Y, V_ACC)],       # V_ACC <- (a * x + b) * y + V_ACC
            [("addi", P_X, P_X, VLEN), ("addi", P_Y, P_Y, VLEN)],
        ]
    prog.append([("vrsum", 4, V_ACC)])
    prog += _finish(L, 4)
    return prog
```

对 64 个元素（8 个分块）来说，这个版本耗费 68 个周期：固定的 12 个周期用于广播常量、归约求和
以及存储结果，之外每个分块还要 7 个周期——比每个分块实际需要的 2 次加载加 2 条向量 ALU 指令要
多。原因是每个分块的第一条 `vmadd` 都要等它依赖的那次 `vload`：这次加载只比它早发射 2 个指令
束，距离 4 周期的加载延迟还差 2 个周期，于是分块在真正开始计算前要先阻塞这 2 个周期。68 个周
期相比基线是 17.99 倍加速。用另一个分块的独立工作填满这些阻塞周期，正是下一步要做的事。

### Step 3 —— 展开循环，填满两个单元

每一轮处理两个分块而不是一个，各自用自己的一对向量寄存器（`V_X0, V_Y0` 和 `V_X1, V_Y1`）：如
果两个分块共用一个寄存器，第二个分块的加载就会在第一个分块的 `vmadd` 读到数据之前把它覆盖掉。
先把这一对分块要用的全部四次加载都发出去，再依次做两个分块各自的一对 `vmadd`：

```python
def build_step3(n):
    assert n % (2 * VLEN) == 0
    L = mem_layout(n)
    V_X0, V_Y0, V_X1, V_Y1 = 3, 4, 5, 6
    prog = _setup(L)
    for _ in range(n // (2 * VLEN)):
        prog += [
            [("vload", V_X0, P_X), ("addi", P_X, P_X, VLEN)],
            [("vload", V_Y0, P_Y), ("addi", P_Y, P_Y, VLEN)],
            [("vload", V_X1, P_X), ("addi", P_X, P_X, VLEN)],
            [("vload", V_Y1, P_Y), ("addi", P_Y, P_Y, VLEN)],
            [("vmadd", V_X0, V_A, V_X0, V_B)],
            [("vmadd", V_ACC, V_X0, V_Y0, V_ACC)],
            [("vmadd", V_X1, V_A, V_X1, V_B)],
            [("vmadd", V_ACC, V_X1, V_Y1, V_ACC)],
        ]
    prog.append([("vrsum", 4, V_ACC)])
    prog += _finish(L, 4)
    return prog
```

等到第一条 `vmadd` 需要用 `V_X0` 时，已经有四个加载指令束发射过了——它自己那次加载，加上另外
三次——所以 `V_X0` 的 4 周期延迟已经完全过去：不会阻塞。这个块里的每一条 `vmadd` 都是如此。但
四次加载和四条 `vmadd` 仍然是两个分开的阶段：四个加载指令束期间向量单元闲着，四个 `vmadd` 指
令束期间加载单元闲着，于是每个单元都只有一半时间在工作。64 个元素耗费 44 个周期，即
`4 * 8 + 12`——每个分块 4 个周期，比 Step 2 的 7 个周期少，阻塞已经消失，但两个单元都没能被持
续占满，这正是下一步要解决的问题。44 个周期相比基线是 27.80 倍加速。

### Step 4 —— 软件流水，并处理尾部

让两个单元在*每一个*周期都有活干，而不是轮流工作，办法是同时处理两个分块：当分块 `i` 的
`vmadd` 正在消费某个轮转寄存器槽位里的数据时，把分块 `i + 2` 的 `vload` 打包进*同一个*指令
束，覆盖掉同一个槽位。读（分块 `i` 已经在那儿的值）和写（分块 `i + 2` 还用不到的值）是这个指
令束里两条不同的指令，所以——正如机器描述里说的——读到的是指令束生效前的旧值：这是安全的，也
正是它让两个向量寄存器就能做到 Step 3 要用四个才能做到的事。一段简短的*前导部分*（prologue）
在稳态开始前把两个槽位都填上分块 0 和分块 1 的数据；一段简短的*收尾部分*（epilogue）在没有什
么可以再预取之后，把已经加载好的最后两个分块处理掉。

```python
def build_step4(n):
    L = mem_layout(n)
    num_chunks, tail = n // VLEN, n % VLEN
    SLOTS = [(3, 4), (5, 6)]        # two rotating (v_x, v_y) register pairs
    V_TMP = 7
    prog = _setup(L)

    if num_chunks == 1:
        vx, vy = SLOTS[0]
        prog.append([("vload", vx, P_X), ("addi", P_X, P_X, VLEN)])
        prog.append([("vload", vy, P_Y), ("addi", P_Y, P_Y, VLEN)])
        prog.append([("vmadd", V_TMP, V_A, vx, V_B)])
        prog.append([("vmadd", V_ACC, V_TMP, vy, V_ACC)])
    elif num_chunks >= 2:
        for k in (0, 1):                                    # prologue: fill both slots
            vx, vy = SLOTS[k]
            prog.append([("vload", vx, P_X), ("addi", P_X, P_X, VLEN)])
            prog.append([("vload", vy, P_Y), ("addi", P_Y, P_Y, VLEN)])
        for i in range(num_chunks - 2):                     # steady state: compute i, prefetch i + 2
            vx, vy = SLOTS[i % 2]
            prog.append([("vmadd", V_TMP, V_A, vx, V_B), ("vload", vx, P_X), ("addi", P_X, P_X, VLEN)])
            prog.append([("vmadd", V_ACC, V_TMP, vy, V_ACC), ("vload", vy, P_Y), ("addi", P_Y, P_Y, VLEN)])
        for i in range(max(num_chunks - 2, 0), num_chunks):  # epilogue: drain the last two chunks
            vx, vy = SLOTS[i % 2]
            prog.append([("vmadd", V_TMP, V_A, vx, V_B)])
            prog.append([("vmadd", V_ACC, V_TMP, vy, V_ACC)])

    scalar_acc = 2
    prog.append([("vrsum", scalar_acc, V_ACC)])
    if tail:
        a_val, b_val, addr_a, addr_b = 3, 4, 5, 6
        prog.append([("li", addr_a, L["addr_a"]), ("li", addr_b, L["addr_b"])])
        prog.append([("sload", a_val, addr_a)])
        prog.append([("sload", b_val, addr_b)])
        prod, tmp = 5, 6         # reuse addr_a's and addr_b's registers once a_val/b_val are loaded
        for _ in range(tail):
            prog += [
                [("sload", prod, P_X)],
                [("mul", prod, a_val, prod)],
                [("add", prod, prod, b_val)],
                [("sload", tmp, P_Y)],
                [("mul", prod, prod, tmp)],
                [("add", scalar_acc, scalar_acc, prod)],
                [("addi", P_X, P_X, 1), ("addi", P_Y, P_Y, 1)],
            ]
    prog += _finish(L, scalar_acc)
    return prog
```

稳态阶段现在每个指令束都恰好发射一条 `vmadd`、一次 `vload` 和一条 `addi`：零阻塞，加载单元和
向量单元同时都跑在各自每周期一次的上限上。对 64 个元素（8 个分块，没有尾部）来说，这是
`2 * 8 + 16 = 32` 个周期——相比基线 38.22 倍加速，也是应该作为 `build_optimised` 返回的版本。
向量化部分处理完之后，剩下的 `n % VLEN` 个尾部元素回落到一个类似 Step 1 逐元素循环的版本，
`a`、`b` 只加载一次并在整个尾部复用：每个尾部元素花费的周期数，与 Step 1 稳态每个元素一样都是
13 个周期，此外还要另加 3 个周期去为尾部取一次 `a`、`b`。`build_step4` 对任何 `n` 都能生成正
确的程序，包括 `n` 不是 `VLEN` 倍数、以及 `n` 小到连一个流水线槽位都填不满的情形。

### 下界

每个分块恰好需要 2 次向量加载；两次乘加融合之后，也恰好只需要 2 条向量 ALU 指令。由于加载单
元和向量单元每个周期都最多只能接受 1 条指令，无论指令束怎么打包，由整块 `VLEN` 宽分块构成的
程序都不可能在少于 `2 * (n // VLEN)` 个周期内处理完 `n // VLEN` 个分块。Step 4 在稳态阶段恰
好达到这个下界——Step 3 没有，它稳定停在恰好两倍的位置，两个单元总有一个闲着一半时间——此外
还要再付出一笔固定开销，实测是 16 个周期，用于一次性广播 `a`、`b`，填充和排空这条两级流水
线，以及最后的归约和存储。这笔固定开销不会随 `n` 增长，所以 `n` 越大它的影响就越小：

| `n` | Step 4 周期数 | `2 * (n // VLEN)` | 比值 |
| --- | --- | --- | --- |
| 64 | 32 | 16 | 2.00 |
| 128 | 48 | 32 | 1.50 |
| 256 | 80 | 64 | 1.25 |
| 512 | 144 | 128 | 1.125 |
| 1024 | 272 | 256 | 1.0625 |

决定下一步该做什么，关键是找出哪个单元在闲着：可以直接从指令束的执行轨迹里读出来，也可以对
每个单元，把它的指令总数和 `cycles * UNIT_WIDTH[unit]` 做比较。达到这个乘积的单元已经饱和，
只有减少它自己的指令数才能再带来提升——比如减少加载次数，或者像 Step 2 从 `vmul`/`vadd` 换
成 `vmadd` 那样融合运算。乘积以下的单元还有余量，靠更好的调度——打包、展开、流水——就能挖出
来，完全不用改变内核算的是什么。把每一步测得的周期数，和由机器自身吞吐量推出的下界做对比，
正应该是说明文档收尾的内容。

### 追问

- **更大的 `LOAD_LATENCY`。** 上面的稳态用一条两级流水线正好隐藏了 4 个周期的延迟（每个分块
  2 次加载 × 提前 2 个分块）。如果 `LOAD_LATENCY = 8`，就需要四级轮转——光是在途分块数据就要
  占 8 个向量寄存器，而两次广播、累加器和临时乘积已经用掉了 4 个：加起来超过机器提供的 8 个
  向量寄存器，只能改用更小、隐藏延迟能力更弱的流水线。
- **存储带宽。** 这个内核只在最后存储一次，所以存储单元从来不会成为候选瓶颈。如果一个内核要
  把完整的逐元素结果写回内存，而不只是一个归约值，就需要像 Step 2 权衡加载和向量 ALU 指令那
  样，权衡向量存储和向量加载。
- **求和顺序。** `vrsum` 按通道顺序对一个向量寄存器里的各条通道求和，累加和本身又是一个分块
  一个分块地累积的，所以总和的求和顺序和 `reference` 对 `x`、`y` 从左到右逐项求和并不相同；
  浮点数加法不满足结合律，两者的结果可能在最后几位上有差异，这也是下面的验证代码用容差比较
  而不是判断完全相等的原因。
- **更紧凑的准备阶段。** 累加和初始零值的广播，其实可以在 `a`、`b` 的加载还在途中时就发出，
  而不必等它们各自广播完成，这样能从上面每个版本的固定开销里再省下一两个周期——在每分块速率
  已经最优之后，这是一点锦上添花的收益。
- **更浅的流水线。** 如果每个操作数只留出一个而不是两个空闲的向量寄存器，一级流水线本可以只
  靠提前一个分块预取来隐藏延迟，前提是处理一个分块本身至少要花 `LOAD_LATENCY` 个周期；而这里
  每个分块只要 2 个周期，达不到这个条件，这就是为什么 Step 4 需要两个轮转槽位而不是一个。

<details>
<summary>验证代码（可运行）</summary>

```python
demo = VectorMachine([1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0])
demo_cycles = demo.run([[("li", 0, 0)], [("vload", 0, 0)], [("vadd", 1, 0, 0)]])
assert demo_cycles == 6 and demo.v[1] == [2.0, 4.0, 6.0, 8.0, 10.0, 12.0, 14.0, 16.0]
print("machine-description trace: OK (6 cycles)")

import random

rng = random.Random(0)
n_demo = 64
x_demo = [rng.uniform(-3, 3) for _ in range(n_demo)]
y_demo = [rng.uniform(-3, 3) for _ in range(n_demo)]
a_demo, b_demo = 1.7, -0.4
want_demo = reference(x_demo, y_demo, a_demo, b_demo)

exact_cycles = [(build_baseline, 1223), (build_step1, 838), (build_step2, 68),
                (build_step3, 44), (build_step4, 32)]
for build, expected in exact_cycles:
    got, cycles = run_program(build, n_demo, x_demo, y_demo, a_demo, b_demo)
    assert abs(got - want_demo) < 1e-6, (build.__name__, got, want_demo)
    assert cycles == expected, (build.__name__, cycles, expected)
print("worked example (n = 64): every version matches the reference; cycle counts as quoted")

baseline_cycles = dict(exact_cycles)[build_baseline]
speedups = {build.__name__: baseline_cycles / cycles for build, cycles in exact_cycles}
for name, quoted in [("build_step1", 1.46), ("build_step2", 17.99), ("build_step3", 27.80),
                      ("build_step4", 38.22)]:
    assert abs(speedups[name] - quoted) < 0.01, (name, speedups[name], quoted)
print("speedups match the quoted figures")

# closed-form cycle counts (also checked for correctness, not just cycles)
for chunks in range(1, 13):
    n = chunks * VLEN
    got2, c2 = run_program(build_step2, n, [1.0] * n, [1.0] * n, 1.0, 0.5)
    assert c2 == 7 * chunks + 12 and abs(got2 - reference([1.0] * n, [1.0] * n, 1.0, 0.5)) < 1e-9
    if chunks % 2 == 0:
        got3, c3 = run_program(build_step3, n, [1.0] * n, [1.0] * n, 1.0, 0.5)
        assert c3 == 4 * chunks + 12 and abs(got3 - reference([1.0] * n, [1.0] * n, 1.0, 0.5)) < 1e-9
    got4, c4 = run_program(build_step4, n, [1.0] * n, [1.0] * n, 1.0, 0.5)
    assert c4 == 2 * chunks + 16 and abs(got4 - reference([1.0] * n, [1.0] * n, 1.0, 0.5)) < 1e-9
for n in range(0, 30):
    want_n = reference([1.0] * n, [1.0] * n, 1.0, 0.5)
    gotb, cb = run_program(build_baseline, n, [1.0] * n, [1.0] * n, 1.0, 0.5)
    got1, c1 = run_program(build_step1, n, [1.0] * n, [1.0] * n, 1.0, 0.5)
    assert cb == 19 * n + 7 and abs(gotb - want_n) < 1e-9
    assert c1 == 13 * n + 6 and abs(got1 - want_n) < 1e-9
print("closed-form cycle counts verified for many sizes, against the reference too")

# Step 4 cycles against the derived lower bound, for growing n
for n, expected_cycles in [(64, 32), (128, 48), (256, 80), (512, 144), (1024, 272)]:
    _, cycles = run_program(build_step4, n, [0.3] * n, [0.2] * n, 1.1, -0.2)
    assert cycles == expected_cycles
    assert cycles >= 2 * (n // VLEN)
print("Step 4 cycle counts against the derived lower bound: as quoted")

# random correctness sweep for build_optimised = build_step4: many sizes, including tails and n = 0
rng2 = random.Random(1)
for _ in range(300):
    n = rng2.choice([0, 1, 2, 3, 7, 8, 9, 15, 16, 17]) if rng2.random() < 0.6 else rng2.randint(0, 90)
    x = [rng2.uniform(-5, 5) for _ in range(n)]
    y = [rng2.uniform(-5, 5) for _ in range(n)]
    a, b = rng2.uniform(-4, 4), rng2.uniform(-4, 4)
    got, cycles = run_program(build_step4, n, x, y, a, b)
    want = reference(x, y, a, b)
    assert abs(got - want) < 1e-6 * max(1.0, abs(want)), (n, got, want)
    assert cycles >= 0
print("random correctness sweep (many sizes, including tails and n = 0): OK")


def expect_error(machine, program, exc):
    try:
        machine.run(program)
        raise AssertionError(f"expected {exc.__name__}")
    except exc:
        pass


expect_error(VectorMachine([0.0] * 32), [[("vload", 0, 0), ("vload", 1, 0)]], ValueError)   # 2 loads
expect_error(VectorMachine([0.0] * 32), [[("li", 0, 1), ("li", 0, 2)]], ValueError)          # dup write
oob_machine = VectorMachine([0.0] * 4)
oob_machine.s[0] = 2
expect_error(oob_machine, [[("vload", 0, 0)]], IndexError)                                   # 2 + VLEN > 4
print("the simulator rejects over-width bundles, duplicate writes and out-of-range addresses")

print("all checks passed")
```

</details>

</details>
