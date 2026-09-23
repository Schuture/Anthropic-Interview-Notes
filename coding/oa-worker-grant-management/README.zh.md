# 在线测评：员工考勤、晋升与薪酬

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 在线测评 · 4 关 | ★★☆☆☆ | 中等 | SWE · MLE | oop-design, interval-merge, timestamps, payroll, ranking | 4 关 / 90 分钟 | 在线测评（OA） |
<!-- meta:end -->

## 题目

一个*员工*（worker）由 `worker_id` 标识，这是一个非空字符串，在所有曾经注册过的员工中唯一——一旦
注册，`worker_id` 就在这个实例的整个生命周期内保持注册状态。按顺序实现下面四关：只有当前一关的检查
全部通过，下一关的检查才会开始运行，且每一关都在同一个类上继续扩展，而不是另起炉灶。方法被调用的
顺序，就是对应事件实际发生的顺序；`clock_in` 和 `clock_out` 还各自携带该时刻的 `timestamp`（一个
非负整数），对同一个实例发起的整个 `clock_in`/`clock_out` 调用序列而言，无论涉及哪个员工，时间戳都
不会变小。本题中的每一个区间——一次办公室时段、传给 `calculate_pay` 的窗口，以及一个双倍薪资时段——
都是左闭右开的：`[a, b)` 包含 `a`，不包含 `b`。`hourly_rate` 是一个正整数：与时间戳同单位的一段
时长 `d`，按恒定费率 `r` 计算，收入为 `d * r`。`position` 是一个非空字符串，在员工注册时设置，此后
每次晋升都会和费率一起被替换。

### Level 1 —— 办公室时段

```py
class Workforce:
    def add_worker(self, worker_id: str, position: str, hourly_rate: int) -> bool:
        """Registers a new worker with this position and hourly_rate. Returns True, or False
        without effect, if worker_id is already registered."""

    def clock_in(self, worker_id: str, timestamp: int) -> bool:
        """Opens a new office session for worker_id, starting at timestamp. Returns True, or
        False without effect, if worker_id is not registered, or if worker_id already has an
        open session (a clock_in with no clock_out since)."""

    def clock_out(self, worker_id: str, timestamp: int) -> bool:
        """Closes worker_id's open session as [start, timestamp), start being the timestamp of
        the clock_in that opened it. Returns True, or False without effect, if worker_id is not
        registered, or has no open session."""
```

示例：

```text
add_worker("alice", "Engineer", 15)   # -> True
add_worker("alice", "Manager", 30)    # -> False   -- "alice" 已经注册过
clock_in("alice", 100)                # -> True    -- 打开 [100, ...)
clock_in("alice", 105)                # -> False   -- 已经有一个未结束的时段
clock_out("alice", 140)               # -> True    -- 把这个时段关闭为 [100, 140)
clock_out("alice", 150)               # -> False   -- 没有未结束的时段
clock_in("bob", 120)                  # -> False   -- "bob" 从未注册过
```

### Level 2 —— 总工时与排行榜

一个员工的*已结束*（closed）时段，是指 `clock_out` 已经完成的那些；仍未结束的时段（还没有对应的
`clock_out`）不算在内。

```py
class Workforce:
    def get_total_work_time(self, worker_id: str) -> int | None:
        """Returns the sum of the duration of every closed session of worker_id. Returns None
        if worker_id is not registered. An open session contributes nothing until it is closed."""

    def top_k_workers(self, k: int) -> list[str]:
        """Returns up to k worker_ids, ranked by get_total_work_time descending (workers tied on
        that total are ordered by worker_id ascending), or every registered worker_id in that
        order if fewer than k are registered. k is a positive integer."""
```

示例：

```text
add_worker("alice", "Engineer", 15); add_worker("bob", "Engineer", 20); add_worker("cy", "Designer", 15)
clock_in("alice", 0); clock_out("alice", 40)      # 已结束时段，40 个刻度
clock_in("bob", 0);   clock_out("bob", 25)        # 已结束时段，25 个刻度
clock_in("cy", 0);    clock_out("cy", 40)         # 已结束时段，40 个刻度 -- 与 alice 打平
get_total_work_time("alice")   # -> 40
get_total_work_time("dee")     # -> None   -- 未注册
top_k_workers(2)                # -> ["alice", "cy"]          40 分打平："alice" 排在 "cy" 前面
top_k_workers(10)               # -> ["alice", "cy", "bob"]   只有 3 名员工注册
clock_in("bob", 100)             # bob 的新时段处于未结束状态
top_k_workers(1)                 # -> ["alice"]                bob 未结束的时段仍算作 0
```

### Level 3 —— 晋升与薪酬

`set_promotion` 为一名员工安排一次 `position` 和 `hourly_rate` 的变更；它在该员工*下一次*
`clock_in` 时才生效，既不是立刻生效，也不会追溯到过去——已经处于未结束状态的时段，保留它开始时的
费率。每个员工至多有一个待生效的晋升：在待生效的晋升被消费之前再次调用 `set_promotion`，会把它彻底
替换掉（更早一次调用的 `new_position` 和 `new_hourly_rate` 都被丢弃，而不是逐字段合并）。

```py
class Workforce:
    def set_promotion(self, worker_id: str, new_position: str, new_hourly_rate: int) -> bool:
        """Schedules new_position and new_hourly_rate to replace worker_id's current ones,
        effective at worker_id's next clock_in. Returns True, or False without effect, if
        worker_id is not registered."""

    def calculate_pay(self, worker_id: str, start: int, end: int) -> int | None:
        """Returns worker_id's total pay earned during [start, end): the sum, over every session
        that overlaps [start, end), of the overlap's duration times the rate that was in effect
        when that session's own clock_in happened. A session still open when calculate_pay is
        called is treated as ending exactly at end, as if it were still ongoing through the whole
        query, rather than at any later, real clock_out. Returns None if worker_id is not
        registered. 0 <= start <= end."""
```

示例：

```text
add_worker("alice", "Engineer", 10)
clock_in("alice", 0); clock_out("alice", 50)               # 时段 A：[0, 50)，费率 $10 -> 500
set_promotion("alice", "Senior Engineer", 18)               # 待生效 -- 时段 A 不受影响
clock_in("alice", 80)                                       # 消费待生效的晋升：费率变为 18
clock_out("alice", 100)                                     # 时段 B：[80, 100)，费率 $18 -> 360
calculate_pay("alice", 0, 100)    # -> 860     -- 500（A） + 360（B）
calculate_pay("alice", 10, 130)   # -> 760     -- A 被裁剪为 [10,50)=400；B 完全落在窗口内：360
calculate_pay("bob", 0, 100)      # -> None    -- 未注册
clock_in("alice", 150)                                       # 打开时段 C，仍未结束
calculate_pay("alice", 120, 200)  # -> 900     -- C 被限定到 [150, 200)：50 个刻度，费率 18
calculate_pay("alice", 0, 130)    # -> 860     -- C 被限定到 [150, 130) 为空：clock_in（150）>= end（130）
```

### Level 4 —— 双倍薪资时段

`set_double_pay(start, end)` 把 `[start, end)` 标记为一个*双倍薪资时段*（double-pay period）：在
这段时间里，任何员工赚到的薪资都翻倍。多个双倍薪资时段之间可以互相重叠；两个时段重叠的部分也只会被
加倍一次，绝不会因为重叠而变成 4 倍。`calculate_pay` 相应地被扩展：在按 Level 3 算出的、某个时段
与 `[start, end)` 的重叠部分中，额外落在至少一个双倍薪资时段内的那一小段，按该时段自己费率的两倍
计薪；其余部分仍按该时段自己的费率计薪，不变。

```py
class Workforce:
    def set_double_pay(self, start: int, end: int) -> None:
        """Marks every timestamp in [start, end) as double pay for every worker. 0 <= start <=
        end. Double-pay periods accumulate; a later call never removes or shrinks an earlier one,
        even where they overlap."""
```

`calculate_pay` 的签名和 Level 3 保持一致，变的只是它算出来的值。

示例：

```text
add_worker("alice", "Engineer", 10)
clock_in("alice", 0); clock_out("alice", 50)          # 时段 A：[0, 50)，费率 $10
set_promotion("alice", "Senior Engineer", 20)
clock_in("alice", 60); clock_out("alice", 120)        # 时段 B：[60, 120)，费率 $20
set_double_pay(40, 70)                                 # [40, 70) 双倍薪资
set_double_pay(65, 100)                                # [65, 100) 双倍薪资，与前一个重叠
                                                        # -- 并集是 [40, 100)；重叠部分 [65,70)
                                                        # 也只翻一倍，绝不是 4 倍
calculate_pay("alice", 0, 120)
# 时段 A [0, 50)：[0, 40) 1 倍 -> 40 * 10 = 400；[40, 50) 2 倍 -> 10 * 10 * 2 = 200 -> 600
# 时段 B [60, 120)：[60, 100) 2 倍 -> 40 * 20 * 2 = 1600；[100, 120) 1 倍 -> 20 * 20 = 400 -> 2000
# 合计：600 + 2000 = 2600
```

## 参考解答

<details>
<summary>展开参考解答</summary>

值得事先确认的两点，因为合理的设计在这两点上可能不同：一个还没有 `clock_out` 的未结束时段，
`calculate_pay` 应该怎么处理（这里选择把它限定在查询自己的 `end` 上，因为查询本身并没有单独携带一个
“此刻”的时间）；以及重复调用 `clock_in`（此时已经处于未结束状态）、或者在待生效的晋升被消费之前又
调用一次 `set_promotion`，应该报错还是悄悄地失败或覆盖（这里两者都选择悄悄地失败或覆盖，与
`clock_out` 和 `add_worker` 已经采用的失败信号方式保持一致）。

### Level 1

员工是一个小对象，携带着自己的 `position` 和 `hourly_rate`、一个未结束时段的开始时间戳（如果存在
的话），以及一份已结束时段的列表。已结束的时段以 `(start, end, rate)` 的形式存储，其中的 rate 就是
那整个时段生效的费率——在 `clock_out` 这一刻就把它记下来，不需要额外成本，而这正是 `calculate_pay`
从 Level 3 起需要用到的东西，因为后来的晋升绝不能改变一个已经结束的时段。`clock_in` 还会在打开新
时段之前，先消费掉一个待生效的晋升（如果有的话），这样它打开的这个时段从一开始用的就是新费率。

```python
class _Worker:
    def __init__(self, position: str, hourly_rate: int):
        self.position = position
        self.hourly_rate = hourly_rate
        self.open_clock_in: int | None = None
        self.closed_sessions: list[tuple[int, int, int]] = []   # (start, end, rate), rate fixed at close
        self.pending_promotion: tuple[str, int] | None = None


class Workforce:
    def __init__(self):
        self._workers: dict[str, _Worker] = {}

    def add_worker(self, worker_id: str, position: str, hourly_rate: int) -> bool:
        if worker_id in self._workers:
            return False
        self._workers[worker_id] = _Worker(position, hourly_rate)
        return True

    def clock_in(self, worker_id: str, timestamp: int) -> bool:
        worker = self._workers.get(worker_id)
        if worker is None or worker.open_clock_in is not None:
            return False
        if worker.pending_promotion is not None:      # NOTE: consumed here -- not when set_promotion was called
            worker.position, worker.hourly_rate = worker.pending_promotion
            worker.pending_promotion = None
        worker.open_clock_in = timestamp
        return True

    def clock_out(self, worker_id: str, timestamp: int) -> bool:
        worker = self._workers.get(worker_id)
        if worker is None or worker.open_clock_in is None:
            return False
        # NOTE: worker.hourly_rate cannot have changed since this session opened: clock_in is the only
        # place a promotion is consumed, and a second clock_in is refused while a session is open.
        worker.closed_sessions.append((worker.open_clock_in, timestamp, worker.hourly_rate))
        worker.open_clock_in = None
        return True
```

Level 1 的每个方法都是 $O(1)$。

### Level 2

`get_total_work_time` 只把已结束时段的时长加总，与题目中“已结束”的定义一致；`top_k_workers` 对
所有已注册员工做一次排序，排序键对这个总和取负（这样总时长最大的排在最前面），并保留 `worker_id`
本身作为并列时的次级排序键——因为它没有被取负，所以是升序。

```python
def _closed_time(worker: _Worker) -> int:
    return sum(end - start for start, end, _ in worker.closed_sessions)


def get_total_work_time(self, worker_id: str) -> int | None:
    worker = self._workers.get(worker_id)
    return None if worker is None else _closed_time(worker)


def top_k_workers(self, k: int) -> list[str]:
    ranked = sorted(self._workers.items(), key=lambda item: (-_closed_time(item[1]), item[0]))
    return [worker_id for worker_id, _ in ranked[:k]]


Workforce.get_total_work_time = get_total_work_time
Workforce.top_k_workers = top_k_workers
```

`get_total_work_time` 的开销是 $O(n)$，$n$ 是时段数量；`top_k_workers` 的开销是 $O(m \log m)$，
$m$ 是已注册员工数量（开销主要来自排序；排序之后再截取前 $k$ 个，代价可以忽略不计，在这种规模下不
值得为此单独设计一种堆结构）。

### Level 3

`set_promotion` 永远是把 `pending_promotion` 整个替换掉，所以一个还没被消费的晋升后面再来一个，
留下的只有后一个。`calculate_pay` 需要算出两个左闭右开区间之间重叠的刻度数；同一个 `_overlap`
辅助函数服务于每一个时段，包括未结束的那个——做法是先统一构造出一份 `(start, end, rate)` 三元组
列表，未结束的时段（如果存在）会被赋予 `end` 本身作为它的结束——也就是查询自己的 `end`，而不是之后
某个真实的 `clock_out`。把这个被限定过的区间和其他每个已结束时段一样，送进同一套重叠计算，而不是
单独写一个分支处理它，还带来一个好处：当未结束时段的 `start` 落在查询的 `end` 之上或者正好相等时，
这个被限定的区间本身就是空的，`_overlap` 自然会为它返回 0。

```python
def _overlap(a_start: int, a_end: int, b_start: int, b_end: int) -> int:
    """Length of the overlap between half-open [a_start, a_end) and [b_start, b_end)."""
    return max(0, min(a_end, b_end) - max(a_start, b_start))


def _sessions_as_of(worker: _Worker, query_end: int) -> list[tuple[int, int, int]]:
    """Every closed session, plus the open one (if any) clamped to end at query_end."""
    sessions = list(worker.closed_sessions)
    if worker.open_clock_in is not None:      # NOTE: open at query time -- treated as ongoing through query_end
        sessions.append((worker.open_clock_in, query_end, worker.hourly_rate))
    return sessions


def set_promotion(self, worker_id: str, new_position: str, new_hourly_rate: int) -> bool:
    worker = self._workers.get(worker_id)
    if worker is None:
        return False
    worker.pending_promotion = (new_position, new_hourly_rate)   # NOTE: replaces, does not merge, any older one
    return True


def calculate_pay(self, worker_id: str, start: int, end: int) -> int | None:   # NOTE: redefined by Level 4 below
    worker = self._workers.get(worker_id)
    if worker is None:
        return None
    return sum(_overlap(s, e, start, end) * rate for s, e, rate in _sessions_as_of(worker, end))


Workforce.set_promotion = set_promotion
Workforce.calculate_pay = calculate_pay
```

`set_promotion` 是 $O(1)$；`calculate_pay` 是 $O(n)$，$n$ 是该员工拥有的时段数量，不论已结束还是
未结束。

### Level 4

双倍薪资时段始终以合并过的、互不相交的有序列表形式保存，这样把某一段与每一个已保存时段分别求重叠再
加总，结果就等于它与它们并集的重叠——不会有哪个刻度因为被两个时段同时覆盖就被算两遍。`_merge_intervals`
是标准的扫描做法：按起点排序，只要下一个区间的起点不超过上一个区间的终点，就延长上一个区间，而不是
新开一个（用 `<=` 而不是 `<`，所以像 `[10, 20)` 和 `[20, 30)` 这样首尾相接的区间也会合并成一个——
这不会改变任何区间算出的薪资，只影响时段存储得有多紧凑）。`calculate_pay` 是整个重新定义，而不是
打补丁：对每个时段与 `[start, end)` 的重叠部分，额外落在合并后的时段范围内的那部分按 `2 * rate`
计薪，其余部分按 `rate` 计薪。

```python
def _merge_intervals(intervals: list[tuple[int, int]]) -> list[tuple[int, int]]:
    merged: list[list[int]] = []
    for s, e in sorted(intervals):
        if merged and s <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], e)
        else:
            merged.append([s, e])
    return [(s, e) for s, e in merged]


def set_double_pay(self, start: int, end: int) -> None:
    if not hasattr(self, "_bonus_periods"):     # NOTE: lazy init -- predates __init__, as in Level 1
        self._bonus_periods: list[tuple[int, int]] = []
    self._bonus_periods = _merge_intervals(self._bonus_periods + [(start, end)])


def calculate_pay(self, worker_id: str, start: int, end: int) -> int | None:
    worker = self._workers.get(worker_id)
    if worker is None:
        return None
    bonus_periods = getattr(self, "_bonus_periods", [])
    total = 0
    for s, e, rate in _sessions_as_of(worker, end):
        seg_start, seg_end = max(s, start), min(e, end)
        if seg_start >= seg_end:
            continue
        # NOTE: bonus_periods is already merged, so these overlaps are disjoint and simply add up --
        # a chain of three overlapping double-pay windows still doubles their union once, not 3x or 4x.
        doubled = sum(_overlap(seg_start, seg_end, bs, be) for bs, be in bonus_periods)
        total += (seg_end - seg_start - doubled) * rate + doubled * rate * 2
    return total


Workforce.set_double_pay = set_double_pay
Workforce.calculate_pay = calculate_pay
```

`set_double_pay` 的开销是 $O(p \log p)$，$p$ 是目前已保存的时段数量，开销来自 `_merge_intervals`
内部的排序；`calculate_pay` 每个时段的开销是 $O(n + p)$，$n$ 是时段数量，$p$ 是合并后的时段数量。

### 追问

- 一种变体：双倍薪资时段只有在完整覆盖某个时段时才对它生效，而不是按部分重叠的比例计算——这样
  `calculate_pay` 就会变成逐个时段检查是否被完整覆盖，而不是对 `_bonus_periods` 求 `_overlap`
  再加总，上面按刻度翻倍的做法也会退化成按时段整体、全有或全无的倍率。
- 晋升从某个指定的未来时间戳开始生效，而不是“下一次 `clock_in`”：这就需要让一个时段的费率能在时段
  进行到一半时发生变化，把 `calculate_pay` 里每个时段的重叠部分拆成生效前和生效后两段——就像双倍
  薪资时段现在已经在做的那样。
- 跨所有员工的汇总，比如某个区间内所有被翻倍的刻度对应的薪资总和：这可以直接复用 `_bonus_periods`
  和 `_overlap`，不需要改动，只是把遍历对象从一个员工的时段换成每个员工的时段。
- 同一个员工的并发 `clock_in`/`clock_out`：两者都在没有加锁的情况下读写 `open_clock_in`，所以两个
  线程同时给同一个员工打卡上班，可能都读到它还没设置，于是都各自打开了一个时段。给 `clock_in` 和
  `clock_out` 的整个执行过程加上按员工区分的锁，恰好能把真正会触碰同一员工状态的调用串行化，同时
  不会阻塞其他员工身上的调用。
- 一个时段历史很长、又经常被查询的员工：`calculate_pay` 每次调用都会重新扫描全部时段。把时段按
  `start` 排序一次，再用 `bisect` 只找出可能与 `[start, end)` 重叠的那些并遍历它们，这样开销就只
  取决于真正被触及的时段数量，而不是这名员工的全部历史。

<details>
<summary>验证代码（可运行）</summary>

```python
# ---- Level 1 example ----
w = Workforce()
assert w.add_worker("alice", "Engineer", 15) is True
assert w.add_worker("alice", "Manager", 30) is False
assert w.clock_in("alice", 100) is True
assert w.clock_in("alice", 105) is False
assert w.clock_out("alice", 140) is True
assert w.clock_out("alice", 150) is False
assert w.clock_in("bob", 120) is False

# ---- Level 2 example ----
w = Workforce()
for wid, pos, rate in [("alice", "Engineer", 15), ("bob", "Engineer", 20), ("cy", "Designer", 15)]:
    w.add_worker(wid, pos, rate)
w.clock_in("alice", 0); w.clock_out("alice", 40)
w.clock_in("bob", 0); w.clock_out("bob", 25)
w.clock_in("cy", 0); w.clock_out("cy", 40)
assert w.get_total_work_time("alice") == 40
assert w.get_total_work_time("dee") is None
assert w.top_k_workers(2) == ["alice", "cy"]
assert w.top_k_workers(10) == ["alice", "cy", "bob"]
w.clock_in("bob", 100)
assert w.top_k_workers(1) == ["alice"]

# ---- Level 3 example ----
w = Workforce()
w.add_worker("alice", "Engineer", 10)
w.clock_in("alice", 0); w.clock_out("alice", 50)
w.set_promotion("alice", "Senior Engineer", 18)
w.clock_in("alice", 80)
w.clock_out("alice", 100)
assert w.calculate_pay("alice", 0, 100) == 860
assert w.calculate_pay("alice", 10, 130) == 760
assert w.calculate_pay("bob", 0, 100) is None
w.clock_in("alice", 150)
assert w.calculate_pay("alice", 120, 200) == 900
assert w.calculate_pay("alice", 0, 130) == 860

# ---- Level 4 example ----
w = Workforce()
w.add_worker("alice", "Engineer", 10)
w.clock_in("alice", 0); w.clock_out("alice", 50)
w.set_promotion("alice", "Senior Engineer", 20)
w.clock_in("alice", 60); w.clock_out("alice", 120)
w.set_double_pay(40, 70)
w.set_double_pay(65, 100)
assert w.calculate_pay("alice", 0, 120) == 2600
print("all four levels' worked examples replayed exactly")

# ---- edge cases the examples do not reach ----

# add_worker returning False leaves the original position/rate untouched
w = Workforce()
w.add_worker("a", "Engineer", 10)
w.add_worker("a", "Manager", 999)
w.clock_in("a", 0); w.clock_out("a", 10)
assert w.calculate_pay("a", 0, 10) == 100   # still $10/tick, not 999

# a promotion set while a session is open only reaches the session after the NEXT clock_in
w = Workforce()
w.add_worker("a", "Engineer", 10)
w.clock_in("a", 0)                        # session open
w.set_promotion("a", "Senior", 50)        # pending -- the open session must not see it
w.clock_out("a", 10)
assert w.calculate_pay("a", 0, 10) == 100           # closed at the old rate
w.clock_in("a", 10)                                  # consumes the pending promotion now
w.clock_out("a", 20)
assert w.calculate_pay("a", 10, 20) == 500           # new session at the new rate

# a second set_promotion before the first is consumed: only the later one survives
w = Workforce()
w.add_worker("a", "Engineer", 10)
w.set_promotion("a", "Mid", 20)
w.set_promotion("a", "Senior", 30)        # replaces the pending "Mid"/20 outright
w.clock_in("a", 0); w.clock_out("a", 10)
assert w.calculate_pay("a", 0, 10) == 300

# a zero-duration session (clock_in and clock_out at the same timestamp) contributes nothing
w = Workforce()
w.add_worker("a", "Engineer", 10)
w.clock_in("a", 5); w.clock_out("a", 5)
assert w.get_total_work_time("a") == 0
assert w.calculate_pay("a", 0, 100) == 0

# an open session that starts at or after the query's end contributes nothing
w = Workforce()
w.add_worker("a", "Engineer", 10)
w.clock_in("a", 50)
assert w.calculate_pay("a", 0, 50) == 0
assert w.calculate_pay("a", 0, 49) == 0

# three pairwise-overlapping double-pay periods still double each tick exactly once
w = Workforce()
w.add_worker("a", "Engineer", 10)
w.clock_in("a", 0); w.clock_out("a", 30)
w.set_double_pay(0, 12)
w.set_double_pay(8, 20)
w.set_double_pay(15, 30)                  # union of all three is exactly [0, 30)
assert w.calculate_pay("a", 0, 30) == 30 * 10 * 2

# two separate (non-overlapping) double-pay periods inside one session stay independently additive
w = Workforce()
w.add_worker("a", "Engineer", 10)
w.clock_in("a", 0); w.clock_out("a", 30)
w.set_double_pay(0, 5)      # 5 ticks doubled
w.set_double_pay(20, 25)    # 5 ticks doubled
# [0,5) 2x=100, [5,20) 1x=150, [20,25) 2x=100, [25,30) 1x=50
assert w.calculate_pay("a", 0, 30) == 100 + 150 + 100 + 50

# set_double_pay before any worker, or any session, exists is harmless
w2 = Workforce()
w2.set_double_pay(0, 100)
assert w2.calculate_pay("nobody", 0, 100) is None
w2.add_worker("z", "Engineer", 5)
assert w2.calculate_pay("z", 0, 100) == 0

# top_k_workers with k larger than the number registered, everyone tied at 0
w3 = Workforce()
for wid in ("m", "a", "z"):
    w3.add_worker(wid, "Engineer", 10)
assert w3.top_k_workers(5) == ["a", "m", "z"]   # nobody has clocked in: all tied at 0, ascending id
print("edge cases OK")
```

```python
# ---- independent reference model: tick-by-tick simulation from the raw call log ----
# Shares no code with the Workforce implementation above -- calculate_pay is answered by walking every
# integer tick of the query window one at a time, rather than with interval-overlap arithmetic.
import random


def _slow_replay(log):
    workers: dict[str, dict] = {}
    bonus: list[tuple[int, int]] = []
    for entry in log:
        kind = entry[0]
        if kind == "add_worker":
            _, wid, pos, rate = entry
            if wid not in workers:
                workers[wid] = {"position": pos, "rate": rate, "pending": None, "open": None, "closed": []}
        elif kind == "clock_in":
            _, wid, ts = entry
            rec = workers.get(wid)
            if rec is not None and rec["open"] is None:
                if rec["pending"] is not None:
                    rec["position"], rec["rate"] = rec["pending"]
                    rec["pending"] = None
                rec["open"] = ts
        elif kind == "clock_out":
            _, wid, ts = entry
            rec = workers.get(wid)
            if rec is not None and rec["open"] is not None:
                rec["closed"].append((rec["open"], ts, rec["rate"]))
                rec["open"] = None
        elif kind == "set_promotion":
            _, wid, pos, rate = entry
            rec = workers.get(wid)
            if rec is not None:
                rec["pending"] = (pos, rate)
        else:                                    # "set_double_pay"
            _, s, e = entry
            bonus.append((s, e))
    return workers, bonus


def _slow_total_time(workers, wid):
    return None if wid not in workers else sum(e - s for s, e, _ in workers[wid]["closed"])


def _slow_top_k(workers, k):
    return sorted(workers, key=lambda wid: (-_slow_total_time(workers, wid), wid))[:k]


def _slow_calculate_pay(workers, bonus, wid, start, end):
    if wid not in workers:
        return None
    rec = workers[wid]
    sessions = list(rec["closed"])
    if rec["open"] is not None:
        sessions.append((rec["open"], end, rec["rate"]))
    total = 0
    for tick in range(start, end):
        for s, e, rate in sessions:
            if s <= tick < e:                   # NOTE: one worker's sessions never overlap each other
                total += rate * 2 if any(bs <= tick < be for bs, be in bonus) else rate
                break
    return total


def _apply(workforce, entry):
    getattr(workforce, entry[0])(*entry[1:])


def _run_trial(seed):
    rng = random.Random(seed)
    ids = ["a", "b", "c"]
    workforce, log, ts = Workforce(), [], 0
    for _ in range(rng.randint(15, 25)):
        kind = rng.choices(
            ["add_worker", "clock_in", "clock_out", "set_promotion", "set_double_pay"],
            weights=[2, 5, 5, 3, 2])[0]
        wid = rng.choice(ids)
        if kind == "add_worker":
            entry = ("add_worker", wid, rng.choice(["Eng", "Designer", "Manager"]), rng.randint(2, 12))
        elif kind == "clock_in":
            ts += rng.choice([0, 0, 1, 2])
            entry = ("clock_in", wid, ts)
        elif kind == "clock_out":
            ts += rng.choice([0, 1, 2, 3])
            entry = ("clock_out", wid, ts)
        elif kind == "set_promotion":
            entry = ("set_promotion", wid, rng.choice(["Eng II", "Staff", "Lead"]), rng.randint(2, 12))
        else:
            s = rng.randint(0, min(ts, 40) + 5)
            entry = ("set_double_pay", s, s + rng.randint(0, 8))

        _apply(workforce, entry)
        log.append(entry)
        workers, bonus = _slow_replay(log)

        for wid2 in ids + ["ghost"]:
            assert workforce.get_total_work_time(wid2) == _slow_total_time(workers, wid2), \
                (seed, "get_total_work_time", wid2, log)
        for k in (1, 2, len(ids) + 2):
            assert workforce.top_k_workers(k) == _slow_top_k(workers, k), (seed, "top_k_workers", k, log)
        hi = min(ts + 8, 60)
        for wid2 in ids + ["ghost"]:
            a, b = sorted(rng.choices(range(hi + 1), k=2))
            got, want = workforce.calculate_pay(wid2, a, b), _slow_calculate_pay(workers, bonus, wid2, a, b)
            assert got == want, (seed, "calculate_pay", wid2, a, b, got, want, log)


for seed in range(200):
    _run_trial(seed)
print("cross-validated 200 random operation sequences against an independent tick-by-tick reference model")
print("all checks passed")
```

</details>

</details>
