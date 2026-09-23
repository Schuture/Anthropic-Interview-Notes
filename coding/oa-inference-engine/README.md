# Online Assessment: Completing a Small LLM Inference Engine

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Online assessment · 5 levels | ★☆☆☆☆ | Hard | Fellows | llm-serving, scheduling, kv-cache, prefix-caching, oop-design | 5 levels / 90 min | Online assessment |
<!-- meta:end -->

## Problem

An inference engine serves requests against a language model: each request supplies a *prompt* (a
list of token ids) and a generation limit, and the engine returns the tokens the model generates
after it. The code below already exists and is not to be changed; it is sparsely commented, and
understanding it — what each call does, what it returns, and what it assumes has already happened —
is part of the task. It defines a deterministic stand-in for a language model, a
block-based memory allocator, the engine's record of one request, and a queue. Implement the five
levels that follow it, in order, as methods of one `Engine` class: a level's own tests must pass
before the next level's tests run, and every rule stated by an earlier level continues to hold at
every later one, in addition to whatever that level adds.

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

An `Engine` is constructed with `num_blocks` (the size of its `BlockAllocator`) and, from Level 2 on,
`token_budget` and `max_seqs` (defined there). `add_request(prompt_token_ids, max_tokens)` creates a
`Sequence` in `WAITING` status, enqueues it, and returns its `request_id` — consecutive integers
starting at 0, in call order. Every rule below is stated in terms of `step()`, the engine's only other
public method.

*Prefilling* a sequence means reserving it enough blocks and running it through `next_token` once to
turn its current `token_ids` into one more token; for a sequence admitted for the first time this
produces its first generated token, from its prompt alone. *Decoding* a sequence means running an
already-running sequence through `next_token` once more, adding one token to what it already has.
Both operations append exactly one token to `token_ids` and both must make sure a block exists to
hold it *before* generating it: reserving blocks for a sequence must always leave it enough room for
`len(token_ids) + 1` tokens, counting the one about to be produced. A block's own numeric content is
never inspected by this problem; only which tokens each block was reserved for, and whether a block
is currently reserved, ever matters.

### Level 1 — Admission and prefill

Implement `add_request` and `step`.

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

No request's prompt, together with everything it will ever generate, exceeds `num_blocks * BLOCK_SIZE`
tokens, and `num_blocks` is always enough to hold every sequence that is concurrently `RUNNING` at
once, however many requests are involved; through Level 4, `step` never has to handle running out of
blocks.

Example, `Engine(num_blocks=10)`:

```text
add_request([2, 9, 4, 7, 1], max_tokens=3)   -> 0     # 5-token prompt
add_request([5, 3, 8], max_tokens=3)         -> 1     # 3-token prompt
step()
sequences[0].block_ids            -> [0, 1]           # ceil((5 + 1) / 4) = 2 blocks
sequences[0].generated_token_ids  -> [15]
sequences[1].block_ids            -> [2]               # ceil((3 + 1) / 4) = 1 block
sequences[1].generated_token_ids  -> [3]
```

### Level 2 — A budget-constrained decode loop

From this level on, `Engine.__init__` takes two more arguments, `token_budget` and `max_seqs`, and the
tests call `step()` repeatedly, once per round, instead of once. Redefine `step` so that one call:

1. Gives every currently `RUNNING` sequence one more decoded token.
2. Then admits and prefills requests from the front of the queue, one at a time, for as long as fewer
   than `max_seqs` sequences are `RUNNING` *and* admitting the next one would not push the number of
   tokens processed this step over `token_budget` — counted as one token for every sequence decoded
   in step 1, plus `len(prompt_token_ids)` for every request admitted so far in step 2. The first
   waiting request that would break the budget stops admission for the rest of this step, even if a
   later request in the queue is short enough to fit on its own; nothing is admitted out of order.

```py
class Engine:
    def __init__(self, num_blocks: int, token_budget: int, max_seqs: int) -> None: ...
    def step(self) -> None: ...   # as described above
```

`token_budget` is guaranteed to be at least every request's own prompt length, and at least `max_seqs`.

Example, `Engine(num_blocks=20, token_budget=6, max_seqs=2)`, three requests added before the first
`step()`:

```text
add_request([1, 2, 3, 4], max_tokens=5) -> 0   # A, 4-token prompt
add_request([5, 6, 7], max_tokens=5)    -> 1   # B, 3-token prompt
add_request([8, 9], max_tokens=5)       -> 2   # C, 2-token prompt

step()   # decode: nothing running yet.  admit: A fits (0+4<=6); B does not (4+3=7>6) -- stop
sequences[0].generated_token_ids -> [14]                    # A admitted
sequences[1].generated_token_ids -> []                      # B, C still waiting

step()   # decode: A -> one more token.  admit: B now fits (1+3<=6); running reaches max_seqs=2 -- stop
sequences[0].generated_token_ids -> [14, 13]
sequences[1].generated_token_ids -> [4]                     # B admitted
sequences[2].generated_token_ids -> []                      # C still waits: it would have fit the budget

step()   # decode: A, B -> one more token each.  admit: running already at max_seqs=2 -- C still waits
sequences[0].generated_token_ids -> [14, 13, 0]
sequences[1].generated_token_ids -> [4, 2]
sequences[2].generated_token_ids -> []
```

### Level 3 — Stop conditions and freeing blocks

A sequence stops the moment its most recently generated token satisfies `Sequence.is_stopped()` —
whether that token came from prefill or from decode. On stopping, `step` frees every block the
sequence holds (`block_ids` becomes empty) and sets its status to `FINISHED` before doing anything
else with it; a stopped sequence is never decoded again and never counts against `max_seqs` again.

```py
class Engine:
    def step(self) -> None: ...   # as Level 2, and: stop and free immediately once is_stopped()
```

Example, `Engine(num_blocks=8, token_budget=20, max_seqs=5)`:

```text
add_request([2, 9, 4, 7, 1], max_tokens=2)  -> 0   # A: stops by max_tokens
add_request([0, 3], max_tokens=5)           -> 1   # B: its first generated token is EOS_TOKEN (16)
add_request([6, 6, 6], max_tokens=4)        -> 2   # C: for contrast, keeps running

step()
A: RUNNING,  blocks [0, 1], generated [15]
B: FINISHED, blocks [],     generated [16]          # EOS during its own prefill: never reaches RUNNING
C: RUNNING,  blocks [2],    generated [11]
8 - 2(A) - 1(C) = 5 blocks free

step()
A: FINISHED, blocks [],     generated [15, 9]       # num_generated reaches max_tokens=2: frees [0, 1]
C: RUNNING,  blocks [2, 1], generated [11, 8]        # token count reaches 4: needs a second block
6 blocks free

step(); step()                                       # two more rounds
C: FINISHED, blocks [], generated [11, 8, 3, 6]      # num_generated reaches max_tokens=4
8 blocks free — every block has returned to the allocator
```

### Level 4 — Prefix caching

From this level on, prefill reuses existing blocks instead of always allocating fresh ones. Two blocks
hash equal under `block_hash`, chained from 0, exactly when every token from the start of their
sequences through the end of that block is identical — their *content hash*.

Redefine prefill's block reservation: before allocating anything, compute the sequence's full blocks'
content hashes with `full_block_hashes` and look each one up with `BlockAllocator.lookup_cached`, in
order starting from block 0. Stop at the first block index that is not already cached, or once every
one of the prompt's full blocks has been checked, and reserve fresh blocks for everything from there
on, exactly as before. Whenever a block reaches exactly `BLOCK_SIZE` tokens for the first time —
whether while reserving fresh blocks for a prompt or later while decoding — and it was not already
obtained by a cache hit, register it under its content hash so a later request can reuse it. A block
reused by more than one sequence raises its reference count instead of copying anything; nothing about
a sequence's own generated tokens changes because one of its blocks happens to be shared.

```py
class Engine:
    def step(self) -> None: ...   # as Level 3, prefill's block reservation now cache-aware as above
```

Example, `Engine(num_blocks=4, token_budget=40, max_seqs=5)`:

```text
add_request([5, 5, 5, 5, 2], max_tokens=2)  -> 0   # A
add_request([5, 5, 5, 5, 9], max_tokens=4)  -> 1   # B: shares A's first block, [5, 5, 5, 5]

step()
A: blocks [0, 1], generated [12]
B: blocks [0, 2], generated [2]         # block 0 is A's block, reused, not copied
block 0's ref_count == 2

step()                                   # A reaches max_tokens=2 and stops
A: FINISHED, blocks [], generated [12, 4]
B: RUNNING,  blocks [0, 2], generated [2, 10]
block 0's ref_count == 1                # still held by B alone -- not evicted while referenced

step(); step()                           # B keeps decoding, then stops at max_tokens=4
B: FINISHED, blocks [], generated [2, 10, 12, 4]
block 0 is now cached (ref_count 0, still holding [5, 5, 5, 5]) instead of freed outright

add_request([1, 1, 1, 1, 1, 1, 1, 1], max_tokens=1)  -> 2   # C: unrelated content, needs 3 blocks
step()                                                       # only 2 are free -- evicts the cached block 0
C: FINISHED, generated [12]

add_request([5, 5, 5, 5, 7], max_tokens=1)  -> 3   # D: same first block as A and B
step()
D: FINISHED, generated [0]              # block 0's old content hash is gone: a miss, not a hit
```

### Level 5 — Preemption

From this level on, whenever reserving blocks — for a prefill or for one more decode token — would
raise `OutOfBlocks`, make room by *preempting* running sequences instead of letting the reservation
fail: repeatedly free every block held by the currently `RUNNING` sequence with the most recent
admission time, returning it to `WAITING` at the *front* of the queue, until the reservation would
succeed. A sequence's admission time is set every time it is admitted, including a re-admission after
being preempted, so ties never occur; the victim may be the very sequence whose own reservation
triggered the search, if it is itself the most recently admitted `RUNNING` sequence. Preemption never
discards a sequence's `token_ids` or `num_generated`, only its blocks: a preempted sequence resumes
through the same admission-and-prefill procedure as any other waiting request, over its current (now
longer) `token_ids`, and produces its next token exactly as if it had never stopped, since
`next_token` depends only on `token_ids`.

```py
class Engine:
    def step(self) -> None: ...   # as Level 4, every block reservation now preemption-aware as above
```

`token_budget` is guaranteed to be at least `len(prompt_token_ids) + max_tokens` for every request, so
a request already back in the queue after a preemption can always be re-admitted once room exists.

Example, `Engine(num_blocks=3, token_budget=30, max_seqs=5)`:

```text
add_request([6, 6], max_tokens=5)        -> 0   # P: 2-token prompt
add_request([2, 9, 4, 7], max_tokens=5)  -> 1   # Q: 4-token prompt, admitted after P

step(); step()
P: RUNNING, blocks [0],    generated [6, 11]
Q: RUNNING, blocks [1, 2], generated [11, 8]
0 blocks free — all 3 in use

step()             # P's 3rd token needs a 2nd block; Q is the more recently admitted RUNNING sequence
P: FINISHED, generated [6, 11, 8, 3, 6]           # runs to completion using the freed room
Q: WAITING,  generated [11, 8, 3, 6]              # preempted one token short of max_tokens=5

step()              # Q is re-admitted, its current 6 tokens re-prefilled, and it reaches max_tokens
Q: FINISHED, generated [11, 8, 3, 6, 11]
every block is free or cached — none leaked
```

Given a request's prompt and `max_tokens`, running it through an `Engine` sized to admit it on its own,
with no other request ever added, produces the same `generated_token_ids` as running it inside any
busier workload: admission order, caching and preemption change only which physical blocks are used
and when, never a sequence's own generated tokens.

## Reference solution

<details>
<summary>Show the reference solution</summary>

Worth confirming up front: whether the decode step of an already-running sequence should ever be
skipped for lack of budget (assumed here never, matching a real scheduler sized so the budget can
always cover one token per running sequence, which is why `token_budget >= max_seqs` is given); and
whether a preempted request keeps its place in the queue or goes to the back (here the front, so a
sequence under repeated memory pressure is retried before anything that has not started yet, rather
than risking indefinite starvation).

### Level 1

`add_request` only has to create a `Sequence` and enqueue its id; every later level's work goes into
`step`. Prefilling reserves `ceil((len(token_ids) + 1) / BLOCK_SIZE)` blocks — not
`ceil(len(token_ids) / BLOCK_SIZE)`, which is one short whenever a prompt's length is itself a
multiple of `BLOCK_SIZE`, since prefill is about to append a token that the already-reserved blocks
then have no room for.

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

`add_request` is $O(1)$; `step` is $O(r)$ for $r$ currently waiting requests, since no block is ever
evicted yet and each prefill's own allocation is $O(1)$.

### Level 2

Decode gives every running sequence one token, unconditionally, so `used` starts at the number of
running sequences (one token each, already spent by the decode loop about to run) and only grows from
there as admissions are approved; recomputing it from `len(self.running)` *after* the decode loop,
instead, would double-count nothing here yet, but stops being correct once Level 5 lets a sequence
disappear from `running` mid-loop.

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

`token_budget` and `max_seqs` default to effectively unlimited so that Level 1's examples, which
construct `Engine(num_blocks)` alone, keep working unchanged against this and every later level.
`step` costs $O(m)$ per call, in the number of sequences decoded (bounded by `max_seqs`) plus the
requests newly admitted this round.

### Level 3

A sequence's stop check has to run after *every* call that appends a token — prefill's and decode's
alike — so it belongs in one place both call into, not duplicated in each.

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

`step` is unchanged: it already only iterates `self.running`, which `_finish_if_stopped` now keeps
current. `_finish_if_stopped` itself adds only $O(1)$ amortized work, freeing at most a handful of
blocks, so `step`'s cost stays $O(m)$, as in Level 2.

### Level 4

Two different sequences can legitimately produce the same block content — two requests sharing a
prompt, most simply — and `register`'s contract (stated in its docstring) is to keep only the first
registration when that happens. Calling it for every block that just became full, and letting it
decide, is enough; nothing here has to detect the collision itself. `full_block_hashes` is given, so
the only new logic is *where* to look each hash up and *when* to stop looking.

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

Caching only ever changes *which physical block* a position's KV cache lives in, never a sequence's
own `token_ids`: `lookup_cached` and `allocate` both return an id whose slot is guaranteed to hold
exactly the right content (by construction for a hit, freshly written for a miss), so nothing past
this point needs to know which happened.

Prefill's cache lookups cost $O(f)$ in a sequence's number of full blocks $f$; `_register_if_newly_full`
recomputes the whole hash chain up to the new block on every call, also $O(f)$, rather than the $O(1)$
an incrementally maintained running hash would give.

### Level 5

`_allocate_or_preempt` retries until the allocator can satisfy the request, each time discarding the
running sequence that was admitted most recently — `self.running` is a plain `dict`, and insertion
order is admission order, so that sequence is `next(reversed(self.running))`. The loop always
terminates: each failed attempt removes one sequence from `running`, so after at most
`len(self.running)` rounds either an attempt succeeds or `running` is empty, and an empty engine's
whole pool is free, which Level 1's sizing guarantee — no request's total need ever exceeds
`num_blocks` — ensures is always enough for one more sequence's reservation.

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

`_decode_step` can be asked to extend a sequence that turns out to be the one evicted to make room for
itself — the most recently admitted running sequence is a well-defined choice even when only one
sequence is running. Skipping the append and returning the spare block leaves no trace: the sequence
is already back in the queue, `token_ids` and `num_generated` are untouched, and its next attempt goes
through `_prefill` like any other admission. `step`, `_finish_if_stopped` and the admission loop in
`step`'s second half are all unchanged from Level 3: a preempted sequence disappearing from
`self.running` mid-decode is exactly the case the snapshot-plus-`.get` pattern in `step` was already
written to tolerate.

Each call to `_allocate_or_preempt` costs $O(v)$ in the number of sequences $v$ it evicts before its
own allocation succeeds, plus that allocation's own $O(1)$ amortized cost.

### Follow-ups

- Chunked prefill: this problem only admits a request once its whole prompt fits the current step's
  remaining budget. A production engine instead prefills a long prompt a bounded chunk of tokens at a
  time across several steps, so one very long prompt cannot delay every other sequence's decode step
  until it is admitted in full.
- Swap instead of recompute: preemption here always discards the victim's blocks and regenerates them
  later from `token_ids` alone. Swapping copies a preempted sequence's block contents out to host
  memory and back instead, trading a transfer for the recomputation it avoids — worth it once a
  sequence's prefix is long enough that recomputing it costs more than copying it.
- A different victim policy: preempting the most recently admitted running sequence bounds how long
  any one sequence can be starved, since the sequence that has been running longest is never chosen
  while a newer one remains. Preempting whichever running sequence holds the most blocks instead frees
  more room per preemption, at the cost of being harder for a caller to predict.
- Partial-block caching: a cache entry only ever covers a full block, so two prompts that agree on
  everything except their last few tokens still share every earlier block but never the final, partial
  one. Caching less than a full block would mean handing a sequence a block that a second, diverging
  sequence could still write into.
- Concurrency: `step()` here runs to completion before anything else touches the engine. A server that
  accepts new requests while a step is in flight would need a lock around `queue`, `running` and the
  allocator together, since admitting a request or preempting a sequence changes state that the same
  step's own loop is still in the middle of reading.

<details>
<summary>Checks (runnable)</summary>

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

