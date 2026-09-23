# Online Assessment: Task Manager with Expiry and Snapshots

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Online assessment · 4 levels | ★★☆☆☆ | Medium | SWE · MLE | oop-design, hash-table, ttl, time-travel, bisect | 4 levels / 90 min | Online assessment |
<!-- meta:end -->

## Problem

Implement the four levels below in order: a level's tests must pass before the next level's tests
run, and every level extends the same class rather than replacing it. A *task* belongs to exactly
one *user* (a string user id) and has a `task_id`, a `title`, and an integer `priority` (a larger
priority means a more urgent task); the same title may repeat, within one user's tasks or across
different users, but every `task_id` is unique across the whole instance. Every method that changes
state (`add_task`, `update_task`, `complete_task`) takes an integer *timestamp* as its first
argument, and, across the whole sequence of such calls made to one instance, timestamps never
decrease; two different calls, for the same task or for different tasks, may share a timestamp.
Every read returns a task as a `Task` object, a plain immutable record:

```py
from dataclasses import dataclass


@dataclass(frozen=True)
class Task:
    task_id: str
    title: str
    priority: int
    created_at: int
```

### Level 1 — Core operations

A task is *active* from the moment it is created until, at the latest, the moment it is completed;
`get_task`, `update_task`, and `complete_task` all treat a task that is not active exactly as if its
`task_id` did not exist.

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

Example:

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

### Level 2 — Listing and filters

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

Example:

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

### Level 3 — Expiry

From this level on, `add_task` accepts an optional `ttl`: a positive integer number of time units.
A task created with a `ttl` is active only while `at_timestamp < created_at + ttl`; at
`created_at + ttl` itself, and at every later timestamp, it is no longer active — exactly as if
`complete_task` had been called on it at that instant, for every purpose: it disappears from
`get_task` and `get_task_list`, and `update_task`/`complete_task` on it return `False`. A task
created without a `ttl` (`ttl=None`, the default) never expires. `update_task` never changes a
task's `ttl` or `created_at` — only its title and priority — so a task's expiry deadline, once
`add_task` sets it, is fixed for the task's whole life. Every read applies this rule lazily, by
comparing `created_at + ttl` against `at_timestamp`/`timestamp` at call time; no background process
ever removes an expired task.

```py
class TaskManager:
    def add_task(self, timestamp: int, user: str, title: str, priority: int, ttl: int | None = None) -> str:
        """Same as before, but a positive ttl makes the task active only while
        at_timestamp < timestamp + ttl. ttl=None (the default) means the task never expires."""
```

Example:

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

### Level 4 — Time travel

The guarantee that `at_timestamp` is always the latest timestamp seen so far is lifted:
`get_task_list` may now be called with an `at_timestamp` earlier than one or more `add_task`,
`update_task`, or `complete_task` calls already made — a genuine query into the past. It must
return exactly the list a live `get_task_list` call would have returned at that instant: the tasks
active at `at_timestamp`, reconstructed from only the calls with `timestamp <= at_timestamp` (every
later call — whether it added a task, changed one, or completed one — must be invisible), with
Level 3's expiry rule applied at `at_timestamp` itself, sorted and filtered exactly as in Level 2.
`at_timestamp` may name any moment, including one before `user`'s first task was ever added, in
which case the result is `[]`.

Example:

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

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two points worth confirming before coding: whether a title must be unique (here it need not be —
only `task_id` is guaranteed unique); and whether `update_task` should be able to change a task's
`ttl` (here it cannot — expiry is fixed once at creation, and `update_task` only ever touches title
and priority).

### Level 1

Every task is one mutable record in a dict keyed by `task_id`; a second counter hands out ids in
call order. `_active_record` centralizes the three failure conditions every mutator and `get_task`
share — unknown id, wrong owner, not active — so each of them is one lookup plus one check.
`timestamp` is accepted here, even though nothing in this level reads it yet, so that every level
from here on keeps the same signature; Level 3 is the first to compare it against anything.

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

Every method is $O(1)$.

### Level 2

`get_task_list` scans every task in the instance, keeps the ones owned by `user` that are active and
pass the priority filter, then sorts. The sort key is `(-priority, creation_seq)`: negating priority
turns Python's ascending sort into descending order on priority, and `creation_seq` — the numeric
suffix of `task_id`, i.e. the order `add_task` was called — breaks ties, because two tasks can carry
the same `created_at` timestamp but never the same `task_id`.

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

Reopening `TaskManager` as its own subclass, rather than editing Level 1's block, is how the rest of
this solution extends it level by level. This costs $O(T + m \log m)$, where $T$ is the number of
tasks in the whole instance and $m \le T$ is the number owned by `user` that pass the filter: the
scan touches every task once, and only the matches are sorted. A per-user index would drop the $T$
term; Level 4 builds one anyway; see there for the resulting improvement.

### Level 3

Level 3 adds one field to the record, `ttl` (`None` for a permanent task, otherwise the positive
integer passed to `add_task`), and one more failure condition to check everywhere a task's liveness
matters: `_active_record` becomes `_record`, taking the instant to check liveness at as an explicit
argument instead of assuming "now". Because every caller's failure condition changes, Level 1's and
Level 2's methods are overridden here, not inherited unchanged — inheriting `get_task_list`, for
instance, would keep checking only `rec["completed"]` and let an expired task through.

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

This is lazy expiry: an expired task is never swept out on a timer or on the next write, only
skipped the next time something reads it. It is enough here because nothing in this class needs to
know how many tasks exist without reading them first, so the only thing an eager sweep would buy is
memory, and an expired record here is a handful of scalar fields, not something worth a background
thread or a check squeezed into every mutator just to reclaim early. Every method keeps Level 1's and
Level 2's complexity: $O(1)$ for the mutators, $O(T + m \log m)$ for `get_task_list`.

### Level 4

Levels 1 through 3 keep exactly one mutable record per task, overwritten in place by every
`update_task` or `complete_task` call — ideal for a "now" query, since there is never more than one
answer to look up, but useless for a past one: once a later call has overwritten a field, nothing is
left to say what it held before. Level 4 needs that, so the storage changes from one current record
per task to that task's whole version history: every `(timestamp, title, priority)` it has ever had,
in call order, plus the one timestamp it was completed at, if any (at most one, since `complete_task`
already fails once a task is not active, so it can never succeed twice on the same task). Both are
append-only. Every `add_task`, `update_task`, and `complete_task` call, across the whole instance,
carries a timestamp no smaller than the one before it, so one task's own events — a subsequence of
that global sequence — arrive already sorted by timestamp; recording one is always an append to the
end of a list, never an insertion into its middle.

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

`_version_at` is the bisect step: `bisect_right` finds the insertion point just past every entry
equal to `at_timestamp`, so subtracting `1` lands on the last entry with `timestamp <= at_timestamp`
— the most recent call visible from `at_timestamp`, including one made at exactly `at_timestamp`
itself. `_alive_at` then combines three independent checks — a version must exist yet (`bisect`
returns nothing only if `at_timestamp` is before the task's `created_at`), it must not carry a
completion at or before `at_timestamp`, and it must not have expired by `at_timestamp` — and none of
the three ever inspects a version recorded after `at_timestamp`, which is exactly why a query into
the past cannot see an edit that has not "happened yet" from that moment's point of view. `add_task`
now also fills `_by_user`, a per-user list of task ids in creation order, which is what removes
Level 2's $T$ term: `get_task_list` visits only tasks `user` has ever owned.

`update_task` and `complete_task` stay $O(1)$: each appends to the end of one task's own list, never
searching it, since that task's latest event is always the one most recently appended. `get_task` is
$O(\log v)$ in the number of versions `task_id` has accumulated — one `bisect`. `get_task_list` is
$O(k (\log v + \log k))$, where $k$ is the number of tasks `user` has ever created (active or not —
every one of them has to be checked) and $v$ bounds any one task's version count: one `bisect` per
task, plus sorting the survivors.

The alternative — keep one global, append-only log of every `add_task`/`update_task`/`complete_task`
call ever made, and answer a query by replaying the log events with `timestamp <= at_timestamp` from
scratch, in call order, rebuilding a fresh dict of current records exactly the way Levels 1-3 would
have looked with only that prefix of calls — is also correct, and needs no per-task bisect at all.
But it costs $O(C)$ per query, where $C$ is the *total* number of mutating calls made to the whole
instance so far, across every user: a system with many active users pays for all of them on every
single-user query. The version-history approach only ever walks work proportional to the one user
being asked about, at the cost of $O(1)$ extra space per version to keep it around indefinitely; log
replay uses the same $O(C)$ space for the log but no per-task index, and is the simpler of the two to
get right first, which is why it is also what the checks below use as an independent way to verify
the fast version.

### Follow-ups

- A method that answers one task's whole edit history, not just its state at one instant, is a slice
  of `timestamps`/`titles`/`priorities` between two `bisect` calls, not a fresh scan.
- `get_task` and the mutators never actually require `at_timestamp`/`timestamp` to be the latest
  moment seen — `_record` and `_version_at` work at any instant. Only `get_task_list`'s statement
  promises this, but the same machinery answers a historical `get_task` correctly too, for free.
- Version history grows without bound: a task updated every day for a year carries 365 entries
  forever. If only, say, the last 90 days of look-back is ever needed, versions older than that
  (for a task with at least one newer one) can be dropped once no future query can name a timestamp
  before them — an assumption that a system answering *arbitrary* past queries, as this one does,
  cannot make.
- An explicit, named-snapshot design — a `backup(timestamp)` call that freezes the current state,
  and a `restore` that jumps back to the nearest one made — trades this level's "ask about any
  instant, always" for "ask only about instants someone explicitly saved", in exchange for $O(1)$
  extra bookkeeping per write instead of an ever-growing history.
- Making this thread-safe needs a lock around each task's whole read-check-append sequence in
  `update_task`/`complete_task`, not just around the list appends themselves: two threads racing to
  update the same task could otherwise both read it as active before either one appends.

<details>
<summary>Checks (runnable)</summary>

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
