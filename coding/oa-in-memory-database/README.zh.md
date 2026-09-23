# 在线测评：内存数据库

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 在线测评 · 4 关 | ★★★☆☆ | 中等 | SWE · MLE · Infra Eng | oop-design, hash-table, ttl, snapshotting | 4 关 / 90 分钟 | 在线测评（OA） |
<!-- meta:end -->

## 题目

按顺序实现下面四关：只有当前一关的测试全部通过，下一关的测试才会开始运行，且每一关都在同一个类上继续扩展，
而不是另起炉灶。一条*记录*（record）由一个字符串 `key` 标识，保存一组*字段*（field），每个字段是一个字符串
字段名到字符串值的映射；`key`、字段名和值都是非空字符串，且字段名和值都不包含 `(`、`)` 或 `,`。每个方法都
返回一个字符串；没有实质内容可返回时返回 `""`。

### Level 1 —— 核心操作

```py
class InMemoryDatabase:
    def set(self, key: str, field: str, value: str) -> str:
        """Sets field of the record key to value, creating the record if key is new and
        overwriting any value already held there. Returns ""."""

    def get(self, key: str, field: str) -> str:
        """Returns the value held in field of key, or "" if key does not exist or holds no
        such field."""

    def delete(self, key: str, field: str) -> str:
        """Removes field from key if present. Returns "true" if a field was removed, "false"
        if key did not exist or held no such field."""
```

示例：

```text
db = InMemoryDatabase()
db.set("order-1", "status", "packed")     # -> ""
db.set("order-1", "carrier", "fedex")     # -> ""
db.get("order-1", "status")               # -> "packed"
db.get("order-1", "weight")               # -> ""            -- 这个字段从未被设置过
db.get("order-9", "status")               # -> ""            -- 这个 key 从未被创建过
db.delete("order-1", "carrier")           # -> "true"
db.delete("order-1", "carrier")           # -> "false"        -- 已经被删掉了
db.set("order-1", "status", "shipped")    # -> ""             -- 覆盖写入
db.get("order-1", "status")               # -> "shipped"
```

### Level 2 —— 扫描

```py
class InMemoryDatabase:
    def scan(self, key: str) -> str:
        """Returns every field of key formatted as "name1(value1), name2(value2), ...", with
        fields sorted lexicographically by name (Python's default string ordering); "" if key
        does not exist or holds no fields."""

    def scan_by_prefix(self, key: str, prefix: str) -> str:
        """Same as scan, restricted to the fields of key whose name starts with prefix
        (prefix="" behaves exactly like scan)."""
```

示例：

```text
db = InMemoryDatabase()
db.set("order-1", "carrier", "fedex")
db.set("order-1", "cost", "12.50")
db.set("order-1", "status", "packed")
db.scan_by_prefix("order-1", "c")   # -> "carrier(fedex), cost(12.50)"
db.scan("order-1")                  # -> "carrier(fedex), cost(12.50), status(packed)"
db.scan_by_prefix("order-9", "c")   # -> ""                              -- 这个 key 从未被创建过
```

### Level 3 —— 时间戳与 TTL

从这一关开始，每次调用都发生在一个整数*时间戳*（timestamp）上；对同一个实例发起的整个调用序列而言，无论
时间戳是否显式给出，都不会变小。Level 1 和 Level 2 的五个方法照常工作，语义和上面完全一样，只是它们现在
内部发生在一个时钟的当前值上——这个时钟从 $0$ 开始，每调用一次这样的方法就前进 $1$。这五个方法各自还多出
一个名字以 `_at` 结尾的孪生版本，它把显式的 `timestamp` 作为第一个参数，而不是读时钟；调用其中一个会把
时钟设为 `timestamp`（它不会小于时钟当前的值）。两种调用共用同一个时钟，因此用普通的 `set` 写入的字段
可以被 `get_at` 读到，用 `set_at_with_ttl` 写入的字段也可能在之后某次普通的 `get` 调用时已经过期。

```py
class InMemoryDatabase:
    def set_at(self, timestamp: int, key: str, field: str, value: str) -> str:
        """Same as set, at timestamp. Clears any TTL field previously held, even one that has
        not expired yet -- the field becomes permanent."""

    def set_at_with_ttl(self, timestamp: int, key: str, field: str, value: str, ttl: int) -> str:
        """Same as set_at, but field is visible from timestamp up to, but not including,
        timestamp + ttl (ttl is a positive integer). A later set_at_with_ttl call on the same
        field replaces this window outright, computed fresh from the new call; it does not
        extend the old one."""

    def get_at(self, timestamp: int, key: str, field: str) -> str:
        """Same as get, at timestamp: a field whose TTL window has elapsed by timestamp counts
        as absent."""

    def delete_at(self, timestamp: int, key: str, field: str) -> str:
        """Same as delete, at timestamp; a field whose TTL window has elapsed by timestamp
        counts as already absent, so this returns "false" for it."""

    def scan_at(self, timestamp: int, key: str) -> str:
        """Same as scan, at timestamp."""

    def scan_by_prefix_at(self, timestamp: int, key: str, prefix: str) -> str:
        """Same as scan_by_prefix, at timestamp."""
```

示例：

```text
db = InMemoryDatabase()
db.set_at_with_ttl(10, "order-1", "carrier", "fedex", 5)  # carrier 在 [10, 15) 期间存活
db.set_at_with_ttl(12, "order-1", "carrier", "ups", 6)    # 替换：窗口变为 [12, 18)，不是从 15 延长到 21
db.set_at(13, "order-1", "cost", "12.50")                  # 没有 ttl -> cost 永久存在
db.scan_at(17, "order-1")                # -> "carrier(ups), cost(12.50)"   17 < 18：carrier 仍然存活
db.scan_at(18, "order-1")                # -> "cost(12.50)"                 18 == 18：carrier 刚好过期
db.delete_at(18, "order-1", "carrier")   # -> "false"    已经过期 -- 没有可删除的内容
db.set_at_with_ttl(19, "order-1", "carrier", "usps", 3)    # carrier 再次存活，在 [19, 22) 期间
db.set_at(20, "order-1", "carrier", "dhl")                 # 没有 ttl -> 清除了 TTL，现在永久存在
db.scan_at(22, "order-1")                # -> "carrier(dhl), cost(12.50)"   22 正是旧的过期时刻，但 TTL 已被清除
```

### Level 4 —— 备份与恢复

```py
class InMemoryDatabase:
    def backup(self, timestamp: int) -> str:
        """Deep-snapshots every field currently live at timestamp under Level 3's rules. A
        field with a TTL is stored in the snapshot as the *remaining* time until its expiry
        (its own expiry minus timestamp), not as an absolute expiry; a permanent field is
        stored as permanent. A second backup at the same timestamp replaces the first.
        Returns ""."""

    def restore(self, timestamp: int, timestamp_to_restore: int) -> str:
        """Finds the backup with the largest own timestamp that is <= timestamp_to_restore (at
        least one such backup is guaranteed to have been made) and makes it the database's
        entire live state as of timestamp, discarding every write and delete made since --
        including ones made by an earlier restore. Every field of the restored snapshot that
        carried a remaining-TTL delta d gets a new expiry of timestamp + d: its countdown
        resumes from the moment of this restore, not from the moment of the original backup or
        of the field's original write. No backup is ever deleted, so a later restore call may
        still reach a backup made after this one. Returns ""."""
```

示例：

```text
db = InMemoryDatabase()
db.set_at_with_ttl(0, "order-1", "carrier", "fedex", 10)  # 在 [0, 10) 期间存活
db.set_at(0, "order-1", "cost", "12.50")                   # 永久存在
db.backup(4)                              # -> ""   快照：carrier 剩余 6（10 - 4），cost 永久
db.set_at(5, "order-1", "cost", "15.00")  # 备份之后实时数据库发生了变化；快照不受影响
db.delete_at(6, "order-1", "carrier")     # -> "true"   carrier 在 6 时仍然存活（6 < 10）
db.restore(100, 4)                        # -> ""   在时间戳 100 恢复 ts=4 的那次备份
db.get_at(100, "order-1", "cost")         # -> "12.50"   回到备份之前的值
db.get_at(100, "order-1", "carrier")      # -> "fedex"    恢复出来，剩余 6 个 tick，从 100 重新开始
db.get_at(105, "order-1", "carrier")      # -> "fedex"    100 + 6 = 106：在 105 时仍然存活
db.get_at(106, "order-1", "carrier")      # -> ""         106 正是新的过期时刻——再次不含边界
```

## 参考解答

<details>
<summary>展开参考解答</summary>

值得事先和面试官确认的两点：`get` 在 key 或字段缺失时应该返回什么（这里用 `""`，与 `delete`、`scan` 用
`""` 表示“这里什么都没有”保持一致）；以及一旦引入 TTL，Level 1、Level 2 的无时间戳方法是继续保留，还是
被带时间戳的版本取代（这里选择保留，靠一个内部时钟驱动，因为分关解锁的评测机制会在解锁新一关之后继续跑
之前几关的测试）。

### Level 1

一条记录就是一个普通的 `dict[str, str]`；整个数据库是一个从 key 到记录的 `dict[str, dict[str, str]]`。
`set` 用 `setdefault`，这样第一次用到某个 key 时会自动建出一条空记录；`get` 把两次 `.get` 调用串起来，
每一步都以 `""` 作为默认值，一行代码就同时处理了“key 不存在”和“字段不存在”两种情况；`delete` 只查一次
字段，只有字段存在时才删除，所以重复调用能正确返回 `"false"`。

```python
class InMemoryDatabase:
    def __init__(self) -> None:
        self._records: dict[str, dict[str, str]] = {}

    def set(self, key: str, field: str, value: str) -> str:
        self._records.setdefault(key, {})[field] = value
        return ""

    def get(self, key: str, field: str) -> str:
        return self._records.get(key, {}).get(field, "")

    def delete(self, key: str, field: str) -> str:
        fields = self._records.get(key, {})
        if field in fields:
            del fields[field]
            return "true"
        return "false"
```

每个方法都是 $O(1)$。

### Level 2

`scan` 和 `scan_by_prefix` 共用一个格式化步骤 `_format_fields`：先对通过筛选的字段名排序——`scan` 是全部
字段，另一个是以 `prefix` 开头的那些——再把 `"name(value)"` 这样的片段用 `", "` 连接起来。先筛选再排序，
而不是先给整条记录排序再筛选，让 `scan_by_prefix` 只需要遍历记录一遍，再对匹配到的那部分排序。

```python
def _format_fields(fields: dict[str, str], names) -> str:
    return ", ".join(f"{name}({fields[name]})" for name in names)


class InMemoryDatabase(InMemoryDatabase):
    def scan(self, key: str) -> str:
        fields = self._records.get(key, {})
        return _format_fields(fields, sorted(fields))

    def scan_by_prefix(self, key: str, prefix: str) -> str:
        fields = self._records.get(key, {})
        return _format_fields(fields, sorted(f for f in fields if f.startswith(prefix)))
```

把 `InMemoryDatabase` 重新打开、定义成它自己的子类，而不是直接修改 Level 1 的代码块，这也是本解答后面
逐关扩展这个类的方式。两个方法的开销都是 $O(m \log m)$（$m$ 是这条记录的字段数）：Python 标准库没有
现成的有序字符串索引可以直接定位出 `startswith` 匹配的那些字段。

### Level 3

Level 3 重构了存储方式，而不是在原有基础上叠加：一旦字段可能过期，只保留每个字段的当前值就不够了，于是
每个字段变成一对 `(value, expiry)`——永久字段的 `expiry` 为 `None`，否则是一个绝对时间戳。由于记录字典的
值类型变了，Level 1、Level 2 的方法在这里是被重写，而不是原样继承——比如直接继承 `get` 的话，返回的会是
原始的 `(value, expiry)` 元组，而不是 `value`。一个内部时钟 `_clock` 为无时间戳的方法提供时间：`_tick`
读取并推进它，供普通调用使用；`_bump` 则把一个显式时间戳并入时钟。`_alive` 只检查字段窗口的上界——
$[t, t+ttl)$ 的下界完全不需要单独检查，因为任何一次写入都已经发生在同一个不递减的时间戳序列中更早的
位置，一次查询不可能比它正在读取的写入更早到达。

```python
class InMemoryDatabase(InMemoryDatabase):
    def __init__(self) -> None:
        # _records[key][field] = (value, expiry); expiry is None (permanent) or the first
        # timestamp at which the field is gone.
        self._records: dict[str, dict[str, tuple[str, int | None]]] = {}
        self._clock = 0

    def _tick(self) -> int:
        timestamp = self._clock
        self._clock += 1
        return timestamp

    def _bump(self, timestamp: int) -> None:
        self._clock = max(self._clock, timestamp)   # NOTE: max, not "=" -- defends the ordering guarantee

    @staticmethod
    def _alive(entry: tuple[str, int | None] | None, timestamp: int) -> bool:
        return entry is not None and (entry[1] is None or timestamp < entry[1])   # NOTE: strict "<": t+ttl is dead

    def set(self, key: str, field: str, value: str) -> str:
        return self.set_at(self._tick(), key, field, value)

    def get(self, key: str, field: str) -> str:
        return self.get_at(self._tick(), key, field)

    def delete(self, key: str, field: str) -> str:
        return self.delete_at(self._tick(), key, field)

    def scan(self, key: str) -> str:
        return self.scan_at(self._tick(), key)

    def scan_by_prefix(self, key: str, prefix: str) -> str:
        return self.scan_by_prefix_at(self._tick(), key, prefix)

    def set_at(self, timestamp: int, key: str, field: str, value: str) -> str:
        self._bump(timestamp)
        self._records.setdefault(key, {})[field] = (value, None)   # NOTE: plain-setting always clears a TTL
        return ""

    def set_at_with_ttl(self, timestamp: int, key: str, field: str, value: str, ttl: int) -> str:
        self._bump(timestamp)
        self._records.setdefault(key, {})[field] = (value, timestamp + ttl)   # NOTE: a fresh window, not extended
        return ""

    def get_at(self, timestamp: int, key: str, field: str) -> str:
        self._bump(timestamp)
        entry = self._records.get(key, {}).get(field)
        return entry[0] if self._alive(entry, timestamp) else ""

    def delete_at(self, timestamp: int, key: str, field: str) -> str:
        self._bump(timestamp)
        fields = self._records.get(key, {})
        if self._alive(fields.get(field), timestamp):
            del fields[field]
            return "true"
        return "false"                                            # NOTE: an already-expired field is "false"

    def _live_fields(self, key: str, timestamp: int) -> dict[str, str]:
        fields = self._records.get(key, {})
        return {name: value for name, (value, expiry) in fields.items()
                if self._alive((value, expiry), timestamp)}

    def scan_at(self, timestamp: int, key: str) -> str:
        self._bump(timestamp)
        live = self._live_fields(key, timestamp)
        return _format_fields(live, sorted(live))

    def scan_by_prefix_at(self, timestamp: int, key: str, prefix: str) -> str:
        self._bump(timestamp)
        live = self._live_fields(key, timestamp)
        return _format_fields(live, sorted(f for f in live if f.startswith(prefix)))
```

这是一种惰性过期（lazy expiry）：是否存活在每次读取时才检查，而不是靠定时器或者在每次写入时主动清扫
过期字段。这样已经够用了，因为这个类里没有任何地方需要在不先读取字段的情况下知道字段数量，所以主动清理
（eager purge）省下的只有内存，而这部分内存反正会在下一次 `set_at` 或 `set_at_with_ttl` 覆盖同一个字段时
被回收。主动清扫需要一个后台线程，或者在每次调用里都塞一次检查，才能换来惰性过期本就免费拥有的那点收益。

### Level 4

`backup` 复制的正是 `_alive` 已经判定为在 `timestamp` 时刻可见的那些字段，直接复用这个判断，而不是重新
推导一遍存活性；复制时保存的是每个字段*剩余*的 TTL，而不是它的绝对过期时间，这正是 `restore` 所依赖的
细节。`restore` 用 `max` 加一个生成器表达式找到合适的备份，把每个字段的过期时间相对这次恢复自己的
`timestamp` 重新计算，然后整体替换 `_records`：备份之后才创建的字段根本不在快照里，所以重建出来的字典里
自然也没有它，不需要单独一步去删除。

```python
import copy


class InMemoryDatabase(InMemoryDatabase):
    def __init__(self) -> None:
        super().__init__()
        self._backups: dict[int, dict[str, dict[str, tuple[str, int | None]]]] = {}

    def backup(self, timestamp: int) -> str:
        self._bump(timestamp)
        snapshot: dict[str, dict[str, tuple[str, int | None]]] = {}
        for key, fields in self._records.items():
            live = {name: (value, None if expiry is None else expiry - timestamp)
                    for name, (value, expiry) in fields.items() if self._alive((value, expiry), timestamp)}
            if live:
                snapshot[key] = live
        self._backups[timestamp] = snapshot         # NOTE: a second backup at this timestamp replaces the first
        return ""

    def restore(self, timestamp: int, timestamp_to_restore: int) -> str:
        self._bump(timestamp)
        backup_ts = max(ts for ts in self._backups if ts <= timestamp_to_restore)
        restored: dict[str, dict[str, tuple[str, int | None]]] = {}
        for key, fields in self._backups[backup_ts].items():
            restored[key] = {name: (value, None if delta is None else timestamp + delta)   # NOTE: delta, not the
                              for name, (value, delta) in fields.items()}                   #  backup's own clock
        self._records = copy.deepcopy(restored)      # NOTE: deep copy -- later writes must not mutate the backup
        return ""
```

两个方法的开销都是 $O(\text{数据库大小})$：`backup` 把每个存活字段都走一遍，`restore` 则根据选中的快照
重建整个记录字典。保存剩余 TTL 而不是绝对过期时间，正是让恢复出来的字段的倒计时能从 `timestamp` 重新
开始的原因：如果直接保存过期时间，就等于原封不动地把备份当时的时钟读数带到了未来，于是一个备份时还剩
$6$ 个 tick 的字段，恢复出来后可能已经过期，也可能剩下远不止 $6$ 个 tick，具体取决于恢复之前实际过去了
多久——这和“从恢复的那一刻重新开始倒计时”的含义正好相反。

### 追问

- 回望式（look-back）的变体会用一个历史查询方法取代 `backup`/`restore`：给定 key、字段和一个过去的
  `timestamp`，直接在按字段维护的一份只追加的历史记录上，对写入时间戳做二分查找来回答；它能在
  $O(\log n)$ 内回答某个字段的历史值，完全不需要生成快照，但没有等价于 `restore` 那种“把这一整个状态
  重新变成当前状态”的能力，因为把每个 key 都回滚回去，就意味着要对存在过的每一个字段都重新做一次这样
  的查找。
- 要让这个类线程安全，需要把锁加在每个 `_at` 方法里“读取 - 判断 - 写入”的整个序列外面，而不能只锁住
  其中访问字典的那一步：否则两个线程同时对同一个字段调用 `set_at_with_ttl`，可能会在任何一个线程写入
  新值之前，两个都先读到了旧的条目。
- 这里的 `backup` 和 `restore` 都是对整个实时数据库做复制；如果数据库大到每次备份都整体复制不现实，就
  应该改成与上一次备份做差异对比，或者使用一种持久化、结构共享的数据结构，让一次快照的开销只跟变化的
  部分成正比，而不是跟数据库的总大小成正比。
- 像大多数真实的键值存储那样，对 key、字段或值加上长度上限，就需要 `set`、`set_at`、`set_at_with_ttl`
  校验并拒绝超长的写入，并且要为这种情况规定一个确切的返回值。

<details>
<summary>验证代码（可运行）</summary>

```python
# ---- Level 1 example ----
db = InMemoryDatabase()
assert db.set("order-1", "status", "packed") == ""
assert db.set("order-1", "carrier", "fedex") == ""
assert db.get("order-1", "status") == "packed"
assert db.get("order-1", "weight") == ""
assert db.get("order-9", "status") == ""
assert db.delete("order-1", "carrier") == "true"
assert db.delete("order-1", "carrier") == "false"
assert db.set("order-1", "status", "shipped") == ""
assert db.get("order-1", "status") == "shipped"

# ---- Level 2 example ----
db = InMemoryDatabase()
db.set("order-1", "carrier", "fedex")
db.set("order-1", "cost", "12.50")
db.set("order-1", "status", "packed")
assert db.scan_by_prefix("order-1", "c") == "carrier(fedex), cost(12.50)"
assert db.scan("order-1") == "carrier(fedex), cost(12.50), status(packed)"
assert db.scan_by_prefix("order-9", "c") == ""

# ---- Level 3 example ----
db = InMemoryDatabase()
assert db.set_at_with_ttl(10, "order-1", "carrier", "fedex", 5) == ""
assert db.set_at_with_ttl(12, "order-1", "carrier", "ups", 6) == ""
assert db.set_at(13, "order-1", "cost", "12.50") == ""
assert db.scan_at(17, "order-1") == "carrier(ups), cost(12.50)"
assert db.scan_at(18, "order-1") == "cost(12.50)"
assert db.delete_at(18, "order-1", "carrier") == "false"
assert db.set_at_with_ttl(19, "order-1", "carrier", "usps", 3) == ""
assert db.set_at(20, "order-1", "carrier", "dhl") == ""
assert db.scan_at(22, "order-1") == "carrier(dhl), cost(12.50)"

# ---- Level 4 example ----
db = InMemoryDatabase()
db.set_at_with_ttl(0, "order-1", "carrier", "fedex", 10)
db.set_at(0, "order-1", "cost", "12.50")
assert db.backup(4) == ""
db.set_at(5, "order-1", "cost", "15.00")
db.delete_at(6, "order-1", "carrier")
assert db.restore(100, 4) == ""
assert db.get_at(100, "order-1", "cost") == "12.50"
assert db.get_at(100, "order-1", "carrier") == "fedex"
assert db.get_at(105, "order-1", "carrier") == "fedex"
assert db.get_at(106, "order-1", "carrier") == ""
print("all four levels' worked examples replayed exactly")

# ---- edge cases the examples do not reach ----
db = InMemoryDatabase()
assert db.delete("ghost", "field") == "false"
assert db.scan("ghost") == ""
db.set("k", "alpha", "1")
db.set("k", "beta", "2")
assert db.scan_by_prefix("k", "") == db.scan("k") == "alpha(1), beta(2)"

# plain and _at calls share one clock
db = InMemoryDatabase()
db.set("k", "a", "1")                                 # plain: happens at internal tick 0
assert db.get_at(50, "k", "a") == "1"                  # a later explicit read still sees it
assert db.set_at_with_ttl(60, "k", "b", "2", 3) == ""  # b alive on [60, 63)
for _ in range(3):
    db.get("k", "z")                                   # three plain calls, unrelated field: clock 60 -> 63
assert db.get("k", "b") == ""                          # this plain call queries at exactly 63: b just expired

# a field that never had a TTL survives arbitrarily far into the future
db = InMemoryDatabase()
db.set_at(0, "k", "permanent", "x")
assert db.get_at(10 ** 6, "k", "permanent") == "x"

# multiple backups: restore keeps every one of them reachable, including ones made after an earlier restore
db = InMemoryDatabase()
db.set_at(0, "k", "a", "v0")
db.backup(10)
db.set_at(11, "k", "a", "v1")
db.backup(20)
db.set_at(21, "k", "a", "v2")
assert db.restore(30, 15) == ""                        # -> the backup at ts=10 (the latest <= 15)
assert db.get_at(30, "k", "a") == "v0"
assert db.restore(40, 25) == ""                        # -> the backup at ts=20, still there after the earlier restore
assert db.get_at(40, "k", "a") == "v1"

# a restore discards a field created after the backup, with no explicit delete needed
db = InMemoryDatabase()
db.set_at(0, "k", "a", "v0")
db.backup(5)
db.set_at(6, "k", "brand-new", "x")
db.restore(10, 5)
assert db.get_at(10, "k", "brand-new") == ""
assert db.scan_at(10, "k") == "a(v0)"
print("edge cases OK")
```

```python
import random


class NaiveDatabase:
    """Independent reference model, straight from the statement: every (key, field) is an append-only
    list of (ts, value, ttl) events in call order, where value=None marks a delete. A query answers
    by scanning the whole list for the last event at or before the query time -- "the value is
    whatever it was last set to, alive while ts < set_ts + ttl" -- with no lazy-expiry cleverness."""

    def __init__(self) -> None:
        self.history: dict[tuple[str, str], list[tuple[int, str | None, int | None]]] = {}
        self.backups: dict[int, dict[tuple[str, str], tuple[str, int | None]]] = {}
        self.clock = 0

    def _tick(self) -> int:
        ts = self.clock
        self.clock += 1
        return ts

    def _bump(self, ts: int) -> None:
        self.clock = max(self.clock, ts)

    def _last(self, key: str, field: str, ts: int):
        chosen = None
        for event in self.history.get((key, field), []):
            if event[0] <= ts:
                chosen = event
        return chosen

    def _value(self, key: str, field: str, ts: int) -> str | None:
        chosen = self._last(key, field, ts)
        if chosen is None or chosen[1] is None:
            return None
        e_ts, value, ttl = chosen
        return None if (ttl is not None and ts >= e_ts + ttl) else value

    def _write(self, ts, key, field, value, ttl=None) -> None:
        self.history.setdefault((key, field), []).append((ts, value, ttl))

    def set_at(self, ts, key, field, value) -> str:
        self._bump(ts)
        self._write(ts, key, field, value)
        return ""

    def set_at_with_ttl(self, ts, key, field, value, ttl) -> str:
        self._bump(ts)
        self._write(ts, key, field, value, ttl)
        return ""

    def get_at(self, ts, key, field) -> str:
        self._bump(ts)
        value = self._value(key, field, ts)
        return value if value is not None else ""

    def delete_at(self, ts, key, field) -> str:
        self._bump(ts)
        existed = self._value(key, field, ts) is not None
        self._write(ts, key, field, None)
        return "true" if existed else "false"

    def _live_names(self, key, ts) -> list[str]:
        return sorted(f for (k, f) in self.history if k == key and self._value(k, f, ts) is not None)

    def scan_at(self, ts, key) -> str:
        self._bump(ts)
        names = self._live_names(key, ts)
        return ", ".join(f"{f}({self._value(key, f, ts)})" for f in names)

    def scan_by_prefix_at(self, ts, key, prefix) -> str:
        self._bump(ts)
        names = [f for f in self._live_names(key, ts) if f.startswith(prefix)]
        return ", ".join(f"{f}({self._value(key, f, ts)})" for f in names)

    def backup(self, ts) -> str:
        self._bump(ts)
        snap = {}
        for pair in self.history:
            value = self._value(*pair, ts)
            if value is not None:
                e_ts, _, ttl = self._last(*pair, ts)
                snap[pair] = (value, None if ttl is None else e_ts + ttl - ts)
        self.backups[ts] = snap
        return ""

    def restore(self, ts, ts_to_restore) -> str:
        self._bump(ts)
        best = max(b for b in self.backups if b <= ts_to_restore)
        snapshot = self.backups[best]
        for pair in list(self.history):
            if pair in snapshot:
                value, remaining = snapshot[pair]
                self._write(ts, pair[0], pair[1], value, remaining)
            else:
                self._write(ts, pair[0], pair[1], None)
        return ""

    def set(self, key, field, value) -> str:
        return self.set_at(self._tick(), key, field, value)

    def get(self, key, field) -> str:
        return self.get_at(self._tick(), key, field)

    def delete(self, key, field) -> str:
        return self.delete_at(self._tick(), key, field)

    def scan(self, key) -> str:
        return self.scan_at(self._tick(), key)

    def scan_by_prefix(self, key, prefix) -> str:
        return self.scan_by_prefix_at(self._tick(), key, prefix)


def _run_trial(seed: int, n_ops: int = 150) -> dict[str, int]:
    rng = random.Random(seed)
    real, naive = InMemoryDatabase(), NaiveDatabase()
    keys, fields, values = [f"k{i}" for i in range(3)], [f"f{i}" for i in range(4)], [f"v{i}" for i in range(5)]
    sim_clock, backup_tss = 0, []
    counts = {"plain": 0, "at": 0, "ttl": 0, "backup": 0, "restore": 0, "delete_hit": 0}

    def explicit_ts() -> int:
        nonlocal sim_clock
        sim_clock += rng.randint(0, 3)
        return sim_clock

    for _ in range(n_ops):
        pick, key, field, value = rng.random(), rng.choice(keys), rng.choice(fields), rng.choice(values)
        if pick < 0.12:
            counts["plain"] += 1
            method = rng.choice(["set", "get", "delete", "scan", "scan_by_prefix"])
            args = {"set": (key, field, value), "get": (key, field), "delete": (key, field),
                    "scan": (key,), "scan_by_prefix": (key, field[:1])}[method]
            got, want = getattr(real, method)(*args), getattr(naive, method)(*args)
            assert got == want, (seed, method, args, got, want)
            counts["delete_hit"] += method == "delete" and want == "true"
            sim_clock += 1                                    # mirrors the plain call's own tick
        elif pick < 0.55:
            counts["at"] += 1
            ts = explicit_ts()
            method = rng.choice(["set_at", "get_at", "delete_at", "scan_at", "scan_by_prefix_at"])
            args = {"set_at": (ts, key, field, value), "get_at": (ts, key, field), "delete_at": (ts, key, field),
                    "scan_at": (ts, key), "scan_by_prefix_at": (ts, key, field[:1])}[method]
            got, want = getattr(real, method)(*args), getattr(naive, method)(*args)
            assert got == want, (seed, method, args, got, want)
            counts["delete_hit"] += method == "delete_at" and want == "true"
        elif pick < 0.75:
            counts["ttl"] += 1
            ts, ttl = explicit_ts(), rng.randint(1, 6)
            assert real.set_at_with_ttl(ts, key, field, value, ttl) == naive.set_at_with_ttl(ts, key, field, value, ttl) == ""
        elif pick < 0.85:
            counts["backup"] += 1
            ts = explicit_ts()
            assert real.backup(ts) == naive.backup(ts) == ""
            backup_tss.append(ts)
        elif backup_tss:
            counts["restore"] += 1
            ts = explicit_ts()
            ts_to_restore = rng.choice(backup_tss + [rng.randint(min(backup_tss), max(backup_tss) + 2)])
            assert real.restore(ts, ts_to_restore) == naive.restore(ts, ts_to_restore) == ""
    return counts


totals = {"plain": 0, "at": 0, "ttl": 0, "backup": 0, "restore": 0, "delete_hit": 0}
for seed in range(300):
    trial = _run_trial(seed)
    totals = {k: totals[k] + trial[k] for k in totals}
assert min(totals.values()) > 50, totals   # every kind of call, including a delete that actually removed something
print(f"cross-validated 300 random sequences against an independent reference model: {totals}")
print("all checks passed")
```

</details>

</details>
