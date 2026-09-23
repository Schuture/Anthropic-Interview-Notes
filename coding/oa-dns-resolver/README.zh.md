# 在线测评：DNS 解析器

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 在线测评 · 5 关 | ★★☆☆☆ | 中等 | Fellows | dns, recursion, caching, ttl, concurrency | 5 关 / 90 分钟 | 在线测评（OA） |
<!-- meta:end -->

## 题目

按顺序实现下面五关：只有当前一关的测试全部通过，下一关的测试才会开始运行，且每一关都在同一个
`Resolver` 类上继续扩展，而不是另起炉灶。一个*区域*（zone）是一个 `dict[str, ARecord | CNAMERecord]`，
把一个域名映射到一条*记录*（record）：

```py
class ARecord:
    def __init__(self, addresses: list[str], ttl: int) -> None:
        """addresses is one or more IPv4 addresses, each a non-empty string such as "203.0.113.7",
        kept in the order given. ttl is a positive integer number of seconds; see Level 4."""

class CNAMERecord:
    def __init__(self, target: str, ttl: int) -> None:
        """target is another name -- not necessarily normalised (below), and not required to be a
        key of any zone. ttl is a positive integer number of seconds; see Level 4."""
```

*名称归一化*（name normalisation）先对名称做大小写折叠（每个 ASCII 字母都变成小写），再把结尾的点
收成恰好一个：不管结尾有几个 `.`（包括零个），都会被替换成恰好一个，所以 `"Example.COM"`、
`"example.com."` 和 `"EXAMPLE.com.."` 都会归一化成 `"example.com."`。这道题里凡是要查找的名称——
`resolve` 被调用时传入的名称，以及沿着链（chain）走到的每一个 `CNAMERecord.target`——都会先做归一化。
一个区域自己的 key，会在 `Resolver` 由它构建出来的那一刻统一做一次归一化；如果一个区域里有两个 key
归一化后变成同一个名称，以 `dict` 迭代顺序中较晚出现的那个为准。

`Resolver` 只构建一次，之后会响应许多次 `resolve` 调用：

```py
class DNSResolutionError(Exception):
    """Base class for every error resolve can raise; name is whichever name each subclass below names."""
    name: str

class Resolver:
    def __init__(self, zone: dict[str, ARecord | CNAMERecord],
                 fallback_zone: dict[str, ARecord | CNAMERecord] | None = None,
                 *, max_chain_length: int = 8) -> None: ...

    def resolve(self, name: str) -> list[str]:
        """Returns the IPv4 addresses name resolves to, in the order Level 2 defines."""
```

`fallback_zone` 和 `max_chain_length` 从 Level 3 起才会起作用；在那之前，把每个 `Resolver` 都当成
没有传 `fallback_zone` 一样看待即可。每条记录的 `ttl` 从 Level 4 起才会起作用；在那之前它只是被带在
记录上，从未被读取过。

### Level 1 —— 名称归一化与直接的 A 记录查找

`resolve` 对 `name` 做归一化，在区域里直接查找，并返回它的地址列表。如果归一化后的名称在区域里没有
对应的记录，就会抛出 `NameNotFoundError(name)`（`name` 是归一化后的名称）。Level 1 的区域里只有
`ARecord`；下面的 Level 2 会引入 `CNAMERecord`，以及 `resolve` 该拿它怎么办。

```py
class NameNotFoundError(DNSResolutionError): ...
```

示例：

```text
zone = {
    "api.acme.com.": ARecord(["203.0.113.10"], ttl=60),
    "db.acme.com.": ARecord(["203.0.113.20", "203.0.113.21"], ttl=60),
}
r = Resolver(zone)
r.resolve("API.acme.com")      # -> ["203.0.113.10"]                    -- 做了大小写折叠，并补上了结尾的点
r.resolve("db.acme.com.")      # -> ["203.0.113.20", "203.0.113.21"]
r.resolve("nope.acme.com.")    # -> raises NameNotFoundError("nope.acme.com.")
```

### Level 2 —— CNAME 链、环与链长上限

`resolve` 被重新定义，加入了对 `CNAMERecord` 的处理——它在 Level 1 的区域里从未出现过。解析 `name`
时，从它归一化后的形式出发，沿着一串 `CNAMERecord` 逐步往前走——每一步都走到当前记录的 `target` 归一
化后的名称——直到走到一个记录是 `ARecord` 的名称为止，这时返回它的地址列表；链的长度就是这样走过的
`CNAMERecord` 个数，所以一个本身就是 `ARecord` 的名称，链长为 $0$。`resolve` 会记住这次调用里已经
走过的每一个名称，从归一化后的查询名称开始；一旦某一步的目标是这个集合里已经出现过的名称，就立刻抛出
`ResolutionCycleError(target)`（`target` 就是那个重复出现的名称）——这项检查在每一步都排在其他检查
之前，所以哪怕继续走下去也会超出链长上限，环也总是会先抛出 `ResolutionCycleError`。如果要走到一个
`ARecord` 需要的链比 `max_chain_length` 还长，就会抛出 `ChainTooLongError(name)`（`name` 是最初、
归一化后的查询名称）。和 Level 1 一样，只要沿途走到的某个名称——无论是最初的查询名称，还是之后的
某个 target——不是区域里的 key，就会抛出 `NameNotFoundError(name)`。

```py
class ResolutionCycleError(DNSResolutionError): ...
class ChainTooLongError(DNSResolutionError): ...
```

示例，在上面的区域基础上扩展出一条链：

```text
zone["www.acme.com."] = CNAMERecord("edge.acme.com.", ttl=60)
zone["edge.acme.com."] = CNAMERecord("api.acme.com.", ttl=30)
r = Resolver(zone)
r.resolve("www.acme.com.")
# www.acme.com. -> CNAME edge.acme.com. -> CNAME api.acme.com. -> A ["203.0.113.10"]
# -> ["203.0.113.10"]                      -- 链长为 2，远低于默认的 max_chain_length=8

cycle_zone = {"a.acme.com.": CNAMERecord("b.acme.com.", ttl=60),
              "b.acme.com.": CNAMERecord("a.acme.com.", ttl=60)}
Resolver(cycle_zone).resolve("a.acme.com.")
# a -> CNAME b（visited={a, b}）-> CNAME a：a 已经出现过 -> raises ResolutionCycleError("a.acme.com.")

long_zone = {
    "start.acme.com.": CNAMERecord("mid1.acme.com.", ttl=60),
    "mid1.acme.com.": CNAMERecord("mid2.acme.com.", ttl=60),
    "mid2.acme.com.": CNAMERecord("mid3.acme.com.", ttl=60),
    "mid3.acme.com.": ARecord(["203.0.113.99"], ttl=60),
}
Resolver(long_zone, max_chain_length=2).resolve("start.acme.com.")
# start -> mid1（长度 1）-> mid2（长度 2）-> mid3（长度 3，超出了 max_chain_length=2）
# -> raises ChainTooLongError("start.acme.com.")
```

### Level 3 —— 备用区域

`resolve` 再次被重新定义，开始用上 `fallback_zone`——Level 1 和 Level 2 里它一直没派上用场。它首先
在 `zone` 内部完整地尝试一次 Level 2 的解析。如果这次尝试抛出了 `NameNotFoundError`，并且给出了
`fallback_zone`，`resolve` 就会再多尝试一次：对*最初*那个归一化后的查询名称，在 `fallback_zone` 里
单独重新解析一遍——绝不会接着第一次尝试的链继续往下走，而是用同一个 `max_chain_length` 完全重新
来过——并返回这一次的结果。如果第二次尝试也抛出了 `NameNotFoundError`，`resolve` 就抛出那一个错误
（它指出的是 `fallback_zone` 自己的链里缺失的那个名称）；只要有备用区域可以尝试，第一次尝试的错误就
会被丢弃。如果没有给出 `fallback_zone`，或者第一次尝试抛出的是 `ResolutionCycleError` 或
`ChainTooLongError`，`resolve` 就会立刻抛出那个错误，绝不会去查 `fallback_zone`——只有单纯的
“查无此名”才会回退，链本身有问题就不会。

示例：

```text
primary = {"svc.acme.com.": CNAMERecord("internal.acme.com.", ttl=60)}    # internal.acme.com. 不在 primary 里
fallback = {"svc.acme.com.": ARecord(["198.51.100.5"], ttl=30)}
Resolver(primary, fallback).resolve("svc.acme.com.")
# primary：svc -> CNAME internal.acme.com.，但它缺失 -> NameNotFoundError -- 回退
# fallback：svc.acme.com. -> A ["198.51.100.5"]            -- 重新解析，不是接着 internal 继续走
# -> ["198.51.100.5"]

looping = {"loop.acme.com.": CNAMERecord("loop.acme.com.", ttl=60)}
has_answer = {"loop.acme.com.": ARecord(["198.51.100.9"], ttl=60)}
Resolver(looping, has_answer).resolve("loop.acme.com.")
# raises ResolutionCycleError("loop.acme.com.") -- fallback 根本不会被尝试，哪怕它本来就能给出答案

Resolver({}, {}).resolve("nowhere.acme.com.")
# -> raises NameNotFoundError("nowhere.acme.com.")          -- 两个区域里都没有
```

### Level 4 —— 建在人工时钟上的 TTL 缓存

`Resolver` 拥有了一个人工时钟，只有 `advance_clock` 能推动它；它从 $0$ 开始。

```py
    def advance_clock(self, seconds: int) -> None:
        """Adds seconds (a non-negative integer) to the resolver's clock."""
```

`resolve` 被重新定义，会把一次成功的结果按归一化后的查询名称缓存起来：对一个名称的第一次调用会执行
Level 3 的解析，把它的地址列表连同一个过期时间一起存下来；此后只要时钟还没到这个过期时间，对同一个
名称的每一次调用都会直接返回缓存里的地址列表，*两个区域都不会碰*。一条缓存记录的过期时间，是它被
存入时的时钟读数，加上产生这个结果时读到的每一条记录的 `ttl` 中的*最小值*——包括沿途经过的每一个
`CNAMERecord`，以及最后那条 `ARecord`，不管最终是 Level 3 的（至多两次）尝试里的哪一次成功的。过期
是惰性检查的，只有在查找一个名称的时候才会检查：不会有任何东西主动扫描缓存去清理过期条目；一条被后来
的 `advance_clock` 调用落在了过期时间之后的记录，只会在它的名称下一次被查询时不再算作命中，然后像任何
其他未命中一样被重新解析并覆盖掉。一次抛出 `NameNotFoundError`、`ResolutionCycleError` 或
`ChainTooLongError` 的解析不会被缓存；下一次对这个名称的调用会重新完整解析一遍。缓存只作用于 `resolve`
最初被调用时的那个名称——解析 `"www.acme.com."` 绝不会为 `"edge.acme.com."` 或者链上路过的其他任何
名称建立缓存条目。

示例，接着 Level 2 的那条链（`www.acme.com.` 的 ttl 是 $60$，$\to$ `edge.acme.com.` 的 ttl 是 $30$，
$\to$ `api.acme.com.` 的 ttl 是 $60$，是一条 `ARecord`）：

```text
r = Resolver(zone)              # zone 就是 Level 2 扩展出的那个；时钟从 0 开始
r.resolve("www.acme.com.")      # -> ["203.0.113.10"]；缓存的过期时间是 0 + min(60, 30, 60) = 30
r.advance_clock(29)             # 时钟 = 29
r.resolve("www.acme.com.")      # -> ["203.0.113.10"]     -- 29 < 30：命中缓存，两个区域都没碰
r.advance_clock(1)              # 时钟 = 30
r.resolve("www.acme.com.")      # -> ["203.0.113.10"]     -- 30 == 30：已过期，重新解析了一遍
```

### Level 5 —— 并发解析

`resolve` 会被许多线程同时调用，它们共用同一个 `Resolver`。从每个调用者自己的角度看，它必须和 Level 4
所规定的完全一致，再加一条规则：当某个名称的 Level 3 解析（缓存未命中之后要做的那份工作）正在进行、
由最先到达的那次调用发起时，在它完成之前到达的、针对*同一个归一化名称*的其他每一次调用，都会等待这
一次解析，而不是各自另外发起一次多余的解析，然后拿到它的结果——同样的地址列表，或者同一个在每一个
等待的调用里都被抛出的异常——就好像是自己做了这次解析一样。一次 `resolve` 调用绝不会为另一个名称的
解析买单：两个同时到达的不同名称，哪怕它们的链都经过同一个 `CNAMERecord`，也是各自独立解析的，可以
并发进行。`resolve` 返回的每一个列表都是这次调用自己的副本：修改它不会影响缓存，也不会影响任何其他
调用的结果。这一关新增的缓存、时钟和记账，全都可以在调用者不用自己加锁的情况下，安全地被多个线程使用。

举例：目前还没有解析过任何名称，线程 $t_1, \dots, t_5$ 都在彼此返回之前调用了 `resolve("www.acme.com.")`。
不管这五次调用如何交错，只要其中还有调用没有结束，`zone` 里的这条链对 `"www.acme.com."` 最多只会被
走一次，而且这五次调用最终都会返回 `["203.0.113.10"]`。第六个线程 $t_6$ 在同一时刻调用
`resolve("shop.acme.com.")`，这个名称恰好也经过 `"edge.acme.com."`，但它不会加入 $t_1, \dots, t_5$
的等待，可以和它们并发解析。

## 参考解答

<details>
<summary>展开参考解答</summary>

值得事先确认的两点：一次失败的解析要不要也被缓存下来，比如配一个较短的负值 TTL（这里假设不需要——
只有成功的结果才会被缓存，Level 4 里已经说明）；以及并发的合并是否也要覆盖那些只是共享了链的一部分、
但查询名称本身并不相同的调用（这里假设不需要，只按查询名称本身来合并，Level 5 里已经说明）。

### Level 1

`ARecord` 和 `CNAMERecord` 就按给定的样子保存字段；`DNSResolutionError` 及其 Level 1 的子类都把出错
的名称存成 `.name`。`_normalize` 用一个表达式就实现了大小写折叠和结尾点的规则；`_normalized_zone` 在
构建时一次性地对输入区域的每个 key 都跑一遍这个函数，这样 `resolve` 自己就只需要对它被调用时传入的
那一个名称做归一化。

```python
class DNSResolutionError(Exception):
    """Base class for every error resolve can raise."""

    def __init__(self, name: str) -> None:
        super().__init__(name)
        self.name = name


class NameNotFoundError(DNSResolutionError):
    """name has no record anywhere it was looked up."""


class ARecord:
    def __init__(self, addresses: list[str], ttl: int) -> None:
        self.addresses = list(addresses)
        self.ttl = ttl


class CNAMERecord:
    def __init__(self, target: str, ttl: int) -> None:
        self.target = target
        self.ttl = ttl


def _normalize(name: str) -> str:
    return name.lower().rstrip(".") + "."   # NOTE: case-fold, then collapse to exactly one trailing dot


def _normalized_zone(zone):
    out = {}
    for name, record in zone.items():
        out[_normalize(name)] = record       # NOTE: a later duplicate normalised key wins, like a dict literal
    return out


class Resolver:
    def __init__(self, zone, fallback_zone=None, *, max_chain_length=8):
        self._zone = _normalized_zone(zone)
        self._fallback_zone = _normalized_zone(fallback_zone) if fallback_zone is not None else None
        self.max_chain_length = max_chain_length

    def resolve(self, name: str) -> list[str]:
        current = _normalize(name)           # NOTE: normalise before ever touching the zone (or, later, the cache)
        record = self._zone.get(current)
        if record is None:
            raise NameNotFoundError(current)
        return list(record.addresses)        # NOTE: a fresh copy -- the caller must not be able to mutate the zone
```

Level 1 的每个操作都是 $O(1)$：一次归一化加一次字典查找。

### Level 2

`_follow_chain` 把整套沿链解析的规则只实现一次，接收一个显式的 `zone` 参数，而不是总用 `self._zone`，
这样 Level 3 就能直接拿它去处理 `self._fallback_zone`，不用再重复一遍。它维护着 `visited`——一个以
起点为初始元素的集合——以及一个不断更新的最小 `ttl`；这里就把它算出来了，尽管在 Level 4 之前没有任何
地方会用到它，因为要是等到那时候再算，就得把同一条链再走一遍。环的检查在每一步都排在长度检查之前，
和 Problem 部分给出的顺序一致；`origin` 单独存在自己的变量里，这样即使 `current` 已经往前走了，
`ChainTooLongError` 依然能指出最初的查询名称。

```python
class ResolutionCycleError(DNSResolutionError):
    """a step's target is a name already visited earlier in this resolution."""


class ChainTooLongError(DNSResolutionError):
    """reaching an A record needed more than max_chain_length CNAME hops."""


class Resolver(Resolver):
    def _follow_chain(self, zone: dict, name: str) -> tuple[list[str], int]:
        origin = _normalize(name)
        current = origin
        visited = {current}
        ttl = None
        hops = 0
        while True:
            record = zone.get(current)
            if record is None:
                raise NameNotFoundError(current)
            ttl = record.ttl if ttl is None else min(ttl, record.ttl)   # NOTE: running min ttl, needed at Level 4
            if isinstance(record, ARecord):
                return list(record.addresses), ttl
            target = _normalize(record.target)
            if target in visited:                # NOTE: cycle is checked before the length cap, on every step
                raise ResolutionCycleError(target)
            hops += 1
            if hops > self.max_chain_length:
                raise ChainTooLongError(origin)   # NOTE: names the query, not the hop where the cap was hit
            visited.add(target)
            current = target

    def resolve(self, name: str) -> list[str]:
        addresses, _ttl = self._follow_chain(self._zone, name)
        return addresses
```

`resolve` 现在是 $O(k)$，其中 $k \le$ `max_chain_length` 是链的长度：每一跳一次字典查找、一次 `ttl`
比较和一次集合成员检查。

### Level 3

`resolve` 再次被重新打开：对 `self._zone` 尝试 `_follow_chain`，只捕获 `NameNotFoundError`；
`ResolutionCycleError` 和 `ChainTooLongError` 没有出现在 `except` 子句里，所以 Python 会让它们自己
继续往外传播，根本不会走到查备用区域这一步。回退的那次尝试用的是最初的 `name`（而不是这个方法里已经
不存在的 `current`）去调用 `_follow_chain(self._fallback_zone, name)`——那是一条从头开始的全新的链，
绝不是接着原来的链继续走。

```python
class Resolver(Resolver):
    def resolve(self, name: str) -> list[str]:
        try:
            addresses, _ttl = self._follow_chain(self._zone, name)
            return addresses
        except NameNotFoundError:
            if self._fallback_zone is None:
                raise
            # NOTE: ResolutionCycleError and ChainTooLongError from the try block above are not caught
            # here -- they already propagated out of resolve before this except clause could run.
            addresses, _ttl = self._follow_chain(self._fallback_zone, name)   # NOTE: name, not current: fresh chain
            return addresses
```

依然是 $O(k)$：`_follow_chain` 最多跑两次，每个区域一次。

### Level 4

Level 3 的回退逻辑从 `resolve` 里搬了出来，放进一个新的辅助方法 `_resolve_uncached`，除了名字和
返回类型之外没有变化：它现在会把 `_follow_chain` 早就算出来的 `ttl` 和地址列表一起带出来，而不是把
它扔掉。`resolve` 自己则变成了包在这个辅助方法外面的一层薄薄的缓存检查。用 `self._now < entry.expires_at`
而不是 `<=` 做比较，正对应 Problem 部分的示例：一条以总 `ttl` 为 $t$、在时钟读数 $c$ 时存入的记录，
存活到（但不包括）时钟读数 $c + t$ 为止。

```python
class _CacheEntry:
    __slots__ = ("addresses", "expires_at")

    def __init__(self, addresses: list[str], expires_at: int) -> None:
        self.addresses = addresses
        self.expires_at = expires_at


class Resolver(Resolver):
    def __init__(self, zone, fallback_zone=None, *, max_chain_length=8):
        super().__init__(zone, fallback_zone, max_chain_length=max_chain_length)
        self._now = 0
        self._cache: dict[str, _CacheEntry] = {}

    def advance_clock(self, seconds: int) -> None:
        self._now += seconds

    def _resolve_uncached(self, name: str) -> tuple[list[str], int]:
        try:
            return self._follow_chain(self._zone, name)
        except NameNotFoundError:
            if self._fallback_zone is None:
                raise
            return self._follow_chain(self._fallback_zone, name)

    def resolve(self, name: str) -> list[str]:
        key = _normalize(name)                                     # NOTE: cache is keyed by the normalised name
        entry = self._cache.get(key)
        if entry is not None and self._now < entry.expires_at:     # NOTE: lazy expiry -- only checked on a read
            return list(entry.addresses)
        addresses, ttl = self._resolve_uncached(key)
        self._cache[key] = _CacheEntry(list(addresses), self._now + ttl)
        return list(addresses)
```

命中缓存是 $O(1)$；未命中的开销是 Level 3 的 $O(k)$ 再加一次字典写入，而且没有任何东西会去扫描整个
缓存，所以一条过期之后再也没被查询过的记录，就一直安安静静地留在那里，无伤大雅。

### Level 5

所有可能被不止一个线程碰到的可变状态——`_cache`、`_in_flight`、`_now`——都只会在 `with self._lock:`
内部被读取或写入；`_zone`、`_fallback_zone` 和 `max_chain_length` 不需要锁，因为只有 `__init__` 会去
设置它们。缓存检查和领头/等待的判定都发生在同一个 `with` 块里，所以不会有两次针对同一个 key 的调用
同时看到未命中、又各自创建一个 `Future`：谁先跑到这个块，谁就会留下一个 `Future`，供之后到达的每一
次调用去发现。`_resolve_uncached`——就像等待方那边的 `future.result()` 一样——是在锁已经释放的情况下
运行的，所以一个名称的解析不会挡住另一个名称，一个正在等待某次较慢查找的调用，也不会同时挡住
`advance_clock` 或者某个不相关的 `resolve` 调用。

```python
import threading
from concurrent.futures import Future


class Resolver(Resolver):
    def __init__(self, zone, fallback_zone=None, *, max_chain_length=8):
        super().__init__(zone, fallback_zone, max_chain_length=max_chain_length)
        self._lock = threading.Lock()
        self._in_flight: dict[str, Future] = {}

    def advance_clock(self, seconds: int) -> None:
        with self._lock:
            self._now += seconds

    def resolve(self, name: str) -> list[str]:
        key = _normalize(name)
        with self._lock:
            entry = self._cache.get(key)
            if entry is not None and self._now < entry.expires_at:
                return list(entry.addresses)
            future = self._in_flight.get(key)
            if future is None:
                future = self._in_flight[key] = Future()
                leader = True
            else:
                leader = False
        if not leader:
            return list(future.result())    # NOTE: re-raises the leader's own exception here too, if it failed
        try:
            addresses, ttl = self._resolve_uncached(key)
        except Exception as exc:
            with self._lock:
                del self._in_flight[key]     # NOTE: not cached (Level 4) -- the next caller starts a fresh attempt
            future.set_exception(exc)
            raise
        with self._lock:
            self._cache[key] = _CacheEntry(list(addresses), self._now + ttl)
            del self._in_flight[key]
        future.set_result(list(addresses))   # NOTE: wakes every follower parked in future.result() above
        return list(addresses)
```

失败的时候，领头的调用会*先*删掉自己在 `_in_flight` 里的条目，*再*调用 `future.set_exception`，这样
紧随其后到达的调用，既看不到缓存条目，也看不到进行中的条目，会重新发起一次全新的尝试，和 Level 4 里
对未缓存失败的要求完全一致——而 `set_exception` 接下来会唤醒每一个及时赶到、正在等待的调用，
`future.result()` 会在它们各自那里重新抛出同一个异常对象。`resolve` 的开销是 $O(1)$ 的加锁，再加上——
对最终成为领头的那次调用而言——Level 4 本身的开销；同一个名称的其他并发调用，付出的代价只是等待一个
`Future`。

### 追问

- 负值缓存——把“查无此名”本身也缓存下来，通常配一个较短的、属于它自己的 TTL——需要 `resolve` 在
  `NameNotFoundError` 往外传播之前先捕获它，缓存一个标记，供之后的调用重新抛出；还需要它自己的淘汰
  机制，这样一个后来真的存在了的名称才不会被永远挡在外面。
- 想把合并扩展到两个不同查询名称之间——只要它们的链经过同一个 `CNAMERecord`——就需要按沿途碰到的每
  一个名称去建索引，而不只是按最初被问到的那一个，还需要一条规则来说明它们各自的回退结果不一样时该
  怎么办。
- 用一个 `threading.Condition` 搭配一个存“完成”标志的普通字典，效果和用 `Future` 做合并一样好，代价
  是要自己手写 wait/notify 那套记账逻辑，而不是直接复用 `Future` 自带的、线程安全的结果/异常机制。
- 超过两个区域——依次尝试一串备用区域——是把 Level 3 的 `except NameNotFoundError: ...` 推广成一个对
  区域的循环，在第一次成功或者最后一个区域自己的错误处停下来。
- 一个大到缓存不能无限增长的区域，就需要淘汰机制，比如 LRU，叠加在——而不是取代——Level 4 已经实现的
  惰性、读时过期之上。

<details>
<summary>验证代码（可运行）</summary>

```python
def expect(exc, fn, *args):
    try:
        fn(*args)
    except exc as e:
        return e
    raise AssertionError(f"expected {exc.__name__} from a call with args {args!r}")


# ---- Level 1 example, replayed exactly ----
zone = {
    "api.acme.com.": ARecord(["203.0.113.10"], ttl=60),
    "db.acme.com.": ARecord(["203.0.113.20", "203.0.113.21"], ttl=60),
}
r = Resolver(zone)
assert r.resolve("API.acme.com") == ["203.0.113.10"]
assert r.resolve("db.acme.com.") == ["203.0.113.20", "203.0.113.21"]
assert expect(NameNotFoundError, r.resolve, "nope.acme.com.").name == "nope.acme.com."
print("Level 1 example replayed")

# ---- Level 2 example, replayed exactly ----
zone["www.acme.com."] = CNAMERecord("edge.acme.com.", ttl=60)
zone["edge.acme.com."] = CNAMERecord("api.acme.com.", ttl=30)
r = Resolver(zone)
assert r.resolve("www.acme.com.") == ["203.0.113.10"]

cycle_zone = {"a.acme.com.": CNAMERecord("b.acme.com.", ttl=60),
              "b.acme.com.": CNAMERecord("a.acme.com.", ttl=60)}
assert expect(ResolutionCycleError, Resolver(cycle_zone).resolve, "a.acme.com.").name == "a.acme.com."

long_zone = {
    "start.acme.com.": CNAMERecord("mid1.acme.com.", ttl=60),
    "mid1.acme.com.": CNAMERecord("mid2.acme.com.", ttl=60),
    "mid2.acme.com.": CNAMERecord("mid3.acme.com.", ttl=60),
    "mid3.acme.com.": ARecord(["203.0.113.99"], ttl=60),
}
assert expect(ChainTooLongError, Resolver(long_zone, max_chain_length=2).resolve,
              "start.acme.com.").name == "start.acme.com."
assert Resolver(long_zone, max_chain_length=3).resolve("start.acme.com.") == ["203.0.113.99"]
print("Level 2 example replayed")

# ---- Level 3 example, replayed exactly ----
primary = {"svc.acme.com.": CNAMERecord("internal.acme.com.", ttl=60)}
fallback = {"svc.acme.com.": ARecord(["198.51.100.5"], ttl=30)}
assert Resolver(primary, fallback).resolve("svc.acme.com.") == ["198.51.100.5"]

looping = {"loop.acme.com.": CNAMERecord("loop.acme.com.", ttl=60)}
has_answer = {"loop.acme.com.": ARecord(["198.51.100.9"], ttl=60)}
assert expect(ResolutionCycleError, Resolver(looping, has_answer).resolve,
              "loop.acme.com.").name == "loop.acme.com."

assert expect(NameNotFoundError, Resolver({}, {}).resolve, "nowhere.acme.com.").name == "nowhere.acme.com."

# a differing fallback chain names ITS OWN missing target, not the primary's
primary3 = {"x.acme.com.": CNAMERecord("y.acme.com.", ttl=10)}
fallback3 = {"x.acme.com.": CNAMERecord("z.acme.com.", ttl=10)}
assert expect(NameNotFoundError, Resolver(primary3, fallback3).resolve, "x.acme.com.").name == "z.acme.com."
print("Level 3 example replayed")

# ---- Level 4 example, replayed exactly, with the zone instrumented to prove caching actually happens ----
class _CountingZone(dict):
    """A dict that counts every .get() call, standing in for Resolver's own _zone after construction,
    to prove whether a resolve() call touched the zone at all."""

    def __init__(self, data):
        super().__init__(data)
        self.reads = 0

    def get(self, key, default=None):
        self.reads += 1
        return super().get(key, default)


r4 = Resolver(zone)
assert r4.resolve("www.acme.com.") == ["203.0.113.10"]     # cached with expiry 0 + min(60, 30, 60) = 30
r4._zone = _CountingZone(r4._zone)
r4.advance_clock(29)
assert r4.resolve("www.acme.com.") == ["203.0.113.10"]
assert r4._zone.reads == 0, "a live cache entry must not touch the zone at all"
r4.advance_clock(1)
assert r4.resolve("www.acme.com.") == ["203.0.113.10"]
assert r4._zone.reads > 0, "an expired entry must be resolved again, not served stale"
print("Level 4 example replayed")

# ---- edge cases the examples do not reach ----
# a failed resolution is never cached: each retry re-touches the zone
rb = Resolver({"only.acme.com.": ARecord(["203.0.113.1"], ttl=5)})
rb._zone = _CountingZone(rb._zone)
expect(NameNotFoundError, rb.resolve, "missing.acme.com.")
first_reads = rb._zone.reads
expect(NameNotFoundError, rb.resolve, "missing.acme.com.")
assert rb._zone.reads > first_reads

# a zone key that only differs by case or trailing dots collapses to one entry: the later one wins
dup_zone = {"Dup.acme.com.": ARecord(["203.0.113.200"], ttl=10), "dup.acme.com": ARecord(["203.0.113.201"], ttl=10)}
assert Resolver(dup_zone).resolve("dup.acme.com.") == ["203.0.113.201"]

# a chain of exactly max_chain_length hops is allowed; one more is not
edge_zone = {"h0.t.": CNAMERecord("h1.t.", ttl=9), "h1.t.": ARecord(["203.0.113.7"], ttl=9)}
assert Resolver(edge_zone, max_chain_length=1).resolve("h0.t.") == ["203.0.113.7"]
expect(ChainTooLongError, Resolver(edge_zone, max_chain_length=0).resolve, "h0.t.")
print("edge cases OK")
```

```python
import random
from collections import Counter


def _ref_normalize(name: str) -> str:
    return name.lower().rstrip(".") + "."


class _SlowResolver:
    """Independent reference model, written from the statement alone: no cache, no coalescing, and no
    code shared with Resolver -- it re-walks the chain (and, on failure, the fallback chain) from
    scratch on every call."""

    def __init__(self, zone, fallback_zone=None, max_chain_length=8):
        self.zone = {_ref_normalize(k): v for k, v in zone.items()}
        self.fallback_zone = ({_ref_normalize(k): v for k, v in fallback_zone.items()}
                               if fallback_zone is not None else None)
        self.max_chain_length = max_chain_length

    def _chain(self, zone, name):
        origin = _ref_normalize(name)
        current = origin
        visited = {current}
        ttl = None
        hops = 0
        while True:
            if current not in zone:
                raise NameNotFoundError(current)
            record = zone[current]
            ttl = record.ttl if ttl is None else min(ttl, record.ttl)
            if isinstance(record, ARecord):
                return list(record.addresses), ttl
            target = _ref_normalize(record.target)
            if target in visited:
                raise ResolutionCycleError(target)
            hops += 1
            if hops > self.max_chain_length:
                raise ChainTooLongError(origin)
            visited.add(target)
            current = target

    def resolve(self, name):
        try:
            return self._chain(self.zone, name)
        except NameNotFoundError:
            if self.fallback_zone is None:
                raise
            return self._chain(self.fallback_zone, name)


def _random_zone(rng, n_names=10, prefix="n"):
    names = [f"{prefix}{i}.test." for i in range(n_names)]
    zone = {}
    for nm in names:
        if rng.random() < 0.6:
            zone[nm] = CNAMERecord(rng.choice(names), ttl=rng.randint(1, 50))   # may chain, loop or self-cycle
        else:
            addrs = [f"203.0.113.{rng.randint(1, 254)}" for _ in range(rng.randint(1, 3))]
            zone[nm] = ARecord(addrs, ttl=rng.randint(1, 50))
    if rng.random() < 0.3:
        del zone[rng.choice(names)]        # leave a gap: some name becomes entirely absent
    return zone, names


def _run_trial(seed):
    rng = random.Random(seed)
    prim, prim_names = _random_zone(rng, prefix="p")
    use_fallback = rng.random() < 0.6
    fb, fb_names = _random_zone(rng, n_names=6, prefix="f") if use_fallback else (None, [])
    max_len = rng.choice([1, 2, 3, 5, 8])
    real = Resolver(prim, fb, max_chain_length=max_len)
    slow = _SlowResolver(prim, fb, max_chain_length=max_len)
    candidates = prim_names + fb_names + ["ghost.test."]
    outcomes = Counter()

    for _ in range(15):
        base = rng.choice(candidates)
        spelling = rng.choice([base, base.upper(), base[:-1]])   # exercises case-fold and trailing-dot rules too
        try:
            want_addrs, want_ttl = slow.resolve(spelling)
        except DNSResolutionError as want_exc:
            got_exc = expect(type(want_exc), real.resolve, spelling)
            assert got_exc.name == want_exc.name, (seed, spelling, got_exc.name, want_exc.name)
            outcomes[type(want_exc).__name__] += 1
            continue
        got_addrs = real.resolve(spelling)
        assert got_addrs == want_addrs, (seed, spelling, got_addrs, want_addrs)
        outcomes["ok"] += 1

        # cross-check caching around the independently computed expiry, on a resolver of its own
        fresh = Resolver(prim, fb, max_chain_length=max_len)
        assert fresh.resolve(spelling) == want_addrs
        fresh.advance_clock(want_ttl - 1)
        assert fresh.resolve(spelling) == want_addrs        # one tick before the computed expiry: still cached
        fresh.advance_clock(1)
        assert fresh.resolve(spelling) == want_addrs         # at the expiry: recomputed, same static answer
    return outcomes


totals = Counter()
for seed in range(300):
    totals += _run_trial(seed)
assert min(totals[k] for k in ("ok", "NameNotFoundError", "ResolutionCycleError", "ChainTooLongError")) > 50, totals
print(f"cross-validated 300 random zones against an independent reference resolver: {dict(totals)}")
```

```python
class _GatedResolver(Resolver):
    """Test-only subclass: the upstream resolution of one chosen name blocks on an Event until
    released, so at least one concurrent caller is guaranteed to find it still in flight."""

    def __init__(self, *args, gate_name=None, **kwargs):
        super().__init__(*args, **kwargs)
        self.upstream_calls = Counter()
        self._count_lock = threading.Lock()
        self._gate_key = _normalize(gate_name) if gate_name is not None else None
        self.entered = threading.Event()
        self._release = threading.Event()

    def _resolve_uncached(self, name):
        with self._count_lock:
            self.upstream_calls[name] += 1
        if name == self._gate_key:
            self.entered.set()
            assert self._release.wait(timeout=5.0), "test never released the gate (deadlock?)"
        return super()._resolve_uncached(name)


def _start_calls(resolver, names):
    """Starts one thread per name, each recording ("ok", addresses) or ("err", exception); returns
    (threads, results) without waiting, so the caller can act while the threads are still running."""
    results = [None] * len(names)

    def call(i, nm):
        try:
            results[i] = ("ok", resolver.resolve(nm))
        except DNSResolutionError as e:
            results[i] = ("err", e)

    threads = [threading.Thread(target=call, args=(i, nm)) for i, nm in enumerate(names)]
    for t in threads:
        t.start()
    return threads, results


def _join_all(threads, timeout=5.0):
    for t in threads:
        t.join(timeout=timeout)
    assert all(not t.is_alive() for t in threads), "a thread never returned (deadlock?)"


# ---- many concurrent callers of the SAME name coalesce into exactly one upstream lookup ----
chain_zone = {"chain.acme.com.": CNAMERecord("mid.acme.com.", ttl=40),
              "mid.acme.com.": ARecord(["203.0.113.55"], ttl=40)}
gated = _GatedResolver(chain_zone, gate_name="chain.acme.com.")
leader_threads, leader_results = _start_calls(gated, ["chain.acme.com."])
assert gated.entered.wait(timeout=5.0), "no call ever reached the gated upstream lookup"
# NOTE: entered is set only from inside _resolve_uncached, which resolve() reaches only after already
# creating this name's in-flight Future under the lock -- so every follower started from here on is
# guaranteed, deterministically and without relying on any sleep, to find that Future already in place.
follower_threads, follower_results = _start_calls(gated, ["CHAIN.acme.com"] * 9)
gated._release.set()
_join_all(leader_threads + follower_threads)

key = _normalize("chain.acme.com.")
assert gated.upstream_calls == Counter({key: 1}), gated.upstream_calls
results = leader_results + follower_results
assert len(results) == 10 and all(kind == "ok" and value == ["203.0.113.55"] for kind, value in results)
for _, value in results:
    value.append("corrupted")                                     # mutate every returned list
assert gated.resolve("chain.acme.com.") == ["203.0.113.55"]        # unaffected -- each was an independent copy
print("single-name coalescing: 10 concurrent callers, 1 upstream lookup")

# ---- the same rule for a name that fails: every waiter gets the identical error, and it is not cached ----
gated_fail = _GatedResolver({"only.acme.com.": ARecord(["203.0.113.1"], ttl=30)}, gate_name="missing.acme.com.")
leader_threads, leader_results = _start_calls(gated_fail, ["missing.acme.com."])
assert gated_fail.entered.wait(timeout=5.0)
follower_threads, follower_results = _start_calls(gated_fail, ["missing.acme.com."] * 6)
gated_fail._release.set()
_join_all(leader_threads + follower_threads)

fail_key = _normalize("missing.acme.com.")
results = leader_results + follower_results
assert gated_fail.upstream_calls[fail_key] == 1
assert len(results) == 7
assert all(kind == "err" and isinstance(exc, NameNotFoundError) and exc.name == fail_key for kind, exc in results)
expect(NameNotFoundError, gated_fail.resolve, "missing.acme.com.")    # not cached: a later call retries
assert gated_fail.upstream_calls[fail_key] == 2
assert fail_key not in gated_fail._cache
print("failure coalescing: every waiter sees the same error, and a failed lookup is never cached")

# ---- two DIFFERENT queried names are never coalesced together, even when they share a CNAME target ----
shared_zone = {"www.acme.com.": CNAMERecord("edge.acme.com.", ttl=40),
               "shop.acme.com.": CNAMERecord("edge.acme.com.", ttl=40),
               "edge.acme.com.": ARecord(["203.0.113.77"], ttl=40)}
threads, results = _start_calls(Resolver(shared_zone), ["www.acme.com.", "shop.acme.com."] * 5)
_join_all(threads)
assert len(results) == 10 and all(kind == "ok" and value == ["203.0.113.77"] for kind, value in results)


# ---- broad randomized stress: many threads, many rounds, checked against the independent reference ----
def _stress(seed, n_threads=20, n_rounds=60):
    rng = random.Random(seed)
    prim, prim_names = _random_zone(rng, n_names=8, prefix="s")
    fb, fb_names = _random_zone(rng, n_names=4, prefix="g")
    max_len = 4
    resolver = Resolver(prim, fb, max_chain_length=max_len)
    slow = _SlowResolver(prim, fb, max_chain_length=max_len)
    candidates = prim_names + fb_names + ["ghost.test."]
    expected = {}
    for nm in candidates:
        try:
            addrs, _ttl = slow.resolve(nm)
            expected[nm] = ("ok", addrs)
        except DNSResolutionError as e:
            expected[nm] = (type(e), e.name)

    failures = []
    failures_lock = threading.Lock()
    done = [False] * n_threads

    def worker(idx):
        local_rng = random.Random(f"{seed}:{idx}")
        for _ in range(n_rounds):
            name = local_rng.choice(candidates)
            spelling = local_rng.choice([name, name.upper(), name[:-1]])
            kind, value = expected[name]
            try:
                got = resolver.resolve(spelling)
                ok = kind == "ok" and got == value
            except DNSResolutionError as e:
                ok = kind != "ok" and type(e) is kind and e.name == value
            if not ok:
                with failures_lock:
                    failures.append((seed, idx, spelling, kind, value))
            if local_rng.random() < 0.1:
                resolver.advance_clock(local_rng.randint(0, 3))
        done[idx] = True

    threads = [threading.Thread(target=worker, args=(i,)) for i in range(n_threads)]
    for t in threads:
        t.start()
    for t in threads:
        t.join(timeout=15.0)
    assert all(done), f"seed {seed}: a worker never finished within the timeout (deadlock?)"
    assert not failures, failures[:5]


for seed in range(8):
    _stress(seed)
print("concurrent stress: 8 zones, 20 threads x 60 rounds each, every result matched the reference resolver")

print("all checks passed")
```

</details>

</details>
