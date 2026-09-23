# Call Stacks from Profiler Samples

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Coding · stacks, string processing | ★★★★☆ | Medium | SWE · MLE · Infra Eng | call-stack, longest-common-prefix, streaming, debounce, trie, recursion | 3 parts / 55 min | Technical phone screen · Onsite |
<!-- meta:end -->

## Problem

A profiler observes a running program by taking periodic *samples*: at certain instants it records the
full call stack — the ordered list of functions currently on the stack, from the outermost (the one that
has been running the longest, called by nothing else in the trace) to the innermost (the one actually
executing at that instant). A `Sample` pairs a timestamp with this list; an `Event` reports one function
beginning or ending its time on the stack.

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

Two samples can report the same function name at two different depths at once — direct or indirect
recursion — so an event's *frame* is identified by its `(depth, name)` pair, not by `name` alone: `depth`
is part of every event below, not a diagnostic extra.

### Part 1 — Samples to events

`convert_samples_to_events(samples, end_time)` takes `samples`, sorted by strictly increasing `ts`, and
returns the list of events that explains how the stack evolved from the first sample up to `end_time` (a
timestamp at least as large as the last sample's `ts`; when `samples` is empty, `end_time` is unused and
the result is `[]`).

At the timestamp of the first sample, every frame in its stack starts, listed outermost to innermost.

Between one sample and the next, compare the two stacks position by position from the outside in and find
their *longest common prefix* (LCP): the largest `p` such that the two stacks agree on every position `0,
1, ..., p - 1`. A frame at a position at or beyond `p` in the earlier stack has left the stack; a frame at
a position at or beyond `p` in the later stack has entered it. Both kinds of change are stamped with the
later sample's `ts`: frames that left end first, in order from the innermost position outward (the call
that was deepest returns first), and only then do the frames that appeared start, in order from `p`
outward to the innermost (a caller has to already be on the stack before anything it calls can start).
Two adjacent samples with identical stacks (`p` equal to both their lengths) contribute no events at all
— this is what compresses a run of unchanged samples down to nothing.

At `end_time`, every frame still in the last sample's stack ends, innermost first; `end_time` is the only
source of closing events for those frames — `convert_samples_to_events` never closes a frame at an
earlier time just because the input runs out.

```py
def convert_samples_to_events(samples: list[Sample], end_time: float) -> list[Event]: ...
```

```text
samples = [
    Sample(0.0, ["main"]),
    Sample(1.0, ["main", "solve"]),
    Sample(2.0, ["main", "solve", "solve"]),           # solve calls itself: two active frames named "solve"
    Sample(2.5, ["main", "solve", "solve"]),           # identical to the previous stack -> no events at 2.5
    Sample(4.0, ["main", "solve", "solve", "solve"]),  # solve recurses again: three active "solve" frames
    Sample(6.0, ["main", "solve", "helper"]),          # the two innermost "solve" frames return; helper is called
]
convert_samples_to_events(samples, end_time=9.0) == [
    Event("start", 0.0, "main", 0),
    Event("start", 1.0, "solve", 1),
    Event("start", 2.0, "solve", 2),      # a second, distinct "solve" frame, one level deeper than the first
    # at 2.5 the common prefix with the previous stack is 3 (the whole stack) -> no events
    Event("start", 4.0, "solve", 3),      # a third, distinct "solve" frame
    # at 6.0 the common prefix with ["main","solve","solve","solve"] is 2: main and solve@1 are unaffected
    Event("end", 6.0, "solve", 3),        # depth 3 leaves first (innermost)
    Event("end", 6.0, "solve", 2),        # then depth 2
    Event("start", 6.0, "helper", 2),     # only then does the new depth-2 frame start
    # end_time = 9.0: main, solve@1 and helper@2 are all still active in the last sample -> close them
    Event("end", 9.0, "helper", 2),
    Event("end", 9.0, "solve", 1),
    Event("end", 9.0, "main", 0),
]
```

### Part 2 — Streaming and noise

A real profiler delivers samples one at a time over what may be a long run; `stream_debounced_events`
must not hold state proportional to the number of samples processed so far. It consumes `samples` as an
iterator — it may only be safe to read once, front to back — and yields events one at a time as they
become certain rather than returning a finished list. Unlike Part 1, it takes no `end_time`: when the
iterator is exhausted, a frame that has not yet been confirmed closed simply produces no further event,
exactly as Part 1 never closes a frame still active in its last sample, generalised here to "not yet
confirmed closed."

A sampled profiler also has noise: an instrument momentarily misreporting a frame, or a call so
short-lived that only a single sample happens to catch it. `stream_debounced_events(samples, k)` filters
this with a debounce rule. Identify frames exactly as Part 1 does, by the position reached along the
longest-common-prefix chain from each sample to the next, so a break in that chain always starts a
brand-new frame at every position it reaches or passes, even if the same name reoccupies a position
immediately afterward (see the example below). A frame produces no event until it has held its position
for `k` consecutive samples (`k >= 1`; `k = 1` needs no debounce and reduces to Part 1's own transitions,
minus the final closing at `end_time`, which Part 2 does not have). Its `start` event is stamped with the
`k`-th of those consecutive samples. If the frame's position later becomes vacant and stays vacant for `k`
consecutive samples in a row, its `end` event is stamped with the `k`-th of *those*. A frame that never
reaches `k` consecutive occupying samples produces neither event; a frame that does reach `k`, but whose
vacancy never reaches `k` consecutive samples before the input ends, produces a `start` but no `end`.

For example, with `k = 1` (every single sample already counts on its own), `["req", "validate"]` followed
by `["req", "validate", "persist"]` has a common prefix of 2 (the whole earlier stack): `req` and
`validate` are unbroken, and only `persist` starts. But `["req", "validate"]` followed by `["worker",
"validate", "req"]` has a common prefix of 0 — position 0 already disagrees, `req` against `worker` — so
`req` and `validate` both end there and a brand-new `req` starts at depth 2, even though the name
`validate` still happens to sit at depth 1 and the name `req` happens to reappear one level deeper:
neither is connected to the earlier occupant of its position, because the chain already broke before
reaching it.

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
    Event("start", 2, "svc", 0),     # depth 0 holds "svc" at samples 1 and 2: 2 consecutive -> confirmed at sample 2
    # depth 1 holds "handle" only at sample 3, then reverts at sample 4: 1 consecutive -> never confirmed, no event
    Event("start", 6, "flush", 1),   # depth 1 holds "flush" at samples 5 and 6: 2 consecutive -> confirmed at sample 6
    Event("end", 8, "flush", 1),     # depth 1 is vacant at samples 7 and 8: 2 consecutive -> closed at sample 8
    # "svc" is still active when the input ends and its vacancy never starts -> no end event for it
]
```

### Part 3 — Suffix matching against known traces

A *complete* stack trace records every frame from the entry point (outermost) to the point of the error
(innermost). `TraceIndex` is built once from a fixed collection of complete, known traces, each identified
by a unique `trace_id`, and then answers queries: given only the innermost frames of an incomplete trace —
a *suffix* — report every known trace whose own last frames, taken to the same length, are exactly that
suffix, frame for frame.

`TraceIndex(traces)` takes `traces`, a list of `(trace_id, frames)` pairs with `frames` ordered entry
point first, error point last, and non-empty. `index.match_suffix(frames)` takes a non-empty query in the
same order and returns the `trace_id`s of every known trace with at least `len(frames)` frames whose own
last `len(frames)` frames equal `frames` exactly, position for position, sorted lexicographically
ascending; if no known trace matches — including every known trace that is simply shorter than the query —
it returns the string `"UNKNOWN"` instead of an empty list.

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
index.match_suffix(["parse", "oom"])                   == ["T1", "T2"]              # both end ..., parse, oom
index.match_suffix(["oom"])                            == ["T1", "T2", "T3", "T4"]  # every trace ends in oom
index.match_suffix(["ingest", "oom"])                  == ["T4"]                    # only T4 ends ..., ingest, oom
index.match_suffix(["boot", "ingest", "parse", "oom"]) == ["T1"]                    # matches only the whole trace
index.match_suffix(["cache", "parse", "oom"])          == "UNKNOWN"                 # no known trace ends this way
```

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two conventions are worth confirming before coding: that the same function name at two different depths
is always two different frames, so identity tracks position along the longest-common-prefix chain rather
than name alone; and that an event's timestamp is the instant at which the change was *observed*, not a
claim about the exact instant the call actually started or returned — a call that starts and finishes
entirely between two samples leaves no trace in the samples at all, and no algorithm working from samples
alone can recover it.

### Part 1

Keep only the previous sample's stack; everything else follows from one helper that walks two stacks
together to find how far they agree.

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

Every position is visited a constant number of times across the whole run (once by whichever
`_common_prefix_len` call stops there, and once more if it turns into an event), so the total time is
linear in the sum of every sample's stack size. Beyond the returned list, the only state carried between
samples is `prev`, so extra memory is $O(D)$, where $D$ is the deepest stack any sample reaches.

### Part 2

A position's presence streak can only ever be extended alongside its parent's: the longest-common-prefix
step that keeps position $d$ in the chain also keeps every shallower position in the chain on that same
step, so a shallower position is always confirmed at the same sample as, or earlier than, any deeper one.
That makes the confirmed positions always a contiguous prefix, trackable as a single integer `open_len`
rather than a set, with a `streak` list covering only the unconfirmed positions beyond it. A confirmed
position that later leaves the chain moves into a small `closing` dict keyed by depth, counting
consecutive absent samples until it reaches `k` — or is abandoned once its depth is reclaimed by a
different candidate, per the corner case above. At most one `closing` entry can exist per depth at a
time, because a replacement candidate needs `k` samples of its own to be confirmed, and by then the old
occupant's own `k`-sample vacancy has already resolved one way or the other. So `streak` and `closing`
together stay $O(D)$, no matter how many samples have already gone by.

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

Each sample does work proportional to its own stack depth (the prefix comparison, the `streak` rebuild,
at most one `closing` dict lookup per position that leaves), so the total time across a run is again
linear in the sum of the sampled stack sizes, and the extra memory beyond whatever the caller keeps from
the yielded events is $O(D)$: `prev` and `streak` are each bounded by the deepest stack seen, and
`closing` by the argument above.

### Part 3

Matching by a *suffix* of the query against a suffix of each known trace is exactly matching by a *prefix*
once every sequence is reversed to innermost-frame-first. Insert every known trace's reversed frame list
into a trie; a query of length $M$, walked the same $M$ steps in the same reversed direction, lands on the
trie node that exactly the traces sharing that $M$-length suffix pass through. Recording, at insertion
time, every `trace_id` that passes through a node — not only the one whose trace ends there — costs one
append per node along each trace's path, so a query then only has to walk to its node and read off a
(pre-sorted) copy of the list already sitting there.

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

Building the trie costs time proportional to the sum of the known traces' lengths (each frame of each
trace becomes one more level of the walk and one dictionary insertion), plus a one-time sort of the id
list stored at every node. `match_suffix` then costs $O(M)$ plus the size of its own result, where $M$ is
the query's length — independent of how many known traces exist or how long any of them are.

### Follow-ups

- **The other direction.** Given a stream of `enter`/`exit` events instead of samples, replay them against
  a plain stack — push on `enter`, pop and check the name on `exit` — and read the active stack off it
  after every event; run-length-compress consecutive identical stacks into one entry carrying a count, or,
  if the compression should be bounded by elapsed time rather than sample count so that a long-idle
  unchanged stack still gets split, carry a duration and start a new entry once too much time has passed
  since the current one began.
- **Only the last $m$ frames of a sample are visible.** If a sample shows just its innermost $m$ frames
  (the caller side is hidden, and $m$ can vary from sample to sample), two consecutive samples cannot be
  diffed from the root the way Parts 1 and 2 do: anchor the comparison at the innermost frame instead and
  extend outward while the two visible suffixes keep agreeing. A frame beyond the first disagreement, or
  beyond the shorter of the two visible windows, cannot be distinguished from a frame that is simply out of
  view, so nothing can safely be reported for it — including, at the very last sample, whichever frame is
  mid-call when observation stops: nothing afterward can confirm when, or whether, it returns.
- **Only the last $N$ samples are retained.** Retaining the $N$ raw samples themselves costs $O(ND)$
  against Part 2's $O(D)$, so it is worth it only once a query needs to reconstruct more than the single
  most recent transition. Either way, what is *knowable* does not change: a call that starts and finishes
  strictly between two retained samples is invisible even though every retained stack looks perfectly
  consistent, and for any frame already active at the window's left edge, only that it started at or
  before that edge is known — not when.
- **Event timestamps are an observation, not a fact.** Stamping a `start` or `end` with a sample's own
  `ts` is a convention this problem adopts for a definite answer, not a claim that the call actually began
  or returned at that exact instant — the true instant lies somewhere in the, possibly large, gap since the
  previous sample. A coarser sampling interval means coarser, not wrong, timestamps; treating them as exact
  is only safe once the interval is already much smaller than the durations being measured.
- **An interval instead of two separate events.** A `start`/`end` pair for one frame can equally be
  reported as a single `(name, depth, start_ts, end_ts)` span, with `end_ts = None` for a frame never
  confirmed closed; the same longest-common-prefix walk produces it, closing a dictionary entry in place of
  yielding a second event, and it reads more directly when the goal is a function's total active time
  rather than a raw event log.

<details>
<summary>Checks (runnable)</summary>

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
