# Weighted Data Batcher with Checkpointing

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Coding · sampling, checkpointing | ★★★☆☆ | Hard | MLE · RE · RS | sampling, checkpointing, iterators, data-loading | 3 parts / 55 min | Technical phone screen · Onsite |
<!-- meta:end -->

## Problem

An *example* is an opaque item drawn from one of several named datasets; nothing about its internal
structure matters here. A `DataRegistry` gives access to $K$ datasets, each identified by a `str` name,
through one method:

```py
class DataRegistry:
    def get_iterator(self, name: str, offset: int = 0):
        """Returns a fresh iterator over dataset `name`, opened directly at position `offset`;
        opening is fast at any offset, it never walks through the examples it skips. Writing L for
        the number of examples in `name`, the iterator yields the example at index offset % L, then
        (offset + 1) % L, then (offset + 2) % L, and so on forever: once it passes the last example
        it wraps back to the first and keeps going. It never raises StopIteration."""
```

Implement `DataBatcher(registry, weights, batch_size)`, where `weights` is a `dict[str, int]` mapping
every dataset name to a positive integer weight (`weights` has at least one entry) and `batch_size` is a
positive integer. Order the names alphabetically as $d_1 < d_2 < \cdots < d_K$ and write $w_k$ for
$\mathrm{weights}[d_k]$ and $W = \sum_{k=1}^{K} w_k$ for the *total weight*. `next_batch()` returns a
`list` of `batch_size` examples drawn from the $K$ datasets in proportion to `weights`. Every part below
keeps the same rule for the order *between* datasets in a batch — ascending alphabetical order of name —
and states its own rule for the order *within* one dataset's contribution.

### Part 1 — Exact weighted batches

Assume for this part that `batch_size` is divisible by $W$. For every $k$, `next_batch()` places exactly
$`\mathrm{batch\_size} \cdot w_k / W`$ of dataset $d_k$'s examples in the batch, consecutively, in the order
its own iterator produced them; the $K$ blocks are concatenated in order $d_1, d_2, \ldots, d_K$.

```py
class DataBatcher:
    def __init__(self, registry: DataRegistry, weights: dict[str, int], batch_size: int) -> None:
        """batch_size is divisible by sum(weights.values()) in this part."""

    def next_batch(self) -> list:
        """Returns batch_size examples: for each name in ascending alphabetical order,
        batch_size * weights[name] // sum(weights.values()) consecutive examples of that dataset."""
```

```text
weights = {"code": 2, "math": 1, "web": 1}      # W = 4
batch_size = 8                                    # 8 is divisible by 4

# code: 8 * 2 // 4 = 4 examples, math: 8 * 1 // 4 = 2, web: 8 * 1 // 4 = 2
next_batch() -> [code-0, code-1, code-2, code-3, math-0, math-1, web-0, web-1]
```

### Part 2 — Checkpoint and resume

Implement `state_dict()` and `load_state_dict(state)`. `state_dict()` returns a `dict` of plain `int`s and
strings — small and JSON-serializable, its size fixed by $K$ and never growing with how many batches have
been produced — capturing exactly enough to resume. Calling `load_state_dict` on a `DataBatcher`
constructed with the same `registry`, `weights` and `batch_size` moves it to the position the state
describes, so that its next `next_batch()` call returns exactly what the batcher the state came from would
have returned next, whether or not the two are the same Python object. Restoring must not re-iterate over
examples already produced: it uses `offset` to reopen each dataset directly where it left off.

```py
    def state_dict(self) -> dict:
        """A small, JSON-serializable dict sufficient to resume from here."""

    def load_state_dict(self, state: dict) -> None:
        """Moves this batcher to the position state describes."""
```

```text
b = DataBatcher(registry, {"code": 2, "math": 1, "web": 1}, batch_size=8)
b.next_batch()             # the batch shown in the Part 1 example
b.state_dict()              # {"offsets": {"code": 4, "math": 2, "web": 2}}
b.next_batch()              # [code-4, code-5, code-6, code-7, math-2, math-3, web-2, web-3]

# a second, independent batcher restored from the saved state continues identically:
b2 = DataBatcher(registry, {"code": 2, "math": 1, "web": 1}, batch_size=8)
b2.load_state_dict({"offsets": {"code": 4, "math": 2, "web": 2}})
b2.next_batch()              # [code-4, code-5, code-6, code-7, math-2, math-3, web-2, web-3]
```

### Part 3 — batch_size not divisible by W

Drop Part 1's divisibility assumption. `next_batch()` still returns `batch_size` examples in proportion to
`weights`, but the per-dataset quota $`\mathrm{batch\_size} \cdot w_k / W`$ is now, in general, not an
integer. `DataBatcher` gains two more constructor arguments, `allocation` and `seed`; whichever value
`allocation` takes, `next_batch()` still falls back to Part 1's exact rule whenever `batch_size` *is*
divisible by $W$.

Either way, when `batch_size` is not divisible by $W$, the batch is built one example at a time,
`batch_size` times, each time choosing which dataset the next example comes from; the batch lists the
`batch_size` examples in the order these choices were made, no longer grouped into per-dataset blocks.
Write $n$ for how many such choices a batcher has made in total since it was constructed (accumulating
across every `next_batch()` call so far, including calls before the current one), and
$`\mathrm{count}_k(n)`$ for how many of the first $n$ choices picked dataset $d_k$; dataset $d_k$'s *ideal
share* of the first $n$ choices is $n w_k / W$, generally not an integer.

- `allocation="deterministic"`: choose with no randomness, from the running per-dataset counts alone, such
  that for every $k$ and every $n$ (not only when $n$ is a multiple of `batch_size`),
  $`\lvert \mathrm{count}_k(n) - n w_k / W \rvert \le C`$ for a constant $C$ that does not depend on $n$.
  State your rule and prove such a $C$ for it.
- `allocation="stochastic"`: build a prefix-sum array over $w_1, \ldots, w_K$ in the same alphabetical
  order. Keep a `random.Random(seed)` instance on the batcher, and make each choice by drawing a uniform
  integer in $[0, W)$ from it and binary-searching the prefix-sum array for the dataset it lands in.
  `state_dict()` must then also carry enough of the generator's own state to make the draws resume
  exactly, not merely to have the correct distribution again.

```py
class DataBatcher:
    def __init__(self, registry: DataRegistry, weights: dict[str, int], batch_size: int,
                 allocation: str = "deterministic", seed: int = 0) -> None:
        """allocation is "deterministic" or "stochastic"; batch_size need not divide
        sum(weights.values()) any more. seed is used only when allocation == "stochastic"."""
```

```text
weights = {"code": 2, "math": 1, "web": 1}, batch_size = 6, allocation = "deterministic"
# W = 4, and 4 does not divide 6

next_batch() -> [code-0, math-0, web-0, code-1, code-2, math-1]
# final counts: code = 3 (ideal 6*2/4 = 3.0), math = 2 (ideal 1.5), web = 1 (ideal 1.5)
```

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two points are worth confirming before coding. Whether a dataset really does wrap around indefinitely
rather than raising once exhausted — assumed yes, exactly as `DataRegistry` states, which is also what
makes the batcher itself never need to know a dataset's length. And whether `weights` is fixed for the
life of one `DataBatcher` — assumed yes below; a run that needs different weights partway through
constructs a new batcher with them and loads the old one's `state_dict()` into it.

### Part 1

Each dataset needs its own read position, since the $K$ datasets are consumed independently and at
different rates; a `dict` keyed by name holds these offsets, and a second `dict` holds one live iterator
per dataset, opened lazily, so that a dataset that `next_batch()` never draws from is never opened at all.
Reopening `get_iterator` on every single call would also work — the registry promises it is fast — but
keeping the iterator alive between calls avoids paying that cost on every one of `batch_size` slots in
Part 3, and it is no harder to write.

```python
class DataBatcher:
    def __init__(self, registry: "DataRegistry", weights: dict[str, int], batch_size: int) -> None:
        if not weights or any(w <= 0 for w in weights.values()):
            raise ValueError("weights must be a non-empty mapping of positive integers")
        if batch_size <= 0:
            raise ValueError("batch_size must be positive")
        self._registry = registry
        self._names = sorted(weights)                      # NOTE: fixed alphabetical order, used everywhere
        self._weights = dict(weights)
        self._total_weight = sum(weights.values())
        self._batch_size = batch_size
        self._offsets = {name: 0 for name in self._names}  # examples drawn so far, per dataset
        self._iters: dict = {}                              # live iterators, opened lazily, never replayed

    def _iterator_for(self, name: str):
        it = self._iters.get(name)
        if it is None:                                       # NOTE: opened once per dataset, at its offset
            it = self._registry.get_iterator(name, self._offsets[name])
            self._iters[name] = it
        return it

    def _take(self, name: str, count: int) -> list:
        it = self._iterator_for(name)
        items = [next(it) for _ in range(count)]
        self._offsets[name] += count
        return items

    def next_batch(self) -> list:
        batch = []
        for name in self._names:
            quota = self._batch_size * self._weights[name] // self._total_weight
            batch.extend(self._take(name, quota))
        return batch
```

`next_batch()` does $`O(\mathrm{batch\_size})`$ work: `_take` never iterates further than the examples it
actually returns, and opening an iterator is assumed $O(1)$. The batcher's own state is $O(K)$, two small
dicts, regardless of how many batches have been produced.

### Part 2

The saved state is the `_offsets` dict alone: two things that might look like state are deliberately left
out of it. The live iterators are not serializable in the first place — a generator cannot be written to
JSON, or even copied — and there is nothing they would add to the state besides the same integer offset
`get_iterator` already accepts; keeping them out is what makes `state_dict()` $O(K)$ instead of growing
with every example produced. `weights` and `batch_size` are also left out, because they are constructor
arguments of the batcher being resumed into, not facts about *where* it has reached; a resumed batcher is
already constructed with them before `load_state_dict` is called.

```python
class DataBatcher(DataBatcher):
    def state_dict(self) -> dict:
        return {"offsets": dict(self._offsets)}             # NOTE: a copy -- callers may hold on to this

    def load_state_dict(self, state: dict) -> None:
        self._offsets = dict(state["offsets"])
        self._iters = {}          # NOTE: dropped, not replayed -- reopened lazily at the restored offsets
```

Reopening lazily rather than immediately inside `load_state_dict` means restoring itself never calls
`get_iterator`: the cost lands on whichever dataset `next_batch()` actually asks for next, once per
dataset it touches, each call landing directly on the saved offset — $O(1)$ per dataset touched, never
$O(n)$ in how many examples came before it, satisfying Part 2's requirement without any special-casing.

### Part 3

**Deterministic.** Track, for each dataset $d_k$, its ratio $`p_k(n) = \mathrm{count}_k(n) / w_k`$ — the
count normalized by weight — and choose whichever dataset currently has the smallest ratio, ties broken
by the alphabetically smaller name. Comparing $p_i(n)$ and $p_j(n)$ by cross-multiplying
$`\mathrm{count}_i(n) \cdot w_j`$ against $`\mathrm{count}_j(n) \cdot w_i`$ keeps the comparison exact: these
are small integers, but float division could still round two mathematically equal ratios to two different
neighbouring floats and break a tie the wrong way.

**Lemma.** For any two datasets $d_i, d_j$ and every $n \ge 0$,
$`\lvert p_i(n) - p_j(n) \rvert \le \max(1/w_i,\, 1/w_j)`$.

*Proof.* Look only at the subsequence of choices that land on $d_i$ or $d_j$; every other choice leaves
both ratios unchanged. Whenever a choice in this subsequence lands on, say, $d_i$, it is because $d_i$'s
ratio was the smallest among *all* $K$ datasets at that moment, in particular no larger than $d_j$'s — so
restricted to this subsequence, the rule is exactly "advance whichever of $d_i$, $d_j$ currently has the
smaller-or-tied ratio, by $1/w_i$ or $1/w_j$ respectively", and the other $K - 2$ datasets play no part in
it. Induct on the length of this subsequence. It starts at length 0 with both ratios 0, difference 0.
Suppose the bound holds and, say, $d_i$'s ratio is the smaller-or-tied one, so it is chosen next: with
$M = \max(1/w_i, 1/w_j)$ and $g$ the gap $p_j(n) - p_i(n)$ just before this choice, the hypothesis gives
$0 \le g \le M$. Afterward the gap is $g - 1/w_i$, which lies in $[-1/w_i,\ M - 1/w_i] \subseteq [-M, M]$
since $1/w_i \le M$. $\blacksquare$

**Corollary.** For every $k$ and every $n$, $`\lvert \mathrm{count}_k(n) - n w_k / W \rvert \le 1 + K`$.

*Proof.* Since $`n = \sum_j \mathrm{count}_j(n) = \sum_j w_j\, p_j(n)`$, dividing by $W$ writes $n / W$ as a
weighted average of the $p_j(n)$ with weights $w_j / W$ summing to 1, so

$$p_k(n) - \frac{n}{W} = \sum\_{j=1}^{K} \frac{w_j}{W}\bigl(p_k(n) - p_j(n)\bigr).$$

By the Lemma, $\lvert p_k(n) - p_j(n) \rvert \le \max(1/w_k, 1/w_j) \le 1/w_k + 1/w_j$, so

$$\left\lvert p_k(n) - \frac{n}{W} \right\rvert \le \sum\_{j=1}^{K} \frac{w_j}{W}\left(\frac{1}{w_k} + \frac{1}{w_j}\right) = \frac{1}{w_k}\underbrace{\sum_j \frac{w_j}{W}}\_{=\\,1} + \underbrace{\sum_j \frac{1}{W}}\_{=\\,K/W} = \frac{1}{w_k} + \frac{K}{W}.$$

Multiplying by $w_k$ gives $`\lvert \mathrm{count}_k(n) - n w_k / W \rvert = w_k \lvert p_k(n) - n/W \rvert
\le 1 + K w_k / W \le 1 + K`$, since $w_k \le W$. $\blacksquare$

The bound is loose in practice: across the many random weight sets and thousands of slots exercised by the
checks below, the worst deviation ever observed is under 2, regardless of $K$.

```python
import random
from bisect import bisect_right


class DataBatcher(DataBatcher):
    def __init__(self, registry: "DataRegistry", weights: dict[str, int], batch_size: int,
                 allocation: str = "deterministic", seed: int = 0) -> None:
        if allocation not in ("deterministic", "stochastic"):
            raise ValueError(f"unknown allocation {allocation!r}")
        super().__init__(registry, weights, batch_size)
        self._allocation = allocation
        self._prefix: list[int] = []          # cumulative weight, same alphabetical order as self._names
        running = 0
        for name in self._names:
            running += self._weights[name]
            self._prefix.append(running)
        self._rng = random.Random(seed)

    def next_batch(self) -> list:
        if self._batch_size % self._total_weight == 0:
            return super().next_batch()                        # NOTE: Part 1's exact rule, unchanged
        pick = self._pick_deterministic if self._allocation == "deterministic" else self._pick_stochastic
        batch = []
        for _ in range(self._batch_size):
            batch.extend(self._take(pick(), 1))
        return batch

    def _pick_deterministic(self) -> str:
        best = self._names[0]
        for name in self._names[1:]:
            # count[name] / weights[name] < count[best] / weights[best], compared by cross-multiplying
            lhs = self._offsets[name] * self._weights[best]
            rhs = self._offsets[best] * self._weights[name]
            if lhs < rhs or (lhs == rhs and name < best):
                best = name
        return best

    def _pick_stochastic(self) -> str:
        draw = self._rng.randrange(self._total_weight)   # NOTE: an int in [0, W) -- no float edge case
        return self._names[bisect_right(self._prefix, draw)]

    def state_dict(self) -> dict:
        state = super().state_dict()
        if self._allocation == "stochastic":
            version, ints, gauss = self._rng.getstate()
            state["rng_state"] = [version, list(ints), gauss]    # NOTE: tuple -> list, so json.dumps works
        return state

    def load_state_dict(self, state: dict) -> None:
        super().load_state_dict(state)
        if self._allocation == "stochastic" and "rng_state" in state:
            version, ints, gauss = state["rng_state"]
            self._rng.setstate((version, tuple(ints), gauss))
```

The two allocations trade off differently. The deterministic rule has no variance at all — the Corollary
bounds its worst case, not just its average case — and its state is the same integer offsets Part 2
already saves; nothing about it depends on when it happens to be checkpointed. Its ties are also broken
the same way every time, so on a run of tied ratios (equal or rationally related weights) the same dataset
wins every time that exact tie recurs, making the within-batch pattern itself fixed across epochs of
otherwise-identical data. The stochastic rule is unbiased — $d_k$'s expected count after $n$ draws is
exactly $n w_k / W$ — but its actual count is a sum of $n$ dependent random draws and so has real
variance around that expectation, with nothing bounding one run's deviation the way the Corollary does.
That randomness must itself be reproducible to be checkpointable: the `random.Random` object has already
advanced by the time a checkpoint is taken, so `state_dict()` has to capture its *current* state, not the
original `seed` — resuming from `seed` alone would replay draws already made and repeat batches already
produced instead of continuing past them.

Choosing a dataset costs $O(K)$ per slot for the deterministic scan (or $O(\log K)$ with a heap keyed by
ratio, worthwhile once $K$ is large) against $O(\log K)$ per slot for the stochastic binary search;
`state_dict()` stays $O(K)$ either way, plus a fixed, small amount of extra state for the generator in the
stochastic case.

### Follow-ups

- A dataset shorter than its quota within one batch: wraparound means it simply repeats inside that batch
  — the registry does this invisibly to `DataBatcher`, which never learns a dataset's length. If repeats
  within a single batch are unacceptable, `get_iterator` would have to raise once exhausted instead of
  wrapping, and `next_batch()` would then have to shrink that batch or fail outright: a change to the
  environment's contract, not something `DataBatcher` can fix by itself.
- Weights changing mid-training: construct a new `DataBatcher` with the new `weights` and load the old
  one's `state_dict()` into it — the state is only per-dataset offsets, never tied to any particular
  weights. Part 3's deterministic rule then measures every ratio against the *new* weights from that point
  on, so a dataset whose weight just increased looks artificially far behind and gets a transient run of
  extra picks to catch up; the Corollary's bound still applies, now anchored at $n = 0$ from the moment of
  the change rather than from the batcher's original construction.
- Sharding across data-parallel ranks without overlap: give rank $r$ of $R$ its own slice of every
  dataset's offset range at construction — for a dataset of length $L$, offsets
  $`\bigl[\, r \lfloor L/R \rfloor,\ (r+1)\lfloor L/R \rfloor \,\bigr)`$ — and wrap within that slice's own
  length instead of $L$. Ranks then never call `get_iterator` with an offset another rank will also use, at
  the cost of each rank cycling back to its own start after a slightly different number of examples when
  $R$ does not divide $L$ evenly.
- Before deploying: differential-test `next_batch()` against a brute force on many random `(weights,
  batch_size)` pairs for Part 1, and against the Corollary's bound for Part 3; pause and resume at every
  batch boundary of a long run under both allocations and diff against an uninterrupted run; round-trip
  `state_dict()` through `json.dumps` / `json.loads`; and fuzz dataset lengths shorter than a single slot,
  a lone dataset, and a `batch_size` smaller than $K$, under both `PYTHONHASHSEED` values.

<details>
<summary>Checks (runnable)</summary>

```python
import json
import random


class FakeDataset:
    """A finite dataset that wraps around forever, exactly as DataRegistry promises."""

    def __init__(self, data: list) -> None:
        self.data = list(data)

    def get_iterator(self, offset: int = 0):
        n = len(self.data)
        i = offset % n
        while True:
            yield self.data[i]
            i = (i + 1) % n


class FakeRegistry:
    def __init__(self, datasets: dict) -> None:
        self._datasets = {name: FakeDataset(items) for name, items in datasets.items()}

    def get_iterator(self, name: str, offset: int = 0):
        return self._datasets[name].get_iterator(offset)


def make_registry(names, length=97):
    return FakeRegistry({name: [f"{name}-{i}" for i in range(length)] for name in names})


# --- the worked examples, exactly as stated in the Problem section ---
weights = {"code": 2, "math": 1, "web": 1}

b = DataBatcher(make_registry(weights), weights, batch_size=8)
batch1 = b.next_batch()
assert batch1 == ["code-0", "code-1", "code-2", "code-3", "math-0", "math-1", "web-0", "web-1"]

state = b.state_dict()
assert state == {"offsets": {"code": 4, "math": 2, "web": 2}}
batch2 = b.next_batch()
assert batch2 == ["code-4", "code-5", "code-6", "code-7", "math-2", "math-3", "web-2", "web-3"]

b_resumed = DataBatcher(make_registry(weights), weights, batch_size=8)
b_resumed.load_state_dict(json.loads(json.dumps(state)))
assert b_resumed.next_batch() == batch2

b3 = DataBatcher(make_registry(weights), weights, batch_size=6, allocation="deterministic")
batch3 = b3.next_batch()
assert [x.split("-")[0] for x in batch3] == ["code", "math", "web", "code", "code", "math"]


# --- Part 1: an independent brute force, built from the statement alone ---
def brute_force_batches(registry, weights, batch_size, num_batches):
    names = sorted(weights)
    total = sum(weights.values())
    assert batch_size % total == 0
    iterators = {name: registry.get_iterator(name, 0) for name in names}
    result = []
    for _ in range(num_batches):
        one = []
        for name in names:
            quota = batch_size * weights[name] // total
            one.extend(next(iterators[name]) for _ in range(quota))
        result.append(one)
    return result


rng = random.Random(0)
for _ in range(60):
    names = [f"d{i}" for i in range(rng.randint(1, 5))]
    w = {name: rng.randint(1, 6) for name in names}
    total = sum(w.values())
    bs = total * rng.randint(1, 4)
    length = rng.randint(3, 40)
    n_batches = rng.randint(1, 4)
    expected = brute_force_batches(make_registry(names, length), w, bs, n_batches)
    batcher = DataBatcher(make_registry(names, length), w, bs)
    got = [batcher.next_batch() for _ in range(n_batches)]
    assert got == expected, (names, w, bs)
print("brute-force sweep OK (Part 1)")


# --- Part 2 & 3: pause at every batch boundary of a long run and compare with an uninterrupted
#     run, for both allocations, with the saved state round-tripped through JSON in between and
#     handed to a fresh instance constructed with a different seed ---
def run_and_compare(weights, batch_size, allocation, num_batches, seed, length=53):
    uninterrupted_batcher = DataBatcher(make_registry(weights, length), weights, batch_size,
                                         allocation=allocation, seed=seed)
    uninterrupted = [uninterrupted_batcher.next_batch() for _ in range(num_batches)]
    for k in range(num_batches + 1):
        warm_up = DataBatcher(make_registry(weights, length), weights, batch_size,
                               allocation=allocation, seed=seed)
        produced = [warm_up.next_batch() for _ in range(k)]
        assert produced == uninterrupted[:k]
        saved = json.loads(json.dumps(warm_up.state_dict()))
        resumed = DataBatcher(make_registry(weights, length), weights, batch_size,
                               allocation=allocation, seed=seed + 12345)  # must be overridden by `saved`
        resumed.load_state_dict(saved)
        rest = [resumed.next_batch() for _ in range(num_batches - k)]
        assert rest == uninterrupted[k:], (allocation, k)


rng = random.Random(1)
for _ in range(20):
    names = [f"d{i}" for i in range(rng.randint(1, 4))]
    w = {name: rng.randint(1, 5) for name in names}
    total = sum(w.values())
    bs = rng.choice([total, total * 2, total + 1, total + 3, max(1, total - 1)])
    allocation = rng.choice(["deterministic", "stochastic"])
    run_and_compare(w, bs, allocation, num_batches=rng.randint(2, 6), seed=rng.randint(0, 1000))
print("pause/resume sweep OK (Part 2 and 3, both allocations)")


# --- Part 3: the deterministic allocator's cumulative-error bound, over many batches and random
#     weights; examples are tagged with their dataset's name so the bound can be checked at every
#     single slot, not only at batch boundaries ---
def sweep_deterministic_bound(trials=200, max_slots=600, seed=2):
    rng = random.Random(seed)
    worst = 0.0
    for _ in range(trials):
        k = rng.randint(1, 8)
        names = [f"d{i}" for i in range(k)]
        w = {name: rng.randint(1, 40) for name in names}
        total = sum(w.values())
        reg = make_registry(names, length=rng.randint(200, 400))
        bs = rng.choice([1, 2, 3, 5, 7])
        n_slots = rng.randint(1, max_slots // bs) * bs
        batcher = DataBatcher(reg, w, batch_size=bs, allocation="deterministic")
        produced = 0
        counts = {name: 0 for name in names}
        while produced < n_slots:
            for item in batcher.next_batch():
                counts[item.split("-")[0]] += 1
                produced += 1
                for name in names:
                    ideal = produced * w[name] / total
                    err = abs(counts[name] - ideal)
                    assert err <= 1 + k + 1e-9, (name, produced, counts[name], ideal, err, k)
                    worst = max(worst, err)
    return worst


worst_error = sweep_deterministic_bound()
print(f"deterministic allocator: worst |count - ideal share| = {worst_error:.3f} (proven bound 1 + K)")
assert worst_error < 6   # comfortably inside 1 + K for every K <= 8 exercised above


# --- Part 3: the stochastic allocator is unbiased in expectation, with a generous statistical margin ---
w = {"a": 5, "b": 3, "c": 2}
batcher = DataBatcher(make_registry(w, length=500), w, batch_size=7, allocation="stochastic", seed=7)
counts = {name: 0 for name in w}
total_examples = 0
for _ in range(3000):
    for item in batcher.next_batch():
        counts[item.split("-")[0]] += 1
        total_examples += 1
total_w = sum(w.values())
for name in w:
    assert abs(counts[name] / total_examples - w[name] / total_w) < 0.02, (name, counts)
print("stochastic expectation check OK")


# --- edge cases ---
# a dataset shorter than a single batch's quota wraps around within that batch
tiny_reg = FakeRegistry({"tiny": ["t0", "t1"], "big": [f"big-{i}" for i in range(50)]})
tiny_batch = DataBatcher(tiny_reg, {"tiny": 1, "big": 1}, batch_size=10).next_batch()
assert [x for x in tiny_batch if x in ("t0", "t1")] == ["t0", "t1", "t0", "t1", "t0"]

# a single dataset wins every slot regardless of allocation, and no tie-break ever matters
solo_reg = FakeRegistry({"only": [f"o-{i}" for i in range(5)]})
solo = DataBatcher(solo_reg, {"only": 3}, batch_size=7, allocation="deterministic")
assert solo.next_batch() == ["o-0", "o-1", "o-2", "o-3", "o-4", "o-0", "o-1"]

# batch_size smaller than the number of datasets: some datasets simply get 0 examples that batch
few = DataBatcher(make_registry(["a", "b", "c", "d"]), {"a": 1, "b": 1, "c": 1, "d": 1},
                   batch_size=2, allocation="deterministic")
assert few.next_batch() == ["a-0", "b-0"]   # the two least-served (tied at 0) alphabetically

# constructor validation
def expect_value_error(**kwargs):
    try:
        DataBatcher(FakeRegistry({"a": [1, 2, 3]}), **kwargs)
    except ValueError:
        return
    raise AssertionError(f"expected ValueError for {kwargs!r}")


expect_value_error(weights={}, batch_size=4)
expect_value_error(weights={"a": 0}, batch_size=4)
expect_value_error(weights={"a": -1}, batch_size=4)
expect_value_error(weights={"a": 1}, batch_size=0)
expect_value_error(weights={"a": 1}, batch_size=4, allocation="uniform")

# state_dict() is a snapshot: mutating the returned dict must not affect the batcher
mut_batcher = DataBatcher(make_registry(weights), weights, batch_size=8)
mut_batcher.next_batch()
snapshot = mut_batcher.state_dict()
snapshot["offsets"]["code"] = 999_999
assert mut_batcher.state_dict()["offsets"]["code"] != 999_999
print("edge cases OK")

print("all checks passed")
```

</details>

</details>
