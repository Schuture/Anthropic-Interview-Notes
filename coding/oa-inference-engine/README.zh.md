# 在线测评：补全一个小型 LLM 推理引擎

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 在线测评 · 5 关 | ★☆☆☆☆ | 困难 | Fellows | llm-serving, scheduling, kv-cache, prefix-caching, oop-design | 5 关 / 90 分钟 | 在线测评（OA） |
<!-- meta:end -->

## 题目

推理引擎面向一个语言模型处理请求：每个请求提供一个*提示*（prompt，一串 token id）和一个生成数量上限，
引擎返回模型在其后生成的 token。下面这段代码已经存在，不能改动；它的注释很稀少，
读懂它——每个调用做什么、返回什么、已经假定发生了什么——本身就是任务的一部分。它定义了一个语言模型的
确定性替身、一个基于块的内存分配器、引擎对一个请求的记录，以及一个队列。按顺序实现随后的五个关卡，全部
作为同一个 `Engine` 类的方法：只有当前一关自己的测试通过后，下一关的测试才会开始运行，且前面每一关声明
过的规则在之后每一关都继续成立，再加上那一关新增的规则。

```python
from collections import deque, OrderedDict
from enum import Enum
from typing import Optional

VOCAB_SIZE = 17          # token ids are 0 .. VOCAB_SIZE - 1
EOS_TOKEN = 16            # generating this token ends a sequence
BLOCK_SIZE = 4            # tokens held by one KV-cache block


def next_token(token_ids: list[int]) -> int:
    """Stand-in for a language model's forward pass: the next token is a fixed hash of the whole
    token sequence so far (the prompt and everything generated after it). Deterministic, needs no
    weights. token_ids must be non-empty."""
    h = 0
    for t in token_ids:
        h = (h * 1_000_003 + t + 1) % VOCAB_SIZE
    return h


class OutOfBlocks(Exception):
    """Raised by BlockAllocator.allocate when it cannot produce enough blocks, even after evicting
    every evictable cached block. Nothing about the allocator's state changes when this is raised."""


class Block:
    """One fixed-size slot of KV-cache memory. ref_count is how many sequences currently depend on
    this block's contents. content_hash is set once the block is registered for prefix caching
    (None until then, and again once the block is evicted)."""
    __slots__ = ("block_id", "ref_count", "content_hash")

    def __init__(self, block_id: int):
        self.block_id = block_id
        self.ref_count = 0
        self.content_hash: Optional[int] = None


class BlockAllocator:
    """Owns all num_blocks physical KV-cache blocks. Every block is in exactly one of three states:
    FREE (ref_count 0, content_hash None), CACHED (ref_count 0, content_hash set -- unused right now
    but still holding valid content a future request could reuse), or LIVE (ref_count >= 1)."""

    def __init__(self, num_blocks: int):
        self.blocks = {i: Block(i) for i in range(num_blocks)}
        self._free = list(range(num_blocks - 1, -1, -1))   # pop() hands out block 0 first
        self._cached_by_hash: dict[int, int] = {}            # content_hash -> block_id, every registered block
        self._lru: "OrderedDict[int, None]" = OrderedDict()  # CACHED block ids, least-recently-freed first

    def num_free_or_evictable(self) -> int:
        """Blocks obtainable without preempting a sequence: the free list plus every CACHED block."""
        return len(self._free) + len(self._lru)

    def allocate(self, count: int) -> list[int]:
        """Returns count fresh block ids with ref_count set to 1, evicting the least-recently-freed
        CACHED blocks first if the free list alone is not enough. Raises OutOfBlocks, with no effect,
        if count > num_free_or_evictable()."""
        if count > self.num_free_or_evictable():
            raise OutOfBlocks(f"need {count}, have {self.num_free_or_evictable()}")
        while len(self._free) < count:
            self._evict_one()
        ids = [self._free.pop() for _ in range(count)]
        for block_id in ids:
            self.blocks[block_id].ref_count = 1
        return ids

    def _evict_one(self) -> None:
        block_id, _ = self._lru.popitem(last=False)
        block = self.blocks[block_id]
        del self._cached_by_hash[block.content_hash]
        block.content_hash = None
        self._free.append(block_id)

    def incref(self, block_id: int) -> None:
        """Adds one reference to a block that is CACHED or already LIVE (removing it from the LRU
        list in the former case -- a block referenced by any sequence is LIVE, never CACHED)."""
        block = self.blocks[block_id]
        if block.ref_count == 0:
            del self._lru[block_id]
        block.ref_count += 1

    def free(self, block_ids: list[int]) -> None:
        """Removes one reference from each block. A block whose ref_count reaches 0 becomes CACHED
        if it has a content_hash, or FREE otherwise."""
        for block_id in block_ids:
            block = self.blocks[block_id]
            block.ref_count -= 1
            if block.ref_count == 0:
                if block.content_hash is None:
                    self._free.append(block_id)
                else:
                    self._lru[block_id] = None   # inserted at the recently-freed end

    def lookup_cached(self, content_hash: int) -> Optional[int]:
        """Returns the id of the CACHED or LIVE block already holding this content_hash, first
        calling incref on it -- or None, with no effect, if no block holds it."""
        block_id = self._cached_by_hash.get(content_hash)
        if block_id is not None:
            self.incref(block_id)
        return block_id

    def register(self, block_id: int, content_hash: int) -> None:
        """Marks the LIVE, not-yet-hashed block block_id as eligible for future reuse under
        content_hash; it stays LIVE (ref_count untouched) until every sequence referencing it frees
        it, at which point it becomes CACHED instead of FREE. Does nothing if content_hash is already
        registered to a different block (two sequences produced the same block content independently;
        the existing entry is kept and block_id stays unhashed)."""
        if content_hash in self._cached_by_hash:
            return
        block = self.blocks[block_id]
        block.content_hash = content_hash
        self._cached_by_hash[content_hash] = block_id


def block_hash(prev_hash: int, block_tokens: list[int]) -> int:
    """Content hash of one full block, chained from the hash of the block before it in the same
    sequence (0 for a sequence's first block). Two blocks hash equal iff every token from the start
    of their sequences through the end of this block is identical."""
    h = prev_hash
    for t in block_tokens:
        h = (h * 1_000_003 + t + 1) & 0xFFFFFFFF
    return h


def full_block_hashes(seq: "Sequence", num_full_blocks: int) -> list[int]:
    """The chained content hash of each of seq's first num_full_blocks full blocks, in order."""
    hashes = []
    h = 0
    for block_index in range(num_full_blocks):
        h = block_hash(h, seq.tokens_in_block(block_index))
        hashes.append(h)
    return hashes


class SeqStatus(Enum):
    WAITING = "waiting"      # queued; prefill has not run (or has not re-run since a preemption)
    RUNNING = "running"      # prefill has run; may still be decoding
    FINISHED = "finished"    # a stop condition has fired; every block has been freed


class Sequence:
    """The engine's-eye view of one request: its prompt, what it has generated, and which physical
    blocks currently hold its KV cache. token_ids is prompt_token_ids + every generated token, in
    order -- the only input next_token needs."""

    def __init__(self, request_id: int, prompt_token_ids: list[int], max_tokens: int):
        self.request_id = request_id
        self.prompt_token_ids = list(prompt_token_ids)
        self.max_tokens = max_tokens
        self.token_ids: list[int] = list(prompt_token_ids)
        self.num_generated = 0
        self.status = SeqStatus.WAITING
        self.block_ids: list[int] = []

    @property
    def generated_token_ids(self) -> list[int]:
        return self.token_ids[len(self.prompt_token_ids):]

    def tokens_in_block(self, block_index: int) -> list[int]:
        """The (up to BLOCK_SIZE) tokens belonging to this sequence's block_index-th block."""
        return self.token_ids[block_index * BLOCK_SIZE: (block_index + 1) * BLOCK_SIZE]

    def append_token(self, token_id: int) -> None:
        """Records a newly generated token. Does not touch block_ids or status -- the caller must
        already have made sure a block exists to hold it."""
        self.token_ids.append(token_id)
        self.num_generated += 1

    def is_stopped(self) -> bool:
        """True once max_tokens have been generated, or the last generated token is EOS_TOKEN."""
        return self.num_generated >= self.max_tokens or self.token_ids[-1] == EOS_TOKEN


class RequestQueue:
    """FIFO queue of waiting request ids, with push_front for a request that must be retried before
    anything else waiting."""

    def __init__(self):
        self._dq: deque[int] = deque()

    def push_back(self, request_id: int) -> None:
        self._dq.append(request_id)

    def push_front(self, request_id: int) -> None:
        self._dq.appendleft(request_id)

    def pop_front(self) -> int:
        return self._dq.popleft()

    def peek_front(self) -> Optional[int]:
        return self._dq[0] if self._dq else None

    def __len__(self) -> int:
        return len(self._dq)
```

构造一个 `Engine` 时传入 `num_blocks`（其 `BlockAllocator` 的大小），从 Level 2 起还要传入
`token_budget` 和 `max_seqs`（在那一关定义）。`add_request(prompt_token_ids, max_tokens)`
创建一个处于 `WAITING` 状态的 `Sequence` 并将其入队，返回它的 `request_id`——从 0 开始按调用顺序
连续编号。下面的每条规则都以 `step()` 来表述，它是引擎唯一的另一个公开方法。

*预填充*（prefill）一个序列，是指为它预留足够的块，并让它经过一次 `next_token`，把当前的 `token_ids`
变成多一个 token；对第一次被准入的序列来说，这会仅凭它的提示生成出第一个 token。*解码*（decode）一个
序列，是指让一个已经在运行的序列再经过一次 `next_token`，在它已有的内容上再加一个 token。这两种操作都
会恰好给 `token_ids` 追加一个 token，并且都必须在生成之*前*确保有块可以容纳它：为一个序列预留块时，
必须始终留出能容纳 `len(token_ids) + 1` 个 token 的空间，其中包含即将生成的那一个。这道题中，块本身
的数值内容从不被检查；重要的只是每个块是为哪些 token 预留的，以及一个块当前是否处于被预留状态。

### Level 1 —— 准入与预填充

实现 `add_request` 和 `step`。

```py
class Engine:
    def __init__(self, num_blocks: int) -> None:
        """A fresh engine over a BlockAllocator of num_blocks blocks, with no requests yet."""

    def add_request(self, prompt_token_ids: list[int], max_tokens: int) -> int:
        """Creates a WAITING Sequence for this request and enqueues it. Returns its request_id: 0
        for the first call, 1 for the second, and so on."""

    def step(self) -> None:
        """Prefills every currently waiting request, in queue order, and marks each RUNNING."""
```

任何请求的提示，加上它此后会生成的全部内容，都不会超过 `num_blocks * BLOCK_SIZE` 个 token；无论涉及
多少个请求，`num_blocks` 也总是足够容纳同时处于 `RUNNING` 状态的全部序列；直到第 4 关为止，`step`
都不需要处理块用尽的情况。

示例，`Engine(num_blocks=10)`：

```text
add_request([2, 9, 4, 7, 1], max_tokens=3)   -> 0     # 5 个 token 的提示
add_request([5, 3, 8], max_tokens=3)         -> 1     # 3 个 token 的提示
step()
sequences[0].block_ids            -> [0, 1]           # ceil((5 + 1) / 4) = 2 个块
sequences[0].generated_token_ids  -> [15]
sequences[1].block_ids            -> [2]               # ceil((3 + 1) / 4) = 1 个块
sequences[1].generated_token_ids  -> [3]
```

### Level 2 —— 带预算约束的解码循环

从这一关开始，`Engine.__init__` 多接受两个参数 `token_budget` 和 `max_seqs`，测试也不再只调用一次
`step()`，而是逐轮反复调用。重新定义 `step`，使得一次调用：

1. 给每个当前处于 `RUNNING` 状态的序列解码出多一个 token。
2. 然后从队列头部开始，逐个准入并预填充请求，只要 `RUNNING` 的序列数还少于 `max_seqs`，*并且*准入
   下一个请求不会让这一步处理的 token 数超过 `token_budget`——第 1 步中每解码一个序列记一个 token，
   第 2 步中每准入一个请求再加上它的 `len(prompt_token_ids)`。第一个会突破预算的等待请求会让这一步
   之后的准入全部停止，哪怕队列里后面还有一个单独看足够短、能够放得下的请求；不会有请求被跳过顺序提前
   准入。

```py
class Engine:
    def __init__(self, num_blocks: int, token_budget: int, max_seqs: int) -> None: ...
    def step(self) -> None: ...   # as described above
```

`token_budget` 保证至少等于每个请求自己的提示长度，并且至少等于 `max_seqs`。

示例，`Engine(num_blocks=20, token_budget=6, max_seqs=2)`，在第一次 `step()` 之前添加了三个请求：

```text
add_request([1, 2, 3, 4], max_tokens=5) -> 0   # A，4 个 token 的提示
add_request([5, 6, 7], max_tokens=5)    -> 1   # B，3 个 token 的提示
add_request([8, 9], max_tokens=5)       -> 2   # C，2 个 token 的提示

step()   # 解码：还没有正在运行的序列。准入：A 放得下 (0+4<=6)；B 放不下 (4+3=7>6)——停止
sequences[0].generated_token_ids -> [14]                    # A 被准入
sequences[1].generated_token_ids -> []                      # B、C 仍在等待

step()   # 解码：A -> 多一个 token。准入：B 现在放得下 (1+3<=6)；运行数达到 max_seqs=2——停止
sequences[0].generated_token_ids -> [14, 13]
sequences[1].generated_token_ids -> [4]                     # B 被准入
sequences[2].generated_token_ids -> []                      # C 仍在等待：它本来是放得下预算的

step()   # 解码：A、B 各自多一个 token。准入：运行数已经是 max_seqs=2——C 仍在等待
sequences[0].generated_token_ids -> [14, 13, 0]
sequences[1].generated_token_ids -> [4, 2]
sequences[2].generated_token_ids -> []
```

### Level 3 —— 停止条件与释放块

一个序列在其最近生成的 token 满足 `Sequence.is_stopped()` 的那一刻停止——不论这个 token 来自预填充
还是解码。停止时，`step` 会先释放该序列持有的每一个块（`block_ids` 变为空）并把状态设为 `FINISHED`，
然后才对它做其他任何事；一个已停止的序列不会再被解码，也不会再占用 `max_seqs` 的名额。

```py
class Engine:
    def step(self) -> None: ...   # as Level 2, and: stop and free immediately once is_stopped()
```

示例，`Engine(num_blocks=8, token_budget=20, max_seqs=5)`：

```text
add_request([2, 9, 4, 7, 1], max_tokens=2)  -> 0   # A：因达到 max_tokens 而停止
add_request([0, 3], max_tokens=5)           -> 1   # B：它生成的第一个 token 就是 EOS_TOKEN（16）
add_request([6, 6, 6], max_tokens=4)        -> 2   # C：作为对照，持续运行

step()
A: RUNNING,  blocks [0, 1], generated [15]
B: FINISHED, blocks [],     generated [16]          # 预填充期间就遇到 EOS：从未进入 RUNNING
C: RUNNING,  blocks [2],    generated [11]
8 - 2(A) - 1(C) = 5 个块空闲

step()
A: FINISHED, blocks [],     generated [15, 9]       # num_generated 达到 max_tokens=2：释放 [0, 1]
C: RUNNING,  blocks [2, 1], generated [11, 8]        # token 数达到 4：需要第二个块
6 个块空闲

step(); step()                                       # 再运行两轮
C: FINISHED, blocks [], generated [11, 8, 3, 6]      # num_generated 达到 max_tokens=4
8 个块空闲——每个块都已归还给分配器
```

### Level 4 —— 前缀缓存

从这一关开始，预填充会复用已有的块，而不是总是分配全新的块。两个块在从 0 开始链式计算的 `block_hash`
下取值相等，当且仅当从各自序列开头到该块结尾为止的每一个 token 都完全相同——这就是它们的*内容哈希*。

重新定义预填充预留块的方式：在分配任何东西之前，先用 `full_block_hashes` 算出该序列每个满块的内容
哈希，再从第 0 个块开始按顺序用 `BlockAllocator.lookup_cached` 逐个查找。在第一个还未被缓存的块下标
处停下，或者提示的每个满块都已经查过之后停下，此后的一切仍和之前一样分配全新的块。每当一个块第一次
恰好达到 `BLOCK_SIZE` 个 token——不论是在为提示预留全新的块时，还是之后在解码期间——且它不是通过
缓存命中获得的，就把它按内容哈希注册起来，供之后的请求复用。一个被多个序列复用的块只会提高其引用
计数，不会发生任何复制；一个序列自己生成的 token 不会因为它的某个块恰好被共享而有任何变化。

```py
class Engine:
    def step(self) -> None: ...   # as Level 3, prefill's block reservation now cache-aware as above
```

示例，`Engine(num_blocks=4, token_budget=40, max_seqs=5)`：

```text
add_request([5, 5, 5, 5, 2], max_tokens=2)  -> 0   # A
add_request([5, 5, 5, 5, 9], max_tokens=4)  -> 1   # B：与 A 共享第一个块 [5, 5, 5, 5]

step()
A: blocks [0, 1], generated [12]
B: blocks [0, 2], generated [2]         # 块 0 是 A 的块，被复用，而不是复制
块 0 的 ref_count == 2

step()                                   # A 达到 max_tokens=2 并停止
A: FINISHED, blocks [], generated [12, 4]
B: RUNNING,  blocks [0, 2], generated [2, 10]
块 0 的 ref_count == 1                # 仍被 B 单独持有——被引用期间不会被驱逐

step(); step()                           # B 持续解码，然后在 max_tokens=4 处停止
B: FINISHED, blocks [], generated [2, 10, 12, 4]
块 0 现在是缓存态（ref_count 0，仍保存着 [5, 5, 5, 5]），而不是直接被释放

add_request([1, 1, 1, 1, 1, 1, 1, 1], max_tokens=1)  -> 2   # C：内容无关，需要 3 个块
step()                                                       # 只有 2 个空闲——驱逐缓存中的块 0
C: FINISHED, generated [12]

add_request([5, 5, 5, 5, 7], max_tokens=1)  -> 3   # D：第一个块与 A、B 相同
step()
D: FINISHED, generated [0]              # 块 0 原来的内容哈希已经不在了：这是未命中，不是命中
```

### Level 5 —— 抢占

从这一关开始，每当预留块——无论是为了预填充，还是为了多解码一个 token——会引发 `OutOfBlocks`，就通过
*抢占*正在运行的序列来腾出空间，而不是让这次预留失败：反复释放当前 `RUNNING` 的序列中准入时间最晚的
那一个所持有的每一个块，把它送回 `WAITING`，放在队列的*最前面*，直到这次预留能够成功为止。一个序列的
准入时间在它每次被准入时都会重新设置，包括被抢占后的再次准入，所以不会出现时间相同的情况；如果触发这
次查找的预留请求本身所属的序列，恰好就是准入时间最晚的那个 `RUNNING` 序列，它自己也可能成为被抢占的
对象。抢占不会丢弃一个序列的 `token_ids` 或 `num_generated`，只会丢弃它的块：一个被抢占的序列会像
任何其他等待中的请求一样，重新经过同样的准入与预填充流程，基于它当前（现在更长了）的 `token_ids`，
生成的下一个 token 与它从未停止过时完全一样，因为 `next_token` 只取决于 `token_ids`。

```py
class Engine:
    def step(self) -> None: ...   # as Level 4, every block reservation now preemption-aware as above
```

`token_budget` 保证对每个请求都至少等于 `len(prompt_token_ids) + max_tokens`，所以一个被抢占后已经
回到队列中的请求，只要空间腾出来了，就总能被重新准入。

示例，`Engine(num_blocks=3, token_budget=30, max_seqs=5)`：

```text
add_request([6, 6], max_tokens=5)        -> 0   # P：2 个 token 的提示
add_request([2, 9, 4, 7], max_tokens=5)  -> 1   # Q：4 个 token 的提示，在 P 之后被准入

step(); step()
P: RUNNING, blocks [0],    generated [6, 11]
Q: RUNNING, blocks [1, 2], generated [11, 8]
0 个块空闲——3 个全部在用

step()             # P 的第 3 个 token 需要第 2 个块；Q 是准入时间更晚的 RUNNING 序列
P: FINISHED, generated [6, 11, 8, 3, 6]           # 借助腾出的空间运行到完成
Q: WAITING,  generated [11, 8, 3, 6]              # 在距 max_tokens=5 还差一个 token 时被抢占

step()              # Q 被重新准入，它当前的 6 个 token 被重新预填充，并达到 max_tokens
Q: FINISHED, generated [11, 8, 3, 6, 11]
每个块都空闲或处于缓存态——没有泄漏
```

给定一个请求的提示和 `max_tokens`，把它放进一个足以单独容纳它、且从未添加过其他请求的 `Engine` 中
运行，得到的 `generated_token_ids` 与把它放进任何更繁忙的工作负载中运行完全相同：准入顺序、缓存与
抢占改变的只是使用了哪些物理块、以及何时使用，从不改变一个序列自己生成的 token。

## 参考解答

<details>
<summary>展开参考解答</summary>

值得事先确认的两点：一个已经在运行的序列的解码步骤，是否会因为预算不够而被跳过（这里假设永远不会，这
与一个真实调度器的做法一致——预算的大小总能覆盖每个运行中序列的一个 token，这也是为什么会给出
`token_budget >= max_seqs`）；以及一个被抢占的请求是保留在队列中原来的位置，还是被放到队尾（这里
选择放到最前面，这样一个反复受到内存压力的序列会先于任何还未开始的请求被重试，而不必冒无限期饥饿的
风险）。

### Level 1

`add_request` 只需要创建一个 `Sequence` 并把它的 id 入队；之后每一关的工作都放在 `step` 里。预填充
预留 `ceil((len(token_ids) + 1) / BLOCK_SIZE)` 个块——而不是 `ceil(len(token_ids) / BLOCK_SIZE)`，
后者在提示长度恰好是 `BLOCK_SIZE` 的倍数时会少算一个块，因为预填充马上就要追加一个 token，而这时
已预留的块已经没有空间容纳它了。

```python
class Engine:
    def __init__(self, num_blocks: int):
        self.allocator = BlockAllocator(num_blocks)
        self.queue = RequestQueue()
        self.sequences: dict[int, Sequence] = {}
        self.running: dict[int, Sequence] = {}   # insertion order == admission order, used from Level 5
        self._next_id = 0

    def add_request(self, prompt_token_ids: list[int], max_tokens: int) -> int:
        request_id = self._next_id
        self._next_id += 1
        seq = Sequence(request_id, prompt_token_ids, max_tokens)
        self.sequences[request_id] = seq
        self.queue.push_back(request_id)
        return request_id

    def _prefill(self, seq: Sequence) -> None:
        # NOTE: +1 -- room for the token this call is about to produce, not just for token_ids as it is now
        needed = (len(seq.token_ids) + 1 + BLOCK_SIZE - 1) // BLOCK_SIZE
        seq.block_ids.extend(self.allocator.allocate(needed - len(seq.block_ids)))
        seq.append_token(next_token(seq.token_ids))
        seq.status = SeqStatus.RUNNING
        self.running[seq.request_id] = seq

    def step(self) -> None:
        while len(self.queue) > 0:
            self._prefill(self.sequences[self.queue.pop_front()])
```

`add_request` 是 $O(1)$；`step` 是 $O(r)$，$r$ 是当前等待中的请求数——此时还不会驱逐任何块，
每次预填充自己的分配也是 $O(1)$。

### Level 2

解码会无条件地给每个正在运行的序列一个 token，所以 `used` 从正在运行的序列数开始计（每个序列一个
token，由即将运行的解码循环预先花掉），此后只会随着
准入被批准而增长；如果改成在解码循环*之后*再从 `len(self.running)` 重新计算，在这里还不会重复计数，
但一旦 Level 5 允许一个序列在循环中途从 `running` 里消失，这种做法就不再正确。

```python
class Engine(Engine):
    def __init__(self, num_blocks: int, token_budget: int = 10**9, max_seqs: int = 10**9):
        super().__init__(num_blocks)
        self.token_budget = token_budget
        self.max_seqs = max_seqs

    def _decode_step(self, seq: Sequence) -> None:
        if len(seq.token_ids) % BLOCK_SIZE == 0:            # its last block is exactly full
            seq.block_ids.extend(self.allocator.allocate(1))
        seq.append_token(next_token(seq.token_ids))

    def step(self) -> None:
        used = len(self.running)                             # NOTE: one token per running sequence, spent up front
        for request_id in list(self.running):                # NOTE: snapshot -- see Level 5
            seq = self.running.get(request_id)
            if seq is not None:
                self._decode_step(seq)

        while len(self.running) < self.max_seqs and len(self.queue) > 0:
            seq = self.sequences[self.queue.peek_front()]
            if used + len(seq.token_ids) > self.token_budget:   # NOTE: FIFO -- stop, do not skip to a smaller one
                break
            self.queue.pop_front()
            used += len(seq.token_ids)
            self._prefill(seq)
```

`token_budget` 和 `max_seqs` 默认取一个实际上无限大的值，这样 Level 1 那些只构造 `Engine(num_blocks)`
的示例，在这一关以及之后的每一关都能不做任何改动地继续通过。`step` 每次调用的开销是 $O(m)$，$m$ 是
被解码的序列数（受 `max_seqs` 限制）加上这一轮新准入的请求数。

### Level 3

一个序列的停止检查必须在*每一次*追加 token 的调用之后运行——预填充和解码都一样——所以它应该放在两者
共同调用的一个地方，而不是在各自内部重复一遍。

```python
class Engine(Engine):
    def _finish_if_stopped(self, seq: Sequence) -> bool:
        if not seq.is_stopped():
            return False
        self.allocator.free(seq.block_ids)
        seq.block_ids = []
        seq.status = SeqStatus.FINISHED
        self.running.pop(seq.request_id, None)
        return True

    def _prefill(self, seq: Sequence) -> None:
        needed = (len(seq.token_ids) + 1 + BLOCK_SIZE - 1) // BLOCK_SIZE
        seq.block_ids.extend(self.allocator.allocate(needed - len(seq.block_ids)))
        seq.append_token(next_token(seq.token_ids))
        if not self._finish_if_stopped(seq):                # NOTE: only reaches RUNNING if it did not just stop
            seq.status = SeqStatus.RUNNING
            self.running[seq.request_id] = seq

    def _decode_step(self, seq: Sequence) -> None:
        if len(seq.token_ids) % BLOCK_SIZE == 0:
            seq.block_ids.extend(self.allocator.allocate(1))
        seq.append_token(next_token(seq.token_ids))
        self._finish_if_stopped(seq)
```

`step` 没有变化：它本来就只遍历 `self.running`，而 `_finish_if_stopped` 现在会让这个集合保持最新。
`_finish_if_stopped` 自己只增加 $O(1)$ 的摊还开销（释放至多几个块），所以 `step` 的开销和 Level 2
一样，仍是 $O(m)$。

### Level 4

两个不同的序列完全可能合法地产生相同的块内容——最简单的情形就是两个请求共享同一个提示——而 `register`
的约定（写在它的文档字符串里）是在这种情况发生时只保留最早的那次注册。对每一个刚满的块都调用它，让它
自己去决定即可；这里不需要另外检测冲突。`full_block_hashes` 已经给出，所以唯一要新增的逻辑，是*在
哪里*查找每个哈希、以及*何时*停止查找。

```python
class Engine(Engine):
    def _register_if_newly_full(self, seq: Sequence) -> None:
        if len(seq.token_ids) % BLOCK_SIZE == 0:
            block_index = len(seq.token_ids) // BLOCK_SIZE - 1
            block_hash_ = full_block_hashes(seq, block_index + 1)[-1]
            self.allocator.register(seq.block_ids[block_index], block_hash_)

    def _prefill(self, seq: Sequence) -> None:
        num_full = len(seq.token_ids) // BLOCK_SIZE
        hashes = full_block_hashes(seq, num_full)
        hits = 0
        while hits < num_full:                                # NOTE: stop at the first miss -- see Follow-ups
            cached_id = self.allocator.lookup_cached(hashes[hits])
            if cached_id is None:
                break
            seq.block_ids.append(cached_id)                    # NOTE: lookup_cached already increfs -- no incref here
            hits += 1
        needed = (len(seq.token_ids) + 1 + BLOCK_SIZE - 1) // BLOCK_SIZE
        fresh = self.allocator.allocate(needed - hits)
        seq.block_ids.extend(fresh)
        for offset, block_id in enumerate(fresh):
            if hits + offset < num_full:                       # NOTE: never register the trailing partial block
                self.allocator.register(block_id, hashes[hits + offset])
        seq.append_token(next_token(seq.token_ids))
        self._register_if_newly_full(seq)                      # NOTE: the prompt's own tail can complete right here
        if not self._finish_if_stopped(seq):
            seq.status = SeqStatus.RUNNING
            self.running[seq.request_id] = seq

    def _decode_step(self, seq: Sequence) -> None:
        if len(seq.token_ids) % BLOCK_SIZE == 0:
            seq.block_ids.extend(self.allocator.allocate(1))
        seq.append_token(next_token(seq.token_ids))
        self._register_if_newly_full(seq)
        self._finish_if_stopped(seq)
```

缓存改变的始终只是某个位置的 KV 缓存*存放在哪个物理块里*，从不改变一个序列自己的 `token_ids`：
`lookup_cached` 和 `allocate` 返回的 id，其对应的槽位都保证恰好保存着正确的内容（命中时是由构造
过程保证的，未命中时是刚刚写入的），所以再往后的代码都不需要知道究竟发生的是哪一种。

预填充的缓存查找开销是 $O(f)$，$f$ 是该序列的满块数；`_register_if_newly_full` 每次调用都会把
直到新块为止的整条哈希链重新算一遍，开销同样是 $O(f)$，而不是增量维护一个运行哈希所能达到的
$O(1)$。

### Level 5

`_allocate_or_preempt` 不断重试，直到分配器能够满足这次请求，每次都丢弃准入时间最晚的那个运行中
序列——`self.running` 是一个普通的 `dict`，插入顺序就是准入顺序，所以这个序列就是
`next(reversed(self.running))`。这个循环总会终止：每次失败的尝试都会从 `running` 中移走一个序列，
所以最多 `len(self.running)` 轮之后，要么某次尝试成功，要么 `running` 变空——而一个空引擎的整个
池子都是空闲的，第 1 关的容量保证——任何请求的总需求都不会超过 `num_blocks`——确保这总是足够满足
再多一个序列的这次预留。

```python
class Engine(Engine):
    def _allocate_or_preempt(self, count: int) -> list[int]:
        while True:
            try:
                return self.allocator.allocate(count)
            except OutOfBlocks:
                self._preempt(next(reversed(self.running)))   # NOTE: may be the very sequence asking for room

    def _preempt(self, request_id: int) -> None:
        seq = self.running.pop(request_id)
        self.allocator.free(seq.block_ids)
        seq.block_ids = []
        seq.status = SeqStatus.WAITING
        self.queue.push_front(request_id)

    def _prefill(self, seq: Sequence) -> None:
        num_full = len(seq.token_ids) // BLOCK_SIZE
        hashes = full_block_hashes(seq, num_full)
        hits = 0
        while hits < num_full:
            cached_id = self.allocator.lookup_cached(hashes[hits])
            if cached_id is None:
                break
            seq.block_ids.append(cached_id)
            hits += 1
        needed = (len(seq.token_ids) + 1 + BLOCK_SIZE - 1) // BLOCK_SIZE
        fresh = self._allocate_or_preempt(needed - hits)   # NOTE: seq is never its own victim here -- it is not
        seq.block_ids.extend(fresh)                          #  in self.running until this call returns
        for offset, block_id in enumerate(fresh):
            if hits + offset < num_full:
                self.allocator.register(block_id, hashes[hits + offset])
        seq.append_token(next_token(seq.token_ids))
        self._register_if_newly_full(seq)
        if not self._finish_if_stopped(seq):
            seq.status = SeqStatus.RUNNING
            self.running[seq.request_id] = seq

    def _decode_step(self, seq: Sequence) -> None:
        if len(seq.token_ids) % BLOCK_SIZE == 0:
            fresh = self._allocate_or_preempt(1)
            if seq.request_id not in self.running:      # NOTE: seq preempted itself while making its own room
                self.allocator.free(fresh)                #  -- it is WAITING now; hand the block straight back
                return
            seq.block_ids.append(fresh[0])
        seq.append_token(next_token(seq.token_ids))
        self._register_if_newly_full(seq)
        self._finish_if_stopped(seq)
```

`_decode_step` 可能被要求去延伸一个序列，而这个序列恰好就是为了给它自己腾地方而被驱逐的那一个——
即使只有一个序列在运行，按准入时间最晚来选择运行中序列，依然是一个定义明确的选择。跳过这次追加、把
多出来的块还回去，不会留下任何痕迹：这个序列已经回到了队列中，`token_ids` 和 `num_generated` 都
原封未动，它下一次尝试会像任何其他准入一样经过 `_prefill`。`step`、`_finish_if_stopped`，以及
`step` 后半段的准入循环，相对 Level 3 都没有变化：一个被抢占的序列在解码中途从 `self.running` 中
消失，正是 `step` 里先取快照、再用 `.get` 这种写法本来就准备容忍的情况。

每次调用 `_allocate_or_preempt` 的开销是 $O(v)$，$v$ 是它在自己的分配成功之前驱逐的序列数，
再加上分配本身 $O(1)$ 的摊还开销。

### 追问

- 分块预填充：这道题只有在一个请求的整个提示都能放进当前这一步剩余的预算时，才会准入它。一个生产
  环境的引擎则会把一个很长的提示拆成若干个有上限的 token 块，分几步预填充完，这样一个很长的提示就
  不会拖住其他每个序列的解码步骤，直到它被完整准入为止。
- 用交换代替重新计算：这里的抢占总是丢弃受害者的块，之后仅凭 `token_ids` 重新生成它们。交换则是把
  一个被抢占序列的块内容复制到宿主内存中，之后再复制回来，用一次数据传输换取本可避免的重新计算——
  当一个序列的前缀长到重新计算的代价超过复制的代价时，这就是值得的。
- 另一种受害者策略：抢占准入时间最晚的运行中序列，能限制任何一个序列被饿死的时长，因为只要还有更新
  的序列存在，运行时间最长的那个就永远不会被选中。改为抢占持有块数最多的运行中序列，则能让每次抢占
  腾出更多空间，代价是调用方更难预测结果。
- 部分块缓存：一条缓存记录始终只覆盖一个满块，所以两个提示即使只有最后几个 token 不同，也仍会共享
  它们之前的每一个块，但绝不会共享最后那个不满的块。缓存不足一个块的内容，就意味着把一个块交给某个
  序列，而另一个后续会分叉的序列可能仍会写入这个块。
- 并发：这里的 `step()` 总是运行到完成之后，才会有别的东西碰到这个引擎。一个在某次 `step` 进行中
  仍会接收新请求的服务，需要用一把锁把 `queue`、`running` 和分配器一起保护起来，因为准入一个请求或
  抢占一个序列，都会改变这同一次 `step` 自己的循环仍在读取的状态。

<details>
<summary>验证代码（可运行）</summary>

```python
# ---- Level 1 example ----
eng = Engine(num_blocks=10)
assert eng.add_request([2, 9, 4, 7, 1], max_tokens=3) == 0
assert eng.add_request([5, 3, 8], max_tokens=3) == 1
eng.step()
assert eng.sequences[0].block_ids == [0, 1] and eng.sequences[0].generated_token_ids == [15]
assert eng.sequences[1].block_ids == [2] and eng.sequences[1].generated_token_ids == [3]

# ---- Level 2 example ----
eng = Engine(num_blocks=20, token_budget=6, max_seqs=2)
a = eng.add_request([1, 2, 3, 4], max_tokens=5)
b = eng.add_request([5, 6, 7], max_tokens=5)
c = eng.add_request([8, 9], max_tokens=5)
eng.step()
assert eng.sequences[a].generated_token_ids == [14]
assert eng.sequences[b].generated_token_ids == [] and eng.sequences[c].generated_token_ids == []
eng.step()
assert eng.sequences[a].generated_token_ids == [14, 13]
assert eng.sequences[b].generated_token_ids == [4]
assert eng.sequences[c].generated_token_ids == []
eng.step()
assert eng.sequences[a].generated_token_ids == [14, 13, 0]
assert eng.sequences[b].generated_token_ids == [4, 2]
assert eng.sequences[c].generated_token_ids == []

# ---- Level 3 example ----
eng = Engine(num_blocks=8, token_budget=20, max_seqs=5)
a = eng.add_request([2, 9, 4, 7, 1], max_tokens=2)
b = eng.add_request([0, 3], max_tokens=5)
c = eng.add_request([6, 6, 6], max_tokens=4)
eng.step()
assert eng.sequences[a].status == SeqStatus.RUNNING and eng.sequences[a].block_ids == [0, 1]
assert eng.sequences[a].generated_token_ids == [15]
assert eng.sequences[b].status == SeqStatus.FINISHED and eng.sequences[b].block_ids == []
assert eng.sequences[b].generated_token_ids == [16]
assert eng.sequences[c].status == SeqStatus.RUNNING and eng.sequences[c].block_ids == [2]
assert eng.sequences[c].generated_token_ids == [11]
assert eng.allocator.num_free_or_evictable() == 5
eng.step()
assert eng.sequences[a].status == SeqStatus.FINISHED and eng.sequences[a].block_ids == []
assert eng.sequences[a].generated_token_ids == [15, 9]
assert eng.sequences[c].block_ids == [2, 1] and eng.sequences[c].generated_token_ids == [11, 8]
assert eng.allocator.num_free_or_evictable() == 6
eng.step(); eng.step()
assert eng.sequences[c].status == SeqStatus.FINISHED
assert eng.sequences[c].generated_token_ids == [11, 8, 3, 6]
assert eng.allocator.num_free_or_evictable() == 8

# ---- Level 4 example ----
eng = Engine(num_blocks=4, token_budget=40, max_seqs=5)
a = eng.add_request([5, 5, 5, 5, 2], max_tokens=2)
b = eng.add_request([5, 5, 5, 5, 9], max_tokens=4)
eng.step()
assert eng.sequences[a].block_ids == [0, 1] and eng.sequences[a].generated_token_ids == [12]
assert eng.sequences[b].block_ids == [0, 2] and eng.sequences[b].generated_token_ids == [2]
assert eng.allocator.blocks[0].ref_count == 2
eng.step()
assert eng.sequences[a].status == SeqStatus.FINISHED and eng.sequences[a].generated_token_ids == [12, 4]
assert eng.sequences[b].status == SeqStatus.RUNNING and eng.sequences[b].block_ids == [0, 2]
assert eng.sequences[b].generated_token_ids == [2, 10]
assert eng.allocator.blocks[0].ref_count == 1
eng.step(); eng.step()
assert eng.sequences[b].status == SeqStatus.FINISHED
assert eng.sequences[b].generated_token_ids == [2, 10, 12, 4]
assert eng.allocator.blocks[0].ref_count == 0 and 0 in eng.allocator._lru
shared_hash = block_hash(0, [5, 5, 5, 5])
assert eng.allocator._cached_by_hash[shared_hash] == 0

c = eng.add_request([1, 1, 1, 1, 1, 1, 1, 1], max_tokens=1)
eng.step()
assert eng.sequences[c].status == SeqStatus.FINISHED and eng.sequences[c].generated_token_ids == [12]
assert shared_hash not in eng.allocator._cached_by_hash        # evicted to make room for C's 3 blocks

d = eng.add_request([5, 5, 5, 5, 7], max_tokens=1)
eng.step()
assert eng.sequences[d].generated_token_ids == [0]              # a miss, not a hit: the old hash is gone
assert shared_hash in eng.allocator._cached_by_hash             # D's own first block re-registers it fresh

# ---- Level 5 example ----
eng = Engine(num_blocks=3, token_budget=30, max_seqs=5)
p = eng.add_request([6, 6], max_tokens=5)
q = eng.add_request([2, 9, 4, 7], max_tokens=5)
eng.step(); eng.step()
assert eng.sequences[p].status == SeqStatus.RUNNING and eng.sequences[p].block_ids == [0]
assert eng.sequences[p].generated_token_ids == [6, 11]
assert eng.sequences[q].status == SeqStatus.RUNNING and eng.sequences[q].block_ids == [1, 2]
assert eng.sequences[q].generated_token_ids == [11, 8]
assert eng.allocator.num_free_or_evictable() == 0
eng.step()
assert eng.sequences[p].status == SeqStatus.FINISHED
assert eng.sequences[p].generated_token_ids == [6, 11, 8, 3, 6]
assert eng.sequences[q].status == SeqStatus.WAITING
assert eng.sequences[q].generated_token_ids == [11, 8, 3, 6]
eng.step()
assert eng.sequences[q].status == SeqStatus.FINISHED
assert eng.sequences[q].generated_token_ids == [11, 8, 3, 6, 11]
assert eng.allocator.num_free_or_evictable() == 3
print("all five levels' worked examples replayed exactly")
```

```python
# ---- independent reference model + randomized cross-validation ----
import random


def naive_generate(prompt_token_ids: list[int], max_tokens: int) -> list[int]:
    """What one request generates run entirely on its own -- no queueing, no shared blocks, no
    eviction, nothing but next_token called in a loop. The invariant the whole problem rests on:
    batching, caching and preemption change only which physical blocks are used and when, never
    this."""
    token_ids = list(prompt_token_ids)
    generated: list[int] = []
    while len(generated) < max_tokens and (not generated or generated[-1] != EOS_TOKEN):
        token = next_token(token_ids)
        token_ids.append(token)
        generated.append(token)
    return generated


def _check_invariants(eng: "Engine") -> None:
    allocator = eng.allocator
    assert len(allocator._free) == len(set(allocator._free))   # no block ever freed twice
    free_set, lru_set = set(allocator._free), set(allocator._lru)
    assert not (free_set & lru_set)
    live_count = 0
    for block_id, block in allocator.blocks.items():
        if block.ref_count > 0:
            assert block_id not in free_set and block_id not in lru_set
            live_count += 1
        elif block.content_hash is not None:
            assert block_id in lru_set and allocator._cached_by_hash.get(block.content_hash) == block_id
        else:
            assert block_id in free_set
    assert len(free_set) + len(lru_set) + live_count == len(allocator.blocks)   # no block ever leaked


def _run_trial(seed: int) -> None:
    rng = random.Random(seed)
    max_prompt, max_gen = 6, 6
    min_num_blocks = -(-(max_prompt + max_gen) // BLOCK_SIZE)   # else even an empty pool could not fit one request
    num_blocks = rng.randint(min_num_blocks, min_num_blocks + 4)
    max_seqs = rng.randint(1, 4)
    token_budget = max(max_prompt + max_gen, max_seqs, rng.randint(4, 16))

    eng = Engine(num_blocks=num_blocks, token_budget=token_budget, max_seqs=max_seqs)
    expected = {}
    for _ in range(rng.randint(3, 12)):
        prompt = [rng.randrange(VOCAB_SIZE) for _ in range(rng.randint(1, max_prompt))]
        max_tokens = rng.randint(1, max_gen)
        request_id = eng.add_request(prompt, max_tokens)
        expected[request_id] = naive_generate(prompt, max_tokens)

    steps = 0
    while (len(eng.queue) or eng.running) and steps < 500:
        eng.step()
        _check_invariants(eng)
        steps += 1
    assert steps < 500, seed   # every trial is sized to finish; hitting the cap would mean a scheduling bug

    for request_id, want in expected.items():
        assert eng.sequences[request_id].generated_token_ids == want, (seed, request_id)
    _check_invariants(eng)


for seed in range(1000):
    _run_trial(seed)
print("cross-validated 1000 random workloads against an independent single-request reference, with no block leaks")
print("all checks passed")
```

</details>

</details>
