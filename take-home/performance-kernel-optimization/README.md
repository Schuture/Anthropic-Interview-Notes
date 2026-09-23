# Take-Home: Optimising a Kernel on a Simulated Vector Machine

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Take-home · performance optimisation | ★★☆☆☆ | Hard | Perf Eng | vectorization, loop-unrolling, instruction-scheduling, software-pipelining, simulation | 2 hours | Take-home |
<!-- meta:end -->

## Problem

You are given a simulator for a small vector processor and a kernel that runs correctly but slowly
on it. The task is to rewrite the kernel so that it computes the same result in fewer simulated
cycles, without changing the processor itself.

### The machine

The processor has 8 scalar registers `s0`–`s7`, each holding one number, and 8 vector registers
`v0`–`v7`, each holding `VLEN = 8` lanes of floats. Memory is a flat, word-addressed list of
floats: address `i` names one float, not one byte. There is no branch instruction. A *program* is a
fixed list of *bundles*, built in advance — the way a compiler unrolls a loop whose trip count it
already knows, rather than a stream that a hardware program counter walks with a decision at the
end of every iteration.

A *bundle* is a list of instructions issued together, in a single cycle. Every instruction belongs
to one *issue unit*, and a bundle may carry at most this many instructions of each unit,
simultaneously: 2 scalar-ALU, 1 vector-ALU, 1 load (scalar or vector) and 1 store (scalar or
vector). The instruction set:

| mnemonic | operands | effect | unit |
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
| `vadd` | `vd, vs1, vs2` | `v[vd] = v[vs1] + v[vs2]`, lane by lane | vector |
| `vmul` | `vd, vs1, vs2` | `v[vd] = v[vs1] * v[vs2]`, lane by lane | vector |
| `vmadd` | `vd, vs1, vs2, vs3` | `v[vd] = v[vs1] * v[vs2] + v[vs3]`, lane by lane | vector |
| `vbcast` | `vd, rs` | every lane of `v[vd]` is set to `s[rs]` | vector |
| `vrsum` | `rd, vs` | `s[rd]` = the sum of the `VLEN` lanes of `v[vs]` | vector |

Every instruction in a bundle reads the register file as it stood *before* the bundle, and only
once every instruction has been read from does any of the bundle's writes commit. Two independent
instructions may therefore share a bundle even when one reads a register that another writes in the
same bundle: the reader sees the pre-bundle value. Two instructions may never write the same
register in one bundle.

A written register is not necessarily readable by the very next bundle. `li`, `mov`, `add`, `addi`,
`mul`, `vadd`, `vmul`, `vmadd`, `vbcast` and `vrsum` make their result readable one cycle after the
bundle that produced it issues. `sload` and `vload` take `LOAD_LATENCY = 4` cycles instead. A bundle
whose inputs are not all ready *stalls*: cycles pass with nothing issued until every register the
bundle reads is ready, and only then does it issue. The *cycle count* of a program is one more than
the cycle at which its last bundle issues (an empty program costs 0 cycles, and every bundle —
issued or stalled — advances the cycle counter by at least 1). Memory is not separately
hazard-checked: no program below reads an address before its own prior write to that address has
already completed.

For example, this sequence of bundles:

```text
li(s0, 0)         # no inputs; issues at cycle 0; s0 is readable from cycle 1
vload(v0, s0)     # reads s0 (ready at 1, and cycle 1 is this bundle's earliest slot anyway);
                  #   issues at cycle 1; v0 is a load result, so it is not readable until 1 + 4 = 5
vadd(v1, v0, v0)  # reads v0; its earliest slot would be cycle 2, but v0 is not ready until 5 --
                  #   stalls through cycles 2, 3 and 4, then issues at cycle 5
```

costs 6 cycles: bundles issue at cycles 0, 1 and 5, and the count is `5 + 1`.

The simulator below implements exactly this machine.

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

### The kernel to optimise

Given arrays `x` and `y` of the same length `n` and scalars `a` and `b`, the kernel computes

```math
S = \sum_{i=0}^{n-1} (a \cdot x_i + b) \cdot y_i .
```

```python
def reference(x, y, a, b):
    """The exact quantity the kernel must produce."""
    return sum((a * xi + b) * yi for xi, yi in zip(x, y))
```

A program receives its input through memory and must leave its output there too, laid out as
follows: `mem[0:n]` holds `x`, `mem[n:2n]` holds `y`, `mem[2n]` holds `a`, `mem[2n + 1]` holds `b`,
and the program must write `S` to `mem[2n + 2]`. `n` can be any non-negative integer, including one
that is not a multiple of `VLEN`.

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

The baseline kernel below is correct but naive: one instruction per bundle, one element of `x` and
`y` at a time, and it re-reads `a` and `b` from memory on every iteration instead of keeping them in
a register.

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

Running `run_program(build_baseline, n, x, y, a, b)` reproduces `reference(x, y, a, b)` for every
`n`, at a cycle cost that grows with `n`.

### Deliverables and scoring

You have 2 hours. AI assistants — Claude, ChatGPT, Copilot or any other — may be used throughout.

Write a function `build_optimised(n)` with the same contract as `build_baseline` above: given `n`,
it returns a program that, run through `run_program`, writes `reference(x, y, a, b)` to
`mem[2n + 2]` for every `n`, including sizes that are not a multiple of `VLEN`. Its score is the
*speedup*

```math
\text{speedup} = \frac{\text{cycles of build\_baseline}(n)}{\text{cycles of build\_optimised}(n)} ,
```

both measured by `run_program` on the same `n`. Submit `build_optimised` together with a short
write-up (under one page) that states, for each change made: the cycle count measured before and
after it, at a size tested; which issue unit was believed to be the bottleneck at that point and
why; and, once the kernel is vectorised, how the final cycle count compares to a lower bound derived
from the machine's own per-unit throughput.

## Reference solution

<details>
<summary>Show the reference solution</summary>

The kernel keeps a running *vector* accumulator and reduces it to a single scalar only once, at the
very end — reducing after every chunk would itself spend a vector-ALU instruction, competing with
the arithmetic for the machine's one vector issue slot. Steps 1 through 3 also assume `n` is a
multiple of `VLEN`, so that each technique can be introduced without the leftover elements
complicating it; Step 4 removes that assumption and is the version to submit as `build_optimised`.
All five versions below are measured on the same worked example: 64 elements, drawn once with a
fixed seed.

### Step 1 — Pack independent operations and hoist the constants

The baseline spends one whole bundle each on two things it does not have to: incrementing the two
pointers, one at a time, though they are independent; and — since `a` and `b` never change — reading
them back from memory on every iteration. Packing `addi(S_PTR_X, ...)` and `addi(S_PTR_Y, ...)` into
a single bundle removes one bundle per element. Loading `a` and `b` once, before the loop, rather
than once per element, removes two more per element — and, being loads, those were the more
expensive kind to repeat, since only one may issue per cycle regardless of how many scalar-ALU slots
are free.

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

`run_program(build_step1, 64, x, y, a, b)` costs 838 cycles against the baseline's 1223 — a 1.46×
speedup. Every remaining bundle but one sits on a single dependency chain: load `x[i]`, multiply,
add, multiply, add, each step needing the previous one's result. The exception is the load of
`y[i]`, which only needs `S_PTR_Y` — but it cannot move any earlier in this schedule either, because
it reuses `S_TMP`, the very register `x[i]`'s load used, so moving it up would overwrite `x[i]`
before the first multiply has read it. Giving independent operands independent registers, so that an
independent load really can run ahead of the arithmetic waiting on a different one, is the deeper
version of this idea that Steps 3 and 4 use. Here, though, the chain is serial regardless of
scheduling, and vectorising — replacing one lane of work per instruction with eight — is the far
larger win, and is what the next step does.

### Step 2 — Vectorise

Switch from one element to one `VLEN`-wide chunk at a time. `a` and `b` are broadcast into vector
registers once, before the loop — the same hoisting as Step 1, now applied to the broadcast rather
than the scalar value. The running sum is kept as a vector, `V_ACC`, and reduced to a scalar only
once, with `vrsum`, right before the store.

Each chunk needs exactly 2 loads (one for `x`'s chunk, one for `y`'s) and, because `vmadd` fuses a
multiply and an add into one instruction, exactly 2 vector-ALU instructions: `V_X <- a * x + b`,
then `V_ACC <- (a * x + b) * y + V_ACC`, both written with `vmadd`. Writing the same arithmetic with
unfused `vmul`/`vadd` would need 4 vector-ALU instructions per chunk against only 2 loads, making the
vector unit the bottleneck on its own; fusing brings the two units back to the same rate — 2
instructions each per chunk, the fewest either one can be made to do, since the kernel is one
multiply-add feeding a second multiply-add, by construction.

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

For 64 elements (8 chunks), this costs 68 cycles: a fixed 12 for broadcasting the constants,
reducing the sum and storing it, plus 7 per chunk — above the 2 loads and 2 vector-ALU instructions
each chunk actually needs. The reason is that every chunk's first `vmadd` waits out the `vload` that
feeds it: the load issued only 2 bundles earlier, 2 cycles short of its 4-cycle latency, so the
chunk stalls for those 2 cycles before it can even start computing. 68 cycles is a 17.99× speedup
over the baseline. Filling those stalled cycles with another chunk's independent work is what the
next step does.

### Step 3 — Unroll to fill both units

Process two chunks per pass instead of one, each in its own pair of vector registers (`V_X0, V_Y0`
and `V_X1, V_Y1`): reusing one register for both chunks would let the second chunk's load overwrite
the first chunk's data before the first chunk's `vmadd` has read it. Issue all four loads for the
pair first, then both chunks' pair of `vmadd`s:

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

By the time the first `vmadd` needs `V_X0`, four load bundles have already issued — this one's own
load, plus three more — so `V_X0`'s 4-cycle latency has fully elapsed: no stall. The same holds for
every `vmadd` in the block. But the four loads and the four `vmadd`s are still two separate phases:
the vector unit is idle for all four load bundles, and the load unit is idle for all four `vmadd`
bundles, so each unit is busy only half the time. 44 cycles for 64 elements is `4 * 8 + 12` — 4
cycles per chunk, down from Step 2's 7: the stalls are gone, but neither unit is kept continuously
busy, which is exactly what the next step fixes. 44 cycles is a 27.80× speedup over the baseline.

### Step 4 — Software-pipeline, and handle the tail

Keep both units busy on *every* cycle, instead of alternating between them, by working on two
chunks at once: while chunk `i`'s `vmadd` consumes the data sitting in one of two rotating register
slots, pack a `vload` for chunk `i + 2` into the *same* bundle, overwriting that same slot. The read
(of chunk `i`'s value, already there) and the write (of chunk `i + 2`'s value, not yet needed) are
two different instructions in the bundle, so — as stated in the machine description — the read sees
the pre-bundle value: this is safe, and it is what lets two vector registers do the work that Step 3
spent four on. A short *prologue* fills both slots with chunks 0 and 1 before the steady state
starts; a short *epilogue* drains the last two chunks, already loaded, once there is nothing left to
prefetch.

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

The steady state now issues exactly one `vmadd`, one `vload` and one `addi` in every bundle: zero
stalls, with the load unit and the vector unit both at their one-per-cycle limit simultaneously. For
64 elements (8 chunks, no tail), this is `2 * 8 + 16 = 32` cycles — a 38.22× speedup over the
baseline, and the version to return as `build_optimised`. The `n % VLEN` leftover elements, once the
vectorised part is done, fall back to a version of Step 1's per-element loop, with `a` and `b`
already loaded once and reused for the whole tail: each leftover element costs the same 13 cycles
Step 1's steady state costs, after 3 more cycles to fetch `a` and `b` for the tail. `build_step4`
produces a correct program for every `n`, including `n` not a multiple of `VLEN` and `n` too small
to fill even one pipeline slot.

### The lower bound

Every chunk needs exactly 2 vector loads and, once the two multiply-adds are fused, exactly 2
vector-ALU instructions. Since the load unit and the vector unit each admit at most 1 instruction
per cycle, no program built from whole `VLEN`-wide chunks can process `n // VLEN` of them in under
`2 * (n // VLEN)` cycles, no matter how the bundles are packed. Step 4 reaches this bound exactly in
its steady state — Step 3 does not, sitting at exactly twice it, one unit idle half the time — and
pays a further fixed cost on top, measured at 16 cycles, for the one-off broadcast of `a` and `b`,
filling and draining the two-deep pipeline, the final reduction and the store. That fixed cost does
not grow with `n`, so it matters less as `n` grows:

| `n` | Step 4 cycles | `2 * (n // VLEN)` | ratio |
| --- | --- | --- | --- |
| 64 | 32 | 16 | 2.00 |
| 128 | 48 | 32 | 1.50 |
| 256 | 80 | 64 | 1.25 |
| 512 | 144 | 128 | 1.125 |
| 1024 | 272 | 256 | 1.0625 |

Deciding what to try next means finding which unit is idle: either read it off a bundle trace, or
compare, for each unit, its total instruction count against `cycles * UNIT_WIDTH[unit]`. A unit at
that product is saturated, and only reducing its own instruction count helps further — fewer loads,
or fusing arithmetic, as Step 2's move from `vmul`/`vadd` to `vmadd` does. A unit below it still has
slack that better scheduling can recover — packing, unrolling, pipelining — without changing what
the kernel computes at all. This comparison, cycles measured at each step against the bound derived
from the machine's own throughput, is what the write-up should end with.

### Follow-ups

- **A larger `LOAD_LATENCY`.** The steady state above hides exactly 4 cycles of latency with a
  2-deep pipeline (2 loads per chunk × 2 chunks of lookahead). `LOAD_LATENCY = 8` would need a 4-deep
  rotation — 8 vector registers just for in-flight chunk data, on top of the 4 already spent on the
  two broadcasts, the accumulator and the temporary product: more than the machine's 8 vector
  registers provide, forcing a smaller, less latency-hiding pipeline instead.
- **Store bandwidth.** This kernel stores once, at the very end, so the store unit is never a
  candidate bottleneck. A kernel that wrote a full elementwise result back to memory, rather than
  only a reduction, would need to weigh vector stores against vector loads the same way Step 2 weighs
  loads against vector-ALU instructions.
- **Summation order.** `vrsum` sums a vector register's lanes in lane order, and the running sum
  itself accumulates chunk by chunk, so the total is not summed in the same order as
  `reference`'s left-to-right sum over `x` and `y`; floating-point addition is not associative, so
  the two can differ in the last few bits, which is why the checks below compare with a tolerance
  rather than for exact equality.
- **A tighter setup.** The broadcast of the running sum's initial zero could be issued while `a` and
  `b`'s loads are still in flight, rather than after their broadcasts, shaving a cycle or two off the
  fixed part of every version above — a small further win once the per-chunk rate is already optimal.
- **A shallower pipeline.** With only one vector register free per operand instead of two, a 1-deep
  pipeline could still hide latency by prefetching one chunk ahead, provided a chunk's own processing
  took at least `LOAD_LATENCY` cycles on its own; at 2 cycles per chunk here, it does not, which is
  why Step 4 needs two rotating slots rather than one.

<details>
<summary>Checks (runnable)</summary>

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
