# LRU 记忆化缓存：修 bug 与持久化

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 调试，持久化 | ★★★★☆ | 中等 | SWE · MLE | caching, hashing, persistence, concurrency, serialization | 3 个部分 / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

`Memoize` 包装一个可调用对象 `func`，缓存它的返回值：用相同参数发起的后续调用直接返回缓存的结果，不再调用
`func`——写法上模仿 `functools.lru_cache`。它最多保存 `capacity`（一个正整数）个不同的调用；新增一个调用
超出这个数目时，先淘汰*最近最少使用*（least recently used，即 LRU）的那一项——也就是隔了最久没有被再次
调用过的那个缓存调用。已经在缓存里的调用记为*命中*（hit），不在缓存里的记为*未命中*（miss）；判定谁是
“最近最少使用”时，命中和未命中都算作使用了各自涉及的那一项。

两次调用当且仅当在相同位置上传入相等的位置参数、并且传入的关键字参数按名字和值比较都相等时，才落到同一个
缓存项——不管调用处把关键字写成什么顺序。如果一次调用把某个参数当关键字传入，另一次把它当位置参数传入，
哪怕是给同一个形参传值，这两次调用也绝不算同一次：`generate_key` 只根据拿到的 `*args`/`**kwargs` 构造
键，从不查看 `func` 的签名，所以 `f(1, 2)` 和 `f(1, b=2)` 是两次不同的调用，可以占据不同的缓存项。每一个
位置参数和关键字参数的值都必须是可哈希的，因为它会成为缓存键的一部分；传入一个不可哈希的值（比如一个
`list`）会失败，失败方式和直接把这个值当 `dict` 的键使用完全一样。

下面给出的骨架代码，写法上模仿 `functools.lru_cache`：

```py
from collections import OrderedDict


class Memoize:
    """Wraps func so that repeated calls with the same arguments return a cached result instead of
    calling func again. Holds at most capacity entries; a call that would add a new one beyond that
    evicts the least recently used entry first. Both a hit and a miss count as a "use" of the key
    they touch, for the purpose of deciding which entry is least recently used."""

    def __init__(self, func, capacity):
        self.func = func
        self.capacity = capacity
        self.cache = OrderedDict()
        self.hits = 0
        self.misses = 0

    def generate_key(self, *args, **kwargs):
        key = args + tuple(kwargs.items())
        if len(key) == 1:
            return key[0]
        return key

    def __call__(self, *args, **kwargs):
        key = self.generate_key(*args, **kwargs)
        if key in self.cache:
            self.hits += 1
            return self.cache[key]
        self.misses += 1
        result = self.func(*args, **kwargs)
        self.cache[key] = result
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)
        return result
```

### Part 1 —— 找出并修复三个 bug

上面的代码有三个 bug：两个在 `generate_key` 里，一个在 `__call__` 里。其中两个是写错的行，一个是缺失的
行。三个都修好之后，`generate_key` 和 `__call__` 合起来要满足上面陈述的每一条规则：关键字参数换一种顺序
给出，仍然落到同一个缓存项；不能把两次无关的调用错误地当成同一次，哪怕它们展开后的参数序列一样，只是在位
置参数和关键字参数之间的切分方式不同——`f(1, 2)`（两个位置参数）和 `f((1, 2))`（一个位置参数，它本身是
一个二元组）绝不能碰撞到同一项；而且命中要和未命中一样更新最近使用顺序。对每一个 bug，都要指出它的位置，
说明它让哪些调用错误地碰撞、或错误地没有碰撞，第三个 bug 则要说明它对淘汰顺序的影响，并说明原因；然后修
复它。

```text
cache = Memoize(lambda *args, **kwargs: (args, kwargs), capacity=2)
cache(1, b=2, c=3)   # 未命中                                      cache (LRU -> MRU)：[f(1,b=2,c=3)]
cache(1, c=3, b=2)   # 命中——关键字参数顺序不同：仍是同一次调用，同一个缓存项
cache((1, 2))        # 未命中——只有一个位置参数，它本身是一个二元组
                      #                                    cache：[f(1,b=2,c=3), f((1,2))]
cache(1, 2)           # 未命中——两个位置参数，和 f((1,2)) 不是同一次调用；capacity 已超出，
                      # 所以 f(1,b=2,c=3)（从第一步起就没再被用过）被淘汰
                      #                                    cache：[f((1,2)), f(1,2)]
cache((1, 2))          # 命中——再次用到 f((1,2))，把它移到 MRU 端
                        #                                    cache：[f(1,2), f((1,2))]
cache(1, 3)             # 未命中——capacity 再次超出，所以 f(1,2)（插入之后再没被用过）被淘汰，
                        # 而不是刚被命中过的 f((1,2))：命中和未命中一样会更新最近使用顺序
                        #                                    cache：[f((1,2)), f(1,3)]
```

### Part 2 —— 持久化缓存

从这里开始，`Memoize` 是给定的，三个 bug 都已修好。`DurableMemoize(func, capacity, path=None)` 在此基础
上扩展，使得即便发生崩溃——进程在任意一个时刻被杀死，没有机会再执行后续代码——之前已经完成的调用也不会丢
失，恢复出来的缓存和进程崩溃那一刻持有的缓存无法区分。`path`（如果给出）是这个缓存持久化和恢复所用的基础
路径；默认值 `path=None` 完全关闭持久化，行为和 `Memoize` 一样。当 `path` 指向一个已有的存储时，恢复发生
在 `__init__` 内部，在它返回之前完成。

不允许靠重写整个缓存来持久化每一次调用：记录一次调用所花的时间和空间，必须只和这一次调用本身成正比，绝不
能和 `capacity` 或者之前发生过的调用次数相关。这要求的是一份*只追加日志*（append-only log）：一个只会被
追加、从不就地重写的文件，直到显式调用下面的 `compact()` 为止。

在 `__call__` 返回之前，这次调用的效果——未命中时新增一个缓存项，命中时更新最近使用顺序——必须是*持久
的*（durable）：它的记录已经追加到日志里，并且已经 `flush` 和 `fsync` 过。这一刻之后发生崩溃，这次调用绝
不会丢失；这一刻之前发生崩溃，它会完全丢失，绝不会只丢失一部分，就好像这次调用从未发生过一样。针对这样一
个 `path` 构造 `DurableMemoize(func, capacity, path)`，恢复出来的缓存里的条目、值，以及最近使用顺序（从
最少到最多），都和早先那个实例在它最后一次持久化的调用完成之后所持有的状态完全一致；唯一的例外是记录还没
写完就中断的那次调用，它的效果视同从未发生：进程可能在写一条记录的过程中崩溃，恢复时必须能识别这一
点——凭借记录声明的长度和一个校验和——并丢弃这条记录以及日志里在它之后的所有内容，既不能抛异常，也不能
信任其中任何一个字节。`hits` 和 `misses` 只是统计用的簿记，不会被持久化，每个新实例里都从 0 开始。

`compact()` 可以在任何时候被调用；缓存自己从不会主动调用它。它返回之后，存储的数据量不再取决于 `path` 第
一次被使用以来发生过多少次调用，只取决于当前缓存着的（至多 `capacity` 个）条目。用新的存储状态替换旧的这
个过程是原子的：不管进程在 `compact()` 执行到哪个瞬间崩溃，之后的恢复要么重现 `compact()` 被调用前一刻的
状态，要么重现它正要提交的新状态，绝不会是两者的混合，也绝不会丢失一个此前已经持久化的条目。

参与构成键的位置参数或关键字参数，以及 `func` 的返回值，都必须是*可 JSON 序列化的*（JSON-serialisable）：
`None`、`bool`、`int`、`float`、`str`，或者递归地由这些类型构成的 `list`/`dict`（`dict` 的键本身也必须是
`str`）。参数的值可以精确地往返（round-trip），哪怕其中嵌套着 `tuple`。`func` 的返回值只受这条较弱的限
制：其中嵌套的 `tuple`，在经历一次崩溃和重启之后，会变成元素相同的 `list`，而不再是 `tuple`，因为 JSON
并不单独记录原来是哪一种。

```py
class DurableMemoize(Memoize):
    def __init__(self, func, capacity, path: str | None = None) -> None:
        """path is the base path this cache persists to and recovers from; None (the default)
        disables persistence entirely. Recovery, if path names an existing store, happens here,
        before __init__ returns."""

    def compact(self) -> None:
        """Rewrites the stored state so that its size no longer depends on call history, only on
        the entries currently cached. Safe to call at any time, including right before a crash."""
```

例如：

```text
w = DurableMemoize(lambda n: n * n, capacity=2, path="cache.log")
w(3)                     # 未命中——返回前持久化记录 (3,) -> 9
w(4)                     # 未命中——持久化记录 (4,) -> 16；   cache (LRU -> MRU)：[(3,), (4,)]
w(3)                     # 命中——持久化记录 (3,) 又被用了一次；   cache：[(4,), (3,)]

# 进程在这里被杀死，正好在持久化记录下一次调用 w(5) 的过程中——磁盘上的日志以一个残片结尾：
# 一条格式完好的记录声明了自己的长度，但后面跟着的字节数比这个长度承诺的要少

r = DurableMemoize(lambda n: n * n, capacity=2, path="cache.log")
# r 重放日志：w(3)、w(4) 的记录，以及对 3 的那次触碰（touch）记录都完好并被应用；
# 末尾的残片没有通过长度/校验和检查，被丢弃，它之后可能记录过的任何内容也一并丢弃。
# r 的缓存现在把调用 n=4 映射到 16、调用 n=3 映射到 9，顺序正是从最近最少使用到最近
# 最多使用——也就是崩溃前一刻 w 所处的顺序
```

### Part 3 —— 线程安全的缓存

`ThreadSafeMemoize` 在 `DurableMemoize` 的基础上扩展，在任意数量的线程并发调用它时——不管是同一个键还是
不同的键，不管以怎样的顺序交错，不管调度如何——保持同样可观察到的约定，并多加一条保证：对于给定的一个
键，`func` 绝不会同时运行超过一次，哪怕有好几个线程在这个键还没有缓存项的时候同时用它调用
`ThreadSafeMemoize`。一个线程到达时如果发现另一个线程正在计算这个键的值，它会等待那次计算的结果，而不是
再发起一次多余、重复的计算；这个等待绝不能占着某个无关键调用所需要的锁，因为 `func` 可能耗费任意、未知长
度的时间（一次网络调用，一次很慢的计算），而一个在负载下把互不相关的键都串行化在后面的缓存，就不再有缓存
该有的样子了。`func` 抛出的异常会传播给所有在等这次调用的线程，不会传给其他任何线程；这次调用不会有任何
东西被缓存或持久化。

```py
class ThreadSafeMemoize(DurableMemoize):
    def __init__(self, func, capacity, path: str | None = None) -> None:
        """Same contract as DurableMemoize.__init__."""
```

例如：

```text
sf = ThreadSafeMemoize(fetch_profile, capacity=100, path="cache.log")

# 线程 A：sf(user_id=7)   # 未命中——成为键 (user_id=7,) 的负责者，调用 fetch_profile，
#                        # 且不持有这个缓存的任何锁
# 线程 B：sf(user_id=7)   # 在 A 的调用仍在进行时到达；发现 (user_id=7,) 已经在被计算，
#                        # 于是等待 A 的结果，而不是自己再调用一次 fetch_profile
# 线程 C：sf(user_id=9)   # 一个不同的键——立刻继续执行，不必等待 A 或 B

# fetch_profile(user_id=7) 只运行一次，不管在它执行期间有多少个线程调用了 sf(user_id=7)；
# 它们每一个都能拿到自己的结果，而 fetch_profile(user_id=9) 可以和它同时自由运行
```

## 参考解答

<details>
<summary>展开参考解答</summary>

动手写代码之前，有两点值得先确认。第一，一次把值当位置参数传给某个形参的调用，和一次把同样的值当关键字传
给同一个形参的调用，是否应该落到同一个缓存项——这里的答案是不会，因为 `generate_key` 拿不到 `func` 的签
名（先用 `inspect.signature(func)` 绑定参数就可以把两者统一起来；见后面的追问）。第二，“持久”要多严
格：崩溃之后，最近使用顺序是否可能被看到出现倒退（这里的答案是不可以——命中和未命中一样持久，代价是
Part 2 复杂度小节给出的那次 `fsync` 开销），以及值是否可以限制为 JSON、而不是 `pickle`（这里选择了
JSON，因为反序列化 `pickle` 会对
磁盘上恰好留存的任意字节执行任意代码，一条被崩溃损坏的记录在决定是否信任它之前就已经不安全，而一条损坏的
JSON 记录只会解析失败）。

### Part 1

Bug 1，在 `generate_key` 里：`kwargs.items()` 被按 Python 迭代它的顺序——也就是插入顺序，即调用者恰好把
关键字写成的那个顺序——折叠进键里。先按名字排序就能去掉这种依赖：`sorted(kwargs.items())` 逐个元素比较
这些元组，而由于字典的键是唯一的，这样构造出来的两个元组永远不会比较到第一个元素之后，所以哪怕参数值本身
不可比较大小，排序也不会失败。

Bug 2，同样在 `generate_key` 里：把 `args` 和 `kwargs.items()` 拍平成一个元组，一旦这个拍平后的元组恰好
只有一个元素，两者之间的边界就丢失了——原因是 `len(key) == 1` 这条特殊处理会把 `key[0]` 解包返回。
`f(1, 2)` 单靠 `args` 就构造出二元组 `(1, 2)`；`f((1, 2))` 构造出的是一元组 `((1, 2),)`，而这条特殊处理
会把它解包成一模一样的 `(1, 2)`。把 `args` 和排序后的关键字参数对保留成一个外层元组里两个分开的元素，而
不是把它们拍平在一起，就从根源上修复了这个问题（见下面代码里的注释）。

Bug 3，在 `__call__` 里：命中时直接返回 `self.cache[key]`，没有调用 `self.cache.move_to_end(key)`，于是
`OrderedDict` 的顺序反映的始终只是*插入*顺序，从不反映*使用*顺序——同一个键反复命中，它在字典里的位置
始终停留在第一次插入的地方，`popitem(last=False)` 淘汰的也就一直是插入最早的那个键；一旦有任何键被使用超
过一次，这就变成了先进先出（FIFO），而不是 LRU。

```python
from collections import OrderedDict
from typing import Any, Callable


class Memoize:
    """Wraps func so that repeated calls with the same arguments return a cached result instead of
    calling func again. Holds at most capacity entries; a call that would add a new one beyond that
    evicts the least recently used entry first. Both a hit and a miss count as a "use" of the key
    they touch, for the purpose of deciding which entry is least recently used."""

    def __init__(self, func: Callable, capacity: int) -> None:
        self.func = func
        self.capacity = capacity
        self.cache: "OrderedDict[Any, Any]" = OrderedDict()
        self.hits = 0
        self.misses = 0

    def generate_key(self, *args: Any, **kwargs: Any) -> Any:
        # NOTE: args and the sorted kwargs stay two separate elements of one outer tuple, rather
        # than being flattened together, so no split between them can ever collide with another
        return (args, tuple(sorted(kwargs.items())))

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        key = self.generate_key(*args, **kwargs)
        if key in self.cache:
            self.hits += 1
            self.cache.move_to_end(key)  # NOTE: a hit is a use too -- without this the cache degrades to FIFO
            return self.cache[key]
        self.misses += 1
        result = self.func(*args, **kwargs)
        self.cache[key] = result
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)
        return result
```

`generate_key` 和 `__call__` 期望情况下都是 $O(1)$，主要开销是对键做哈希和常数次字典操作；
`sorted(kwargs.items())` 的开销是 $O(k \log k)$，$`k`$ 是这一次调用传入的关键字参数个数，和 `capacity` 无
关。

### Part 2

每次调用都往一个单一的日志文件里追加一条*记录*（record）：未命中时是一条 `PUT` 记录（键和值都写入），命
中时是一条更轻量的 `TOUCH` 记录（只写键——值已经知道了，不需要再写一遍）。恢复时，在最近一份快照（第一
次是空的）的基础上按顺序重放日志：一条 `PUT` 设置这个键的值，并把它移到最近使用的一端，如果因此超出容
量，就按 `__call__` 用的同一条容量规则淘汰；一条 `TOUCH` 把一个已经存在的键移到最近使用的一端。没有任何
记录直接标记一次淘汰（参见后面的追问）。

每条记录的格式是固定的 8 字节大端长度，加一个 4 字节的 CRC-32 校验和，两者都在负载（payload）前面——负
载是一个 JSON 文档，装着操作类型和键，`PUT` 的话还有值。校验和覆盖的是长度字段本身，不只是负载：如果校验
和只覆盖负载，那么一次刚好写了 12 个零字节就被截断的写入——这正是崩溃写到一半时一种说得通的形态——会被
当成一条合法的、空负载的记录通过校验，因为 `zlib.crc32(b"")` 正好是 `0`，恰好和被清零的校验和字段一致；
把长度字段也纳入校验，就能让同样这个全零的头部校验和自身失败。只有当完整的长度字段存在、后面跟着的负载字
节数不少于这个长度、并且校验和匹配时，一条记录才会被信任；重放遇到第一条不满足这些条件的记录就停止，日志
会被截断回最后一条完好记录的末尾，因为损坏片段之后写入的任何内容，后续的重放都不会再触及。

一个键是 `(args, sorted_kwargs)`，本身由若干元组构成；`json.dumps` 输出时已经会把嵌套的 `tuple` 变成
JSON 数组，所以编码不需要任何特殊处理。解码则必须在每一处元组可能出现过的地方把这个转换反过来，否则恢复
出来的键里会含有 `list`——它不可哈希，也永远不可能等于同样参数下实时调用 `generate_key` 产生的那个元
组——所以 `_decode_key` 把任意深度的每一个 JSON 数组都还原成 `tuple`。这个过程绝不会把一个 `list` 类型
的参数误当成 `tuple`，因为 `list` 类型的参数从一开始就不可能被用在这里：`generate_key` 的结果必须可哈
希，而一个内部任何地方含有 `list` 的元组是不可哈希的，所以只要实时调用传入这样的参数，`key in
self.cache` 在涉及持久化之前就已经会抛出 `TypeError`。返回值不受这条限制——它从来不是键——所以会原样保
留 `json.loads` 解析出来的结果，这也是为什么其中的 `tuple` 在崩溃之后会变成 `list` 存活下来。

```python
import json
import os
import zlib

PUT, TOUCH = "put", "touch"


def _decode_key(value: Any) -> Any:
    if isinstance(value, list):
        return tuple(_decode_key(item) for item in value)
    return value


def _encode_record(op: str, key: Any, value: Any = None) -> bytes:
    body = {"op": op, "key": key}
    if op == PUT:
        body["value"] = value
    payload = json.dumps(body, separators=(",", ":")).encode("utf-8")
    header = len(payload).to_bytes(8, "big")
    checksum = zlib.crc32(header + payload).to_bytes(4, "big")  # NOTE: covers the header too -- see above
    return header + checksum + payload


def _read_record(data: bytes, pos: int):
    """Returns (op, key, value, end) for the record starting at pos, or None if no complete,
    checksum-valid record starts there."""
    if pos + 12 > len(data):
        return None                                        # not even a full header and checksum remain
    length = int.from_bytes(data[pos:pos + 8], "big")
    end = pos + 12 + length
    if end > len(data):
        return None                                         # cut off mid-record, or a damaged (huge) length
    payload = data[pos + 12:end]
    if zlib.crc32(data[pos:pos + 8] + payload) != int.from_bytes(data[pos + 8:pos + 12], "big"):
        return None                                         # enough bytes, but not the ones that were written
    try:
        body = json.loads(payload)
        op, key = body["op"], _decode_key(body["key"])
    except (json.JSONDecodeError, UnicodeDecodeError, KeyError, TypeError):
        return None                                          # checksum-valid but not one of our records
    return op, key, body.get("value"), end


def _fsync_directory(path: str) -> None:
    dir_fd = os.open(os.path.dirname(os.path.abspath(path)) or ".", os.O_RDONLY)
    try:
        os.fsync(dir_fd)   # NOTE: the rename below is durable only once the directory entry is synced too
    finally:
        os.close(dir_fd)


class DurableMemoize(Memoize):
    def __init__(self, func: Callable, capacity: int, path: str | None = None) -> None:
        super().__init__(func, capacity)
        self.path = path
        self._log = None
        if path is not None:
            self._load()
            self._log = open(path, "ab")   # NOTE: kept open and appended to for the life of this instance

    def _snapshot_path(self) -> str:
        return self.path + ".snapshot"

    def _load(self) -> None:
        snapshot = self._snapshot_path()
        if os.path.exists(snapshot):
            with open(snapshot, "r", encoding="utf-8") as f:
                for key, value in json.load(f):          # oldest (LRU) first, exactly as compact() wrote it
                    self.cache[_decode_key(key)] = value
        if not os.path.exists(self.path):
            return
        with open(self.path, "rb") as f:
            data = f.read()
        pos = 0
        while (record := _read_record(data, pos)) is not None:
            op, key, value, pos = record
            if op == PUT:
                self.cache[key] = value
                self.cache.move_to_end(key)               # NOTE: reassigning an existing key does not reorder it
                if len(self.cache) > self.capacity:
                    self.cache.popitem(last=False)
            elif key in self.cache:                        # TOUCH; always true for a log this class wrote
                self.cache.move_to_end(key)
        if pos < len(data):                                # NOTE: cut the bad tail, or later appends hide behind it
            with open(self.path, "r+b") as f:
                f.truncate(pos)
                f.flush()
                os.fsync(f.fileno())

    def _append(self, op: str, key: Any, value: Any = None) -> None:
        if self._log is None:
            return
        self._log.write(_encode_record(op, key, value))
        self._log.flush()
        os.fsync(self._log.fileno())          # NOTE: the durability point -- __call__ does not return before this

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        key = self.generate_key(*args, **kwargs)
        if key in self.cache:
            self.hits += 1
            self.cache.move_to_end(key)
            self._append(TOUCH, key)
            return self.cache[key]
        self.misses += 1
        result = self.func(*args, **kwargs)
        self._append(PUT, key, result)         # NOTE: durable before the entry is even visible in memory
        self.cache[key] = result
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)
        return result

    def compact(self) -> None:
        if self.path is None:
            return
        snapshot = [[key, value] for key, value in self.cache.items()]   # oldest (LRU) first
        tmp = self._snapshot_path() + ".tmp"
        with open(tmp, "w", encoding="utf-8") as f:
            json.dump(snapshot, f, separators=(",", ":"))
            f.flush()
            os.fsync(f.fileno())
        os.replace(tmp, self._snapshot_path())   # NOTE: the commit point -- atomic on the same filesystem
        _fsync_directory(self._snapshot_path())
        self._log.close()
        open(self.path, "wb").close()             # NOTE: safe only once the snapshot above is durably committed
        self._log = open(self.path, "ab")
```

如果崩溃发生在 `compact()` 的改名（rename）和截断之间，刚被替换下来的旧日志会和新快照一起留在磁盘上；下
一次恢复会先加载快照，再把这份整体上已经多余的旧日志重放一遍。这样重现出来的还是同一个状态，而不是别的状
态：把日志连续重放两遍，等价于把它和自身拼接成一份日志、整体重放一次；而一次完整重放最终得到的顺序和被淘
汰的集合，只取决于每个键在这个序列里*最后*出现的位置——这个相对位置，在拼接出的第二份日志里和单独一份日
志里完全一样，哪怕重放过程中每一条记录都会把缓存暂时打乱成不同的样子。`_load` 和 `compact()` 都不会调用
`func`，所以最坏情况也只是浪费一些解析的开销，绝不会得到错误的结果。

`__call__` 在 `Memoize` 的 $O(1)$ 之外多付出一次 `fsync` 的代价，不管命中还是未命中，每次调用都要往持久
化存储走一个来回。`compact()` 是 $O(\text{capacity})$，主要开销是把当前缓存的条目写出去；它从不会在
`__call__` 内部被调用，所以从不给某次调用增加自己的开销。

### Part 3

用一把锁把整次调用（包括 `func`）整个包起来，是最简单的线程安全包装：这样做是正确的，但它把每一次调用都
串行地排在当前正在运行的那次调用后面，不管是已经缓存了的键，还是根本没人碰过的键，都逃不掉。`func` 本身
的开销——一次网络调用，一次很慢的计算——就这样变成了每一个其他调用者也要付出的开销。

```python
import threading
import time


class NaiveLockedMemoize(DurableMemoize):
    """One lock around the whole call, func included -- correct, but no two calls ever overlap."""

    def __init__(self, func: Callable, capacity: int, path: str | None = None) -> None:
        super().__init__(func, capacity, path)
        self._lock = threading.Lock()

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        with self._lock:
            return super().__call__(*args, **kwargs)
```

修复办法是只用一把锁围住簿记工作——检查缓存、追加日志、更新 `OrderedDict`——而把 `func` 完全放在锁外面
执行。但这样一来就重新打开了一个 `Memoize` 从不需要考虑的竞态：两个线程可能在彼此都还没算完之前，都看到
同一个键未命中，于是都去调用 `func`、都插入一个结果——缓存最终依然是合法的，所以这是正确的，但很浪费，而
且对于一个重复调用代价不能白白重复付的 `func`（比如一次支付，一次对另一个系统的写入）来说是错的。要避免
这一点——也就是*single-flight*——还需要一份额外的共享状态：哪些键正有计算在进行中，以及让后到的调用者
等待第一个调用者的结果，而不是自己再起一次计算的办法。

`ThreadSafeMemoize` 为每一个正在被计算的键保留一个 `threading.Event`，放在一个和缓存共用同一把锁保护的
字典里。一个线程如果发现自己的键已经有这样一个事件，就不持有锁等待它，等事件触发后读取领头者（leader）的
结果；如果发现没有，就创建一个、成为这个键的领头者，释放锁，调用 `func`，然后重新获取锁，只用来安装结
果、在 `capacity` 要求时淘汰、并追加持久化记录——之后才唤醒所有等待者，这样一个等待者拿到的结果和领头者
的一样持久，尽管只有领头者会写日志记录。

追加日志这一步共用了领头者的临界区，而不是单独用一把自己的锁，原因是日志在磁盘上的顺序正是恢复时用来重建
最近使用顺序的依据（一个键的*最后一条*记录决定它的位置），所以这个顺序必须和调用在内存里真正生效的顺序一
致：如果专门为日志设一把锁，和串行化缓存更新的那把锁分开获取，两个线程就可能以相反的顺序获取这两把
锁——一个线程的缓存更新先生效，它的日志记录却后落盘——恢复出来的最近使用顺序就会是活着的缓存从未有过的
顺序。一把在每个键第一次被碰到时才创建的按键加锁（per-key lock），同样去不掉这把共享锁：`OrderedDict`
的最近使用链是所有键共享的，保护它仍然需要一把横跨所有键的锁（按键加锁能带来什么好处，留给后面的追问）。

```python
class _InFlight:
    """Coordinates one key's in-progress computation: a thread that is not the leader waits on
    event, then reads result (or re-raises exc)."""

    __slots__ = ("event", "result", "exc")

    def __init__(self) -> None:
        self.event = threading.Event()
        self.result: Any = None
        self.exc: BaseException | None = None


class ThreadSafeMemoize(DurableMemoize):
    def __init__(self, func: Callable, capacity: int, path: str | None = None) -> None:
        super().__init__(func, capacity, path)
        self._lock = threading.Lock()
        self._in_flight: dict[Any, _InFlight] = {}

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        key = self.generate_key(*args, **kwargs)
        with self._lock:
            if key in self.cache:
                self.hits += 1
                self.cache.move_to_end(key)
                self._append(TOUCH, key)
                return self.cache[key]
            waiting = self._in_flight.get(key)
            if waiting is None:
                waiting = self._in_flight[key] = _InFlight()
                leader = True
            else:
                leader = False

        if not leader:
            waiting.event.wait()                  # NOTE: outside the lock -- other keys keep making progress
            if waiting.exc is not None:
                raise waiting.exc
            return waiting.result

        try:
            result = self.func(*args, **kwargs)    # NOTE: outside the lock -- func's own latency is not serialised
        except BaseException as exc:
            with self._lock:
                del self._in_flight[key]
            waiting.exc = exc
            waiting.event.set()
            raise

        with self._lock:
            self.misses += 1
            self._append(PUT, key, result)         # NOTE: same critical section as the cache update -- see above
            self.cache[key] = result
            if len(self.cache) > self.capacity:
                self.cache.popitem(last=False)
            del self._in_flight[key]
        waiting.result = result
        waiting.event.set()                         # NOTE: only after the result is fully cached and durable
        return result
```

`__call__` 在命中和等待时期望情况下都是 $O(1)$；一个领头者自己的开销是 `func` 的开销加一次 `fsync`，不
管有多少线程汇聚到它身上，这个开销只需要付一次。

### 追问

- 在哈希之前先用 `inspect.signature(func).bind(*args, **kwargs).apply_defaults()` 绑定参数，可以把
  `f(1, 2)` 和 `f(1, b=2)` 统一成同一次调用，代价是键的计算变慢，而且需要 `func` 有一个真实的签
  名——如果 `func` 只被声明成 `(*args, **kwargs)`，就没有这样的签名可用。
- 淘汰不需要专门的记录：在固定的 `capacity` 下，重放会从 `PUT`/`TOUCH` 的历史里推导出每一次淘汰，和
  `__call__` 实时运行时完全一样。如果没有 `compact()`，这样一条记录就会有用了——它可以标出一个分界
  点，在这个点之前某个键的旧记录都可以安全跳过。
- 把几条待写记录攒在一起，每隔几次调用或者每隔几毫秒才 `fsync` 一次，是用持久性换吞吐量：这样一来，崩
  溃时最多可能丢失这一批里的所有调用，“不丢数据”也就变成了“最多丢最后一批还没 flush 的数据”。
- 第二个写入进程绝不能和第一个抢着写：要么用一把操作系统级的建议锁（`fcntl.flock`），在一次追加和它的
  `fsync` 期间一直持有，把两者串行化，要么让所有调用都经过唯一拥有这份日志的那个写入进程，其他进程通
  过队列或者 socket 和它通信。
- 一次命中的开销主要来自它的 `fsync`——不管查找本身多便宜，这都是一次毫秒级的、到存储设备的往
  返——所以命中这条路径是 I/O 密集型的；只要 `func` 本身做 I/O，未命中也是 I/O 密集型的，只有当
  `func` 是纯计算时，未命中才是 CPU 密集型的。
- 按键加锁能缩小的，是 single-flight 判定本身的争用，只有在同时未命中的键很多时才值得，因为这一步本
  来就已经是 $O(1)$ 且不碰磁盘；它帮不上淘汰这一步的忙。要想彻底去掉围绕淘汰的最后一把锁，就要把缓存分
  片成若干个独立的 LRU，各自有自己的锁和一部分 `capacity`，用键的哈希值选用哪一片——一旦瓶颈变成这把
  锁本身的争用，而不是磁盘 I/O，这就是标准做法。

<details>
<summary>验证代码（可运行）</summary>

```python
import random

# the Part 1 worked example, asserted directly
trace = Memoize(lambda *args, **kwargs: (args, kwargs), capacity=2)
trace(1, b=2, c=3)                                     # miss
trace(1, c=3, b=2)                                     # hit: kwargs in a different order
trace((1, 2))                                          # miss: one positional argument, a tuple
trace(1, 2)                                            # miss: two positional arguments; evicts f(1,b=2,c=3)
trace((1, 2))                                          # hit: touches f((1,2)) again
trace(1, 3)                                            # miss: evicts f(1,2), not the more recently hit f((1,2))
assert trace.hits == 2 and trace.misses == 4
assert list(trace.cache.keys()) == [trace.generate_key((1, 2)), trace.generate_key(1, 3)]


class ModelMemoize:
    """Independent reference model built straight from the statement: a plain list of [key, value]
    pairs, oldest (LRU) first, never touching Memoize's own data structures."""

    def __init__(self, func: Callable, capacity: int) -> None:
        self.func = func
        self.capacity = capacity
        self.entries: list = []
        self.misses = 0

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        key = (args, tuple(sorted(kwargs.items())))
        for i, (k, v) in enumerate(self.entries):
            if k == key:
                self.entries.append(self.entries.pop(i))
                return v
        self.misses += 1
        value = self.func(*args, **kwargs)
        self.entries.append((key, value))
        if len(self.entries) > self.capacity:
            self.entries.pop(0)
        return value


def echo(*args: Any, **kwargs: Any) -> Any:
    return (args, kwargs)


NAMES = ["a", "b", "c"]
VALUES = [0, 1, 2, (1, 2), (3,), "x", None, 1.5]


def random_call(rng: random.Random):
    args = tuple(rng.choice(VALUES) for _ in range(rng.randint(0, 3)))
    names = rng.sample(NAMES, k=rng.randint(0, len(NAMES)))
    kwargs = {n: rng.choice(VALUES) for n in names}
    if kwargs and rng.random() < 0.5:               # sometimes rebuild kwargs in a different order
        items = list(kwargs.items())
        rng.shuffle(items)
        kwargs = dict(items)
    return args, kwargs


for trial in range(300):
    rng = random.Random(trial)
    capacity = rng.randint(1, 5)
    real, model = Memoize(echo, capacity), ModelMemoize(echo, capacity)
    for _ in range(60):
        args, kwargs = random_call(rng)
        assert real(*args, **kwargs) == model(*args, **kwargs)
    assert real.misses == model.misses
    assert list(real.cache.keys()) == [k for k, _ in model.entries]
print("Part 1 checks OK (", trial + 1, "random trials against a brute-force model )")
```

```python
import tempfile


def json_echo(*args: Any, **kwargs: Any) -> Any:
    """Like echo, but returns a JSON-native structure (no tuples), so a durable round trip is exact."""
    return [list(args), dict(kwargs)]


JSON_VALUES = [0, 1, 2, "x", "y", None, 1.5, True]   # no tuples: durability checks keep values JSON-exact


def random_json_call(rng: random.Random):
    args = tuple(rng.choice(JSON_VALUES) for _ in range(rng.randint(0, 3)))
    names = rng.sample(NAMES, k=rng.randint(0, len(NAMES)))
    kwargs = {n: rng.choice(JSON_VALUES) for n in names}
    if kwargs and rng.random() < 0.5:
        items = list(kwargs.items())
        rng.shuffle(items)
        kwargs = dict(items)
    return args, kwargs


# the Part 2 worked example, asserted directly: two misses, a hit, a clean reopen, then a crash mid-record
with tempfile.TemporaryDirectory() as d:
    p = os.path.join(d, "cache.log")
    w = DurableMemoize(lambda n: n * n, capacity=2, path=p)
    w(3)                                                # miss
    w(4)                                                # miss
    w(3)                                                # hit
    clean = DurableMemoize(lambda n: n * n, capacity=2, path=p)
    assert list(clean.cache.items()) == list(w.cache.items())
    assert clean.hits == 0 and clean.misses == 0        # bookkeeping counters are not persisted

    fifth_record = _encode_record(PUT, w.generate_key(5), 25)
    with open(p, "ab") as f:
        f.write(fifth_record[:len(fifth_record) // 2])  # crash mid-record, as in the example
    r = DurableMemoize(lambda n: n * n, capacity=2, path=p)
    assert list(r.cache.items()) == [(w.generate_key(4), 16), (w.generate_key(3), 9)]

    tuple_returning = DurableMemoize(lambda: (1, 2), capacity=1, path=os.path.join(d, "t.log"))
    tuple_returning()
    reread = DurableMemoize(lambda: (1, 2), capacity=1, path=os.path.join(d, "t.log"))
    only_value = next(iter(reread.cache.values()))
    assert only_value == [1, 2] and isinstance(only_value, list)   # a tuple survives as a list
print("Part 2 worked-example and basic checks OK")

# crash simulation: truncate the real, on-disk log at every byte offset and reopen. states[n] / ends[n]
# are an independent Memoize's cache and the on-disk log size after n calls, precomputed once.
with tempfile.TemporaryDirectory() as d:
    p = os.path.join(d, "cache.log")
    rng = random.Random(0)
    history, ref = DurableMemoize(json_echo, capacity=4, path=p), Memoize(json_echo, capacity=4)
    states, ends = [OrderedDict()], [0]
    for _ in range(50):
        a, k = random_json_call(rng)
        history(*a, **k)
        ref(*a, **k)
        states.append(OrderedDict(ref.cache))
        ends.append(os.path.getsize(p))
    full_log = open(p, "rb").read()

    for cut in range(len(full_log) + 1):
        with tempfile.TemporaryDirectory() as d2:
            p2 = os.path.join(d2, "cache.log")
            with open(p2, "wb") as f:
                f.write(full_log[:cut])
            n_complete = max(i for i, e in enumerate(ends) if e <= cut)
            recovered = DurableMemoize(json_echo, capacity=4, path=p2)
            assert list(recovered.cache.items()) == list(states[n_complete].items()), cut
            assert os.path.getsize(p2) <= cut          # never left holding bytes past the crash point
    print("Part 2 crash-truncation check OK (", len(full_log) + 1, "byte offsets )")

    garbage_rng = random.Random(99)
    for n in (1, 2, 8, 11, 12, 13, 40):               # garbage appended after the last good record
        for tail in (bytes(n), bytes(garbage_rng.randrange(256) for _ in range(n))):
            with tempfile.TemporaryDirectory() as d3:
                p3 = os.path.join(d3, "cache.log")
                with open(p3, "wb") as f:
                    f.write(full_log + tail)
                recovered = DurableMemoize(json_echo, capacity=4, path=p3)
                assert list(recovered.cache.items()) == list(states[-1].items())
                assert os.path.getsize(p3) == len(full_log)
    print("Part 2 garbage-tail check OK")

# compaction: same observable state, bounded size, and safe even if the crash lands between the
# snapshot rename and the log truncation, or during the temporary file write before either
with tempfile.TemporaryDirectory() as d:
    p = os.path.join(d, "cache.log")
    rng = random.Random(7)
    live = DurableMemoize(json_echo, capacity=5, path=p)
    for _ in range(30):
        a, k = random_json_call(rng)
        live(*a, **k)
    before_compact = list(live.cache.items())
    old_log_bytes = open(p, "rb").read()
    live.compact()
    assert list(live.cache.items()) == before_compact
    new_log_bytes = open(p, "rb").read()
    snapshot_bytes = open(p + ".snapshot", "rb").read()
    assert len(new_log_bytes) == 0
    assert len(snapshot_bytes) < len(old_log_bytes)   # no longer grows with the whole call history

    fresh = DurableMemoize(json_echo, capacity=5, path=p)
    assert list(fresh.cache.items()) == before_compact

    with tempfile.TemporaryDirectory() as d2:          # crash after the rename, before the truncation
        p2 = os.path.join(d2, "cache.log")
        open(p2, "wb").write(old_log_bytes)
        open(p2 + ".snapshot", "wb").write(snapshot_bytes)
        recovered = DurableMemoize(json_echo, capacity=5, path=p2)
        assert list(recovered.cache.items()) == before_compact

    with tempfile.TemporaryDirectory() as d3:          # crash while still writing the temp file
        p3 = os.path.join(d3, "cache.log")
        open(p3, "wb").write(old_log_bytes)
        open(p3 + ".snapshot.tmp", "wb").write(b"{not valid json, a crash landed here")
        recovered = DurableMemoize(json_echo, capacity=5, path=p3)
        assert list(recovered.cache.items()) == before_compact
print("Part 2 compaction checks OK")
```

```python
from collections import Counter

# timing: unrelated keys proceed concurrently only when func() is called outside the lock
def make_slow(delay: float):
    def slow(x):
        time.sleep(delay)
        return x * 2
    return slow


def timed_run(cls, n_keys: int, path: str) -> float:
    cache = cls(make_slow(DELAY), capacity=10, path=path)
    threads = [threading.Thread(target=cache, args=(i,)) for i in range(n_keys)]
    t0 = time.perf_counter()
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    return time.perf_counter() - t0


N_KEYS, DELAY = 6, 0.1
with tempfile.TemporaryDirectory() as d:
    naive_elapsed = timed_run(NaiveLockedMemoize, N_KEYS, os.path.join(d, "c.log"))
with tempfile.TemporaryDirectory() as d:
    fast_elapsed = timed_run(ThreadSafeMemoize, N_KEYS, os.path.join(d, "c.log"))

assert naive_elapsed > 1.8 * fast_elapsed, (naive_elapsed, fast_elapsed)  # generous margin -- see the text
print(f"Part 3 timing check OK (whole-call lock {naive_elapsed:.2f}s, single-flight {fast_elapsed:.2f}s)")

# single-flight: many threads request overlapping keys; func must run exactly once per key
KEYS = list(range(5))
call_counts: Counter = Counter()
count_lock = threading.Lock()


def counted_slow(x):
    with count_lock:
        call_counts[x] += 1
    time.sleep(0.03)
    return x * 2


with tempfile.TemporaryDirectory() as d:
    single_flight = ThreadSafeMemoize(counted_slow, capacity=10, path=os.path.join(d, "c.log"))
    n_threads = 40
    barrier = threading.Barrier(n_threads)
    results: list = [None] * n_threads
    errors: list = []

    def worker(i: int) -> None:
        key = KEYS[i % len(KEYS)]
        barrier.wait()                # every thread reaches sf(...) at effectively the same instant
        try:
            results[i] = single_flight(key)
        except Exception as exc:
            errors.append(exc)

    threads = [threading.Thread(target=worker, args=(i,)) for i in range(n_threads)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    assert not errors
    assert all(results[i] == KEYS[i % len(KEYS)] * 2 for i in range(n_threads))
    assert all(call_counts[k] == 1 for k in KEYS), call_counts     # exactly once per key, not once per call
    assert single_flight.misses == len(KEYS)
print("Part 3 single-flight check OK, invocation counts:", dict(call_counts))

# stress: many threads, overlapping keys, some calls raise; the cache stays within capacity and a
# fresh recovery afterwards matches the live cache exactly
def flaky(x):
    if x % 13 == 0:
        raise ValueError("boom")
    return x * x


with tempfile.TemporaryDirectory() as d:
    p = os.path.join(d, "c.log")
    stressed = ThreadSafeMemoize(flaky, capacity=7, path=p)
    errors = []

    def stress_worker(seed: int) -> None:
        rng = random.Random(seed)
        for _ in range(50):
            x = rng.randint(0, 25)
            try:
                assert stressed(x) == x * x
            except ValueError:
                pass
            except Exception as exc:
                errors.append(exc)

    threads = [threading.Thread(target=stress_worker, args=(s,)) for s in range(16)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    assert not errors
    assert len(stressed.cache) <= 7
    recovered = DurableMemoize(flaky, capacity=7, path=p)
    assert list(recovered.cache.items()) == list(stressed.cache.items())
print("Part 3 stress check OK")

print("all checks passed")
```

</details>

</details>
