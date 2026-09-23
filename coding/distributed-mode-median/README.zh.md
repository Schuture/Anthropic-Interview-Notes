# 分布式众数、中位数与排序

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 分布式系统，消息传递 | ★★★☆☆ | 中等 | SWE · MLE · Infra Eng | distributed-systems, message-passing, bandwidth, hashing, sorting | 3 个部分 / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

一个集群由 $P = 10$ 个 rank 组成，编号 $0$ 到 $9$，共同持有一份整数数据集。rank $r$ 的*分片*（shard）是一个
包含零个或多个整数的列表 `shard`，其中可能有重复值；各分片的长度可以不同，完整数据集是 0 号分片、1 号分片、
……、9 号分片按此顺序拼接的结果。至少有一个分片非空。每个 rank 预先知道 $P = 10$ 以及自己的 `rank` 编号，
并且每个 rank 都只运行同一个函数一次，调用时传入自己的 `rank`、自己的 `shard`，以及三个已经绑定到该 rank
的原语，因此每次调用都不需要说明是谁在调用：

- `send(dst: int, obj) -> None` 把 `obj` 发给 rank `dst`，随即立刻返回；它不会等待 `dst` 收到这条消息。
- `recv(src: int) -> object` 会阻塞，直到 rank `src` 发来的值可用，然后返回它。同一个 `src` 发给同一个
  rank 的两个对象，到达顺序和调用 `send` 的顺序一致。
- `barrier() -> None` 会阻塞调用它的 rank，直到全部 10 个 rank 都调用了 `barrier()`，然后把它们一起释放。

本题只计一种代价：网络流量。一次 `send(dst, obj)` 调用的代价是 `len(pickle.dumps(obj))` 字节，所有 rank
的每一次调用加总，就是评判方案好坏的依据——有时看总量，在下文特别说明之处，则看单个 rank 发送或接收的最大量。
读取或扫描本地 `shard`、对它排序、在它上面建一个字典，或任何其他本地计算，在这个代价模型下都不花代价，
不论 shard 有多大；只有跨越网络、在 rank 之间传输的部分才计入代价。

只检查 rank 0 的返回值；其余 rank 的返回值一律忽略，但每个 rank 仍必须返回，不能永远阻塞下去。数据集的
*众数*（mode）是十个分片合计计数最高的那个值；如果有多个值并列最高，众数取其中最小的一个。记 $N$ 为十个
分片元素总数，$k = \lfloor (N-1)/2 \rfloor$，*中位数*（median）定义为数据集按非降序排序后、（从 0 开始
计数）下标为 $k$ 的元素——$N$ 为奇数时就是通常意义下的中间元素，$N$ 为偶数时则取中间两个元素中较小的那个
（*下中位数*，lower median）。

### Part 1 —— 全局众数

实现 `global_mode`。记 $D$ 为十个分片里（rank、不同取值）组合的总数——形式化地写作 $D = \sum_{r=0}^{9} d_r$，
其中 $d_r$ 是分片 $r$ 里不同取值的个数，一个值在几个分片里出现，就计几次。正确的方案还必须保证，整个调用
过程中任何单个 rank 接收的字节数都不超过 $O(D/P + P)$。

```py
def global_mode(rank: int, shard: list[int], send, recv, barrier) -> int | None:
    """Every rank calls this once. After every rank has returned, rank 0's return value is the global
    mode of the ten shards combined; every other rank's return value is ignored."""
```

```text
rank 0: [4, 1, 4]
rank 1: [2, 4, 2]
rank 2: [1, 1]
rank 3-9: []
# 合并后：[4, 1, 4, 2, 4, 2, 1, 1] -> 计数为 {4: 3, 1: 3, 2: 2}
# 4 和 1 并列最高，计数都是 3；平局规则保留较小的那个值
global_mode(...) -> 1   # rank 0 上的返回值
```

### Part 2 —— 全局中位数

实现 `global_median`。记 $R$ 为十个分片里最大值与最小值之差再加一。正确的方案还必须保证，所有 rank
发送的字节数总和是 $O\big(P \cdot (\log_2 R + 1)\big)$——尤其是不能随 $N$ 增长。

```py
def global_median(rank: int, shard: list[int], send, recv, barrier) -> int | None:
    """Same calling convention as global_mode. After every rank has returned, rank 0's return value is
    the median as defined above; every other rank's return value is ignored."""
```

```text
rank 0: [10, 1]
rank 1: [7, 3]
rank 2: [15, 6]
rank 3: [8, 2]
rank 4-9: []
# 合并并排序后：[1, 2, 3, 6, 7, 8, 10, 15]（N = 8，为偶数）
# k = (8 - 1) // 2 = 3（从 0 开始计数）-> 取下标为 3 的元素
global_median(...) -> 6   # 中间两个值 6 和 7 里较小的那个
```

### Part 3 —— 分布式排序

实现 `sample_sort`。约定每个 rank 返回后，`result[r]` 表示 rank $r$ 的返回值。把
`result[0], result[1], ..., result[9]` 按 rank 顺序拼接起来，必须不多不少地还原出每个分片的每个元素，
且整体非降序排列——等价地说，`result[r]` 本身有序，并且 `result[r]` 里的每个元素都不大于 `result[r+1]`
里的每个元素。除了这条正确性要求以外，只要各分片的大小和取值都不是极端倾斜，任何单个 rank 的收发流量
都不应超过 $N/P$ 字节的一个较小常数倍，其中 $N$ 是十个分片的元素总数；如果数据被某个超大分片或某个
重复值主导，流量、或者某些 `result[r]`，出现不均衡也是允许的，此时不受带宽要求约束，但仍然必须满足
正确性要求。

```py
def sample_sort(rank: int, shard: list[int], send, recv, barrier) -> list[int]:
    """Every rank calls this once. After every rank has returned, rank r's return value is result[r]
    as defined above."""
```

```text
rank 0: [5, 1]
rank 1: [9, 2]
rank 2: [4]
rank 3-9: []
# 合并并排序后：[1, 2, 4, 5, 9]
# 一种合法结果：result[0] = [1, 2]，result[1] = [4, 5]，result[2] = [9]，result[3..9] = []
# 把这 5 个值切分成另一组同样有序、连续的 10 段，同样合法
```

## 参考解答

<details>
<summary>展开参考解答</summary>

动手写代码之前，有两点值得和面试官确认。一是每个 rank 是否已经把 $P = 10$ 和自己的 `rank` 当作预先知道
的常量，而不是运行时才发现——下面假设是的，与设定完全一致。二是这些值是否为普通的、量级有界的整数，从而
保证 Part 2 里在值域上的搜索一定能落在某个真实元素上——下面假设是的；任意浮点数留作追问。代价模型只计
网络字节数，因此某个 rank 对自己 `shard` 做一次 $O(n \log n)$ 的本地排序是免费的；只有跨 rank 传输的
部分才计费。

### Part 1

把每个元素都发给 rank 0，代价是 $O(N)$ 字节，只要不同取值比元素总数少——而这恰好正是建一个分片直方图
值得做的场景——这个代价就不会比 $O(D)$ 更好。换成把每个 rank 的完整本地直方图发给 rank 0——每个 rank
发一条消息，内容是若干 `(value, count)` 条目——则每个 rank 的发送代价只有 $O(\min(n_r, V))$ 字节，
其中 $V$ 是合并后数据集里不同取值的个数；当取值范围较小（$V \ll N$）时，这已经足够便宜也足够简单，
rank 0 只需要把收到的十份直方图按 key 相加即可。但这些条目最终全部落在 rank 0 上：$V$ 越大，rank 0
承担的流量份额和合并工作量就越大，其余 rank 却始终闲着。

按取值而不是按 rank 来分区，可以解决这个问题。给每个取值指定一个固定的*所有者*（owner）——只由这个值
本身决定、在每个 rank 上都算出同一个结果的某个函数——让每个 rank 只把某个值的本地计数转发给这个值的
所有者，而不是转发给某个固定的协调者。等所有 rank 都这样做完之后，每个所有者会从其余每个 rank 那里，
收到该 rank 对自己负责的每个值的贡献；把这些计数和自己的本地计数加在一起，所有者现在就掌握了自己名下
每个值的*真实*全局计数，完全不需要听到任何与自己无关的值的消息。所有者可以立刻找出自己名下计数最高的
值——一个*局部胜者*（local winner）——最终只需要让每个所有者的一个 `(value, count)` 元组、总共十个数，
到达 rank 0 参与最后的比较。

```python
NUM_RANKS = 10


def _local_histogram(shard: list[int]) -> dict[int, int]:
    counts: dict[int, int] = {}
    for v in shard:
        counts[v] = counts.get(v, 0) + 1
    return counts


def _owner(value: int) -> int:
    return value % NUM_RANKS   # NOTE: Python's % is already non-negative here, negative values included


def global_mode(rank, shard, send, recv, barrier):
    local = _local_histogram(shard)
    outgoing = [[] for _ in range(NUM_RANKS)]
    for value, count in local.items():
        outgoing[_owner(value)].append((value, count))
    for dst in range(NUM_RANKS):
        if dst != rank:
            send(dst, outgoing[dst])    # NOTE: send even [] -- recv(src) below waits specifically for dst

    owned = dict(outgoing[rank])
    for src in range(NUM_RANKS):
        if src != rank:
            for value, count in recv(src):
                owned[value] = owned.get(value, 0) + count

    if owned:
        best_count = max(owned.values())
        winner = min(v for v, c in owned.items() if c == best_count)
        candidate = (winner, owned[winner])
    else:
        candidate = None                # NOTE: this owner's slice of the key space is empty

    if rank != 0:
        send(0, candidate)
        return None
    candidates = [candidate] + [recv(src) for src in range(1, NUM_RANKS)]
    best_value, _ = min((c for c in candidates if c is not None), key=lambda vc: (-vc[1], vc[0]))
    return best_value
```

每个 rank 本地直方图里的每个 `(value, count)` 条目都只发送一次，发给它唯一的所有者，所以整个 shuffle
总共搬运 $D$ 个条目；只要取值不是集中在同一个模 $P$ 的余数上，每个所有者收到的条目数就接近 $D/P$，加上
最后十候选人那一轮给每个 rank 再添的 $O(P)$，正好是题目要求的 $O(D/P + P)$ 上界。实测在 $N = 4{,}000$、
取值有 2,000 个不同值时，这个方案里最忙的 rank 收到 2,551 字节，而让每个 rank 把直方图直接发给 rank 0，
则要收到 22,576 字节——几乎是前者的九倍。当不同取值只有 5 个时，发送原始元素而不是直方图要花费 7,344
字节，对比直方图的 414 字节，几乎是它的十八倍，说明在取值稀少时真正重要的是压缩成计数，而不是这些计数
落在哪个 rank 上。

### Part 2

把原始元素收集起来的代价是 $O(N)$ 字节，这正是要避免的。先在本地给每个分片排序，再在 rank 0 上合并十个
有序列表，并不能解决问题：合并这一步在 CPU 时间上确实更便宜，但每个元素依然要实实在在地跨网络到达
rank 0，字节代价并没有变。真正能避免搬运数据的做法，是永远不问“数据是什么”，只问“比某个猜测值 $g$ 小
的数据有多少”——这是每个 rank 只凭自己的分片就能回答的问题，答案只是两个小数字。

这样一来，求中位数就变成了在*取值*范围上做二分查找，而不是在数组下标上做二分。每个 rank 先把自己的
本地最小值、最大值和元素个数报给 rank 0，rank 0 把它们合并成全局范围 $[\mathit{lo}, \mathit{hi}]$ 和
全局个数 $N$，并一次性提前算出 $k = \lfloor(N-1)/2\rfloor$——也就是答案在排序结果里的*目标名次*，它在
后续搜索过程中始终不变（对比数组上的 quickselect：它每次都递归进一个不断缩小的子数组，每一步都要调整
$k$；这里的 $k$ 始终是相对于同一份完整数据集来衡量的，所以保持不变）。每一轮，rank 0 把猜测值 $g$ 发给
其余每个 rank；每个 rank 回报自己有多少元素小于 $g$、多少元素等于 $g$；rank 0 把十个 rank 的这两个计数
分别求和。如果小于 $g$ 的元素超过 $k$ 个，说明答案更小，搜索范围收缩到 $[\mathit{lo}, g - 1]$。否则，
如果小于等于 $g$ 的元素已经覆盖了第 $k$ 个位置，那么 $g$ 本身就是答案——它的若干次出现恰好跨越了排序后
第 $k$ 个位置。否则答案更大，搜索范围收缩到 $[g + 1, \mathit{hi}]$。每一轮都会让 $[\mathit{lo}, \mathit{hi}]$
严格变小，所以整个过程会在 $O(\log_2 R)$ 轮内收敛，其中 $R = \mathit{hi} - \mathit{lo} + 1$ 是初始
范围的大小。

```python
def global_median(rank, shard, send, recv, barrier):
    local_sorted = sorted(shard)
    local_min = local_sorted[0] if local_sorted else None
    local_max = local_sorted[-1] if local_sorted else None

    if rank != 0:
        send(0, (local_min, local_max, len(shard)))
    else:
        lo, hi, total_n = local_min, local_max, len(shard)
        for src in range(1, NUM_RANKS):
            r_min, r_max, r_len = recv(src)
            total_n += r_len
            if r_len:                      # NOTE: an empty shard contributes no bound on lo/hi
                lo = r_min if lo is None else min(lo, r_min)
                hi = r_max if hi is None else max(hi, r_max)
    barrier()                              # every rank now has (or, on rank 0, knows) the global range

    if rank != 0:
        while True:
            guess = recv(0)
            if guess is None:              # NOTE: rank 0's signal that the answer is already found
                return None
            less = sum(1 for v in local_sorted if v < guess)
            equal = sum(1 for v in local_sorted if v == guess)
            send(0, (less, equal))

    k = (total_n - 1) // 2
    answer = None
    while answer is None:
        guess = (lo + hi) // 2
        for dst in range(1, NUM_RANKS):
            send(dst, guess)
        total_less = sum(1 for v in local_sorted if v < guess)
        total_equal = sum(1 for v in local_sorted if v == guess)
        for src in range(1, NUM_RANKS):
            r_less, r_equal = recv(src)
            total_less += r_less
            total_equal += r_equal
        if total_less > k:
            hi = guess - 1
        elif total_less + total_equal > k:
            answer = guess
        else:
            lo = guess + 1
    for dst in range(1, NUM_RANKS):
        send(dst, None)                    # NOTE: every worker is blocked in recv(0) -- release them all
    return answer
```

每一轮的代价是 $O(P)$ 字节——发出一个猜测值，收回其余九个 rank 各自的 `(less, equal)` 数对——一共
$O(\log_2 R)$ 轮，再加上最初交换本地范围的 $O(P)$：总计 $O\big(P \cdot (\log_2 R + 1)\big)$，与 $N$
无关。实测把取值范围固定在 1,000，让 $N$ 从 200 增长到 100,000，二分法的字节总量从 3,105 移动到
2,988——基本持平，因为它由 $R$ 决定，与 $N$ 无关——而直接收集全部数据则从 636 增长到 247,318，多了
将近 390 倍，几乎跟上了 $N$ 自身 500 倍的增长；在最大规模下，收集全部数据的代价是二分法的八十多倍。
不过这个固定开销并不是免费的：当只有 200 个元素时，收集全部数据（636 字节）仍然比二分法十轮的开销
（3,105 字节）更便宜，所以这套协议只有在 $N$ 大到让 $O(P \log_2 R)$ 真正低于 $O(N)$ 时才值得使用，
而不是任何情况下都划算。

### Part 3

把所有数据收集到 rank 0、在那里排序、再把连续的一段发回给每个 rank，这样做是正确的，而且总搬运量
已经是理论最小值——每个元素进来时跨一次网络，为了到达最终归属的 rank，出去时再跨一次。问题在于这些
流量都流向了同一个地方：全部 $2N$ 字节都要经过 rank 0，它既是发送最繁忙的一方，也是接收最繁忙的一方，
其余九个 rank 只能等着。

*样本排序*（sample sort）保留了每个元素跨网络两次的次数，但把它们分散到全部十个 rank 上，而不是都挤
过同一个。每个 rank 从自己的分片里抽取一个小的本地样本，发给 rank 0 的是这个样本——不是整个分片；
rank 0 把这些样本合并、排序，并按等间距的位置挑出九个*分割点*（splitter），把取值范围切成十个*桶*
（bucket）：不超过第一个分割点的值属于 0 号桶，介于第一、第二个分割点之间的值属于 1 号桶，依此类推。
rank 0 把这些分割点广播给所有人。之后每个 rank 按桶划分*自己*的分片，并通过一轮全对全（all-to-all）
交换，把每个桶对应的那部分发给拥有该桶的 rank——一个元素直接从它的来源走到它最终归属的 rank，除非
rank 0 恰好是来源或归属之一，否则完全不经过 rank 0。最后每个 rank 把自己收到的、属于自己那个桶的数据
排序。

```python
import bisect
import random

SAMPLE_SIZE = 8


def _bucket_of(value: int, splitters: list[int]) -> int:
    return bisect.bisect_right(splitters, value)   # NOTE: ties go to the bucket starting at that splitter


def sample_sort(rank, shard, send, recv, barrier):
    rng = random.Random(rank)     # NOTE: seeded by rank alone, so the sample is reproducible regardless
    sample = rng.sample(shard, min(len(shard), SAMPLE_SIZE))   #      of thread-scheduling order

    if rank != 0:
        send(0, sample)
    else:
        combined = sorted(sample + [v for src in range(1, NUM_RANKS) for v in recv(src)])
        splitters = [combined[i * len(combined) // NUM_RANKS] for i in range(1, NUM_RANKS)] if combined else []
        for dst in range(1, NUM_RANKS):
            send(dst, splitters)

    if rank != 0:
        splitters = recv(0)
    barrier()                     # everyone now has the same splitters before the exchange begins

    outgoing = [[] for _ in range(NUM_RANKS)]
    for v in shard:
        outgoing[_bucket_of(v, splitters)].append(v)
    for dst in range(NUM_RANKS):
        if dst != rank:
            send(dst, outgoing[dst])   # NOTE: send even [] -- every rank recv's from every other rank once
    mine = outgoing[rank]
    for src in range(NUM_RANKS):
        if src != rank:
            mine.extend(recv(src))
    mine.sort()
    return mine
```

抽样和广播分割点的代价是每个 rank $O(P \cdot \min(S, n_r))$，其中样本大小 $S$ 是一个小常数；交换阶段
把每个元素恰好搬运一次到它最终归属的 rank，总计 $O(N)$，并且——只要抽样得到的分割点把取值范围分得
大致均匀——经过任何一个 rank 的流量都接近 $N/P$，而不是像之前那样全部 $N$ 个都堆在 rank 0 上。一个
rank 自己的分片大小也是它必须发送的下限：不管用什么分区方式，都不可能让一个起始就有 $n_r$ 个元素的
rank 发送少于 $n_r$ 字节，所以这个保证还额外要求各分片大小本身就在 $N/P$ 的常数倍以内——这一点不同于
Part 1 和 Part 2，它们始终只搬运一份分片的摘要，从不搬运分片本身，因此完全不依赖分片大小。实测在
$N = 50{,}000$ 个均匀分布的取值、且分片大小均匀的情况下，样本排序里最忙的 rank 流量是 31,365 字节，
而先收集、排序、再分发的方案则是 219,390 字节——几乎是它的七倍。但这个保证的好坏同时取决于样本和输入
本身：如果数据集被某一个重复值极度主导，不管分割点落在哪里，这个值的所有出现都会落进同一个桶；如果
数据集被某一个超大分片极度主导，不管桶怎么划分，这个 rank 都必须发送其中的大部分。不论哪种情况，结果
依然正确——每个 `result[r]` 仍然有序，各 rank 之间仍然非降序排列——只是均衡性没有了，而这恰恰是题目
里带宽要求所豁免的情形。

### 追问

- 某个 rank 的分片占了 90% 的数据。Part 1 和 Part 2 的网络开销只取决于不同取值和取值范围的规模，与
  分片大小无关，所以不受影响；受影响的是这个超载 rank 的本地计算量，以及 Part 3 里固定的单 rank 样本
  量——它对这个大分片抽样不足，会让分割点失真。按 `len(shard)` 的比例抽样，或者在运行这三个算法之前，
  先把过大的分片重新分给空闲的 rank，两个问题都能解决。
- 取值是浮点数，且中位数必须精确。Part 2 的搜索依赖答案是二分能够精确落到的一个值；浮点数没有这样的
  保证。这时退回到 Part 3：先跑一遍 `sample_sort`，再让 rank 0 收集十个 `len(result[r])` 值（十个整数），
  直接定位排序后第 $k$ 个位置落在哪个 rank 的那一段里——只需一趟 $O(N)$ 字节，而且结果精确，不必再做
  一次原始数据的收集。
- 取值范围太大，使得 $\log_2 R$ 轮也不便宜，或者取值本身是连续的。这时改用可合并的摘要结构，例如
  t-digest 或 KLL sketch：每个 rank 用 $O(n_r)$ 时间在本地建一个，只发送这个摘要（大小与 $n_r$ 无关），
  rank 0 把十个摘要合并后在合并结果上查询——得到一个误差有界的近似中位数，只需一轮，而不是 $O(\log_2 R)$
  轮。
- Part 1 里的所有者函数，或者 Part 3 里的分割点，在对抗性输入下发生严重碰撞——比如很多不同取值恰好
  共享同一个模 $P$ 余数。乘法哈希（`(value * 2654435761) % NUM_RANKS`）比 `value % NUM_RANKS` 更能把
  相关联的取值分散开，计算代价却同样低；在 Part 3 里，一个明显大于 $N/P$ 的桶还可以在自己更窄的取值
  范围内再做一轮抽样，进一步拆分。
- $P$ 大到连每轮 $O(P)$ 条经过 rank 0 的消息都成问题。这时把广播和归约都改成走一棵 rank 组成的树，
  而不是以一个 rank 为中心的星形结构，就像集合通信里的 all-reduce：每个 rank 只转发给固定数量的子
  节点，等它们都回复之后再回复自己的上级，把集中在一个 rank 上的每轮 $O(P)$ 扇入，换成分摊在每个
  rank 上的 $O(\log P)$ 深度。

<details>
<summary>验证代码（可运行）</summary>

```python
import pickle
import queue
import threading


class Cluster:
    """A minimal synchronous message-passing simulator for NUM_RANKS ranks, run over real threads so a
    blocking recv() genuinely blocks until the matching send() has been made. send(dst, obj) never blocks:
    it serializes obj and enqueues it for dst. recv(src) blocks until a message from that specific src is
    available, then returns it. Every rank's function receives its own send/recv/barrier already bound to
    its rank, so it never has to say who it is."""

    def __init__(self, num_ranks: int = NUM_RANKS):
        self.num_ranks = num_ranks
        self._inboxes = [[queue.Queue() for _ in range(num_ranks)] for _ in range(num_ranks)]  # [dst][src]
        self._barrier = threading.Barrier(num_ranks)
        self._lock = threading.Lock()
        self.bytes_sent = 0
        self.bytes_received = [0] * num_ranks   # bytes landing at each rank -- the per-rank bottleneck

    def run(self, fn, shards: list) -> list:
        results = [None] * self.num_ranks
        errors = [None] * self.num_ranks

        def make_send(rank):
            def send(dst, obj):
                nbytes = len(pickle.dumps(obj))
                with self._lock:
                    self.bytes_sent += nbytes
                    self.bytes_received[dst] += nbytes
                self._inboxes[dst][rank].put(obj)
            return send

        def make_recv(rank):
            def recv(src):
                return self._inboxes[rank][src].get()
            return recv

        def worker(rank):
            try:
                results[rank] = fn(rank, shards[rank], make_send(rank), make_recv(rank), self._barrier.wait)
            except Exception as e:   # surface the real traceback instead of a silent hang on join()
                errors[rank] = e

        threads = [threading.Thread(target=worker, args=(r,)) for r in range(self.num_ranks)]
        for t in threads:
            t.start()
        for t in threads:
            t.join()
        for rank, e in enumerate(errors):
            if e is not None:
                raise RuntimeError(f"rank {rank} raised") from e
        return results


def reference_mode(values: list[int]) -> int:
    counts: dict[int, int] = {}
    for v in values:
        counts[v] = counts.get(v, 0) + 1
    best = max(counts.values())
    return min(v for v, c in counts.items() if c == best)


def reference_median(values: list[int]) -> int:
    s = sorted(values)
    return s[(len(s) - 1) // 2]


def _gather_to_rank0(rank, shard, send, recv):
    """Baseline building block: every rank ships its raw shard to rank 0, which concatenates them.
    Used only to measure the cost of centralizing the data, never as part of an accepted solution."""
    if rank == 0:
        merged = list(shard)
        for src in range(1, NUM_RANKS):
            merged.extend(recv(src))
        return merged
    send(0, shard)
    return None


def gather_all_to_rank0(rank, shard, send, recv, barrier):
    return _gather_to_rank0(rank, shard, send, recv)


def naive_mode_gather_histograms(rank, shard, send, recv, barrier):
    """Baseline: every rank ships its full local histogram to rank 0, which merges them -- cheaper than
    gather_all_to_rank0 when there are few distinct values, but still funnels everything through rank 0."""
    local = _local_histogram(shard)
    if rank == 0:
        merged = dict(local)
        for src in range(1, NUM_RANKS):
            for v, c in recv(src):
                merged[v] = merged.get(v, 0) + c
        best = max(merged.values())
        return min(v for v, c in merged.items() if c == best)
    send(0, list(local.items()))
    return None


def _split_into_contiguous_chunks(items: list, k: int) -> list:
    n = len(items)
    base, extra = divmod(n, k)
    chunks, start = [], 0
    for i in range(k):
        size = base + (1 if i < extra else 0)
        chunks.append(items[start:start + size])
        start += size
    return chunks


def gather_sort_scatter(rank, shard, send, recv, barrier):
    """Baseline: gather everything to rank 0, sort it there, hand each rank a contiguous slice back --
    correct, but every one of the 2N bytes moved passes through rank 0."""
    if rank == 0:
        merged = list(shard)
        for src in range(1, NUM_RANKS):
            merged.extend(recv(src))
        merged.sort()
        chunks = _split_into_contiguous_chunks(merged, NUM_RANKS)
        for dst in range(1, NUM_RANKS):
            send(dst, chunks[dst])
        return chunks[0]
    send(0, shard)
    return recv(0)


def make_shards(rng: random.Random, total_n: int, kind: str = "uniform", value_range: int = 1000) -> list:
    sizes = [total_n // NUM_RANKS] * NUM_RANKS
    for i in range(total_n - sum(sizes)):
        sizes[i] += 1
    if kind == "skewed_sizes" and total_n > 0:
        big = int(total_n * 0.7)
        rest = total_n - big
        sizes = [big] + [rest // (NUM_RANKS - 1)] * (NUM_RANKS - 1)
        for i in range(total_n - sum(sizes)):
            sizes[1 + i] += 1
    shards = []
    for size in sizes:
        if kind == "duplicate_heavy":
            shards.append([7 if rng.random() < 0.85 else rng.randint(0, 5) for _ in range(size)])
        elif kind == "all_same":
            shards.append([42] * size)
        else:
            shards.append([rng.randint(0, value_range - 1) for _ in range(size)])
    return shards


# --- the three worked examples of the statement ---
example1 = [[4, 1, 4], [2, 4, 2], [1, 1], [], [], [], [], [], [], []]
assert Cluster().run(global_mode, example1)[0] == 1

example2 = [[10, 1], [7, 3], [15, 6], [8, 2], [], [], [], [], [], []]
assert Cluster().run(global_median, example2)[0] == 6

example3 = [[5, 1], [9, 2], [4], [], [], [], [], [], [], []]
res3 = Cluster().run(sample_sort, example3)
assert [v for r in res3 for v in r] == [1, 2, 4, 5, 9]
for i in range(NUM_RANKS - 1):
    if res3[i] and res3[i + 1]:
        assert res3[i][-1] <= res3[i + 1][0]

# --- correctness sweeps against independent references, several data shapes, fixed seeds ---
rng = random.Random(0)
for _ in range(30):
    kind = rng.choice(["uniform", "skewed_sizes", "duplicate_heavy", "all_same"])
    n = rng.randint(0, 300)
    shards = make_shards(rng, n, kind=kind, value_range=rng.choice([3, 10, 500]))
    flat = [v for s in shards for v in s]
    if not flat:
        continue
    expected = reference_mode(flat)
    for fn in (global_mode, naive_mode_gather_histograms):
        assert Cluster().run(fn, shards)[0] == expected, (kind, n, fn.__name__)

rng = random.Random(1)
for _ in range(30):
    kind = rng.choice(["uniform", "skewed_sizes", "duplicate_heavy", "all_same"])
    n = rng.randint(1, 300)
    shards = make_shards(rng, n, kind=kind, value_range=rng.choice([2, 10, 500]))
    flat = [v for s in shards for v in s]
    if not flat:
        continue
    assert Cluster().run(global_median, shards)[0] == reference_median(flat), (kind, n)

rng = random.Random(2)  # negative values
for _ in range(15):
    shards = [[rng.randint(-1000, 1000) for _ in range(rng.randint(0, 30))] for _ in range(NUM_RANKS)]
    flat = [v for s in shards for v in s]
    if not flat:
        continue
    assert Cluster().run(global_median, shards)[0] == reference_median(flat)
    assert Cluster().run(global_mode, shards)[0] == reference_mode(flat)

rng = random.Random(3)
for _ in range(20):
    kind = rng.choice(["uniform", "skewed_sizes", "duplicate_heavy", "all_same"])
    n = rng.randint(0, 400)
    shards = make_shards(rng, n, kind=kind, value_range=rng.choice([2, 5, 500]))
    flat_sorted = sorted(v for s in shards for v in s)
    for fn in (sample_sort, gather_sort_scatter):
        results = Cluster().run(fn, shards)
        assert [v for r in results for v in r] == flat_sorted, (kind, n, fn.__name__)
        for i in range(NUM_RANKS - 1):
            if results[i] and results[i + 1]:
                assert results[i][-1] <= results[i + 1][0]

rng = random.Random(4)  # tiny totals, including all-empty and single-element
for n in range(16):
    shards = make_shards(rng, n, kind="uniform", value_range=3)
    flat_sorted = sorted(v for s in shards for v in s)
    assert [v for r in Cluster().run(sample_sort, shards) for v in r] == flat_sorted
    if flat_sorted:
        assert Cluster().run(global_mode, shards)[0] == reference_mode(flat_sorted)
        assert Cluster().run(global_median, shards)[0] == reference_median(flat_sorted)

# --- communication claims, measured ---

# Part 2: bisection stays flat as N grows (range fixed); gathering grows with N
median_records = []
for n in (200, 2_000, 20_000, 100_000):
    rng = random.Random(42)
    shards = make_shards(rng, n, kind="uniform", value_range=1000)
    c1, c2 = Cluster(), Cluster()
    c1.run(global_median, shards)
    c2.run(gather_all_to_rank0, shards)
    median_records.append((n, c1.bytes_sent, c2.bytes_sent))
assert median_records[-1][1] < 5 * median_records[0][1], median_records
assert median_records[-1][2] > 50 * median_records[0][2], median_records
assert median_records[-1][2] > 10 * median_records[-1][1], median_records

# Part 1: the busiest rank's bytes under the shuffle grow much more slowly than under histogram-gather
mode_records = []
for v_count in (5, 50, 500, 2_000):
    rng = random.Random(7)
    shards = make_shards(rng, 4_000, kind="uniform", value_range=v_count)
    c1, c2 = Cluster(), Cluster()
    c1.run(global_mode, shards)
    c2.run(naive_mode_gather_histograms, shards)
    mode_records.append((v_count, max(c1.bytes_received), max(c2.bytes_received)))
assert mode_records[-1][2] > 3 * mode_records[-1][1], mode_records

rng = random.Random(11)  # low cardinality: raw gather vs. histogram gather
shards = make_shards(rng, 4_000, kind="uniform", value_range=5)
c1, c2 = Cluster(), Cluster()
c1.run(gather_all_to_rank0, shards)
c2.run(naive_mode_gather_histograms, shards)
assert c1.bytes_sent > 10 * c2.bytes_sent, (c1.bytes_sent, c2.bytes_sent)

# Part 3: the busiest rank's bytes under sample sort stay near N / P; gather-sort-scatter concentrates at rank 0
sort_records = []
for n in (2_000, 10_000, 50_000):
    rng = random.Random(9)
    shards = make_shards(rng, n, kind="uniform", value_range=1_000_000)
    c1, c2 = Cluster(), Cluster()
    c1.run(sample_sort, shards)
    c2.run(gather_sort_scatter, shards)
    sort_records.append((n, max(c1.bytes_received), max(c2.bytes_received)))
assert sort_records[-1][2] > 5 * sort_records[-1][1], sort_records

print("all checks passed")
```

</details>

</details>
