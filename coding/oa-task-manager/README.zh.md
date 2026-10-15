# 在线测评：带过期与快照的任务管理

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 在线测评 · 4 关 | ★★☆☆☆ | 中等 | SWE · MLE | oop-design, hash-table, ttl, time-travel, bisect | 4 关 / 90 分钟 | 在线测评（OA） |
<!-- meta:end -->

## 题目

按顺序实现下面四关：只有当前一关的测试全部通过，下一关的测试才会开始运行，且每一关都在同一个类上继续
扩展，而不是另起炉灶。一个*任务*（task）恰好属于一个*用户*（user，一个字符串形式的用户 id），拥有
`task_id`、`title`（标题）和一个整数 `priority`（优先级，数值越大表示任务越紧急）；同一个用户的多个
任务之间、或者不同用户的任务之间，标题都可以重复，但 `task_id` 在整个实例范围内始终唯一。每一个会修改
状态的方法（`add_task`、`update_task`、`complete_task`）都把一个整数*时间戳*（timestamp）作为第一个
参数；对同一个实例发起的这一类调用而言，整个序列中时间戳不会变小——同一个任务的两次调用，或者不同任务
的两次调用，都可能共用同一个时间戳。每次读取都把任务作为一个 `Task` 对象返回，它是一条普通的不可变记录：

```py
from dataclasses import dataclass


@dataclass(frozen=True)
class Task:
    task_id: str
    title: str
    priority: int
    created_at: int
```

### Level 1 —— 核心操作

一个任务从被创建的那一刻起就是*活跃*（active）的，直到它最迟在被标记完成的那一刻为止；`get_task`、
`update_task` 和 `complete_task` 都把一个不活跃的任务当作它的 `task_id` 根本不存在一样处理。

```py
class TaskManager:
    def add_task(self, timestamp: int, user: str, title: str, priority: int) -> str:
        """Creates a new task owned by user, with the given title and priority, created at
        timestamp. Returns the new task's id. Ids are assigned from one counter shared by every
        user, in call order: the first call to add_task on this instance returns "task-1", the
        second "task-2", and so on -- ids are always unique, whatever title or user they carry."""

    def update_task(self, timestamp: int, user: str, task_id: str, title: str, priority: int) -> bool:
        """Replaces the title and priority of task_id, leaving created_at unchanged. Returns True
        on success. Returns False, and changes nothing, if task_id does not exist, is not owned by
        user, or is not active."""

    def complete_task(self, timestamp: int, user: str, task_id: str) -> bool:
        """Marks task_id completed: from this call on, it is no longer active. Returns True on
        success. Returns False, and changes nothing, if task_id does not exist, is not owned by
        user, or is already not active (including a task that is already completed)."""

    def get_task(self, timestamp: int, user: str, task_id: str) -> Task | None:
        """Returns task_id's current title, priority, and creation timestamp if it exists, is
        owned by user, and is active. None otherwise."""
```

示例：

```text
tm = TaskManager()
tm.add_task(10, "alice", "write report", 3)     # -> "task-1"
tm.add_task(12, "alice", "review PR", 5)        # -> "task-2"
tm.add_task(14, "bob", "write report", 1)       # -> "task-3"   -- title reused; owned by a different user
tm.get_task(15, "alice", "task-1")   # -> Task("task-1", "write report", 3, 10)
tm.get_task(15, "alice", "task-3")   # -> None                  -- task-3 belongs to bob, not alice
tm.update_task(16, "alice", "task-1", "write Q3 report", 4)   # -> True
tm.get_task(17, "alice", "task-1")   # -> Task("task-1", "write Q3 report", 4, 10)   -- created_at unchanged
tm.complete_task(18, "alice", "task-1")   # -> True
tm.complete_task(19, "alice", "task-1")   # -> False             -- already completed
tm.get_task(20, "alice", "task-1")   # -> None                   -- a completed task is not active
tm.update_task(21, "alice", "task-1", "x", 1)   # -> False       -- cannot update a task that is not active
```

### Level 2 —— 列表与过滤

```py
class TaskManager:
    def get_task_list(self, at_timestamp: int, user: str, min_priority: int | None = None) -> list[Task]:
        """Returns every active task owned by user whose priority is >= min_priority (every active
        task owned by user, if min_priority is None), evaluated at at_timestamp. Through Level 3,
        at_timestamp is guaranteed to equal the greatest timestamp passed to any call made so far
        on this instance -- every list here asks about the present, never the past. Tasks are
        sorted by priority descending; ties are broken by creation order ascending -- the order
        add_task was called, i.e. by task_id's numeric suffix, not by timestamp, since two tasks
        may be created with the same timestamp. [] if no task of user's matches."""
```

示例：

```text
tm = TaskManager()
tm.add_task(0, "alice", "write report", 3)      # -> "task-1"
tm.add_task(1, "alice", "review PR", 5)         # -> "task-2"
tm.add_task(2, "alice", "file expenses", 5)     # -> "task-3"     -- same priority as task-2, added later
tm.add_task(3, "alice", "renew badge", 1)       # -> "task-4"
tm.get_task_list(3, "alice")
# -> [Task("task-2", "review PR", 5, 1), Task("task-3", "file expenses", 5, 2),
#     Task("task-1", "write report", 3, 0), Task("task-4", "renew badge", 1, 3)]
#    priority 5 before 3 before 1; within priority 5, task-2 (created first) before task-3
tm.get_task_list(3, "alice", min_priority=3)
# -> [Task("task-2", ...), Task("task-3", ...), Task("task-1", ...)]   -- renew badge (priority 1) filtered out
tm.get_task_list(3, "carol")   # -> []                                 -- carol owns no tasks
```

### Level 3 —— 过期

从这一关开始，`add_task` 接受一个可选的 `ttl`：一个正整数，表示存活的时间长度。带 `ttl` 创建的任务只在
`at_timestamp < created_at + ttl` 期间活跃；到了 `created_at + ttl` 那一刻，以及之后的任何时刻，它就不
再活跃——就好像在那一刻对它调用了 `complete_task` 一样，各方面都是如此：它会从 `get_task` 和
`get_task_list` 中消失，`update_task`/`complete_task` 对它调用也会返回 `False`。没有带 `ttl` 创建的任务
（默认 `ttl=None`）永不过期。`update_task` 永远不会改变一个任务的 `ttl` 或 `created_at`——只会改变它的
标题和优先级——所以一个任务的过期截止时刻，一旦被 `add_task` 设定，就在它的整个生命周期内保持不变。每次
读取都惰性地应用这条规则，在调用时把 `created_at + ttl` 与 `at_timestamp`/`timestamp` 做比较；不会有任何
后台进程主动去清除一个已过期的任务。

```py
class TaskManager:
    def add_task(self, timestamp: int, user: str, title: str, priority: int, ttl: int | None = None) -> str:
        """Same as before, but a positive ttl makes the task active only while
        at_timestamp < timestamp + ttl. ttl=None (the default) means the task never expires."""
```

示例：

```text
tm = TaskManager()
tm.add_task(0, "alice", "flush cache", 2, ttl=5)   # -> "task-1"   active while at_timestamp < 5
tm.add_task(0, "alice", "renew badge", 1)           # -> "task-2"   no ttl -> never expires
tm.get_task_list(4, "alice")   # -> [task-1, task-2]     4 < 5: task-1 still active
tm.get_task_list(5, "alice")   # -> [task-2]             5 == 0 + 5: task-1 has just expired
tm.get_task(5, "alice", "task-1")             # -> None
tm.update_task(5, "alice", "task-1", "x", 9)  # -> False               -- expired, behaves unknown
tm.complete_task(6, "alice", "task-2")        # -> True
tm.get_task_list(6, "alice")   # -> []
```

### Level 4 —— 时间回溯

“`at_timestamp` 始终等于目前为止见过的最大时间戳”这条保证被取消了：`get_task_list` 现在可以用一个比
某些已经发生过的 `add_task`、`update_task` 或 `complete_task` 调用更早的 `at_timestamp` 来调用——这是
一次真正意义上对过去的查询。它必须精确返回一次实时的 `get_task_list` 调用在那一刻本该返回的列表：只根据
`timestamp <= at_timestamp` 的那些调用重建出来、在 `at_timestamp` 时刻活跃的任务（任何时间戳更大的调用
——无论它是新增了一个任务、修改了一个任务，还是完成了一个任务——都必须不可见），并在 `at_timestamp` 这一
时刻本身应用 Level 3 的过期规则，排序和过滤方式与 Level 2 完全相同。`at_timestamp` 可以指向任意时刻，
包括早于 `user` 第一个任务被创建之前的时刻，此时结果为 `[]`。

示例：

```text
tm = TaskManager()
tm.add_task(0, "alice", "write report", 3)                    # -> "task-1"
tm.add_task(2, "alice", "review PR", 5, ttl=10)                # -> "task-2"   active while at_timestamp < 12
tm.update_task(4, "alice", "task-1", "write Q3 report", 4)     # -> True
tm.complete_task(6, "alice", "task-2")                          # -> True
tm.get_task_list(20, "alice")
# -> [Task("task-1", "write Q3 report", 4, 0)]           -- "now": task-2 completed, task-1 already renamed
tm.get_task_list(1, "alice")
# -> [Task("task-1", "write report", 3, 0)]              -- before task-2 existed (created at 2 > 1) and
#                                                             before task-1's rename (at 4 > 1)
tm.get_task_list(3, "alice")
# -> [Task("task-2", "review PR", 5, 2), Task("task-1", "write report", 3, 0)]
#    task-2 exists and outranks by priority; task-1's rename (at 4) has not happened yet at 3
tm.get_task_list(6, "alice")
# -> [Task("task-1", "write Q3 report", 4, 0)]           -- task-2's completion at 6 already applies (inclusive)
```

## 参考解答

<details>
<summary>展开参考解答</summary>

值得事先确认的两点：标题是否必须唯一（这里不必——只有 `task_id` 保证唯一）；以及 `update_task` 是否
应该能修改一个任务的 `ttl`（这里不能——过期时间在创建时一次性确定，`update_task` 只会改动标题和优先级）。

### Level 1

每个任务是一个以 `task_id` 为键、存在 dict 里的可变记录；另一个计数器按调用顺序发放 id。
`_active_record` 把每个修改方法和 `get_task` 共用的三种失败情形集中到一处——未知 id、owner 不对、不
活跃——这样每个方法就只是一次查找加一次判断。`timestamp` 从这里就被接收了，尽管这一关还没有任何地方
读取它，这是为了让后面每一关都保持同样的签名；Level 3 才是第一个真正拿它去做比较的地方。

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Task:
    task_id: str
    title: str
    priority: int
    created_at: int


class TaskManager:
    def __init__(self) -> None:
        self._tasks: dict[str, dict] = {}   # task_id -> mutable record
        self._next_id = 1

    def add_task(self, timestamp: int, user: str, title: str, priority: int) -> str:
        task_id = f"task-{self._next_id}"
        self._next_id += 1
        self._tasks[task_id] = {"user": user, "title": title, "priority": priority,
                                 "created_at": timestamp, "completed": False}
        return task_id

    def _active_record(self, user: str, task_id: str) -> dict | None:
        rec = self._tasks.get(task_id)
        if rec is None or rec["user"] != user or rec["completed"]:
            return None
        return rec

    def update_task(self, timestamp: int, user: str, task_id: str, title: str, priority: int) -> bool:
        rec = self._active_record(user, task_id)
        if rec is None:
            return False
        rec["title"], rec["priority"] = title, priority
        return True

    def complete_task(self, timestamp: int, user: str, task_id: str) -> bool:
        rec = self._active_record(user, task_id)
        if rec is None:
            return False
        rec["completed"] = True
        return True

    def get_task(self, timestamp: int, user: str, task_id: str) -> Task | None:
        rec = self._active_record(user, task_id)
        return None if rec is None else Task(task_id, rec["title"], rec["priority"], rec["created_at"])
```

每个方法都是 $O(1)$。

### Level 2

`get_task_list` 扫描实例里的每一个任务，保留那些属于 `user` 且活跃、又通过优先级过滤的，然后排序。
排序键是 `(-priority, creation_seq)`：对优先级取负，把 Python 的升序排序变成按优先级降序；
`creation_seq`——`task_id` 的数字后缀，也就是 `add_task` 被调用的顺序——用来打破平局，因为两个任务
可能共享同一个 `created_at` 时间戳，但绝不会共享同一个 `task_id`。

```python
def _creation_seq(task_id: str) -> int:
    return int(task_id.rsplit("-", 1)[1])


class TaskManager(TaskManager):
    def get_task_list(self, at_timestamp: int, user: str, min_priority: int | None = None) -> list[Task]:
        matches = [(task_id, rec) for task_id, rec in self._tasks.items()
                   if rec["user"] == user and not rec["completed"]
                   and (min_priority is None or rec["priority"] >= min_priority)]
        matches.sort(key=lambda pair: (-pair[1]["priority"], _creation_seq(pair[0])))
        return [Task(task_id, rec["title"], rec["priority"], rec["created_at"]) for task_id, rec in matches]
```

把 `TaskManager` 重新打开、定义成它自己的子类，而不是直接修改 Level 1 的代码块，这也是本解答后面
逐关扩展这个类的方式。这里的开销是 $O(T + m \log m)$，其中 $T$ 是整个实例中的任务总数，
$m \le T$ 是属于 `user` 且通过过滤的任务数：扫描会碰到每一个任务一次，只有匹配到的那些才会被排序。
按用户建索引可以去掉这个 $T$ 项；Level 4 反正要建这样一个索引，其带来的改进留到那里再说。

### Level 3

Level 3 给记录多加了一个字段 `ttl`（永久任务为 `None`，否则就是传给 `add_task` 的那个正整数），
也给每一处判断任务是否存活的地方多加了一种失败情形：`_active_record` 变成了 `_record`，把“在哪个
时刻判断存活”作为一个显式参数传入，而不是默认为“现在”。由于每个调用方的失败条件都变了，Level 1 和
Level 2 的方法在这里是被重写，而不是原样继承——比如直接继承 `get_task_list` 的话，就只会继续检查
`rec["completed"]`，从而放过一个已经过期的任务。

```python
class TaskManager(TaskManager):
    def add_task(self, timestamp: int, user: str, title: str, priority: int, ttl: int | None = None) -> str:
        task_id = super().add_task(timestamp, user, title, priority)
        self._tasks[task_id]["ttl"] = ttl
        return task_id

    @staticmethod
    def _alive(rec: dict, at_timestamp: int) -> bool:
        if rec["completed"]:
            return False
        ttl = rec.get("ttl")
        return ttl is None or at_timestamp < rec["created_at"] + ttl   # NOTE: "<", not "<=" -- created_at + ttl
                                                                         #  is already expired, not the last alive instant

    def _record(self, user: str, task_id: str, at_timestamp: int) -> dict | None:
        rec = self._tasks.get(task_id)
        if rec is None or rec["user"] != user or not self._alive(rec, at_timestamp):
            return None
        return rec

    def update_task(self, timestamp: int, user: str, task_id: str, title: str, priority: int) -> bool:
        rec = self._record(user, task_id, timestamp)
        if rec is None:
            return False
        rec["title"], rec["priority"] = title, priority
        return True

    def complete_task(self, timestamp: int, user: str, task_id: str) -> bool:
        rec = self._record(user, task_id, timestamp)
        if rec is None:
            return False
        rec["completed"] = True
        return True

    def get_task(self, timestamp: int, user: str, task_id: str) -> Task | None:
        rec = self._record(user, task_id, timestamp)
        return None if rec is None else Task(task_id, rec["title"], rec["priority"], rec["created_at"])

    def get_task_list(self, at_timestamp: int, user: str, min_priority: int | None = None) -> list[Task]:
        matches = [(task_id, rec) for task_id, rec in self._tasks.items()
                   if rec["user"] == user and self._alive(rec, at_timestamp)
                   and (min_priority is None or rec["priority"] >= min_priority)]
        matches.sort(key=lambda pair: (-pair[1]["priority"], _creation_seq(pair[0])))
        return [Task(task_id, rec["title"], rec["priority"], rec["created_at"]) for task_id, rec in matches]
```

这是一种惰性过期（lazy expiry）：一个过期的任务不会被定时器或者下一次写入主动清扫掉，只会在下一次被
读取时被跳过。这样已经够用了，因为这个类里没有任何地方需要在不先读取任务的情况下知道任务总数，所以主动
清扫能换来的只有内存——而这里一条过期记录不过是几个标量字段，不值得为此专门起一个后台线程，或者在每个
修改方法里都塞一次检查来提前回收它。每个方法都保持着 Level 1 和 Level 2 的开销：修改方法是 $O(1)$，
`get_task_list` 是 $O(T + m \log m)$。

### Level 4

Level 1 到 Level 3 里，每个任务始终只保留一条可变记录，每次 `update_task` 或 `complete_task` 调用
都直接在原地覆盖它——这对“现在”这一种查询是理想的，因为永远只需要查一个答案；但对“过去”毫无用处：
一旦后面的调用已经覆盖了某个字段，就没有任何东西能说出它之前是什么样子。Level 4 需要这个能力，所以
存储方式从“每个任务一条当前记录”变成了这个任务的完整版本历史：它曾经拥有过的每一个
`(timestamp, title, priority)`，按调用顺序排列，再加上它被完成时的那一个时间戳（如果有的话；最多只
会有一个，因为一旦任务不再活跃，`complete_task` 就会失败，所以它不可能在同一个任务上成功两次）。两者
都是只追加的。因为整个实例范围内的每一次 `add_task`、`update_task`、`complete_task` 调用，携带的时间戳
都不小于前一次的时间戳，所以单个任务自己的事件——作为这个全局序列的一个子序列——到达时天然就已经按
时间戳排好序了；记录一个事件永远是往列表末尾追加，从来不需要插入到中间。

```python
from bisect import bisect_right


@dataclass
class _History:
    user: str
    ttl: int | None
    created_at: int
    timestamps: list[int]     # version timestamps, non-decreasing, bisected against at_timestamp
    titles: list[str]         # titles[i] / priorities[i] held from timestamps[i] up to the next entry
    priorities: list[int]
    completed_at: int | None = None


class TaskManager(TaskManager):
    def __init__(self) -> None:
        self._history: dict[str, _History] = {}
        self._by_user: dict[str, list[str]] = {}
        self._next_id = 1

    def add_task(self, timestamp: int, user: str, title: str, priority: int, ttl: int | None = None) -> str:
        task_id = f"task-{self._next_id}"
        self._next_id += 1
        self._history[task_id] = _History(user, ttl, timestamp, [timestamp], [title], [priority])
        self._by_user.setdefault(user, []).append(task_id)
        return task_id

    def _version_at(self, hist: _History, at_timestamp: int) -> tuple[str, int] | None:
        i = bisect_right(hist.timestamps, at_timestamp) - 1   # NOTE: bisect_right, not bisect_left -- a version
        if i < 0:                                               #  recorded at exactly at_timestamp must still count
            return None
        return hist.titles[i], hist.priorities[i]

    def _alive_at(self, hist: _History, at_timestamp: int) -> bool:
        if self._version_at(hist, at_timestamp) is None:
            return False
        if hist.completed_at is not None and hist.completed_at <= at_timestamp:
            return False
        return hist.ttl is None or at_timestamp < hist.created_at + hist.ttl

    def _record(self, user: str, task_id: str, at_timestamp: int) -> _History | None:
        hist = self._history.get(task_id)
        if hist is None or hist.user != user or not self._alive_at(hist, at_timestamp):
            return None
        return hist

    def update_task(self, timestamp: int, user: str, task_id: str, title: str, priority: int) -> bool:
        hist = self._record(user, task_id, timestamp)
        if hist is None:
            return False
        hist.timestamps.append(timestamp)
        hist.titles.append(title)
        hist.priorities.append(priority)
        return True

    def complete_task(self, timestamp: int, user: str, task_id: str) -> bool:
        hist = self._record(user, task_id, timestamp)
        if hist is None:
            return False
        hist.completed_at = timestamp
        return True

    def get_task(self, timestamp: int, user: str, task_id: str) -> Task | None:
        hist = self._record(user, task_id, timestamp)
        if hist is None:
            return None
        title, priority = self._version_at(hist, timestamp)
        return Task(task_id, title, priority, hist.created_at)

    def get_task_list(self, at_timestamp: int, user: str, min_priority: int | None = None) -> list[Task]:
        out = []
        for task_id in self._by_user.get(user, []):
            hist = self._history[task_id]
            if not self._alive_at(hist, at_timestamp):   # NOTE: checked at at_timestamp, not "now" -- a later
                continue                                   #  update or completion must never leak into this snapshot
            title, priority = self._version_at(hist, at_timestamp)
            if min_priority is not None and priority < min_priority:
                continue
            out.append((priority, _creation_seq(task_id), task_id, title))
        out.sort(key=lambda x: (-x[0], x[1]))
        return [Task(task_id, title, priority, self._history[task_id].created_at)
                for priority, _, task_id, title in out]
```

`_version_at` 就是二分查找的那一步：`bisect_right` 找到的是紧跟在每一个等于 `at_timestamp` 的条目
之后的插入点，所以减去 `1` 就落在最后一个满足 `timestamp <= at_timestamp` 的条目上——也就是从
`at_timestamp` 这个时刻能看到的最新一次调用，哪怕这次调用恰好发生在 `at_timestamp` 那一刻本身也一样。
`_alive_at` 接着组合了三个互相独立的检查——必须已经存在某个版本了（只有当 `at_timestamp` 早于这个任务
的 `created_at` 时，`bisect` 才会一无所获）、不能带有一个发生在 `at_timestamp` 或更早的完成事件、也
不能在 `at_timestamp` 之前就已经过期——这三条里没有一条会去看任何一个记录在 `at_timestamp` 之后的
版本，这正是“对过去的查询看不到一次尚未发生的编辑”的原因所在。`add_task` 现在也会同步填充 `_by_user`
——一个按用户维护、按创建顺序排列的任务 id 列表，这正是去掉 Level 2 里 $T$ 项的关键：`get_task_list`
只会访问 `user` 曾经拥有过的任务。

`update_task` 和 `complete_task` 仍然是 $O(1)$：都只是往某个任务自己的列表末尾追加，完全不需要
查找，因为这个任务最新的事件永远就是最近一次追加的那个。`get_task` 是 $O(\log v)$（$`v`$ 是 `task_id`
积累的版本数）——一次 `bisect`。`get_task_list` 是 $O(k (\log v + \log k))$，其中 $k$ 是 `user`
曾经创建过的任务总数（不论是否还活跃——每一个都得检查一遍），$`v`$ 是其中任意一个任务的版本数上界：
每个任务一次 `bisect`，再加上对存活下来的那些排序。

另一种做法——维护一份全局的、只追加的日志，记录每一次 `add_task`/`update_task`/`complete_task`
调用，回答一次查询时，按调用顺序重放 `timestamp <= at_timestamp` 的那些日志事件，从零开始重建出一份
当前记录的 dict，正好就是 Level 1-3 在只看到这段调用前缀时会呈现的样子——这样做同样正确，而且完全不需要
按任务二分查找。但它的代价是每次查询 $O(C)$，其中 $C$ 是整个实例到目前为止发生过的*全部*修改调用数，
横跨所有用户：一个有很多活跃用户的系统，每一次针对单个用户的查询都要为所有其他用户的调用买单。版本历史
这种做法只会花费与被查询的那一个用户成正比的工作量，代价是每个版本都要额外花 $O(1)$ 的空间无限期地保留
下去；日志重放为这份日志花费同样是 $O(C)$ 的空间，却不需要任何按任务建的索引，也是两者中更容易一次写对
的那个，这也是下面的验证代码用它作为独立验证快速版本的原因。

### 追问

- 一个回答“某个任务的完整编辑历史”而不只是“它在某一时刻的状态”的方法，是在 `timestamps`/`titles`/
  `priorities` 上做两次 `bisect` 之间的切片，而不是一次重新扫描。
- `get_task` 和几个修改方法其实完全不要求 `at_timestamp`/`timestamp` 一定是见过的最新时刻——
  `_record` 和 `_version_at` 在任意时刻都能正常工作。只有 `get_task_list` 的题面对此作出了承诺，但
  同一套机制其实也顺带正确地支持了对 `get_task` 的历史查询。
- 版本历史会无限增长：一个任务如果每天都被更新一次，一年下来就会永远带着 365 条记录。如果实际上只需要
  回溯最近 90 天，那么对于已经有更新版本的任务，早于这个窗口的旧版本就可以在确定不会再有查询指向它们之后
  丢弃——但这个假设是一个支持*任意*过去时刻查询的系统（也就是这里要实现的系统）无法做出的。
- 一种显式命名快照的设计——一个 `backup(timestamp)` 调用把当前状态冻结下来，一个 `restore` 跳回到
  最近的一次快照——用“只能查询某个人显式保存过的时刻”换来了这一关“随时可以查询任意时刻”的能力，
  代价则是把无限增长的历史换成了每次写入只需 $O(1)$ 的额外记录。
- 要让这个类线程安全，需要给每个任务在 `update_task`/`complete_task` 里“读取 - 判断 - 追加”的整个
  序列加锁，而不能只锁住列表追加这一步本身：否则两个线程同时争着更新同一个任务，可能会在任何一个线程
  追加之前，都读到它还是活跃的。

<details>
<summary>验证代码（可运行）</summary>

```python
# ---- Level 1 example ----
tm = TaskManager()
assert tm.add_task(10, "alice", "write report", 3) == "task-1"
assert tm.add_task(12, "alice", "review PR", 5) == "task-2"
assert tm.add_task(14, "bob", "write report", 1) == "task-3"
assert tm.get_task(15, "alice", "task-1") == Task("task-1", "write report", 3, 10)
assert tm.get_task(15, "alice", "task-3") is None
assert tm.update_task(16, "alice", "task-1", "write Q3 report", 4) is True
assert tm.get_task(17, "alice", "task-1") == Task("task-1", "write Q3 report", 4, 10)
assert tm.complete_task(18, "alice", "task-1") is True
assert tm.complete_task(19, "alice", "task-1") is False
assert tm.get_task(20, "alice", "task-1") is None
assert tm.update_task(21, "alice", "task-1", "x", 1) is False

# ---- Level 2 example ----
tm = TaskManager()
tm.add_task(0, "alice", "write report", 3)
tm.add_task(1, "alice", "review PR", 5)
tm.add_task(2, "alice", "file expenses", 5)
tm.add_task(3, "alice", "renew badge", 1)
assert tm.get_task_list(3, "alice") == [
    Task("task-2", "review PR", 5, 1), Task("task-3", "file expenses", 5, 2),
    Task("task-1", "write report", 3, 0), Task("task-4", "renew badge", 1, 3),
]
assert tm.get_task_list(3, "alice", min_priority=3) == [
    Task("task-2", "review PR", 5, 1), Task("task-3", "file expenses", 5, 2), Task("task-1", "write report", 3, 0),
]
assert tm.get_task_list(3, "carol") == []

# ---- Level 3 example ----
tm = TaskManager()
assert tm.add_task(0, "alice", "flush cache", 2, ttl=5) == "task-1"
assert tm.add_task(0, "alice", "renew badge", 1) == "task-2"
assert tm.get_task_list(4, "alice") == [Task("task-1", "flush cache", 2, 0), Task("task-2", "renew badge", 1, 0)]
assert tm.get_task_list(5, "alice") == [Task("task-2", "renew badge", 1, 0)]
assert tm.get_task(5, "alice", "task-1") is None
assert tm.update_task(5, "alice", "task-1", "x", 9) is False
assert tm.complete_task(6, "alice", "task-2") is True
assert tm.get_task_list(6, "alice") == []

# ---- Level 4 example ----
tm = TaskManager()
tm.add_task(0, "alice", "write report", 3)
tm.add_task(2, "alice", "review PR", 5, ttl=10)
tm.update_task(4, "alice", "task-1", "write Q3 report", 4)
tm.complete_task(6, "alice", "task-2")
assert tm.get_task_list(20, "alice") == [Task("task-1", "write Q3 report", 4, 0)]
assert tm.get_task_list(1, "alice") == [Task("task-1", "write report", 3, 0)]
assert tm.get_task_list(3, "alice") == [Task("task-2", "review PR", 5, 2), Task("task-1", "write report", 3, 0)]
assert tm.get_task_list(6, "alice") == [Task("task-1", "write Q3 report", 4, 0)]
print("all four levels' worked examples replayed exactly")

# ---- edge cases the examples do not reach ----
tm = TaskManager()
assert tm.update_task(0, "alice", "task-999", "x", 1) is False   # unknown id
assert tm.complete_task(0, "alice", "task-999") is False
assert tm.get_task(0, "alice", "task-999") is None
assert tm.get_task_list(0, "ghost-user") == []

t1 = tm.add_task(0, "alice", "shared", 1)
assert tm.update_task(1, "bob", t1, "y", 2) is False              # wrong owner
assert tm.complete_task(1, "bob", t1) is False
assert tm.get_task(1, "bob", t1) is None

assert tm.get_task_list(0, "alice", min_priority=99) == []        # filter excludes every task

# two tasks, same user, same timestamp, same priority: creation (call) order breaks the tie
tm2 = TaskManager()
first = tm2.add_task(9, "alice", "first", 3)
second = tm2.add_task(9, "alice", "second", 3)
assert [t.task_id for t in tm2.get_task_list(9, "alice")] == [first, second]

# an update and a completion racing at the exact same timestamp: the later call wins
tm3 = TaskManager()
t3 = tm3.add_task(0, "alice", "x", 1)
assert tm3.update_task(5, "alice", t3, "x2", 2) is True
assert tm3.complete_task(5, "alice", t3) is True     # same instant, called after the update
assert tm3.get_task(5, "alice", t3) is None           # already inactive at exactly 5
assert tm3.get_task(4, "alice", t3).title == "x"       # a moment earlier: still the original version

# at_timestamp before the user's first task ever existed
tm4 = TaskManager()
tm4.add_task(10, "alice", "x", 1)
assert tm4.get_task_list(9, "alice") == []
print("edge cases OK")
```

```python
import random


class NaiveTaskManager:
    """Independent reference model, straight from the statement: one global, append-only log of
    every add_task/update_task/complete_task call, in call order. Every query rebuilds the state
    from scratch by replaying the events with timestamp <= the query's own timestamp -- no per-task
    history, no bisect."""

    def __init__(self) -> None:
        self._log: list[tuple] = []
        self._next_id = 1

    @staticmethod
    def _alive(rec: dict, at_timestamp: int) -> bool:
        if rec["completed"]:
            return False
        return rec["ttl"] is None or at_timestamp < rec["created_at"] + rec["ttl"]

    def _replay(self, at_timestamp: int) -> dict[str, dict]:
        state: dict[str, dict] = {}
        for kind, ts, *payload in self._log:
            if ts > at_timestamp:
                continue
            if kind == "add":
                task_id, user, title, priority, ttl = payload
                state[task_id] = {"user": user, "title": title, "priority": priority,
                                   "created_at": ts, "ttl": ttl, "completed": False}
            elif kind == "update":
                task_id, user, title, priority = payload
                rec = state.get(task_id)
                if rec is not None and rec["user"] == user and self._alive(rec, ts):
                    rec["title"], rec["priority"] = title, priority
            else:
                task_id, user = payload
                rec = state.get(task_id)
                if rec is not None and rec["user"] == user and self._alive(rec, ts):
                    rec["completed"] = True
        return state

    def add_task(self, timestamp, user, title, priority, ttl=None) -> str:
        task_id = f"task-{self._next_id}"
        self._next_id += 1
        self._log.append(("add", timestamp, task_id, user, title, priority, ttl))
        return task_id

    def update_task(self, timestamp, user, task_id, title, priority) -> bool:
        rec = self._replay(timestamp).get(task_id)
        ok = rec is not None and rec["user"] == user and self._alive(rec, timestamp)
        if ok:
            self._log.append(("update", timestamp, task_id, user, title, priority))
        return ok

    def complete_task(self, timestamp, user, task_id) -> bool:
        rec = self._replay(timestamp).get(task_id)
        ok = rec is not None and rec["user"] == user and self._alive(rec, timestamp)
        if ok:
            self._log.append(("complete", timestamp, task_id, user))
        return ok

    def get_task(self, timestamp, user, task_id) -> Task | None:
        rec = self._replay(timestamp).get(task_id)
        if rec is None or rec["user"] != user or not self._alive(rec, timestamp):
            return None
        return Task(task_id, rec["title"], rec["priority"], rec["created_at"])

    def get_task_list(self, at_timestamp, user, min_priority=None) -> list[Task]:
        state = self._replay(at_timestamp)
        items = [(rec["priority"], int(tid.rsplit("-", 1)[1]), tid, rec) for tid, rec in state.items()
                 if rec["user"] == user and self._alive(rec, at_timestamp)
                 and (min_priority is None or rec["priority"] >= min_priority)]
        items.sort(key=lambda x: (-x[0], x[1]))
        return [Task(tid, rec["title"], rec["priority"], rec["created_at"]) for _, _, tid, rec in items]


def _run_trial(seed: int, n_ops: int = 150) -> dict[str, int]:
    rng = random.Random(seed)
    fast, naive = TaskManager(), NaiveTaskManager()
    users = ["alice", "bob", "carol"]
    titles = ["write report", "review PR", "file expenses", "renew badge"]
    clock = 0
    task_ids: list[str] = []
    counts = {"add": 0, "add_ttl": 0, "update_hit": 0, "complete_hit": 0,
              "get_past": 0, "list_past": 0, "list_filtered": 0}

    for _ in range(n_ops):
        clock += rng.randint(0, 2)
        user = rng.choice(users)
        pick = rng.random()
        if pick < 0.3 or not task_ids:
            ttl = rng.choice([None, None, rng.randint(1, 8)])
            counts["add"] += 1
            counts["add_ttl"] += ttl is not None
            title, priority = rng.choice(titles), rng.randint(1, 5)
            got = fast.add_task(clock, user, title, priority, ttl)
            want = naive.add_task(clock, user, title, priority, ttl)
            assert got == want, (seed, "add_task", got, want)
            task_ids.append(got)
        elif pick < 0.5:
            task_id, title, priority = rng.choice(task_ids), rng.choice(titles), rng.randint(1, 5)
            got = fast.update_task(clock, user, task_id, title, priority)
            want = naive.update_task(clock, user, task_id, title, priority)
            assert got == want, (seed, "update_task", clock, user, task_id, got, want)
            counts["update_hit"] += got
        elif pick < 0.65:
            task_id = rng.choice(task_ids)
            got = fast.complete_task(clock, user, task_id)
            want = naive.complete_task(clock, user, task_id)
            assert got == want, (seed, "complete_task", clock, user, task_id, got, want)
            counts["complete_hit"] += got
        elif pick < 0.8:
            task_id = rng.choice(task_ids)
            ts = clock if rng.random() < 0.5 else rng.randint(0, clock)
            counts["get_past"] += ts < clock
            got = fast.get_task(ts, user, task_id)
            want = naive.get_task(ts, user, task_id)
            assert got == want, (seed, "get_task", ts, user, task_id, got, want)
        else:
            at_ts = clock if rng.random() < 0.4 else rng.randint(0, clock)
            counts["list_past"] += at_ts < clock
            min_p = rng.randint(1, 5) if rng.random() < 0.4 else None
            counts["list_filtered"] += min_p is not None
            got = fast.get_task_list(at_ts, user, min_p)
            want = naive.get_task_list(at_ts, user, min_p)
            assert got == want, (seed, "get_task_list", at_ts, user, min_p, got, want)
    return counts


totals: dict[str, int] = {}
for seed in range(400):
    for key, value in _run_trial(seed).items():
        totals[key] = totals.get(key, 0) + value
assert min(totals.values()) > 80, totals   # every kind of call fired repeatedly, including ttl tasks and
                                            # past-timestamp queries
print(f"cross-validated 400 random sequences against an independent reference model: {totals}")
print("all checks passed")
```

</details>

</details>
