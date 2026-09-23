# 借助 AI 编程助手评审并改进 PR

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 代码评审 · AI 辅助，Python | ★★★☆☆ | 中等 | SWE · Infra Eng | code-review, pull-requests, race-condition, deserialization, testing | 4 个 PR / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

你正在一个带终端的浏览器内置编辑器中工作，打开着一个小型 Python 代码仓库。另有一个类似代码托管平台的页面
列出了针对该仓库的四个待合并 Pull Request（PR）；下面把每一个都以统一 *diff*（一种用 `diff` 格式描述
改动增删了哪些行的文本）的形式给出，并附上其作者写的描述。终端里可以使用一个 AI 编程助手，整个任务的任何
部分都可以借助它，但你提交的每一条评审意见、每一行代码都由你自己负责：如果说不清楚某条评论或某处改动为
什么是对的，就等同于没有做出这个评论或改动。

仓库只有一个模块 `jobqueue.py`：

```python
import heapq
import itertools
import json
import threading


class Job:
    def __init__(self, id, payload, priority=0, status="pending", attempts=0, max_attempts=3):
        self.id = id
        self.payload = payload
        self.priority = priority
        self.status = status                # "pending" | "running" | "done" | "failed"
        self.attempts = attempts
        self.max_attempts = max_attempts

    def to_dict(self):
        return dict(id=self.id, payload=self.payload, priority=self.priority,
                    status=self.status, attempts=self.attempts, max_attempts=self.max_attempts)


class JobQueue:
    """A priority job queue, safe to share between several worker threads in one process. Ready jobs
    are handed out by priority (higher first); jobs of equal priority are handed out in submission order."""

    def __init__(self):
        self._lock = threading.Lock()
        self._jobs = {}                     # id -> Job, every job ever submitted
        self._heap = []                     # (-priority, seq, id), pending jobs only
        self._counter = itertools.count()

    def submit(self, job_id, payload, priority=0, max_attempts=3):
        with self._lock:
            if job_id in self._jobs:
                raise ValueError(f"duplicate job id {job_id!r}")
            job = Job(job_id, payload, priority, "pending", 0, max_attempts)
            self._jobs[job_id] = job
            heapq.heappush(self._heap, (-priority, next(self._counter), job_id))
        return job

    def pop_ready(self):
        """Return the highest-priority pending job, marking it "running" so no other caller can
        receive the same job. Returns None if no job is pending."""
        with self._lock:
            while self._heap:
                _, _, job_id = heapq.heappop(self._heap)
                job = self._jobs[job_id]
                if job.status == "pending":
                    job.status = "running"
                    return job
            return None

    def complete(self, job_id):
        with self._lock:
            self._jobs[job_id].status = "done"

    def fail(self, job_id):
        """Record a failed attempt at a running job. Reschedules it as pending if attempts remain
        (it becomes poppable again immediately), else marks it permanently "failed"."""
        with self._lock:
            job = self._jobs[job_id]
            job.attempts += 1
            if job.attempts < job.max_attempts:
                job.status = "pending"
                heapq.heappush(self._heap, (-job.priority, next(self._counter), job_id))
            else:
                job.status = "failed"

    def get(self, job_id):
        return self._jobs[job_id]

    def counts(self):
        """Number of jobs currently in each status, for monitoring."""
        with self._lock:
            out = {"pending": 0, "running": 0, "done": 0, "failed": 0}
            for job in self._jobs.values():
                out[job.status] += 1
            return out

    def save(self, path):
        with self._lock:
            rows = [job.to_dict() for job in self._jobs.values()]
        with open(path, "w") as f:
            json.dump(rows, f)

    @classmethod
    def load(cls, path):
        queue = cls()
        with open(path) as f:
            rows = json.load(f)
        for row in rows:
            job = Job(**row)
            queue._jobs[job.id] = job
            if job.status == "pending":
                heapq.heappush(queue._heap, (-job.priority, next(queue._counter), job.id))
        return queue
```

仓库里还有 `test_jobqueue.py`，用 `unittest` 写了针对 `submit`（拒绝重复 id）、`pop_ready`（先按优先级、
优先级相同时再按提交顺序）、`fail`（重试到 `max_attempts` 次数用尽后把任务标记为 `"failed"`），以及
`save`/`load`（经过一次文件写入再读回，每个任务的字段都保持不变，且重新加载后只有仍处于 `"pending"` 的
任务才能再被取出）的测试。仓库 `README` 里写明的测试命令是：`python -m unittest -v`。

**PR #1 —— 新增用于批量提交任务的 `submit_many`**（由 `grace-oyelaran` 提出）

> 我们在启动时会把前一天快照里积压的几百个任务重新灌入队列，现在这意味着每个调用方都要自己写一遍循环去
> 调用 `submit`。这个 PR 新增了 `submit_many`：接收调用方本来就会构造的那些 `(job_id, payload, priority)`
> 三元组，一次调用全部提交，省去了那个循环，而且整批只在批次级别取一次锁，不再是每个任务取一次。`submit`
> 和其它方法都没有改动。

```diff
diff --git a/jobqueue.py b/jobqueue.py
--- a/jobqueue.py
+++ b/jobqueue.py
@@ class JobQueue:
             heapq.heappush(self._heap, (-priority, next(self._counter), job_id))
         return job

+    def submit_many(self, items):
+        """items: an iterable of (job_id, payload, priority) triples. Submits each in order and
+        returns the list of created Job objects."""
+        return [self.submit(job_id, payload, priority) for job_id, payload, priority in items]
+
     def pop_ready(self):
         """Return the highest-priority pending job, marking it "running" so no other caller can
diff --git a/test_jobqueue.py b/test_jobqueue.py
--- a/test_jobqueue.py
+++ b/test_jobqueue.py
@@ class TestJobQueue(unittest.TestCase):
     def test_submit_rejects_duplicate_id(self):
         q = JobQueue()
         q.submit("a", {})
         with self.assertRaises(ValueError):
             q.submit("a", {})
+
+    def test_submit_many_creates_all_jobs_in_order(self):
+        q = JobQueue()
+        jobs = q.submit_many([("a", {}, 0), ("b", {}, 5), ("c", {}, 0)])
+        self.assertEqual([j.id for j in jobs], ["a", "b", "c"])
+        self.assertEqual(q.pop_ready().id, "b")
```

**PR #2 —— 降低 `submit()` 的锁竞争**（由 `marcus-kaelin` 提出）

> 在压测下，`submit` 在性能分析里是竞争最激烈的锁：每次调用都要为整个方法持有 `self._lock`，但只要 id
> 最终是空闲的，重复 id 检查其实根本不碰共享可变状态。这个改动把检查挪到取锁之前，这样最常见的情形——id
> 是新的——持锁期间要做的事就更少。行为不变：重复 id 仍然会抛出 `ValueError`。

```diff
diff --git a/jobqueue.py b/jobqueue.py
--- a/jobqueue.py
+++ b/jobqueue.py
@@ class JobQueue:
     def submit(self, job_id, payload, priority=0, max_attempts=3):
-        with self._lock:
-            if job_id in self._jobs:
-                raise ValueError(f"duplicate job id {job_id!r}")
+        if job_id in self._jobs:
+            raise ValueError(f"duplicate job id {job_id!r}")
+        with self._lock:
             job = Job(job_id, payload, priority, "pending", 0, max_attempts)
             self._jobs[job_id] = job
             heapq.heappush(self._heap, (-priority, next(self._counter), job_id))
         return job
```

**PR #3 —— 用 pickle 代替 JSON 持久化队列状态**（由 `owen-t` 提出）

> 有几个内部调用方想在任务的 `payload` 里存更丰富的对象——namedtuple、dataclass，甚至有个场景要存一个
> 小的 numpy 数组——JSON 没法表示这些，所以 `save` 现在对它们其实是悄悄用不了的。Pickle 能处理任何可
> pickle 的对象，我们这边不用做任何 schema 方面的工作，而且是无缝替换：文件不变，调用位置不变，来回读写
> 的流程也不变。把 `save` 和 `load` 都换了过去；文件里其它地方都没有改动。

```diff
diff --git a/jobqueue.py b/jobqueue.py
--- a/jobqueue.py
+++ b/jobqueue.py
@@
 import heapq
 import itertools
-import json
+import pickle
 import threading
@@ class JobQueue:
     def save(self, path):
         with self._lock:
             rows = [job.to_dict() for job in self._jobs.values()]
-        with open(path, "w") as f:
-            json.dump(rows, f)
+        with open(path, "wb") as f:
+            pickle.dump(rows, f)

     @classmethod
     def load(cls, path):
         queue = cls()
-        with open(path) as f:
-            rows = json.load(f)
+        with open(path, "rb") as f:
+            rows = pickle.load(f)
         for row in rows:
             job = Job(**row)
             queue._jobs[job.id] = job
```

**PR #4 —— 可插拔的调度策略**（由 `felix-wu` 提出）

> 优先级加 FIFO 不会一直是我们唯一需要的调度规则——批量导入团队的值班轮换那边已经在要求按租户做轮询
> （round-robin），这样一个吵闹的租户才不会把别的租户饿死。与其以后再把第二条规则硬编码进 `pop_ready`，
> 这个 PR 引入了 `SchedulingPolicy` 接口和一个小型注册表，并把现有行为搬进了 `PriorityPolicy`，因此默认
> 行为不变。另外还给 `Job` 加了一个通用的 `metadata` 字段，因为按租户调度的策略会需要地方存放它自己每个
> 任务的记账信息。

```diff
diff --git a/jobqueue.py b/jobqueue.py
--- a/jobqueue.py
+++ b/jobqueue.py
@@
+from abc import ABC, abstractmethod
+
 import heapq
 import itertools
 import json
 import threading


+class SchedulingPolicy(ABC):
+    """Decides which pending job pop_ready() should hand out next."""
+
+    @abstractmethod
+    def select(self, heap, jobs):
+        """heap: the queue's internal (-priority, seq, job_id) list. jobs: id -> Job.
+        Returns a ready Job, or None if none is ready."""
+
+
+class PriorityPolicy(SchedulingPolicy):
+    def select(self, heap, jobs):
+        while heap:
+            _, _, job_id = heapq.heappop(heap)
+            job = jobs[job_id]
+            if job.status == "pending":
+                return job
+        return None
+
+
+_POLICIES = {"priority": PriorityPolicy}
+
+
+def _resolve_policy(policy):
+    if isinstance(policy, SchedulingPolicy):
+        return policy
+    return _POLICIES[policy or "priority"]()
+
+
 class Job:
-    def __init__(self, id, payload, priority=0, status="pending", attempts=0, max_attempts=3):
+    def __init__(self, id, payload, priority=0, status="pending", attempts=0, max_attempts=3, metadata={}):
         self.id = id
         self.payload = payload
         self.priority = priority
         self.status = status
         self.attempts = attempts
         self.max_attempts = max_attempts
+        self.metadata = metadata
@@ class JobQueue:
-    def __init__(self):
+    def __init__(self, policy=None):
         self._lock = threading.Lock()
         self._jobs = {}
         self._heap = []
         self._counter = itertools.count()
+        self._policy = _resolve_policy(policy)
@@ class JobQueue:
     def pop_ready(self):
         """Return the highest-priority pending job, marking it "running" so no other caller can
         receive the same job. Returns None if no job is pending."""
         with self._lock:
-            while self._heap:
-                _, _, job_id = heapq.heappop(self._heap)
-                job = self._jobs[job_id]
-                if job.status == "pending":
-                    job.status = "running"
-                    return job
-            return None
+            job = self._policy.select(self._heap, self._jobs)
+            if job is not None:
+                job.status = "running"
+            return job
```

### Part 1 —— 审查四个 PR

针对下面给出的四个 PR，按给定顺序依次写出：

1. 它试图解决的产品或工程问题，一到两句话。
2. 它的实现思路概述，以及它改动了 `jobqueue.py`（或其测试）的哪些部分。
3. 评审意见，需要覆盖正确性、边界情况、可维护性、测试、性能以及安全性或兼容性，每一条都要指向具体的行
   或具体的行为；并说明新增或改动的行为缺了哪些测试。
4. 这个 PR 目前是否可以合并；如果不能，具体是什么在阻塞它。

### Part 2 —— 改进一个 PR

从四个 PR 中恰好选出一个来改进；不必是问题最明显的那个。说明选了哪一个、为什么选它，也说明另外三个各自
为什么没有被选中。然后：

1. 直接修改 `jobqueue.py`（以及 `test_jobqueue.py`），把所选 PR 剩下的问题解决掉。
2. 为每一处新增或改动的行为补充、更新或修复测试。除非某个测试所检查的行为已经不存在，否则不要删除它。
3. 运行仓库的测试命令 `python -m unittest -v`，并报告结果。
4. 重写这个 PR 的描述：改动摘要、做出的设计取舍、测试结果，以及仍然存在的已知限制。

## 参考解答

<details>
<summary>展开参考解答</summary>

如果还没有事先说清楚，先和面试官确认：Part 1 要评分的交付物是写下来的评审意见本身（而不是和助手对话的
记录），并且“可以合并”指的是按代码库今天的状态就能合并，而不是经过某个没有说明的后续工作量之后才能合并。

### 与助手协作

时间很紧——四份评审加一次动手修复——所以在这一轮里，该让助手做什么是会变的。

在 Part 1，助手适合用来对每个 diff 做一遍机械的初筛：“列出这个 diff 增删的每一行，以及每一行会改变哪些
可观察的行为”，得到的是一份可以修改的草稿，而不是可以直接粘贴的评论。但它不适合直接给结论：如果笼统地问
PR #3“看起来没问题吧”，它给出的回答往往也很笼统——pickle 文档齐全，diff 也不大。真正能问到要害的问题要
针对这段代码本身很具体：“`load()` 会打开一个由调用方给出的路径——列出这个仓库里所有已经存在的、或者可以
想见会出现的、以一个可能被进程外部的值影响的路径去调用 `save` 或 `load` 的地方。”这个问题可以逐行对照
文件本身来回答；“这样安全吗”这种问法做不到。PR #2 同理：与其问把检查挪到锁外面是否安全，不如让它逐步写出
两个线程的一种交织执行顺序，使两者都在重复 id 检查处看到 `job_id not in self._jobs`。如果它给不出这样
一种顺序，“这个 PR 是安全的”这个说法就没有分量；如果能给出，这个交织顺序本身，几乎原封不动，就是评审意见。

助手在 Part 2 给出的 diff，要按看同事 PR 的方式去读，而不是扫一眼有没有绿勾就算过。让它把 PR #1 的批量
提交改成原子的，一个看起来很合理的初稿会是在 `with self._lock:` 内部循环调用 `self.submit(...)`——因为
这样复用了已有的、有测试覆盖的代码——但这会在第一个多条目的批次上死锁，因为 `threading.Lock` 不可重入，
而 `submit` 在外层 `with` 退出之前又会去取同一把锁。不管有没有靠读 diff 先看出这一点，跑一遍新测试都能
当场抓到。一处改动值得提交，前提是它的作者能用一句话说清楚每一
行为什么需要，且这句话不依赖助手自己给出的解释——如果说不出这句话，这一行就该被去掉或者打回去改。

大约 55 分钟要完成四份评审加一次实现，每份评审大约 5 分钟——把阅读过程中已经得出的结论写下来，而不是在
助手里从头组织——剩下的时间留给 Part 2。这个时间预算本身就是下面选择改进 PR #1 的原因：PR #2 或 PR #3
要正确修好，需要的是证明，而不仅仅是重写一遍，而无论是证明哪一个（一次强制交织的并发测试，或者一套真正
重新设计的持久化方案），在写完四份评审之后剩下的时间里都做不完。

### Part 1

**PR #1 —— `submit_many`。** 解决的是启动时的批量提交问题，把每个调用方自己写的 `submit` 循环换成一次
调用。实现是一行列表推导式，对每个条目调用一次 `self.submit` 并收集结果；没有改动其它方法。

- 描述里的性能说法和 diff 实际做的事对不上：`submit_many` 调用的是 `self.submit`，而 `submit` 每次调用
  仍然要取 `self._lock`，所以整批下来还是每个条目取一次锁，和它要替代的那个循环完全一样。值得留的评论
  不是“这样写错了”——它照样能正常工作——而是描述夸大了改动的效果，一个更相信描述而不去看 diff 的读者会
  误以为存在一个其实并不存在的好处。
- 没有针对空批次（`submit_many([])`；这里其实无害，只是没写明）的测试，也没有针对批次*内部*出现重复
  id、或者批次里的 id 与队列里已有任务重复这两种情况的测试。遇到重复 id 时，`self.submit` 会在第 *n*
  个条目上抛出 `ValueError`，但前面 *n* - 1 个已经提交成功、留在了队列里——这次调用不是原子的，而这里
  也没有说明这是不是预期行为。
- 能否合并：目前还不行。如果遇到重复 id 时允许部分提交本来就是预期行为，那实现不需要改，但这个意图需要
  写清楚并配上测试；不管是哪种情形，上面两个缺失的边界情况都需要补测试；描述里那个有误导性的性能说法也
  应该去掉。

**PR #2 —— `submit` 的锁竞争。** 解决（或者说试图解决）的是突发提交下的锁竞争问题：把重复 id 检查挪到
`with self._lock:` 之前，让最常见的情形（一个全新的 id）持锁的时间更短。只改动了 `submit`；测试文件没
有变化。

- `if job_id in self._jobs` 和紧接着的插入操作不再是同一个原子步骤了。两个线程同时调用
  `submit("x", ...)`，可能在两边都还没把 `"x"` 插入 `self._jobs` 之前就都做完了检查；两边看到的都是
  `"x" not in self._jobs`，都会继续往下走，也都会“成功”——谁都不会抛出 `ValueError`，尽管 PR 自己的
  描述里写着“重复 id 仍然会抛出 `ValueError`”。两次 `self._jobs["x"] = job` 赋值里，后执行的那次会
  生效；另一个调用方的 `Job` 对象就这样被悄悄丢弃了，即便它的 `submit` 调用明明是正常返回的。任何依赖
  这个异常来发现冲突的调用方（比如对同一个任务 id 做重试提交，这是让提交幂等的一种常见做法）恰恰会在
  这个 PR 想要优化的并发负载下失去这层保证。
- 没有任何新测试覆盖并发提交同一个 id 的情形——而这恰恰是这处改动唯一涉及的行为。
- 性能方面的动机本身只是断言，没有实测：描述里提到了性能分析，但 PR 里没有附带任何基准测试，而被挪到锁
  外面的那个检查只是一次字典查找，算不上昂贵的操作，所以用正确性换来的这点收益并没有被证实。
- 能否合并：不能。这个改动是对模块明确承诺的那一条属性（拒绝重复 id）的回归，而且是悄无声息的（没有
  异常、没有日志），修法也不是把这个 diff 削减一下就行——而是要退回到在同一段不中断的 `self._lock`
  持有期内完成检查和插入。

**PR #3 —— pickle 持久化。** 解决的是如何在保存的任务里表示 JSON 不能安全表达的 payload。只改动了
`save` 和 `load`，把 `json.dump`/`json.load` 换成了 `pickle.dump`/`pickle.load`，文件打开模式也从文本
换成了二进制；调用位置没有变化——正因为如此，它乍看之下才像是一次安全的、可以直接替换的重构。

- `pickle.load` 在反序列化的同时会执行代码：一个精心构造的文件可以在 `load` 读取它的那一刻，通过对象的
  `__reduce__` 调用任意 Python 代码。`json.load` 没有这条路——解析不了的内容只会抛出
  `json.JSONDecodeError`，如果字节本身都不是合法文本，则是 `UnicodeDecodeError`，不管哪一种都不会执行
  别的任何东西。这是否可以被利用，取决于会不会有一个并非进程自己刚用
  `save` 写出的 `path` 传到 `load` 里——比如一个恢复出来的存储卷、一份上传的崩溃转储、一个由任务 id
  拼出来的路径——而这个 PR 完全没有讨论这条边界，更谈不上做任何防护。
- 用现在的 `save` 写出的旧快照是纯文本 JSON；新的 `load` 会以二进制模式打开它们，然后对 JSON 字节调用
  `pickle.load`，结果是解析失败而不是正常读出，既没有迁移方案，描述里也完全没有提到这次格式变更会破坏
  已有文件。
- 完全没有针对 `save`/`load` 的新测试；尤其是描述里作为动机举出的那些更丰富的 payload 类型
  （namedtuple、dataclass、numpy 数组）没有任何一个被实际测试过，也就是说这个 PR 声称新支持的那件事
  恰恰没有测试。
- 能否合并：不能，而且不是靠微调这个 diff 就能修好的。如果更丰富的 payload 确实是真实需求，答案应该是
  一种在加载时不可能执行代码的编码方式——比如 JSON 加上一个明确列出、加载器只被允许重建这些类型的小型
  注册表——而不是把一个通用的反序列化器直接指向一个文件路径。

**PR #4 —— 可插拔的调度策略。** 解决的是（针对一个这个仓库里目前还不存在的使用场景）让 `pop_ready` 能
用优先级加 FIFO 以外的调度规则。改动涉及文件顶部（新增 `SchedulingPolicy` 接口和一个注册表）、
`Job.__init__`（新增 `metadata` 字段）、`JobQueue.__init__`（新增 `policy` 参数），以及 `pop_ready`
（改成委托给策略对象）。

- `Job.__init__(..., metadata={})` 给 `metadata` 用了一个可变默认参数。每一个没有显式传 `metadata=`
  创建出来的 `Job`，共享的都是*同一个* dict 对象，所以往一个任务的 `metadata` 里写东西，会悄悄改到所有
  其它任务的 `metadata`。这是一个确凿、可以直接证明的 bug，与怎么评价这个设计无关；而且这个 PR 里根本
  没有任何策略读写过 `metadata`，这个 bug 还是照样带进来了。
- 目前只有 `PriorityPolicy` 这一个具体策略；这个接口、这个注册表，以及新增的构造参数，都还没有第二个
  实现可以拿来检验，所以目前没有任何东西能证明这套抽象的形状，就适合描述里用来当动机的那个按租户轮询
  的场景。
- `SchedulingPolicy.select` 的文档字符串没有说清楚：一个实现在运行时能不能假设 `self._lock` 已经被
  持有（`pop_ready` 是在锁内部调用它的）；`PriorityPolicy` 自己不需要额外加锁，所以这个问题在这里没有
  暴露出来，但以后写新策略的人完全没办法从这个接口本身得到答案。
- `_resolve_policy` 对一个未知的字符串策略名直接用普通下标去查 `_POLICIES`，所以在 `round_robin` 这个
  策略还不存在时调用 `JobQueue(policy="round_robin")`，得到的是一个从内部裸抛出来的 `KeyError`，而不是
  在调用处给出一个清楚的错误。`policy=`、`SchedulingPolicy`、`PriorityPolicy`、`metadata` 都完全没有
  测试覆盖。
- 能否合并：不能。可变默认参数这个 bug 本身就是硬性阻塞；除此之外，这更是一个需要和作者当面讨论范围的
  问题，而不是逐行修一修就行——要么先按 `PriorityPolicy` 今天的行为原样上线，把插件式的接口留到真正有
  第二个策略需要它的时候再加；要么就把接口连同一个真正的第二个实现一起放进同一个 PR，这样才审得动。

### Part 2

PR #1 是剩余工作量有限、并且能在剩下的时间里完全验证清楚的那一个。PR #2 和 PR #3 都不是差一点小修就能
合并——PR #2 需要把并发方面的说法真正证明出来，而不是再断言一遍；PR #3 需要一套不同的持久化方案，而不
是把 diff 缩小一点。PR #4 的可变默认参数 bug 本身一行就能修好，但这个 PR 真正的问题是范围，这是需要和
作者商量的决定，不该由我们单方面替他们做。PR #1 的实现，对它自己声明的（非原子的）行为来说已经是对的；
缺的是把描述里没说清楚的那件事——批次内出现重复 id 时会怎样——决定下来并配上测试，而且既然描述里那个
性能上的说法值得兑现而不是直接删掉，那就顺便把批次真正做成只取一次锁。

修法是在一次不中断的 `self._lock` 持有期内，先校验完所有 id 再插入任何任务，这样一个批次要么完全成功，
要么让队列保持它原来的样子：

```python
def submit_many(self, items):
    """items: an iterable of (job_id, payload, priority) triples. Submits the whole batch atomically:
    if any job_id is already taken -- by a job already in the queue, or by an earlier item in this same
    batch -- the call raises ValueError and no job in the batch is created. Returns the list of created
    Job objects, in the given order, on success."""
    items = list(items)
    with self._lock:
        seen = set()
        for job_id, payload, priority in items:
            if job_id in self._jobs or job_id in seen:
                raise ValueError(f"duplicate job id {job_id!r}")
            seen.add(job_id)
        created = []
        # NOTE: inserts directly rather than calling self.submit(), which would deadlock here --
        # self._lock is a plain threading.Lock, not reentrant, and submit() takes it too.
        for job_id, payload, priority in items:
            job = Job(job_id, payload, priority, "pending", 0, 3)
            self._jobs[job_id] = job
            heapq.heappush(self._heap, (-priority, next(self._counter), job_id))
            created.append(job)
        return created


JobQueue.submit_many = submit_many
```

先校验、再插入，而且两步都在同一次取锁期间完成，这正是它在并发调用下依然安全的原因：检查和插入之间，
没有任何人能观察到队列的中间状态——这正是 PR #2 在单个 `submit` 上没做对的那个性质。提前把 `items`
具体化成一个 list，是为了让一个由生成器构造出来的批次可以被扫描两遍（一遍校验、一遍插入），而不会在
第一遍之后生成器就被耗尽。

四个测试取代了 PR #1 原来那一个只覆盖正常路径的测试：正常路径、空批次、批次内部重复、批次与队列里已有
任务重复——后两种正是原 PR 没有测试覆盖的情况——另外还各自检查了两种失败情形下队列都完全没有变化：

```python
import unittest


class TestSubmitMany(unittest.TestCase):
    def test_happy_path_submits_all_in_order(self):
        q = JobQueue()
        jobs = q.submit_many([("a", {}, 1), ("b", {}, 5), ("c", {}, 0)])
        self.assertEqual([j.id for j in jobs], ["a", "b", "c"])
        self.assertEqual(q.pop_ready().id, "b")          # highest priority first, batch order aside

    def test_empty_batch_returns_empty_list(self):
        q = JobQueue()
        self.assertEqual(q.submit_many([]), [])
        self.assertEqual(q.counts(), {"pending": 0, "running": 0, "done": 0, "failed": 0})

    def test_duplicate_within_batch_submits_nothing(self):
        q = JobQueue()
        with self.assertRaises(ValueError):
            q.submit_many([("a", {}, 0), ("b", {}, 0), ("a", {}, 0)])
        self.assertEqual(q.counts(), {"pending": 0, "running": 0, "done": 0, "failed": 0})

    def test_duplicate_against_existing_job_submits_nothing_new(self):
        q = JobQueue()
        q.submit("a", {})
        with self.assertRaises(ValueError):
            q.submit_many([("b", {}, 0), ("a", {}, 0)])
        self.assertEqual(q.counts(), {"pending": 1, "running": 0, "done": 0, "failed": 0})
```

把这一组测试和题目部分描述过的既有 `TestJobQueue` 测试（为了让验证代码自成一体，下面重新写出了它们）
一起跑一遍仓库的测试命令，结果是：

```text
python -m unittest -v
test_completed_job_reports_done (TestJobQueue) ... ok
test_fail_retries_then_marks_failed (TestJobQueue) ... ok
test_pop_ready_priority_then_submission_order (TestJobQueue) ... ok
test_save_load_roundtrip (TestJobQueue) ... ok
test_submit_rejects_duplicate_id (TestJobQueue) ... ok
test_duplicate_against_existing_job_submits_nothing_new (TestSubmitMany) ... ok
test_duplicate_within_batch_submits_nothing (TestSubmitMany) ... ok
test_empty_batch_returns_empty_list (TestSubmitMany) ... ok
test_happy_path_submits_all_in_order (TestSubmitMany) ... ok

----------------------------------------------------------------------
Ran 9 tests in 0.002s

OK
```

**重写后的 PR #1 描述。**

> **摘要。** `submit_many` 让调用方可以用一次调用提交一批 `(job_id, payload, priority)` 三元组，而不用
> 再循环调用 `submit`。和那个循环不同的是，这一批是原子的：整次调用只取一次 `self._lock`，先校验所有
> id，然后要么把这一批任务全部创建出来，要么——只要有任意一个 id 和已有任务、或者和同一批里更早的条目
> 冲突——一个都不创建，并抛出 `ValueError`。
>
> **取舍。** 原子性的代价是插入前要多扫一遍 `items` 做校验，而且即使调用方传进来的是生成器，也要先把
> `items` 具体化成一个 list，也就是说整个批次会一次性都放在内存里。以这里的批次规模（启动时重放积压
> 任务，几百个量级）来说，这两个代价都不算什么；如果某个调用方要一次性灌入一个非常大的积压任务集合，
> 应该把它分块，而不是整批传一个巨大的批次。
>
> **测试结果。** `python -m unittest -v` 全部通过，9 个测试、0 个失败，如上所示；两个新增的重复 id
> 用例还各自额外断言了队列没有发生变化。
>
> **已知限制。** 原子性只在单个进程内部得到保证；两个仅通过 `save`/`load` 文件共享状态的进程，仍然
> 可能在谁都还没写文件之前各自都接受了同一个 id。发生冲突时的错误信息没有说明冲突对象是已有任务还是
> 同一批里的另一个条目——对今天唯一的调用方（记录日志再重试）来说够用，如果以后有调用方需要区分这两种
> 情况，值得再改一下。

### 追问

- 崩溃会让一些任务永远停在 `"running"`：`load` 按设计只会把 `"pending"` 的任务重新排队，因为一个
  `"running"` 的任务确实有可能还在另一个进程里真实地跑着。一个知道没有任何 worker 从崩溃中存活下来的
  监控进程，需要的是一个单独的 `requeue_stuck(older_than)` 调用，而不是去改 `load` 本身。
- `pop_ready` 立即返回 `None`，意味着空闲的 worker 要么忙轮询，要么在两次调用之间睡眠；另一种做法是让
  `pop_ready` 阻塞直到有任务就绪，代价是要引入一个条件变量，而且 `submit` 和 `fail` 都得记得去通知它。
- `self._lock` 只保护同一个进程内部的线程；两个进程都对同一个路径调用 `save`，或者一个在存的同时另一个
  在读，仍然可能在文件系统这一层发生交织，需要它们自己的协调机制（比如一个锁文件，或者让某一个进程独占
  状态文件），这已经超出了这个模块的范围。

<details>
<summary>验证代码（可运行）</summary>

```python
import io
import json
import os
import pickle
import tempfile
import threading
import unittest


# --- run the Part 2 test suite: the repository's test command, python -m unittest -v ---
class TestJobQueue(unittest.TestCase):
    def test_submit_rejects_duplicate_id(self):
        q = JobQueue()
        q.submit("a", {})
        with self.assertRaises(ValueError):
            q.submit("a", {})

    def test_pop_ready_priority_then_submission_order(self):
        q = JobQueue()
        q.submit("low", {}, priority=0)
        q.submit("high", {}, priority=5)
        q.submit("also-low", {}, priority=0)
        self.assertEqual(q.pop_ready().id, "high")
        self.assertEqual(q.pop_ready().id, "low")         # equal priority: earlier submission first
        self.assertEqual(q.pop_ready().id, "also-low")
        self.assertIsNone(q.pop_ready())

    def test_completed_job_reports_done(self):
        q = JobQueue()
        q.submit("a", {})
        q.complete(q.pop_ready().id)
        self.assertEqual(q.get("a").status, "done")

    def test_fail_retries_then_marks_failed(self):
        q = JobQueue()
        q.submit("a", {}, max_attempts=2)
        q.fail(q.pop_ready().id)
        self.assertEqual(q.get("a").status, "pending")    # one attempt left
        q.fail(q.pop_ready().id)
        self.assertEqual(q.get("a").status, "failed")      # attempts exhausted

    def test_save_load_roundtrip(self):
        q = JobQueue()
        q.submit("a", {"x": 1}, priority=2)
        q.pop_ready()                                       # "a" is now "running"
        q.submit("b", {"y": 2}, priority=1)
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, "state.json")
            q.save(path)
            restored = JobQueue.load(path)
        self.assertEqual(restored.get("a").status, "running")
        self.assertEqual(restored.get("a").payload, {"x": 1})
        self.assertEqual(restored.get("b").status, "pending")
        self.assertEqual(restored.pop_ready().id, "b")      # "a" is running, not re-queued


suite = unittest.TestSuite([
    unittest.defaultTestLoader.loadTestsFromTestCase(TestJobQueue),
    unittest.defaultTestLoader.loadTestsFromTestCase(TestSubmitMany),   # defined in Part 2, above
])
result = unittest.TextTestRunner(verbosity=0, stream=io.StringIO()).run(suite)
assert result.wasSuccessful() and result.testsRun == 9
print(f"python -m unittest -v: {result.testsRun} tests, "
      f"{len(result.failures) + len(result.errors)} failures -- OK")


# --- PR #2: reconstruct the lock-scope change and force the exact bad interleaving ---
class RacyJobQueue(JobQueue):
    """PR #2's submit(): the duplicate-id check runs before the lock is taken. `_checkpoint` runs
    once, right after that check, purely to force a deterministic interleaving below -- production
    code never passes one."""

    def submit(self, job_id, payload, priority=0, max_attempts=3, _checkpoint=lambda: None):
        if job_id in self._jobs:
            raise ValueError(f"duplicate job id {job_id!r}")
        _checkpoint()
        with self._lock:
            job = Job(job_id, payload, priority, "pending", 0, max_attempts)
            self._jobs[job_id] = job
            heapq.heappush(self._heap, (-priority, next(self._counter), job_id))
        return job


def demonstrate_forced_race():
    """Thread A pauses right after its duplicate check, with job "x" still absent; thread B then
    runs its own submit("x", ...) to completion; only then does thread A resume and insert."""
    queue = RacyJobQueue()
    a_at_checkpoint, b_finished = threading.Event(), threading.Event()
    results = {}

    def checkpoint():
        a_at_checkpoint.set()
        assert b_finished.wait(timeout=5), "thread b did not finish in time"

    def run_a():
        try:
            results["a"] = queue.submit("x", {"from": "a"}, _checkpoint=checkpoint)
        except ValueError as e:
            results["a"] = e

    def run_b():
        assert a_at_checkpoint.wait(timeout=5), "thread a never reached its checkpoint"
        try:
            results["b"] = queue.submit("x", {"from": "b"})
        except ValueError as e:
            results["b"] = e
        b_finished.set()

    ta, tb = threading.Thread(target=run_a), threading.Thread(target=run_b)
    ta.start(), tb.start()
    ta.join(timeout=5), tb.join(timeout=5)
    return queue, results


racy_queue, racy_results = demonstrate_forced_race()
assert not isinstance(racy_results.get("a"), Exception) and not isinstance(racy_results.get("b"), Exception), \
    f"PR #2: both callers should wrongly believe they succeeded, got {racy_results}"
assert racy_queue.counts()["pending"] == 1, "PR #2: two successful submits should collapse into one job"
first, second = racy_queue.pop_ready(), racy_queue.pop_ready()
assert first is not None and first.id == "x" and second is None, \
    "PR #2: one of the two submitted payloads should have vanished without a trace"
print("PR #2 (racy submit): two threads both 'succeeded', but only one job was ever dispatched")


def concurrent_double_submit(queue):
    results = {}

    def call(key):
        try:
            results[key] = queue.submit("x", {"from": key})
        except ValueError as e:
            results[key] = e

    ta, tb = threading.Thread(target=call, args=("a",)), threading.Thread(target=call, args=("b",))
    ta.start(), tb.start()
    ta.join(timeout=5), tb.join(timeout=5)
    return results


for trial in range(50):                                      # no forced interleaving needed: the
    fixed_queue = JobQueue()                                  # lock makes every interleaving correct
    outcome = concurrent_double_submit(fixed_queue)
    wins = sum(not isinstance(v, Exception) for v in outcome.values())
    fails = sum(isinstance(v, ValueError) for v in outcome.values())
    assert wins == 1 and fails == 1, f"trial {trial}: fixed submit() let both or neither succeed -- {outcome}"
    assert fixed_queue.counts()["pending"] == 1
print("PR #2 (fixed submit): 50 concurrent trials, always exactly one winner and one ValueError")


# --- PR #3: reconstruct the pickle switch and prove it executes an attacker-controlled file ---
class PickleJobQueue(JobQueue):
    """PR #3's save/load: pickle instead of json, otherwise identical to JobQueue."""

    def save(self, path):
        with self._lock:
            rows = [job.to_dict() for job in self._jobs.values()]
        with open(path, "wb") as f:
            pickle.dump(rows, f)

    @classmethod
    def load(cls, path):
        queue = cls()
        with open(path, "rb") as f:
            rows = pickle.load(f)
        for row in rows:
            job = Job(**row)
            queue._jobs[job.id] = job
            if job.status == "pending":
                heapq.heappush(queue._heap, (-job.priority, next(queue._counter), job.id))
        return queue


_rce_marker = []


def _record(tag):     # a module-level function so pickle can reference it by name, like any real one
    _rce_marker.append(tag)
    return []          # PickleJobQueue.load then iterates an empty "rows": no crash, no trace either


class MaliciousState:
    """What an attacker-controlled "saved state" file can contain: __reduce__ makes unpickling it
    call an arbitrary function with arbitrary arguments, before any application code runs."""

    def __reduce__(self):
        return (_record, ("pwned",))


with tempfile.TemporaryDirectory() as d:
    path = os.path.join(d, "state")
    with open(path, "wb") as f:
        pickle.dump(MaliciousState(), f)

    PickleJobQueue.load(path)                                  # merely loading the file runs _record
    assert _rce_marker == ["pwned"], "PR #3: unpickling should have executed the payload"

    try:
        JobQueue.load(path)                                    # the json-based load on the same bytes
        raised = None
    except Exception as e:
        raised = e
    assert raised is not None, "PR #3 control: pickle bytes are not valid JSON and must fail to parse"
    assert _rce_marker == ["pwned"], "json.load must never execute the file's content, however it fails"
print(f"PR #3 (pickle load): crafted file executed code on load; "
      f"the json-based load only raised {type(raised).__name__}")


# --- PR #4: reconstruct the mutable-default bug on Job.metadata ---
class BuggyJob:
    """PR #4's change to Job.__init__: metadata={} is a mutable default argument."""

    def __init__(self, id, metadata={}):
        self.id = id
        self.metadata = metadata


buggy_a, buggy_b = BuggyJob("a"), BuggyJob("b")
buggy_a.metadata["owner"] = "tenant-1"
assert buggy_b.metadata == {"owner": "tenant-1"}, "PR #4: every Job should wrongly share one dict"


class FixedJob:                       # metadata=None, defaulted inside __init__: the standard fix
    def __init__(self, id, metadata=None):
        self.id = id
        self.metadata = metadata if metadata is not None else {}


fixed_a, fixed_b = FixedJob("a"), FixedJob("b")
fixed_a.metadata["owner"] = "tenant-1"
assert fixed_b.metadata == {}, "a fresh dict per job must not leak between jobs"
print("PR #4 (mutable default): confirmed the shared-dict leak, and that metadata=None does not have it")

print("all checks passed")
```

</details>

</details>
