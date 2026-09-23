# 带断点续跑的加权数据批采样器

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 采样，断点续训 | ★★★☆☆ | 困难 | MLE · RE · RS | sampling, checkpointing, iterators, data-loading | 3 个部分 / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

*样本*（example）是取自若干个命名数据集之一的一个不透明的项，它内部的具体结构在这里无关紧要。
`DataRegistry` 通过一个方法提供对 $K$ 个数据集的访问，每个数据集用一个 `str` 类型的名字标识：

```py
class DataRegistry:
    def get_iterator(self, name: str, offset: int = 0):
        """Returns a fresh iterator over dataset `name`, opened directly at position `offset`;
        opening is fast at any offset, it never walks through the examples it skips. Writing L for
        the number of examples in `name`, the iterator yields the example at index offset % L, then
        (offset + 1) % L, then (offset + 2) % L, and so on forever: once it passes the last example
        it wraps back to the first and keeps going. It never raises StopIteration."""
```

实现 `DataBatcher(registry, weights, batch_size)`，其中 `weights` 是一个 `dict[str, int]`，把每个数据集的
名字映射到一个正整数权重（`weights` 至少有一项），`batch_size` 是一个正整数。把这些名字按字母顺序排列为
$d_1 < d_2 < \cdots < d_K$，记 $w_k$ 为 $\mathrm{weights}[d_k]$，记 $W = \sum_{k=1}^{K} w_k$ 为*总权重*
（total weight）。`next_batch()` 返回一个长度为 `batch_size` 的 `list`，其中的样本按 `weights` 的比例从这
$K$ 个数据集中抽取。下面每一问都遵循同一条关于数据集*之间*顺序的规则——按名字的字母升序排列——并各自给出
数据集*内部*样本顺序的规则。

### Part 1 —— 精确加权批次

这一问假设 `batch_size` 能被 $W$ 整除。对每个 $k$，`next_batch()` 恰好在 batch 中放入数据集 $d_k$ 的
$\mathrm{batch\_size} \cdot w_k / W$ 个样本，这些样本连续排列，顺序就是该数据集自己的迭代器产出的顺序；
这 $K$ 段依次按 $d_1, d_2, \ldots, d_K$ 的顺序拼接。

```py
class DataBatcher:
    def __init__(self, registry: DataRegistry, weights: dict[str, int], batch_size: int) -> None:
        """batch_size is divisible by sum(weights.values()) in this part."""

    def next_batch(self) -> list:
        """Returns batch_size examples: for each name in ascending alphabetical order,
        batch_size * weights[name] // sum(weights.values()) consecutive examples of that dataset."""
```

```text
weights = {"code": 2, "math": 1, "web": 1}      # W = 4
batch_size = 8                                    # 8 能被 4 整除

# code: 8 * 2 // 4 = 4 个样本，math: 8 * 1 // 4 = 2 个，web: 8 * 1 // 4 = 2 个
next_batch() -> [code-0, code-1, code-2, code-3, math-0, math-1, web-0, web-1]
```

### Part 2 —— 断点保存与续跑

实现 `state_dict()` 和 `load_state_dict(state)`。`state_dict()` 返回一个只含普通 `int` 和字符串的
`dict`——体积很小、可以直接 JSON 序列化，大小只由 $K$ 决定，不会随着已经产出的 batch 数量增长——其中恰好
记录了续跑所需的全部信息。用同样的 `registry`、`weights`、`batch_size` 构造出的 `DataBatcher` 调用
`load_state_dict` 后，会移动到该状态描述的位置，使得它下一次调用 `next_batch()` 返回的结果，与该状态取自
的那个 batcher 接下来会返回的结果完全一致，无论两者是不是同一个 Python 对象。恢复过程不能对已经产出过的
样本重新迭代一遍：它要用 `offset` 直接把每个数据集重新定位到上次结束的地方。

```py
    def state_dict(self) -> dict:
        """A small, JSON-serializable dict sufficient to resume from here."""

    def load_state_dict(self, state: dict) -> None:
        """Moves this batcher to the position state describes."""
```

```text
b = DataBatcher(registry, {"code": 2, "math": 1, "web": 1}, batch_size=8)
b.next_batch()             # 即 Part 1 例子中展示的那个 batch
b.state_dict()              # {"offsets": {"code": 4, "math": 2, "web": 2}}
b.next_batch()              # [code-4, code-5, code-6, code-7, math-2, math-3, web-2, web-3]

# 另一个独立的 batcher，从保存的状态恢复后，续跑结果完全相同：
b2 = DataBatcher(registry, {"code": 2, "math": 1, "web": 1}, batch_size=8)
b2.load_state_dict({"offsets": {"code": 4, "math": 2, "web": 2}})
b2.next_batch()              # [code-4, code-5, code-6, code-7, math-2, math-3, web-2, web-3]
```

### Part 3 —— batch_size 不能整除 W

去掉 Part 1 中「能整除」的假设。`next_batch()` 仍然返回 `batch_size` 个按 `weights` 比例抽取的样本，但每个
数据集的配额 $\mathrm{batch\_size} \cdot w_k / W$ 现在一般不再是整数。`DataBatcher` 增加两个构造参数，
`allocation` 和 `seed`；不论 `allocation` 取什么值，只要 `batch_size` 恰好能被 $W$ 整除，`next_batch()`
仍然退回 Part 1 的精确规则。

不能整除时，batch 改为逐个样本地构造，一共做 `batch_size` 次选择，每次决定下一个样本取自哪个数据集；batch
中的 `batch_size` 个样本按这些选择发生的顺序排列，不再按数据集分块。记 $n$ 为一个 batcher 自构造以来一共
做过多少次这样的选择（累计所有 `next_batch()` 调用，包括当前这次之前的调用），记 $\mathrm{count}_k(n)$ 为
前 $n$ 次选择中选中数据集 $d_k$ 的次数；数据集 $d_k$ 在前 $n$ 次选择中的*理想份额*（ideal share）是
$n w_k / W$，一般不是整数。

- `allocation="deterministic"`：只根据当前各数据集的计数、不引入任何随机性来做选择，使得对每个 $k$ 和每个
  $n$（不仅是 $n$ 恰为 `batch_size` 倍数的时刻），都有 $\lvert \mathrm{count}_k(n) - n w_k / W \rvert \le C$
  成立，其中常数 $C$ 不依赖于 $n$。写出你的规则，并证明这样一个 $C$。
- `allocation="stochastic"`：按同样的字母顺序，对 $w_1, \ldots, w_K$ 建立一个前缀和数组。在 batcher 上保留
  一个 `random.Random(seed)` 实例，每次选择时从中抽取一个 $[0, W)$ 内的均匀整数，再对前缀和数组做二分查找，
  确定它落在哪个数据集上。这时 `state_dict()` 还必须携带足够多的生成器自身状态，使得抽取结果能被精确地
  续接，而不只是让分布重新变得正确。

```py
class DataBatcher:
    def __init__(self, registry: DataRegistry, weights: dict[str, int], batch_size: int,
                 allocation: str = "deterministic", seed: int = 0) -> None:
        """allocation is "deterministic" or "stochastic"; batch_size need not divide
        sum(weights.values()) any more. seed is used only when allocation == "stochastic"."""
```

```text
weights = {"code": 2, "math": 1, "web": 1}, batch_size = 6, allocation = "deterministic"
# W = 4，而 4 不能整除 6

next_batch() -> [code-0, math-0, web-0, code-1, code-2, math-1]
# 最终计数：code = 3（理想值 6*2/4 = 3.0），math = 2（理想值 1.5），web = 1（理想值 1.5）
```

## 参考解答

<details>
<summary>展开参考解答</summary>

开工前有两点值得确认。一是数据集是否真的会无限地循环下去，而不是在耗尽时抛出异常——这里假设会，
正如 `DataRegistry` 所声明的那样，这也是 batcher 本身完全不需要知道数据集长度的原因。二是 `weights` 在一个
`DataBatcher` 的生命周期内是否固定——下面假设是；如果某次运行中途需要换一套权重，就用新权重构造一个新的
batcher，再把旧 batcher 的 `state_dict()` 加载进去。

### Part 1

每个数据集都需要自己的读取位置，因为这 $K$ 个数据集是各自独立、按不同速度被消费的；一个以名字为键的 `dict`
保存这些偏移量，另一个 `dict` 为每个数据集保存一个存活的迭代器，按需惰性打开，这样 `next_batch()` 从未用到
的数据集就完全不会被打开。每次调用都重新 `get_iterator` 也同样可行——registry 保证这很快——但让迭代器在
调用之间保持存活，可以避免在 Part 3 的每一个 `batch_size` 槽位上都重新付出这个开销，写起来也不会更难。

```python
class DataBatcher:
    def __init__(self, registry: "DataRegistry", weights: dict[str, int], batch_size: int) -> None:
        if not weights or any(w <= 0 for w in weights.values()):
            raise ValueError("weights must be a non-empty mapping of positive integers")
        if batch_size <= 0:
            raise ValueError("batch_size must be positive")
        self._registry = registry
        self._names = sorted(weights)                      # NOTE: fixed alphabetical order, used everywhere
        self._weights = dict(weights)
        self._total_weight = sum(weights.values())
        self._batch_size = batch_size
        self._offsets = {name: 0 for name in self._names}  # examples drawn so far, per dataset
        self._iters: dict = {}                              # live iterators, opened lazily, never replayed

    def _iterator_for(self, name: str):
        it = self._iters.get(name)
        if it is None:                                       # NOTE: opened once per dataset, at its offset
            it = self._registry.get_iterator(name, self._offsets[name])
            self._iters[name] = it
        return it

    def _take(self, name: str, count: int) -> list:
        it = self._iterator_for(name)
        items = [next(it) for _ in range(count)]
        self._offsets[name] += count
        return items

    def next_batch(self) -> list:
        batch = []
        for name in self._names:
            quota = self._batch_size * self._weights[name] // self._total_weight
            batch.extend(self._take(name, quota))
        return batch
```

`next_batch()` 的开销是 $O(\mathrm{batch\_size})$：`_take` 从不会比它实际返回的样本多迭代一步，打开一个
迭代器则按假设是 $O(1)$。batcher 自身的状态是 $O(K)$，就是两个小 `dict`，与已经产出过多少个 batch 无关。

### Part 2

保存的状态就只有 `_offsets` 这一个 dict：另外两样看起来也像是「状态」的东西被刻意排除在外。存活的迭代器
本身就无法序列化——生成器既不能写成 JSON，也不能被复制——而且它们能给状态添加的信息，也不过是
`get_iterator` 本来就接受的那同一个整数偏移量；把它们排除在外，正是 `state_dict()` 能保持 $O(K)$、而不随
产出样本数增长的原因。`weights` 和 `batch_size` 也被排除在外，因为它们是「被恢复」那个 batcher 的构造
参数，而不是「它已经走到哪里」这一事实的一部分；一个被恢复的 batcher 在调用 `load_state_dict` 之前，早已
用它们构造完毕。

```python
class DataBatcher(DataBatcher):
    def state_dict(self) -> dict:
        return {"offsets": dict(self._offsets)}             # NOTE: a copy -- callers may hold on to this

    def load_state_dict(self, state: dict) -> None:
        self._offsets = dict(state["offsets"])
        self._iters = {}          # NOTE: dropped, not replayed -- reopened lazily at the restored offsets
```

在 `load_state_dict` 里选择惰性重开、而不是立即重开，意味着恢复这一步本身从不调用 `get_iterator`：开销
落在 `next_batch()` 接下来真正用到的数据集上，每个被用到的数据集只需付出一次，且每次都直接落在保存的
偏移量上——每个被用到的数据集是 $O(1)$，永远不会是 $O(n)$（$n$ 为此前已产出的样本数），不需要任何特殊
处理就满足了 Part 2 的要求。

### Part 3

**确定性做法。** 为每个数据集 $d_k$ 维护它的比值 $p_k(n) = \mathrm{count}_k(n) / w_k$——即按权重归一化
后的计数——每次选择当前比值最小的数据集，比值相同时按字母序较小的名字打破平局。用
$\mathrm{count}_i(n) \cdot w_j$ 与 $\mathrm{count}_j(n) \cdot w_i$ 交叉相乘来比较 $p_i(n)$ 与 $p_j(n)$，
可以让比较保持精确：这里都是不大的整数，但用浮点除法仍可能把两个数学上相等的比值舍入到两个不同的相邻
浮点数，从而把平局判错方向。

**引理。** 对任意两个数据集 $d_i, d_j$ 和每个 $n \ge 0$，都有
$\lvert p_i(n) - p_j(n) \rvert \le \max(1/w_i,\, 1/w_j)$。

*证明。* 只看落在 $d_i$ 或 $d_j$ 上的那些选择构成的子序列；其余选择不改变这两个比值。子序列中落在
（比如）$d_i$ 上的每一次选择，都是因为当时 $d_i$ 的比值在*全部* $K$ 个数据集中最小，特别是不大于 $d_j$
的比值——所以限制在这个子序列上看，规则恰好就是「$d_i$、$d_j$ 两者中当前比值更小或打平的那个，按
$1/w_i$ 或 $1/w_j$ 前进一步」，其余 $K - 2$ 个数据集完全不参与。对这个子序列的长度做归纳。长度为 0 时
两个比值都是 0，差为 0。假设结论成立，且（比如）$d_i$ 的比值更小或打平，于是它被选中：记
$M = \max(1/w_i, 1/w_j)$，$g$ 为这次选择前的差 $p_j(n) - p_i(n)$，归纳假设给出 $0 \le g \le M$。选择
之后差变为 $g - 1/w_i$，它落在 $[-1/w_i,\ M - 1/w_i] \subseteq [-M, M]$ 中，因为 $1/w_i \le M$。
$\blacksquare$

**推论。** 对每个 $k$ 和每个 $n$，都有 $\lvert \mathrm{count}_k(n) - n w_k / W \rvert \le 1 + K$。

*证明。* 由于 $n = \sum_j \mathrm{count}_j(n) = \sum_j w_j\, p_j(n)$，两边除以 $W$，就把 $n / W$ 写成了
$p_j(n)$ 以 $w_j / W$（和为 1）为权重的加权平均，于是

```math
p_k(n) - \frac{n}{W} = \sum_{j=1}^{K} \frac{w_j}{W}\bigl(p_k(n) - p_j(n)\bigr).
```

由引理，$\lvert p_k(n) - p_j(n) \rvert \le \max(1/w_k, 1/w_j) \le 1/w_k + 1/w_j$，于是

```math
\left\lvert p_k(n) - \frac{n}{W} \right\rvert
  \le \sum_{j=1}^{K} \frac{w_j}{W}\left(\frac{1}{w_k} + \frac{1}{w_j}\right)
  = \frac{1}{w_k}\underbrace{\sum_j \frac{w_j}{W}}_{=\,1} + \underbrace{\sum_j \frac{1}{W}}_{=\,K/W}
  = \frac{1}{w_k} + \frac{K}{W}.
```

两边乘以 $w_k$，得到 $\lvert \mathrm{count}_k(n) - n w_k / W \rvert = w_k \lvert p_k(n) - n/W \rvert
\le 1 + K w_k / W \le 1 + K$，因为 $w_k \le W$。$\blacksquare$

这个界在实践中相当宽松：在下面验证代码遍历的大量随机权重组合和数千个槽位上，观察到的最大偏差都不到 2，
与 $K$ 无关。

```python
import random
from bisect import bisect_right


class DataBatcher(DataBatcher):
    def __init__(self, registry: "DataRegistry", weights: dict[str, int], batch_size: int,
                 allocation: str = "deterministic", seed: int = 0) -> None:
        if allocation not in ("deterministic", "stochastic"):
            raise ValueError(f"unknown allocation {allocation!r}")
        super().__init__(registry, weights, batch_size)
        self._allocation = allocation
        self._prefix: list[int] = []          # cumulative weight, same alphabetical order as self._names
        running = 0
        for name in self._names:
            running += self._weights[name]
            self._prefix.append(running)
        self._rng = random.Random(seed)

    def next_batch(self) -> list:
        if self._batch_size % self._total_weight == 0:
            return super().next_batch()                        # NOTE: Part 1's exact rule, unchanged
        pick = self._pick_deterministic if self._allocation == "deterministic" else self._pick_stochastic
        batch = []
        for _ in range(self._batch_size):
            batch.extend(self._take(pick(), 1))
        return batch

    def _pick_deterministic(self) -> str:
        best = self._names[0]
        for name in self._names[1:]:
            # count[name] / weights[name] < count[best] / weights[best], compared by cross-multiplying
            lhs = self._offsets[name] * self._weights[best]
            rhs = self._offsets[best] * self._weights[name]
            if lhs < rhs or (lhs == rhs and name < best):
                best = name
        return best

    def _pick_stochastic(self) -> str:
        draw = self._rng.randrange(self._total_weight)   # NOTE: an int in [0, W) -- no float edge case
        return self._names[bisect_right(self._prefix, draw)]

    def state_dict(self) -> dict:
        state = super().state_dict()
        if self._allocation == "stochastic":
            version, ints, gauss = self._rng.getstate()
            state["rng_state"] = [version, list(ints), gauss]    # NOTE: tuple -> list, so json.dumps works
        return state

    def load_state_dict(self, state: dict) -> None:
        super().load_state_dict(state)
        if self._allocation == "stochastic" and "rng_state" in state:
            version, ints, gauss = state["rng_state"]
            self._rng.setstate((version, tuple(ints), gauss))
```

这两种分配方式的取舍并不相同。确定性方案完全没有方差——推论约束的是它的最坏情形，而不只是平均情形——
它的状态就是 Part 2 已经保存的那些整数偏移量，什么时候做检查点对它没有任何影响。它打破平局的方式也每次
都一样，所以在一串比值打平的选择上（权重相等或成比例时），每次出现完全相同的平局，赢的都是同一个数据
集，使得同样的数据在不同 epoch 里，batch 内部的选择模式本身也是固定的。随机方案是无偏的——$d_k$ 在 $n$
次抽取后的期望计数恰好是 $n w_k / W$——但它的实际计数是 $n$ 次相互依赖的随机抽取之和，围绕这个期望值
存在真实的方差，没有任何东西能像推论那样约束某一次具体运行的偏差。这种随机性本身也必须是可复现的，才
谈得上可以做检查点：等做检查点的时候，`random.Random` 对象早已往前走了若干步，所以 `state_dict()` 必须
捕获它*当前*的状态，而不是最初的 `seed`——只靠 `seed` 去恢复，会把已经抽取过的结果重新抽一遍、把已经
产出过的 batch 重新产出一遍，而不是接着往下走。

确定性方案每个槽位选一个数据集的开销是 $O(K)$（线性扫描；也可以用一个按比值排序的堆做到 $O(\log K)$，
在 $K$ 较大时更划算），随机方案每个槽位的二分查找则是 $O(\log K)$；`state_dict()` 无论哪种方案都保持
$O(K)$，随机方案只是再加上一份固定的、很小的生成器状态。

### 追问

- 某个数据集比它在一个 batch 里的配额还短：循环回绕意味着它会在这个 batch 内部直接重复出现——这是
  registry 对 `DataBatcher` 不可见地完成的，后者根本不知道任何数据集的长度。如果一个 batch 内部不能接受
  重复，`get_iterator` 就得改成耗尽时抛异常而不是回绕，`next_batch()` 也就得相应地把这个 batch 缩短或者
  直接失败：这是对环境契约本身的改动，不是 `DataBatcher` 自己能解决的。
- 训练中途权重发生变化：用新的 `weights` 构造一个新的 `DataBatcher`，再把旧 batcher 的 `state_dict()`
  加载进去——状态只是各数据集的偏移量，从不绑定任何具体的权重。Part 3 的确定性规则此后会用*新*权重去
  衡量每个比值，因此权重刚被调高的那个数据集会显得人为地落后，从而经历一段短暂的、用来追赶的额外抽取；
  推论给出的界依然成立，只是现在的 $n = 0$ 锚定在权重变化的那一刻，而不是 batcher 最初构造的时刻。
- 在数据并行的多个 rank 之间分片、且互不重叠：在构造时就给 $R$ 个 rank 中的每个 rank $r$ 分配每个数据集
  偏移量范围里专属的一段——对长度为 $L$ 的数据集，偏移量落在
  $\bigl[\, r \lfloor L/R \rfloor,\ (r+1)\lfloor L/R \rfloor \,\bigr)$ 之内——并且回绕时只在这一段自己的
  长度内进行，而不是对整个 $L$ 取模。这样一来，任何一个 rank 调用 `get_iterator` 用到的偏移量，都不会有
  另一个 rank 也用到；代价是当 $R$ 不能整除 $L$ 时，各个 rank 要经过略有不同的样本数才会回到自己那一段
  的起点。
- 上线之前：针对 Part 1，在大量随机的 `(weights, batch_size)` 组合上用暴力解法对 `next_batch()` 做差分
  测试；针对 Part 3，用推论给出的界做同样的校验；在两种分配方式下，对一次长时间运行的每一个 batch 边界
  都做暂停与恢复，并与不中断的运行结果逐一比对；把 `state_dict()` 完整地过一遍 `json.dumps` /
  `json.loads`；再模糊测试短于单个槽位的数据集长度、只有一个数据集、以及小于 $K$ 的 `batch_size` 这些
  情形，并且在两种 `PYTHONHASHSEED` 下都跑一遍。

<details>
<summary>验证代码（可运行）</summary>

```python
import json
import random


class FakeDataset:
    """A finite dataset that wraps around forever, exactly as DataRegistry promises."""

    def __init__(self, data: list) -> None:
        self.data = list(data)

    def get_iterator(self, offset: int = 0):
        n = len(self.data)
        i = offset % n
        while True:
            yield self.data[i]
            i = (i + 1) % n


class FakeRegistry:
    def __init__(self, datasets: dict) -> None:
        self._datasets = {name: FakeDataset(items) for name, items in datasets.items()}

    def get_iterator(self, name: str, offset: int = 0):
        return self._datasets[name].get_iterator(offset)


def make_registry(names, length=97):
    return FakeRegistry({name: [f"{name}-{i}" for i in range(length)] for name in names})


# --- the worked examples, exactly as stated in the Problem section ---
weights = {"code": 2, "math": 1, "web": 1}

b = DataBatcher(make_registry(weights), weights, batch_size=8)
batch1 = b.next_batch()
assert batch1 == ["code-0", "code-1", "code-2", "code-3", "math-0", "math-1", "web-0", "web-1"]

state = b.state_dict()
assert state == {"offsets": {"code": 4, "math": 2, "web": 2}}
batch2 = b.next_batch()
assert batch2 == ["code-4", "code-5", "code-6", "code-7", "math-2", "math-3", "web-2", "web-3"]

b_resumed = DataBatcher(make_registry(weights), weights, batch_size=8)
b_resumed.load_state_dict(json.loads(json.dumps(state)))
assert b_resumed.next_batch() == batch2

b3 = DataBatcher(make_registry(weights), weights, batch_size=6, allocation="deterministic")
batch3 = b3.next_batch()
assert [x.split("-")[0] for x in batch3] == ["code", "math", "web", "code", "code", "math"]


# --- Part 1: an independent brute force, built from the statement alone ---
def brute_force_batches(registry, weights, batch_size, num_batches):
    names = sorted(weights)
    total = sum(weights.values())
    assert batch_size % total == 0
    iterators = {name: registry.get_iterator(name, 0) for name in names}
    result = []
    for _ in range(num_batches):
        one = []
        for name in names:
            quota = batch_size * weights[name] // total
            one.extend(next(iterators[name]) for _ in range(quota))
        result.append(one)
    return result


rng = random.Random(0)
for _ in range(60):
    names = [f"d{i}" for i in range(rng.randint(1, 5))]
    w = {name: rng.randint(1, 6) for name in names}
    total = sum(w.values())
    bs = total * rng.randint(1, 4)
    length = rng.randint(3, 40)
    n_batches = rng.randint(1, 4)
    expected = brute_force_batches(make_registry(names, length), w, bs, n_batches)
    batcher = DataBatcher(make_registry(names, length), w, bs)
    got = [batcher.next_batch() for _ in range(n_batches)]
    assert got == expected, (names, w, bs)
print("brute-force sweep OK (Part 1)")


# --- Part 2 & 3: pause at every batch boundary of a long run and compare with an uninterrupted
#     run, for both allocations, with the saved state round-tripped through JSON in between and
#     handed to a fresh instance constructed with a different seed ---
def run_and_compare(weights, batch_size, allocation, num_batches, seed, length=53):
    uninterrupted_batcher = DataBatcher(make_registry(weights, length), weights, batch_size,
                                         allocation=allocation, seed=seed)
    uninterrupted = [uninterrupted_batcher.next_batch() for _ in range(num_batches)]
    for k in range(num_batches + 1):
        warm_up = DataBatcher(make_registry(weights, length), weights, batch_size,
                               allocation=allocation, seed=seed)
        produced = [warm_up.next_batch() for _ in range(k)]
        assert produced == uninterrupted[:k]
        saved = json.loads(json.dumps(warm_up.state_dict()))
        resumed = DataBatcher(make_registry(weights, length), weights, batch_size,
                               allocation=allocation, seed=seed + 12345)  # must be overridden by `saved`
        resumed.load_state_dict(saved)
        rest = [resumed.next_batch() for _ in range(num_batches - k)]
        assert rest == uninterrupted[k:], (allocation, k)


rng = random.Random(1)
for _ in range(20):
    names = [f"d{i}" for i in range(rng.randint(1, 4))]
    w = {name: rng.randint(1, 5) for name in names}
    total = sum(w.values())
    bs = rng.choice([total, total * 2, total + 1, total + 3, max(1, total - 1)])
    allocation = rng.choice(["deterministic", "stochastic"])
    run_and_compare(w, bs, allocation, num_batches=rng.randint(2, 6), seed=rng.randint(0, 1000))
print("pause/resume sweep OK (Part 2 and 3, both allocations)")


# --- Part 3: the deterministic allocator's cumulative-error bound, over many batches and random
#     weights; examples are tagged with their dataset's name so the bound can be checked at every
#     single slot, not only at batch boundaries ---
def sweep_deterministic_bound(trials=200, max_slots=600, seed=2):
    rng = random.Random(seed)
    worst = 0.0
    for _ in range(trials):
        k = rng.randint(1, 8)
        names = [f"d{i}" for i in range(k)]
        w = {name: rng.randint(1, 40) for name in names}
        total = sum(w.values())
        reg = make_registry(names, length=rng.randint(200, 400))
        bs = rng.choice([1, 2, 3, 5, 7])
        n_slots = rng.randint(1, max_slots // bs) * bs
        batcher = DataBatcher(reg, w, batch_size=bs, allocation="deterministic")
        produced = 0
        counts = {name: 0 for name in names}
        while produced < n_slots:
            for item in batcher.next_batch():
                counts[item.split("-")[0]] += 1
                produced += 1
                for name in names:
                    ideal = produced * w[name] / total
                    err = abs(counts[name] - ideal)
                    assert err <= 1 + k + 1e-9, (name, produced, counts[name], ideal, err, k)
                    worst = max(worst, err)
    return worst


worst_error = sweep_deterministic_bound()
print(f"deterministic allocator: worst |count - ideal share| = {worst_error:.3f} (proven bound 1 + K)")
assert worst_error < 6   # comfortably inside 1 + K for every K <= 8 exercised above


# --- Part 3: the stochastic allocator is unbiased in expectation, with a generous statistical margin ---
w = {"a": 5, "b": 3, "c": 2}
batcher = DataBatcher(make_registry(w, length=500), w, batch_size=7, allocation="stochastic", seed=7)
counts = {name: 0 for name in w}
total_examples = 0
for _ in range(3000):
    for item in batcher.next_batch():
        counts[item.split("-")[0]] += 1
        total_examples += 1
total_w = sum(w.values())
for name in w:
    assert abs(counts[name] / total_examples - w[name] / total_w) < 0.02, (name, counts)
print("stochastic expectation check OK")


# --- edge cases ---
# a dataset shorter than a single batch's quota wraps around within that batch
tiny_reg = FakeRegistry({"tiny": ["t0", "t1"], "big": [f"big-{i}" for i in range(50)]})
tiny_batch = DataBatcher(tiny_reg, {"tiny": 1, "big": 1}, batch_size=10).next_batch()
assert [x for x in tiny_batch if x in ("t0", "t1")] == ["t0", "t1", "t0", "t1", "t0"]

# a single dataset wins every slot regardless of allocation, and no tie-break ever matters
solo_reg = FakeRegistry({"only": [f"o-{i}" for i in range(5)]})
solo = DataBatcher(solo_reg, {"only": 3}, batch_size=7, allocation="deterministic")
assert solo.next_batch() == ["o-0", "o-1", "o-2", "o-3", "o-4", "o-0", "o-1"]

# batch_size smaller than the number of datasets: some datasets simply get 0 examples that batch
few = DataBatcher(make_registry(["a", "b", "c", "d"]), {"a": 1, "b": 1, "c": 1, "d": 1},
                   batch_size=2, allocation="deterministic")
assert few.next_batch() == ["a-0", "b-0"]   # the two least-served (tied at 0) alphabetically

# constructor validation
def expect_value_error(**kwargs):
    try:
        DataBatcher(FakeRegistry({"a": [1, 2, 3]}), **kwargs)
    except ValueError:
        return
    raise AssertionError(f"expected ValueError for {kwargs!r}")


expect_value_error(weights={}, batch_size=4)
expect_value_error(weights={"a": 0}, batch_size=4)
expect_value_error(weights={"a": -1}, batch_size=4)
expect_value_error(weights={"a": 1}, batch_size=0)
expect_value_error(weights={"a": 1}, batch_size=4, allocation="uniform")

# state_dict() is a snapshot: mutating the returned dict must not affect the batcher
mut_batcher = DataBatcher(make_registry(weights), weights, batch_size=8)
mut_batcher.next_batch()
snapshot = mut_batcher.state_dict()
snapshot["offsets"]["code"] = 999_999
assert mut_batcher.state_dict()["offsets"]["code"] != 999_999
print("edge cases OK")

print("all checks passed")
```

</details>

</details>
