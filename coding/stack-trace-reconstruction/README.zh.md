# 从采样还原调用栈事件

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 调用栈，字符串处理 | ★★★★☆ | 中等 | SWE · MLE · Infra Eng | call-stack, longest-common-prefix, streaming, debounce, trie, recursion | 3 个部分 / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

性能分析器（profiler）通过周期性地采集*样本*（sample）来观察一个正在运行的程序：在某些时刻记录下完整的调用
栈——按顺序列出当前栈上的所有函数，从最外层（运行时间最长、在这段追踪里不是被任何函数调用的那个）到最内层
（此刻正在执行的那个）。一个 `Sample` 把一个时间戳和这份列表配对；一个 `Event` 表示某个函数开始或结束其在
栈上的这段时间。

```py
from typing import Literal, NamedTuple


class Sample(NamedTuple):
    ts: float
    stack: list[str]   # outermost (root) first, innermost (currently executing) last


class Event(NamedTuple):
    kind: Literal["start", "end"]
    ts: float
    name: str
    depth: int   # 0-based position in the stack
```

同一个函数名可以在同一时刻的两个不同深度上被报告——直接或间接递归——因此一个事件的*帧*（frame）由它的
`(depth, name)` 二元组来标识，而不能只看 `name`：下文每个事件里的 `depth` 都是标识的一部分，不是附带的调
试信息。

### Part 1 —— 从样本到事件

`convert_samples_to_events(samples, end_time)` 接收按 `ts` 严格递增排序的 `samples`，返回一份事件列表，
说明调用栈从第一个样本到 `end_time`（一个不早于最后一个样本 `ts` 的时间戳）之间是如何演变的；如果
`samples` 为空，`end_time` 不会被用到，结果是 `[]`。

在第一个样本的时间戳上，其调用栈里的每一帧都会启动（start），顺序从最外层到最内层。

相邻两个样本之间，从外到内逐个位置比较两个调用栈，找出它们的*最长公共前缀*（longest common prefix，
LCP）：使两个栈在位置 `0, 1, ..., p - 1` 上都一致的最大 `p`。较早那个栈里位置在 `p` 或更深处的帧，已经离开
了调用栈；较晚那个栈里位置在 `p` 或更深处的帧，则是新进入的。这两类变化都打上较晚那个样本的 `ts` 时间戳：
离开的帧先结束（end），顺序从最内层往外（调用最深的那个先返回）；然后新出现的帧才开始（start），顺序从
`p` 往内（调用者必须先出现在栈上，它调用的函数才能开始）。如果相邻两个样本的调用栈完全相同（`p` 等于两者
的长度），则不产生任何事件——这正是让一段不变的样本序列被压缩成空的原因。

在 `end_time` 时刻，最后一个样本调用栈里仍然存在的每一帧都会结束，顺序从最内层到最外层；对这些帧来说，
`end_time` 是它们结束事件的唯一来源——`convert_samples_to_events` 绝不会仅仅因为输入结束，就在更早的时间
点关闭某一帧。

```py
def convert_samples_to_events(samples: list[Sample], end_time: float) -> list[Event]: ...
```

```text
samples = [
    Sample(0.0, ["main"]),
    Sample(1.0, ["main", "solve"]),
    Sample(2.0, ["main", "solve", "solve"]),           # solve 调用了自己：两个名为 "solve" 的帧同时活动
    Sample(2.5, ["main", "solve", "solve"]),           # 与上一个样本完全相同 -> 2.5 时刻不产生任何事件
    Sample(4.0, ["main", "solve", "solve", "solve"]),  # solve 又递归了一层：三个 "solve" 帧同时活动
    Sample(6.0, ["main", "solve", "helper"]),          # 两个最内层的 "solve" 帧返回；转而调用了 helper
]
convert_samples_to_events(samples, end_time=9.0) == [
    Event("start", 0.0, "main", 0),
    Event("start", 1.0, "solve", 1),
    Event("start", 2.0, "solve", 2),      # 第二个、独立的 "solve" 帧，比第一个深一层
    # 2.5 时刻，与上一个栈的公共前缀是 3（整个栈）-> 不产生事件
    Event("start", 4.0, "solve", 3),      # 第三个、独立的 "solve" 帧
    # 6.0 时刻，与 ["main","solve","solve","solve"] 的公共前缀是 2：main 和 solve@1 不受影响
    Event("end", 6.0, "solve", 3),        # 深度 3 先离开（最内层）
    Event("end", 6.0, "solve", 2),        # 然后是深度 2
    Event("start", 6.0, "helper", 2),     # 之后新的深度 2 帧才开始
    # end_time = 9.0：main、solve@1、helper@2 在最后一个样本里都仍然活动 -> 全部关闭
    Event("end", 9.0, "helper", 2),
    Event("end", 9.0, "solve", 1),
    Event("end", 9.0, "main", 0),
]
```

### Part 2 —— 流式处理与去噪

真实的性能分析器会在一次可能很长的运行过程中逐条地送出样本；`stream_debounced_events` 持有的状态不能随着
已经处理过的样本数量增长。它把 `samples` 当作一个迭代器来消费——这个迭代器可能只能从头到尾安全地读取一
遍——并且随着事件变得确定就逐个 yield 出来，而不是返回一份完整列表。和 Part 1 不同，它不接收 `end_time`：
当迭代器耗尽时，一个尚未被确认关闭的帧不会再产生任何事件，这正是 Part 1 中“绝不关闭仍在最后一个样本里活动
的帧”这条规则的推广——这里推广成了“尚未被确认关闭”。

采样得到的性能分析结果也会带有噪声：仪器偶尔误报了某一帧，或者某次调用短到只被一个样本恰好捕捉到。
`stream_debounced_events(samples, k)` 用一条去抖（debounce）规则过滤掉这些噪声。帧的识别方式和 Part 1 完
全一样，按照从一个样本到下一个样本、沿最长公共前缀链所到达的位置来判断；因此这条链上的任何一次断裂，都会
在它触及或越过的每一个位置上开启一个全新的帧，哪怕同一个名字紧接着又重新占据了这个位置（参见下面的例
子）。只有当一帧在某个位置上连续保持了 `k` 个样本（`k >= 1`；`k = 1` 表示不需要去抖，退化为 Part 1 本身的
那些变化，只是不包含 `end_time` 处的最终关闭——Part 2 根本没有 `end_time` 这个参数）之后，它才会被报告。它的
`start` 事件打上
这连续 `k` 个样本中第 `k` 个的时间戳。如果这一帧所在的位置之后变为空缺，并连续 `k` 个样本保持空缺，它的
`end` 事件就打上*这* `k` 个样本中第 `k` 个的时间戳。一帧如果始终没有连续占据满 `k` 个样本，就既不会有
start 事件，也不会有 end 事件；一帧如果达到了 `k`，但它的空缺在输入结束前始终没有连续满 `k` 个样本，就只
有 start 事件、没有 end 事件。

举例来说，取 `k = 1`（每一个样本自己就已经足够）：`["req", "validate"]` 之后跟着
`["req", "validate", "persist"]`，公共前缀是 2（也就是前一个栈的全部）：`req` 和 `validate` 都没有中断，
只有 `persist` 是新开始的。但 `["req", "validate"]` 之后如果跟着 `["worker", "validate", "req"]`，公共
前缀是 0——位置 0 上已经不一致，`req` 对 `worker`——于是 `req` 和 `validate` 都在此处结束，一个全新的
`req` 则在深度 2 上开始，即便名字 `validate` 碰巧仍然占据着深度 1、名字 `req` 碰巧又在更深一层重新出
现：它们都和各自位置上先前的占用者没有关系，因为这条链早在到达它们之前就已经断裂。

```text
stream_debounced_events([Sample(1, ["req", "validate"]), Sample(2, ["req", "validate", "persist"])], k=1)
== [Event("start", 1, "req", 0), Event("start", 1, "validate", 1), Event("start", 2, "persist", 2)]

stream_debounced_events([Sample(1, ["req", "validate"]), Sample(2, ["worker", "validate", "req"])], k=1)
== [Event("start", 1, "req", 0), Event("start", 1, "validate", 1),
    Event("end", 2, "validate", 1), Event("end", 2, "req", 0),
    Event("start", 2, "worker", 0), Event("start", 2, "validate", 1), Event("start", 2, "req", 2)]
```

```py
from collections.abc import Iterable, Iterator


def stream_debounced_events(samples: Iterable[Sample], k: int) -> Iterator[Event]: ...
```

```text
samples = [Sample(1, ["svc"]), Sample(2, ["svc"]), Sample(3, ["svc", "handle"]), Sample(4, ["svc"]),
           Sample(5, ["svc", "flush"]), Sample(6, ["svc", "flush"]), Sample(7, ["svc"]),
           Sample(8, ["svc"]), Sample(9, ["svc"])]
list(stream_debounced_events(samples, k=2)) == [
    Event("start", 2, "svc", 0),     # 深度 0 在样本 1、2 上都是 "svc"：连续 2 个 -> 在样本 2 处被确认
    # 深度 1 只在样本 3 上是 "handle"，样本 4 又变了回去：只连续了 1 个 -> 从未被确认，没有事件
    Event("start", 6, "flush", 1),   # 深度 1 在样本 5、6 上都是 "flush"：连续 2 个 -> 在样本 6 处被确认
    Event("end", 8, "flush", 1),     # 深度 1 在样本 7、8 上都空缺：连续 2 个 -> 在样本 8 处被关闭
    # 输入结束时 "svc" 仍然活动，它的空缺从未开始 -> 没有它的 end 事件
]
```

### Part 3 —— 针对已知调用栈的后缀匹配

一份*完整*的调用栈追踪记录了从入口点（最外层）到出错点（最内层）之间的每一帧。`TraceIndex` 由一批固定
的、已知的完整追踪一次性构建而成，每条追踪都有一个唯一的 `trace_id`；构建完成后用来回答查询：只给出一份不
完整追踪的最内层若干帧——即它的*后缀*（suffix）——报告每一条已知追踪，只要把它自己的末尾若干帧截取到同样
的长度，就与这份后缀逐帧完全相同。

`TraceIndex(traces)` 接收 `traces`：一份 `(trace_id, frames)` 二元组的列表，其中 `frames` 按入口点在
前、出错点在后排列，且非空。`index.match_suffix(frames)` 接收一份同样顺序、非空的查询，返回每一条帧数不
少于 `len(frames)` 且自身末尾 `len(frames)` 帧与 `frames` 逐位置完全相等的已知追踪的 `trace_id`，按字典
序升序排列；如果没有任何已知追踪匹配——包括那些帧数本身就比查询短的追踪——则返回字符串 `"UNKNOWN"`，而不
是空列表。

```py
class TraceIndex:
    def __init__(self, traces: list[tuple[str, list[str]]]) -> None: ...
    def match_suffix(self, frames: list[str]) -> list[str] | str: ...
```

```text
traces = [
    ("T1", ["boot", "ingest", "parse", "oom"]),
    ("T2", ["worker", "parse", "oom"]),
    ("T3", ["boot", "cache", "oom"]),
    ("T4", ["boot", "ingest", "oom"]),
]
index = TraceIndex(traces)
index.match_suffix(["parse", "oom"])                   == ["T1", "T2"]              # 两者都以 ..., parse, oom 结尾
index.match_suffix(["oom"])                            == ["T1", "T2", "T3", "T4"]  # 每条追踪都以 oom 结尾
index.match_suffix(["ingest", "oom"])                  == ["T4"]                    # 只有 T4 以 ..., ingest, oom 结尾
index.match_suffix(["boot", "ingest", "parse", "oom"]) == ["T1"]                    # 只有匹配整条追踪才成立
index.match_suffix(["cache", "parse", "oom"])          == "UNKNOWN"                 # 没有已知追踪这样结尾
```

## 参考解答

<details>
<summary>展开参考解答</summary>

动手之前有两点约定值得先确认：同一个函数名在两个不同深度上永远是两个不同的帧，因此帧的身份要沿着最长公共
前缀链按位置追踪，而不能只看名字；以及一个事件的时间戳是变化被*观测到*的那一刻，而不是在断言调用真的是在
那个精确瞬间开始或返回的——一次调用如果完全发生在两个样本之间、开始又结束，样本里根本不会留下任何痕迹，
任何只依赖样本的算法都无法把它找回来。

### Part 1

只需要保留上一个样本的调用栈；其余的一切都来自同一个辅助函数——让两个栈一起前进，看它们能保持一致到多远。

```python
from collections.abc import Iterable, Iterator
from typing import Literal, NamedTuple


class Sample(NamedTuple):
    ts: float
    stack: list[str]   # outermost (root) first, innermost (currently executing) last


class Event(NamedTuple):
    kind: Literal["start", "end"]
    ts: float
    name: str
    depth: int   # 0-based position in the stack -- the same name at two depths is two different frames


def _common_prefix_len(a: list[str], b: list[str]) -> int:
    n = 0
    while n < len(a) and n < len(b) and a[n] == b[n]:
        n += 1
    return n


def convert_samples_to_events(samples: list[Sample], end_time: float) -> list[Event]:
    if not samples:
        return []
    events: list[Event] = []
    first = samples[0]
    for depth, name in enumerate(first.stack):
        events.append(Event("start", first.ts, name, depth))   # NOTE: only the very first sample gets this;
    prev = first.stack                                          # Part 2 has no such special case
    for sample in samples[1:]:
        lcp = _common_prefix_len(prev, sample.stack)
        for depth in range(len(prev) - 1, lcp - 1, -1):        # innermost first: a callee returns before its caller
            events.append(Event("end", sample.ts, prev[depth], depth))
        for depth in range(lcp, len(sample.stack)):             # outermost first: a caller starts before its callee
            events.append(Event("start", sample.ts, sample.stack[depth], depth))
        prev = sample.stack
    for depth in range(len(prev) - 1, -1, -1):                  # innermost first, same rule as any other close
        events.append(Event("end", end_time, prev[depth], depth))
    return events
```

整个运行过程中，每个位置只会被访问常数次（被某次停在那里的 `_common_prefix_len` 调用访问一次，如果它变成
了一个事件，再被访问一次），所以总时间与每个样本调用栈大小之和成线性关系。除了返回的列表之外，在各个样本
之间需要保留的状态只有 `prev`，所以额外内存是 $O(D)$，其中 $D$ 是任意样本达到过的最大栈深度。

### Part 2

一个位置的连续在场计数，只能跟着它父位置的计数一起延续：让位置 $d$ 保持在链上的那一步最长公共前缀计算，
在同一步里也会让所有比它更浅的位置保持在链上，所以更浅的位置总是在同一个样本、或更早的样本上被确认，不会
晚于更深的位置。这意味着已确认的位置永远是一段连续的前缀，可以用一个整数 `open_len` 来追踪，而不需要一个
集合；再用一份 `streak` 列表，只覆盖这段前缀之外那些尚未确认的位置。一个已确认的位置一旦之后离开了这条
链，就会进入一个以深度为键的小字典 `closing`，统计连续缺席的样本数，直到达到 `k`——或者，按上面那条边界
情形，一旦它的深度被另一个候选者占据，就被放弃。同一个深度上任意时刻最多只会存在一条 `closing` 记录，因
为一个替补的候选者自己也需要 `k` 个样本才能被确认，而到那时，原先占用者自己那连续 `k` 个样本的空缺早已经
有了结果。因此不论已经经过了多少个样本，`streak` 和 `closing` 加在一起始终是 $O(D)$。

```python
def stream_debounced_events(samples: Iterable[Sample], k: int) -> Iterator[Event]:
    prev: list[str] = []
    streak: list[int] = []           # streak[d]: consecutive samples (ending at prev) position d has held prev[d]
    open_len = 0                     # positions [0, open_len) are confirmed -- their start has already been emitted
    closing: dict[int, list] = {}    # depth -> [name, consecutive samples absent since] for a confirmed frame just vacated

    for sample in samples:
        cur = sample.stack
        lcp = _common_prefix_len(prev, cur)

        for entry in closing.values():
            entry[1] += 1                                             # every pending closure ages by one sample
        for depth in range(len(prev) - 1, lcp - 1, -1):                # positions leaving the chain, innermost first
            if depth < open_len:
                closing[depth] = [prev[depth], 1]                      # NOTE: overwrites -- no stale entry can coexist here
        open_len = min(open_len, lcp)
        for depth in sorted((d for d, entry in closing.items() if entry[1] >= k), reverse=True):
            name, _count = closing.pop(depth)
            yield Event("end", sample.ts, name, depth)

        streak = [streak[d] + 1 if d < lcp else 1 for d in range(len(cur))]
        while open_len < len(cur) and streak[open_len] >= k:            # NOTE: streak is non-increasing in depth,
            yield Event("start", sample.ts, cur[open_len], open_len)    # so this never skips a confirmable position
            open_len += 1
        prev = cur
```

每个样本所做的工作都与它自身的栈深度成正比（前缀比较、重建 `streak`、每个离开的位置至多一次 `closing`
字典查找），所以整个运行过程的总时间同样与采样到的栈大小之和成线性关系；除去调用方从 yield 出的事件里自
行保留的部分，额外内存是 $O(D)$：`prev` 和 `streak` 都以见过的最大栈深度为界，`closing` 则由上面的论证
给出界限。

### Part 3

按查询的*后缀*去匹配每条已知追踪的后缀，一旦把每个序列都反转成最内层帧在前，就恰好变成了按*前缀*匹配。把
每条已知追踪反转后的帧列表插入一棵字典树（trie）；一份长度为 $M$ 的查询，沿同样反转的方向走同样的 $M$
步，落在的那个字典树节点，恰好就是所有与它共享这段长度为 $M$ 的后缀的追踪都会经过的节点。在插入时就记录
下经过每个节点的全部 `trace_id`——而不只是恰好在那里结束的那一条——每条追踪的路径上每个节点只多花一次追
加操作，这样一次查询只需要走到对应的节点，读出已经放在那里的（预先排好序的）一份列表副本即可。

```python
class TraceIndex:
    """Indexes a fixed set of complete stack traces for exact-suffix lookup."""

    def __init__(self, traces: list[tuple[str, list[str]]]) -> None:
        self._root: dict = {}
        for trace_id, frames in traces:
            node = self._root
            for frame in reversed(frames):                      # innermost (error point) first
                node = node.setdefault(frame, {})
                node.setdefault("$", []).append(trace_id)        # NOTE: every node on the path, not just the last
        self._sort_ids(self._root)

    def _sort_ids(self, node: dict) -> None:
        if "$" in node:
            node["$"].sort()
        for key, child in node.items():
            if key != "$":
                self._sort_ids(child)

    def match_suffix(self, frames: list[str]) -> list[str] | str:
        node = self._root
        for frame in reversed(frames):
            if frame not in node:
                return "UNKNOWN"          # NOTE: no known trace even has this many matching trailing frames
            node = node[frame]
        matches = node.get("$", [])
        return list(matches) if matches else "UNKNOWN"
```

构建这棵字典树所需的时间，与所有已知追踪的长度之和成正比（每条追踪的每一帧都对应遍历中的一层、一次字典
插入），再加上对每个节点上累积的 id 列表做一次性排序。之后每次 `match_suffix` 的开销是 $O(M)$ 加上它自
身结果的大小，其中 $M$ 是查询的长度——与已知追踪有多少条、每条有多长都无关。

### 追问

- **另一个方向。** 如果输入换成一串 `enter`/`exit` 事件而不是样本，用一个普通的栈来重放它们——`enter` 时
  压栈，`exit` 时弹栈并核对名字——每处理完一个事件就读出此刻的活动栈；把连续相同的栈行程压缩
  （run-length compress）成一条带计数的记录，或者，如果压缩应该按经过的时间而不是样本个数来限定（这样一
  段长时间不变但确实空闲的栈仍会被切开），就改为带一个持续时长，一旦从当前记录开始算起过去了太久，就另起
  一条新记录。
- **一个样本只能看到最内层的 $m$ 帧。** 如果一个样本只显示它最内层的 $m$ 帧（调用者那一侧被隐藏，而且
  $m$ 在不同样本之间还可能变化），相邻两个样本就不能再像 Part 1、Part 2 那样从根部开始比较：应该把比较
  的锚点放在最内层的帧上，向外扩展，只要两份可见的后缀还保持一致就继续。超出第一处不一致、或者超出两份可
  见窗口中较短那一份的帧，就没办法和一个单纯不在视野内的帧区分开，因此没有办法安全地为它报告任何事
  件——包括观测停止那一刻，最后一个样本里那个正处于调用中途的帧：之后再也没有观测能确认它何时返回，甚至
  是否会返回。
- **只保留最后 $N$ 个样本。** 直接保留这 $N$ 个原始样本本身，开销是 $O(ND)$，相比 Part 2 的 $O(D)$ 高出
  不少，所以只有当查询确实需要重建不止一次最近的变化时，才值得这样做。但无论怎么保留，*能够知道什么*并
  不会因此改变：一次调用如果完全发生在两个被保留的样本之间、开始又结束，即使每个被保留的栈看起来都完全
  一致，这次调用依然是不可见的；而对于任何在窗口左边界就已经活动的帧，能知道的也只是它在这个边界或更早
  就已经开始——而不是具体什么时候。
- **事件的时间戳是一种观测，而不是事实。** 用样本自身的 `ts` 去标记一个 `start` 或 `end`，是本题为了得
  到一个确定答案而采用的约定，而不是在断言调用真的是在那个精确瞬间开始或返回的——真实的时刻落在与上一个
  样本之间那段、可能相当长的间隔里的某处。采样间隔更粗，只会让时间戳更粗略，而不是错误；只有当采样间隔
  已经远小于被测量的持续时间时，把这些时间戳当作精确值才是安全的。
- **用一个区间代替两个独立事件。** 同一帧的 `start`/`end` 事件对，完全可以等价地报告成单独一条
  `(name, depth, start_ts, end_ts)` 的区间（span）；对于从未被确认关闭的帧，`end_ts = None`。同样的最长
  公共前缀遍历就能产生它，只是把 yield 第二个事件换成了关闭一个字典条目；当目标是每个函数的总活动时长而
  不是一份原始事件日志时，这种表示读起来也更直接。

<details>
<summary>验证代码（可运行）</summary>

```python
import itertools
import random


# --- Part 1: the worked example ---
samples1 = [
    Sample(0.0, ["main"]),
    Sample(1.0, ["main", "solve"]),
    Sample(2.0, ["main", "solve", "solve"]),
    Sample(2.5, ["main", "solve", "solve"]),
    Sample(4.0, ["main", "solve", "solve", "solve"]),
    Sample(6.0, ["main", "solve", "helper"]),
]
expected1 = [
    Event("start", 0.0, "main", 0), Event("start", 1.0, "solve", 1), Event("start", 2.0, "solve", 2),
    Event("start", 4.0, "solve", 3), Event("end", 6.0, "solve", 3), Event("end", 6.0, "solve", 2),
    Event("start", 6.0, "helper", 2), Event("end", 9.0, "helper", 2), Event("end", 9.0, "solve", 1),
    Event("end", 9.0, "main", 0),
]
assert convert_samples_to_events(samples1, 9.0) == expected1
assert convert_samples_to_events([], 5.0) == []

# --- Part 2: the worked examples, including the prefix-preserved / prefix-reset corner case ---
assert list(stream_debounced_events(
    [Sample(1, ["req", "validate"]), Sample(2, ["req", "validate", "persist"])], 1)) == [
    Event("start", 1, "req", 0), Event("start", 1, "validate", 1), Event("start", 2, "persist", 2)]
assert list(stream_debounced_events(
    [Sample(1, ["req", "validate"]), Sample(2, ["worker", "validate", "req"])], 1)) == [
    Event("start", 1, "req", 0), Event("start", 1, "validate", 1),
    Event("end", 2, "validate", 1), Event("end", 2, "req", 0),
    Event("start", 2, "worker", 0), Event("start", 2, "validate", 1), Event("start", 2, "req", 2)]

samples2 = [Sample(1, ["svc"]), Sample(2, ["svc"]), Sample(3, ["svc", "handle"]), Sample(4, ["svc"]),
            Sample(5, ["svc", "flush"]), Sample(6, ["svc", "flush"]), Sample(7, ["svc"]),
            Sample(8, ["svc"]), Sample(9, ["svc"])]
assert list(stream_debounced_events(samples2, 2)) == [
    Event("start", 2, "svc", 0), Event("start", 6, "flush", 1), Event("end", 8, "flush", 1)]
assert list(stream_debounced_events(iter([]), 3)) == []

# --- Part 3: the worked example ---
traces3 = [
    ("T1", ["boot", "ingest", "parse", "oom"]), ("T2", ["worker", "parse", "oom"]),
    ("T3", ["boot", "cache", "oom"]), ("T4", ["boot", "ingest", "oom"]),
]
index3 = TraceIndex(traces3)
for query, expected in [(["parse", "oom"], ["T1", "T2"]), (["oom"], ["T1", "T2", "T3", "T4"]),
                        (["ingest", "oom"], ["T4"]), (["boot", "ingest", "parse", "oom"], ["T1"]),
                        (["cache", "parse", "oom"], "UNKNOWN")]:
    assert index3.match_suffix(query) == expected, (query, index3.match_suffix(query))


# --- Part 1: an independent reading of the rule (whole-stack comparison via itertools.takewhile, not
# _common_prefix_len), plus a round-trip replay that checks the events reproduce every sampled stack ---
def _events_brute_force(samples: list[Sample], end_time: float) -> list[Event]:
    if not samples:
        return []
    events: list[Event] = []
    bounds = [samples[0].ts] + [s.ts for s in samples[1:]] + [end_time]
    stacks = [[]] + [s.stack for s in samples]
    for prev_stack, cur_stack, ts in zip(stacks, stacks[1:], bounds):
        lcp = len(list(itertools.takewhile(lambda pair: pair[0] == pair[1], zip(prev_stack, cur_stack))))
        for depth in reversed(range(lcp, len(prev_stack))):
            events.append(Event("end", ts, prev_stack[depth], depth))
        for depth in range(lcp, len(cur_stack)):
            events.append(Event("start", ts, cur_stack[depth], depth))
    for depth in reversed(range(len(samples[-1].stack))):
        events.append(Event("end", end_time, samples[-1].stack[depth], depth))
    return events


def _replay_and_check(samples: list[Sample], end_time: float, events: list[Event]) -> None:
    active: dict[int, str] = {}
    i = 0
    for sample in samples:
        while i < len(events) and events[i].ts <= sample.ts:
            e = events[i]
            if e.kind == "start":
                assert e.depth not in active, (sample.ts, e)
                active[e.depth] = e.name
            else:
                assert active.get(e.depth) == e.name, (sample.ts, e, active)
                del active[e.depth]
            i += 1
        assert [active[d] for d in range(len(active))] == sample.stack, (sample.ts, active, sample.stack)
    while i < len(events):
        e = events[i]
        assert e.kind == "end" and active.get(e.depth) == e.name
        del active[e.depth]
        i += 1
    assert not active


def _random_stack_samples(rng: random.Random, n_samples: int, names: list[str]) -> list[Sample]:
    """Simulates a call stack with random push/pop (names can repeat at nested depths -- recursion) and
    takes n_samples snapshots at increasing integer timestamps."""
    stack: list[str] = []
    samples = []
    ts = 0
    for _ in range(n_samples):
        for _ in range(rng.randint(0, 3)):
            if stack and rng.random() < 0.5:
                stack.pop()
            else:
                stack.append(rng.choice(names))
        ts += rng.randint(1, 3)
        samples.append(Sample(float(ts), list(stack)))
    return samples


for seed in range(400):
    rng = random.Random(seed)
    samples = _random_stack_samples(rng, rng.randint(1, 12), ["f", "g", "h"])   # small alphabet -> frequent recursion
    end_time = samples[-1].ts + rng.randint(1, 5) if samples else 0.0
    got = convert_samples_to_events(samples, end_time)
    assert got == _events_brute_force(samples, end_time), seed
    _replay_and_check(samples, end_time, got)


# --- Part 2: an independent reading of the debounce rule -- finds every maximal run of consecutive
# samples during which one position holds one name (direct comparison, no shared helper), keeps only
# runs of at least k samples, and time-stamps each by its k-th sample ---
def _debounced_brute_force(samples: list[Sample], k: int) -> list[Event]:
    n = len(samples)
    runs = []                                  # (depth, name, start_idx, end_idx or None)
    open_runs: dict[int, tuple[str, int]] = {}
    prev_stack: list[str] = []
    for idx, sample in enumerate(samples):
        cur = sample.stack
        common = 0
        while common < len(prev_stack) and common < len(cur) and prev_stack[common] == cur[common]:
            common += 1
        for depth in range(len(prev_stack) - 1, common - 1, -1):
            name, start_idx = open_runs.pop(depth)
            runs.append((depth, name, start_idx, idx))
        for depth in range(common, len(cur)):
            open_runs[depth] = (cur[depth], idx)
        prev_stack = cur
    for depth, (name, start_idx) in open_runs.items():
        runs.append((depth, name, start_idx, None))

    events = []
    for depth, name, start_idx, end_idx in runs:
        length = (end_idx if end_idx is not None else n) - start_idx
        if length < k:
            continue
        events.append(Event("start", samples[start_idx + k - 1].ts, name, depth))
        if end_idx is not None and end_idx + k <= n:
            events.append(Event("end", samples[end_idx + k - 1].ts, name, depth))
    events.sort(key=lambda e: (e.ts, e.kind == "start", -e.depth if e.kind == "end" else e.depth))
    return events


for seed in range(400):
    rng = random.Random(1000 + seed)
    samples = _random_stack_samples(rng, rng.randint(1, 15), ["f", "g", "h"])
    k = rng.randint(1, 4)
    got = list(stream_debounced_events(iter(samples), k))          # iter(): exercises a true one-pass iterator
    assert got == _debounced_brute_force(samples, k), (seed, k)

# k = 1 reduces to Part 1's own transitions, minus the final end_time closing batch
for seed in range(150):
    rng = random.Random(2000 + seed)
    samples = _random_stack_samples(rng, rng.randint(1, 10), ["f", "g", "h"])
    if not samples:
        continue
    end_time = samples[-1].ts + 7
    full = convert_samples_to_events(samples, end_time)
    assert list(stream_debounced_events(iter(samples), 1)) == [e for e in full if e.ts != end_time], seed


# --- Part 3: an independent brute force that compares suffixes directly with slicing ---
def _match_suffix_brute_force(traces: list[tuple[str, list[str]]], frames: list[str]) -> list[str] | str:
    m = len(frames)
    matches = sorted(trace_id for trace_id, trace_frames in traces
                      if len(trace_frames) >= m and trace_frames[len(trace_frames) - m:] == frames)
    return matches if matches else "UNKNOWN"


for seed in range(300):
    rng = random.Random(3000 + seed)
    names = ["a", "b", "c"]                    # small alphabet -> frequent recursion and cross-trace collisions
    traces = [(f"id{i}", [rng.choice(names) for _ in range(rng.randint(1, 6))]) for i in range(rng.randint(1, 10))]
    index = TraceIndex(traces)
    for _ in range(8):
        if rng.random() < 0.7:                                       # a suffix of a real trace: exercises matches
            source = rng.choice(traces)[1]
        else:                                                         # an unrelated string: exercises UNKNOWN
            source = [rng.choice(names) for _ in range(rng.randint(1, 6))]
        m = rng.randint(1, len(source))
        query = source[len(source) - m:]
        assert index.match_suffix(query) == _match_suffix_brute_force(traces, query), (seed, query)

print("all checks passed")
```

</details>

</details>
