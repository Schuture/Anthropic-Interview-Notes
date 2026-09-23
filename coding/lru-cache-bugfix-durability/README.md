# Memoising LRU Cache: Bug Fix and Durability

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Coding · debugging, persistence | ★★★★☆ | Medium | SWE · MLE | caching, hashing, persistence, concurrency, serialization | 3 parts / 55 min | Technical phone screen · Onsite |
<!-- meta:end -->

## Problem

`Memoize` wraps a callable `func` and caches the value it returns, so a later call with the same
arguments returns the cached value instead of calling `func` again — in the style of
`functools.lru_cache`. It holds at most `capacity` distinct calls, a positive integer; adding one beyond
that evicts the *least recently used* entry first — the cached call that has gone longest without being
made again. Both a call already cached (a *hit*) and one that is not (a *miss*) count as using the entry
they touch when deciding which one is least recently used.

Two calls resolve to the same cache entry exactly when they pass equal positional arguments in the same
positions and equal keyword arguments — compared by name and value, regardless of the order the
keywords were written in at the call site. A call is never the same as one that passes, as a keyword, an
argument the other passed positionally, even for the same-named parameter: `generate_key` builds a key
from the `*args`/`**kwargs` it receives and never inspects `func`'s signature, so `f(1, 2)` and
`f(1, b=2)` are different calls that may occupy separate entries. Every positional and keyword argument
value must be hashable, since it becomes part of a cache key; passing one that is not (a `list`, for
instance) fails exactly as it would from using that value as a `dict` key directly.

The skeleton below is provided, written in the style of `functools.lru_cache`:

```py
from collections import OrderedDict


class Memoize:
    """Wraps func so that repeated calls with the same arguments return a cached result instead of
    calling func again. Holds at most capacity entries; a call that would add a new one beyond that
    evicts the least recently used entry first. Both a hit and a miss count as a "use" of the key
    they touch, for the purpose of deciding which entry is least recently used."""

    def __init__(self, func, capacity):
        self.func = func
        self.capacity = capacity
        self.cache = OrderedDict()
        self.hits = 0
        self.misses = 0

    def generate_key(self, *args, **kwargs):
        key = args + tuple(kwargs.items())
        if len(key) == 1:
            return key[0]
        return key

    def __call__(self, *args, **kwargs):
        key = self.generate_key(*args, **kwargs)
        if key in self.cache:
            self.hits += 1
            return self.cache[key]
        self.misses += 1
        result = self.func(*args, **kwargs)
        self.cache[key] = result
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)
        return result
```

### Part 1 — Find and fix three bugs

The code above has three bugs: two in `generate_key`, one in `__call__`. Two are wrong lines; one is a
missing line. With all three fixed, `generate_key` and `__call__` together satisfy every rule stated
above: kwargs given in a different order still resolve to the same entry; a call is never confused with
an unrelated one that flattens to the same argument sequence through a different split between
positional and keyword arguments — `f(1, 2)` (two positional arguments) and `f((1, 2))` (one positional
argument, itself a two-element tuple) must never collide — and a hit updates recency exactly as a miss
does. For each bug, give its location, state which calls it makes wrongly collide, wrongly fail to
collide, or (for the third) what it does to eviction order, and why; then fix it.

```text
cache = Memoize(lambda *args, **kwargs: (args, kwargs), capacity=2)
cache(1, b=2, c=3)   # miss                                       cache (LRU -> MRU): [f(1,b=2,c=3)]
cache(1, c=3, b=2)   # hit -- kwargs reordered: the same call, the same entry
cache((1, 2))        # miss -- ONE positional argument, itself a two-element tuple
                      #                                    cache: [f(1,b=2,c=3), f((1,2))]
cache(1, 2)           # miss -- TWO positional arguments, not the same call as f((1,2)); capacity
                      # exceeded, so f(1,b=2,c=3) (unused since step 1) is evicted
                      #                                    cache: [f((1,2)), f(1,2)]
cache((1, 2))          # hit -- touches f((1,2)) again, moving it to MRU
                        #                                    cache: [f(1,2), f((1,2))]
cache(1, 3)             # miss -- capacity exceeded again, so f(1,2) (never touched again after
                        # insertion) is evicted, not the more recently touched f((1,2)): a hit
                        # updates recency exactly like a miss
                        #                                    cache: [f((1,2)), f(1,3)]
```

### Part 2 — Durable cache

From here on `Memoize` is a given, with all three bugs fixed. `DurableMemoize(func, capacity,
path=None)` extends it so a crash — the process killed at an arbitrary point, with no chance to run
further code — loses no call that finished beforehand, and recovers a cache indistinguishable from the
one the process held at that instant. `path`, if given, is the base path this cache persists to and
recovers from; `path=None` disables persistence entirely, identical to `Memoize`. When `path` names an
existing store, recovery happens inside `__init__`, before it returns.

Persisting every call by rewriting the whole cache is disallowed: recording one call must cost time and
space proportional to that call alone, never to `capacity` or to the calls before it. This requires an
*append-only log*: a file only ever added to, never rewritten in place, until it is explicitly compacted
by `compact()` below.

Before `__call__` returns, the call's effect — a new entry on a miss, a recency update on a hit — is
*durable*: its record has been appended to the log, flushed, and `fsync`ed. A crash after that instant
can never lose it; a crash before it loses it completely, never partially, as if the call had not been
made. Constructing `DurableMemoize(func, capacity, path)` against such a `path` recovers a cache whose
entries, values, and recency order — least to most recently used — match exactly what the earlier
instance held after its last durable call, except that a call whose record never finished being written
counts as never made: recovery must detect a crash mid-record — from the record's declared length and a
checksum — and discard it and everything after it in the log, without raising or trusting any byte of
it. `hits` and `misses` are bookkeeping only, never persisted, and start at 0 in every new instance.

`compact()` may be called at any time; the cache never calls it on its own. After it returns, the amount
of data stored no longer depends on how many calls have been made since `path` was first used, only on
the (at most `capacity`) entries currently cached. Replacing the old stored state with the new one is
atomic: whichever instant the process crashes at during `compact()`, a subsequent recovery reproduces
either the state from just before it was called or the state it was about to commit, never a mixture,
and never loses an entry that was durable beforehand.

A positional or keyword argument that takes part in a key, and the value `func` returns, must be
*JSON-serialisable*: `None`, a `bool`, an `int`, a `float`, a `str`, or a `list`/`dict` built from these,
recursively (a `dict`'s keys must themselves be `str`). An argument's value round-trips exactly,
including a `tuple` nested inside it; a returned value is under the weaker restriction alone, so a
`tuple` inside it survives a crash and a restart as a `list` with the same elements instead, since JSON
keeps no separate record of which one it was.

```py
class DurableMemoize(Memoize):
    def __init__(self, func, capacity, path: str | None = None) -> None:
        """path is the base path this cache persists to and recovers from; None (the default)
        disables persistence entirely. Recovery, if path names an existing store, happens here,
        before __init__ returns."""

    def compact(self) -> None:
        """Rewrites the stored state so that its size no longer depends on call history, only on
        the entries currently cached. Safe to call at any time, including right before a crash."""
```

For example:

```text
w = DurableMemoize(lambda n: n * n, capacity=2, path="cache.log")
w(3)                     # miss -- durably records (3,) -> 9 before returning
w(4)                     # miss -- durably records (4,) -> 16;   cache (LRU -> MRU): [(3,), (4,)]
w(3)                     # hit -- durably records that (3,) was used again;   cache: [(4,), (3,)]

# the process is killed here, mid-way through durably recording the next call, w(5) -- the log
# on disk ends in a fragment: a well-formed record's declared length, followed by fewer bytes
# than that length promises

r = DurableMemoize(lambda n: n * n, capacity=2, path="cache.log")
# r replays the log: the records for w(3), w(4), and the touch of 3 are intact and applied; the
# trailing fragment fails its length/checksum check and is discarded. r's cache now maps n=4 to 16
# and n=3 to 9, in exactly that order from least to most recently used -- the order right before the crash
```

### Part 3 — Thread-safe cache

`ThreadSafeMemoize` extends `DurableMemoize` with the same observable contract under any number of
threads calling it concurrently — for the same key, for different keys, in any interleaving, under any
scheduling — plus one more guarantee: `func` is never running more than once at a time for a given key,
even when several threads call it with that key while no cached entry for it yet exists. A thread that
arrives while another is already computing that key's value waits for that result instead of starting a
second, redundant computation; it must not do so while holding any lock a call for an unrelated key
needs, since `func` may take an arbitrary, unknown amount of time (a network call, a slow computation),
and a cache that serialises unrelated keys behind it stops behaving like a cache under load. An
exception `func` raises propagates to every thread waiting on that call, and to none of the others;
nothing is cached or persisted for that call.

```py
class ThreadSafeMemoize(DurableMemoize):
    def __init__(self, func, capacity, path: str | None = None) -> None:
        """Same contract as DurableMemoize.__init__."""
```

For example:

```text
sf = ThreadSafeMemoize(fetch_profile, capacity=100, path="cache.log")

# Thread A: sf(user_id=7)   # miss -- becomes responsible for key (user_id=7,); calls fetch_profile
#                           # without holding any lock this cache keeps
# Thread B: sf(user_id=7)   # arrives while A's call is still running; finds (user_id=7,) already
#                           # being computed, and waits for A's result instead of calling it itself
# Thread C: sf(user_id=9)   # a different key -- proceeds immediately, never waiting on A or B

# fetch_profile(user_id=7) runs exactly once, however many threads called sf(user_id=7) while it
# was in flight; every one of them returns its result, and fetch_profile(user_id=9) runs freely
# at the same time as it
```

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two points are worth confirming before coding. First, whether a positional argument and an equal-valued
keyword argument to the same parameter should hit the same entry — here they do not, since
`generate_key` has no access to `func`'s signature (binding against `inspect.signature(func)` first
would unify them; see the follow-ups). Second, how strict "durable" must be: whether a crash may ever
be seen to have rewound the recency order (here it may not — a hit is exactly as durable as a miss, at
the `fsync` cost the complexity note in Part 2 gives) and whether values may be restricted to JSON
rather than `pickle` (chosen here, since unpickling runs arbitrary code for whatever bytes happen to be
on disk, so a record damaged by a crash cannot be safely inspected before deciding whether to trust it,
while a damaged JSON record just fails to parse).

### Part 1

Bug 1, in `generate_key`: `kwargs.items()` is folded into the key in whatever order Python iterates
it, which is insertion order — the order the caller happened to write the keywords in. Sorting by name
first removes the dependence: `sorted(kwargs.items())` compares its tuples element by element, and
since dictionary keys are unique, two tuples built this way are never compared past their first
element, so sorting never fails even when the argument values themselves are not orderable.

Bug 2, also in `generate_key`: folding `args` and `kwargs.items()` into one flat tuple loses the
boundary between them the moment that flat tuple happens to have exactly one element, because of the
`len(key) == 1` special case that returns `key[0]` unwrapped. `f(1, 2)` builds the two-element tuple
`(1, 2)` from `args` alone; `f((1, 2))` builds the one-element tuple `((1, 2),)`, which the special case
then unwraps to that very same `(1, 2)`. Keeping `args` and the sorted keyword pairs as two separate
elements of one outer tuple, instead of flattening them together, fixes this at the root (see the code
comment below).

Bug 3, in `__call__`: a hit returns `self.cache[key]` without calling `self.cache.move_to_end(key)`, so
the `OrderedDict`'s order only ever reflects *insertion*, never *use* — repeated hits on the same key
leave it exactly where it was first inserted, and `popitem(last=False)` then evicts whichever key was
inserted longest ago, which is FIFO, not LRU, the moment any key is used more than once.

```python
from collections import OrderedDict
from typing import Any, Callable


class Memoize:
    """Wraps func so that repeated calls with the same arguments return a cached result instead of
    calling func again. Holds at most capacity entries; a call that would add a new one beyond that
    evicts the least recently used entry first. Both a hit and a miss count as a "use" of the key
    they touch, for the purpose of deciding which entry is least recently used."""

    def __init__(self, func: Callable, capacity: int) -> None:
        self.func = func
        self.capacity = capacity
        self.cache: "OrderedDict[Any, Any]" = OrderedDict()
        self.hits = 0
        self.misses = 0

    def generate_key(self, *args: Any, **kwargs: Any) -> Any:
        # NOTE: args and the sorted kwargs stay two separate elements of one outer tuple, rather
        # than being flattened together, so no split between them can ever collide with another
        return (args, tuple(sorted(kwargs.items())))

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        key = self.generate_key(*args, **kwargs)
        if key in self.cache:
            self.hits += 1
            self.cache.move_to_end(key)  # NOTE: a hit is a use too -- without this the cache degrades to FIFO
            return self.cache[key]
        self.misses += 1
        result = self.func(*args, **kwargs)
        self.cache[key] = result
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)
        return result
```

`generate_key` and `__call__` are both $O(1)$ expected, dominated by hashing the key and a constant
number of dictionary operations; `sorted(kwargs.items())` costs $O(k \log k)$ in the number of keyword
arguments passed to that one call, never in `capacity`.

### Part 2

Each call appends one *record* to a single log file: a `PUT` record on a miss (key and value), a
lightweight `TOUCH` record on a hit (key alone — the value is already known and does not need to be
written again). Recovery replays the log in order on top of the most recent snapshot (empty, the first
time): a `PUT` sets the key's value and moves it to the most-recently-used end, evicting under the same
capacity rule `__call__` uses if that overflows; a `TOUCH` moves an already-present key to that end. No
record marks an eviction directly (see the follow-ups).

Each record is framed by a fixed 8-byte big-endian length and a 4-byte CRC-32 ahead of the payload — a
JSON document holding the operation and the key, plus the value for a `PUT`. The checksum covers the
length field, not just the payload: over the payload alone, a torn write of exactly 12 zero bytes — a
plausible shape for a crash mid-write — would pass as a valid, empty-payload record, since
`zlib.crc32(b"")` is `0` and matches a zeroed checksum field by coincidence; folding the length in makes
that same all-zero header fail its own checksum instead. A record is trusted only once its declared
length is followed by that many bytes and the checksum matches; replay stops at the first record that
fails, and the log is truncated back to the end of the last good one, since nothing past a bad fragment
would ever be reached by a later replay.

A key is `(args, sorted_kwargs)`, built from tuples; `json.dumps` already turns a nested `tuple` into a
JSON array, so encoding needs no special handling. Decoding must reverse that everywhere a tuple could
have been — a recovered key holding a `list` would be unhashable and could never equal the tuple a live
call to `generate_key` produces for the same arguments — so `_decode_key` turns every JSON array, at any
depth, back into a `tuple`. This can never mistake a genuine `list` argument for a `tuple` one, because
such an argument was already unusable here: `generate_key`'s result must be hashable, so a live call
with a `list` anywhere in its arguments would already raise `TypeError` from `key in self.cache`, before
persistence enters into it. A returned value is under no such requirement, since it is never a key, so
it is left exactly as `json.loads` produces it — why a `tuple` inside one survives a crash as a `list`.

```python
import json
import os
import zlib

PUT, TOUCH = "put", "touch"


def _decode_key(value: Any) -> Any:
    if isinstance(value, list):
        return tuple(_decode_key(item) for item in value)
    return value


def _encode_record(op: str, key: Any, value: Any = None) -> bytes:
    body = {"op": op, "key": key}
    if op == PUT:
        body["value"] = value
    payload = json.dumps(body, separators=(",", ":")).encode("utf-8")
    header = len(payload).to_bytes(8, "big")
    checksum = zlib.crc32(header + payload).to_bytes(4, "big")  # NOTE: covers the header too -- see above
    return header + checksum + payload


def _read_record(data: bytes, pos: int):
    """Returns (op, key, value, end) for the record starting at pos, or None if no complete,
    checksum-valid record starts there."""
    if pos + 12 > len(data):
        return None                                        # not even a full header and checksum remain
    length = int.from_bytes(data[pos:pos + 8], "big")
    end = pos + 12 + length
    if end > len(data):
        return None                                         # cut off mid-record, or a damaged (huge) length
    payload = data[pos + 12:end]
    if zlib.crc32(data[pos:pos + 8] + payload) != int.from_bytes(data[pos + 8:pos + 12], "big"):
        return None                                         # enough bytes, but not the ones that were written
    try:
        body = json.loads(payload)
        op, key = body["op"], _decode_key(body["key"])
    except (json.JSONDecodeError, UnicodeDecodeError, KeyError, TypeError):
        return None                                          # checksum-valid but not one of our records
    return op, key, body.get("value"), end


def _fsync_directory(path: str) -> None:
    dir_fd = os.open(os.path.dirname(os.path.abspath(path)) or ".", os.O_RDONLY)
    try:
        os.fsync(dir_fd)   # NOTE: the rename below is durable only once the directory entry is synced too
    finally:
        os.close(dir_fd)


class DurableMemoize(Memoize):
    def __init__(self, func: Callable, capacity: int, path: str | None = None) -> None:
        super().__init__(func, capacity)
        self.path = path
        self._log = None
        if path is not None:
            self._load()
            self._log = open(path, "ab")   # NOTE: kept open and appended to for the life of this instance

    def _snapshot_path(self) -> str:
        return self.path + ".snapshot"

    def _load(self) -> None:
        snapshot = self._snapshot_path()
        if os.path.exists(snapshot):
            with open(snapshot, "r", encoding="utf-8") as f:
                for key, value in json.load(f):          # oldest (LRU) first, exactly as compact() wrote it
                    self.cache[_decode_key(key)] = value
        if not os.path.exists(self.path):
            return
        with open(self.path, "rb") as f:
            data = f.read()
        pos = 0
        while (record := _read_record(data, pos)) is not None:
            op, key, value, pos = record
            if op == PUT:
                self.cache[key] = value
                self.cache.move_to_end(key)               # NOTE: reassigning an existing key does not reorder it
                if len(self.cache) > self.capacity:
                    self.cache.popitem(last=False)
            elif key in self.cache:                        # TOUCH; always true for a log this class wrote
                self.cache.move_to_end(key)
        if pos < len(data):                                # NOTE: cut the bad tail, or later appends hide behind it
            with open(self.path, "r+b") as f:
                f.truncate(pos)
                f.flush()
                os.fsync(f.fileno())

    def _append(self, op: str, key: Any, value: Any = None) -> None:
        if self._log is None:
            return
        self._log.write(_encode_record(op, key, value))
        self._log.flush()
        os.fsync(self._log.fileno())          # NOTE: the durability point -- __call__ does not return before this

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        key = self.generate_key(*args, **kwargs)
        if key in self.cache:
            self.hits += 1
            self.cache.move_to_end(key)
            self._append(TOUCH, key)
            return self.cache[key]
        self.misses += 1
        result = self.func(*args, **kwargs)
        self._append(PUT, key, result)         # NOTE: durable before the entry is even visible in memory
        self.cache[key] = result
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)
        return result

    def compact(self) -> None:
        if self.path is None:
            return
        snapshot = [[key, value] for key, value in self.cache.items()]   # oldest (LRU) first
        tmp = self._snapshot_path() + ".tmp"
        with open(tmp, "w", encoding="utf-8") as f:
            json.dump(snapshot, f, separators=(",", ":"))
            f.flush()
            os.fsync(f.fileno())
        os.replace(tmp, self._snapshot_path())   # NOTE: the commit point -- atomic on the same filesystem
        _fsync_directory(self._snapshot_path())
        self._log.close()
        open(self.path, "wb").close()             # NOTE: safe only once the snapshot above is durably committed
        self._log = open(self.path, "ab")
```

A crash between the rename and the truncation in `compact()` leaves the just-replaced log sitting next
to the new snapshot; the next recovery loads the snapshot, then replays that whole, now-redundant log on
top of it. That reproduces the very same state, not a different one: replaying the log twice in a row is
the same as replaying it once concatenated with itself, and a full replay's final order and evicted set
depend only on each key's *last* occurrence in the sequence — which falls at the same relative position
in that second copy as in the log alone, even though individual records reorder the cache differently
along the way. Neither `_load` nor `compact()` ever calls `func`, so at worst this wastes some parsing,
never returns a wrong answer.

`__call__` costs one `fsync` beyond `Memoize`'s $O(1)$, a round trip to durable storage on every single
call, hit or miss alike. `compact()` is $O(\text{capacity})$, dominated by writing out the entries
currently cached, and is never called from inside `__call__`, so it never adds its own cost to a call.

### Part 3

A single lock could wrap the entire call, `func` included — correct, and the simplest thread-safe
wrapper, but it serialises every call behind whichever one is running, including calls for already-cached
keys and keys nobody else is touching: `func`'s own cost (a network call, a slow computation) becomes
every other caller's cost too.

```python
import threading
import time


class NaiveLockedMemoize(DurableMemoize):
    """One lock around the whole call, func included -- correct, but no two calls ever overlap."""

    def __init__(self, func: Callable, capacity: int, path: str | None = None) -> None:
        super().__init__(func, capacity, path)
        self._lock = threading.Lock()

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        with self._lock:
            return super().__call__(*args, **kwargs)
```

The fix is to hold a lock only around the bookkeeping — checking the cache, appending to the log,
updating the `OrderedDict` — and run `func` outside it entirely. That alone reopens a race `Memoize`
never had to consider: two threads can both see a miss for the same key before either finishes
computing it, so both call `func` and both insert a result — correct, since the cache still ends up
valid, but wasteful, and wrong when `func`'s repeated calls are not free to duplicate (a payment, a
write to another system). Avoiding that — *single-flight* — needs one more piece of shared state: which
keys have a computation in progress, and a way for a later caller to wait on the first one's result
instead of starting its own.

`ThreadSafeMemoize` keeps one `threading.Event` per key currently being computed, in a dict guarded by
the same lock as the cache. A thread that finds its key already has one waits on it, without holding the
lock, and reads the leader's result once the event fires; a thread that finds none creates one, becomes
that key's leader, releases the lock, calls `func`, then reacquires the lock only to install the result,
evict if `capacity` demands it, and append the durability record — waking every waiter only afterwards,
so a waiter's return is exactly as durable as the leader's, though only the leader ever writes a log
record.

That log append shares the leader's critical section rather than a lock of its own because the log's
on-disk order is what recovery uses to reconstruct recency (a key's *last* record fixes its position), so
it must agree with the order calls actually took effect in memory: a lock dedicated only to the log,
acquired independently of the one serialising cache updates, could let two threads acquire the two in
opposite orders — one thread's cache update lands first while its log record lands second — and recovery
would then reconstruct a recency order the live cache never had. A per-key lock, created the first time
each key is touched, would not remove this shared lock either: an `OrderedDict`'s recency chain is
shared by every key, so protecting it still needs one lock spanning all of them (the follow-ups return
to what a per-key lock would buy instead).

```python
class _InFlight:
    """Coordinates one key's in-progress computation: a thread that is not the leader waits on
    event, then reads result (or re-raises exc)."""

    __slots__ = ("event", "result", "exc")

    def __init__(self) -> None:
        self.event = threading.Event()
        self.result: Any = None
        self.exc: BaseException | None = None


class ThreadSafeMemoize(DurableMemoize):
    def __init__(self, func: Callable, capacity: int, path: str | None = None) -> None:
        super().__init__(func, capacity, path)
        self._lock = threading.Lock()
        self._in_flight: dict[Any, _InFlight] = {}

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        key = self.generate_key(*args, **kwargs)
        with self._lock:
            if key in self.cache:
                self.hits += 1
                self.cache.move_to_end(key)
                self._append(TOUCH, key)
                return self.cache[key]
            waiting = self._in_flight.get(key)
            if waiting is None:
                waiting = self._in_flight[key] = _InFlight()
                leader = True
            else:
                leader = False

        if not leader:
            waiting.event.wait()                  # NOTE: outside the lock -- other keys keep making progress
            if waiting.exc is not None:
                raise waiting.exc
            return waiting.result

        try:
            result = self.func(*args, **kwargs)    # NOTE: outside the lock -- func's own latency is not serialised
        except BaseException as exc:
            with self._lock:
                del self._in_flight[key]
            waiting.exc = exc
            waiting.event.set()
            raise

        with self._lock:
            self.misses += 1
            self._append(PUT, key, result)         # NOTE: same critical section as the cache update -- see above
            self.cache[key] = result
            if len(self.cache) > self.capacity:
                self.cache.popitem(last=False)
            del self._in_flight[key]
        waiting.result = result
        waiting.event.set()                         # NOTE: only after the result is fully cached and durable
        return result
```

`__call__` is $O(1)$ expected on a hit and while waiting; a leader's own cost is `func`'s cost plus one
`fsync`, paid once regardless of how many threads coalesced onto it.

### Follow-ups

- Binding arguments against `inspect.signature(func).bind(*args, **kwargs).apply_defaults()` before
  hashing would unify `f(1, 2)` with `f(1, b=2)`, at the cost of a slower key and a real signature for
  `func` — unavailable when `func` is declared only as `(*args, **kwargs)`.
- An eviction needs no explicit record: replay derives every eviction from the `PUT`/`TOUCH` history
  under a fixed `capacity`, exactly as `__call__` does live. It would matter without `compact()`, where
  such a record would mark the point before which a key's older records are safe to skip.
- Batching several pending records and `fsync`ing once every few calls or milliseconds trades
  durability for throughput: up to that many calls can now be lost on a crash, so "no data loss" means
  "at most the last unflushed batch."
- A second writer process must not race the first: serialise both with an OS advisory lock
  (`fcntl.flock`) held across an append and its `fsync`, or route every call through one writer
  process that owns the log, with the others talking to it over a queue or a socket.
- A hit's cost is dominated by its `fsync` — a millisecond-scale round trip to storage regardless of
  lookup cost — so that path is I/O-bound; a miss is too whenever `func` does I/O, and CPU-bound only
  when `func` is pure computation.
- A per-key lock would shrink contention on the single-flight decision itself, worthwhile only under
  many concurrently-missing keys, since that step is already $O(1)$ and touches no disk; it would not
  touch eviction. Removing that last lock for good instead means sharding into independent LRUs, each
  with its own lock and a slice of `capacity` chosen by hashing the key — the standard fix once
  contention on the lock, not disk I/O, is the bottleneck.

<details>
<summary>Checks (runnable)</summary>

```python
import random

# the Part 1 worked example, asserted directly
trace = Memoize(lambda *args, **kwargs: (args, kwargs), capacity=2)
trace(1, b=2, c=3)                                     # miss
trace(1, c=3, b=2)                                     # hit: kwargs in a different order
trace((1, 2))                                          # miss: one positional argument, a tuple
trace(1, 2)                                            # miss: two positional arguments; evicts f(1,b=2,c=3)
trace((1, 2))                                          # hit: touches f((1,2)) again
trace(1, 3)                                            # miss: evicts f(1,2), not the more recently hit f((1,2))
assert trace.hits == 2 and trace.misses == 4
assert list(trace.cache.keys()) == [trace.generate_key((1, 2)), trace.generate_key(1, 3)]


class ModelMemoize:
    """Independent reference model built straight from the statement: a plain list of [key, value]
    pairs, oldest (LRU) first, never touching Memoize's own data structures."""

    def __init__(self, func: Callable, capacity: int) -> None:
        self.func = func
        self.capacity = capacity
        self.entries: list = []
        self.misses = 0

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        key = (args, tuple(sorted(kwargs.items())))
        for i, (k, v) in enumerate(self.entries):
            if k == key:
                self.entries.append(self.entries.pop(i))
                return v
        self.misses += 1
        value = self.func(*args, **kwargs)
        self.entries.append((key, value))
        if len(self.entries) > self.capacity:
            self.entries.pop(0)
        return value


def echo(*args: Any, **kwargs: Any) -> Any:
    return (args, kwargs)


NAMES = ["a", "b", "c"]
VALUES = [0, 1, 2, (1, 2), (3,), "x", None, 1.5]


def random_call(rng: random.Random):
    args = tuple(rng.choice(VALUES) for _ in range(rng.randint(0, 3)))
    names = rng.sample(NAMES, k=rng.randint(0, len(NAMES)))
    kwargs = {n: rng.choice(VALUES) for n in names}
    if kwargs and rng.random() < 0.5:               # sometimes rebuild kwargs in a different order
        items = list(kwargs.items())
        rng.shuffle(items)
        kwargs = dict(items)
    return args, kwargs


for trial in range(300):
    rng = random.Random(trial)
    capacity = rng.randint(1, 5)
    real, model = Memoize(echo, capacity), ModelMemoize(echo, capacity)
    for _ in range(60):
        args, kwargs = random_call(rng)
        assert real(*args, **kwargs) == model(*args, **kwargs)
    assert real.misses == model.misses
    assert list(real.cache.keys()) == [k for k, _ in model.entries]
print("Part 1 checks OK (", trial + 1, "random trials against a brute-force model )")
```

```python
import tempfile


def json_echo(*args: Any, **kwargs: Any) -> Any:
    """Like echo, but returns a JSON-native structure (no tuples), so a durable round trip is exact."""
    return [list(args), dict(kwargs)]


JSON_VALUES = [0, 1, 2, "x", "y", None, 1.5, True]   # no tuples: durability checks keep values JSON-exact


def random_json_call(rng: random.Random):
    args = tuple(rng.choice(JSON_VALUES) for _ in range(rng.randint(0, 3)))
    names = rng.sample(NAMES, k=rng.randint(0, len(NAMES)))
    kwargs = {n: rng.choice(JSON_VALUES) for n in names}
    if kwargs and rng.random() < 0.5:
        items = list(kwargs.items())
        rng.shuffle(items)
        kwargs = dict(items)
    return args, kwargs


# the Part 2 worked example, asserted directly: two misses, a hit, a clean reopen, then a crash mid-record
with tempfile.TemporaryDirectory() as d:
    p = os.path.join(d, "cache.log")
    w = DurableMemoize(lambda n: n * n, capacity=2, path=p)
    w(3)                                                # miss
    w(4)                                                # miss
    w(3)                                                # hit
    clean = DurableMemoize(lambda n: n * n, capacity=2, path=p)
    assert list(clean.cache.items()) == list(w.cache.items())
    assert clean.hits == 0 and clean.misses == 0        # bookkeeping counters are not persisted

    fifth_record = _encode_record(PUT, w.generate_key(5), 25)
    with open(p, "ab") as f:
        f.write(fifth_record[:len(fifth_record) // 2])  # crash mid-record, as in the example
    r = DurableMemoize(lambda n: n * n, capacity=2, path=p)
    assert list(r.cache.items()) == [(w.generate_key(4), 16), (w.generate_key(3), 9)]

    tuple_returning = DurableMemoize(lambda: (1, 2), capacity=1, path=os.path.join(d, "t.log"))
    tuple_returning()
    reread = DurableMemoize(lambda: (1, 2), capacity=1, path=os.path.join(d, "t.log"))
    only_value = next(iter(reread.cache.values()))
    assert only_value == [1, 2] and isinstance(only_value, list)   # a tuple survives as a list
print("Part 2 worked-example and basic checks OK")

# crash simulation: truncate the real, on-disk log at every byte offset and reopen. states[n] / ends[n]
# are an independent Memoize's cache and the on-disk log size after n calls, precomputed once.
with tempfile.TemporaryDirectory() as d:
    p = os.path.join(d, "cache.log")
    rng = random.Random(0)
    history, ref = DurableMemoize(json_echo, capacity=4, path=p), Memoize(json_echo, capacity=4)
    states, ends = [OrderedDict()], [0]
    for _ in range(50):
        a, k = random_json_call(rng)
        history(*a, **k)
        ref(*a, **k)
        states.append(OrderedDict(ref.cache))
        ends.append(os.path.getsize(p))
    full_log = open(p, "rb").read()

    for cut in range(len(full_log) + 1):
        with tempfile.TemporaryDirectory() as d2:
            p2 = os.path.join(d2, "cache.log")
            with open(p2, "wb") as f:
                f.write(full_log[:cut])
            n_complete = max(i for i, e in enumerate(ends) if e <= cut)
            recovered = DurableMemoize(json_echo, capacity=4, path=p2)
            assert list(recovered.cache.items()) == list(states[n_complete].items()), cut
            assert os.path.getsize(p2) <= cut          # never left holding bytes past the crash point
    print("Part 2 crash-truncation check OK (", len(full_log) + 1, "byte offsets )")

    garbage_rng = random.Random(99)
    for n in (1, 2, 8, 11, 12, 13, 40):               # garbage appended after the last good record
        for tail in (bytes(n), bytes(garbage_rng.randrange(256) for _ in range(n))):
            with tempfile.TemporaryDirectory() as d3:
                p3 = os.path.join(d3, "cache.log")
                with open(p3, "wb") as f:
                    f.write(full_log + tail)
                recovered = DurableMemoize(json_echo, capacity=4, path=p3)
                assert list(recovered.cache.items()) == list(states[-1].items())
                assert os.path.getsize(p3) == len(full_log)
    print("Part 2 garbage-tail check OK")

# compaction: same observable state, bounded size, and safe even if the crash lands between the
# snapshot rename and the log truncation, or during the temporary file write before either
with tempfile.TemporaryDirectory() as d:
    p = os.path.join(d, "cache.log")
    rng = random.Random(7)
    live = DurableMemoize(json_echo, capacity=5, path=p)
    for _ in range(30):
        a, k = random_json_call(rng)
        live(*a, **k)
    before_compact = list(live.cache.items())
    old_log_bytes = open(p, "rb").read()
    live.compact()
    assert list(live.cache.items()) == before_compact
    new_log_bytes = open(p, "rb").read()
    snapshot_bytes = open(p + ".snapshot", "rb").read()
    assert len(new_log_bytes) == 0
    assert len(snapshot_bytes) < len(old_log_bytes)   # no longer grows with the whole call history

    fresh = DurableMemoize(json_echo, capacity=5, path=p)
    assert list(fresh.cache.items()) == before_compact

    with tempfile.TemporaryDirectory() as d2:          # crash after the rename, before the truncation
        p2 = os.path.join(d2, "cache.log")
        open(p2, "wb").write(old_log_bytes)
        open(p2 + ".snapshot", "wb").write(snapshot_bytes)
        recovered = DurableMemoize(json_echo, capacity=5, path=p2)
        assert list(recovered.cache.items()) == before_compact

    with tempfile.TemporaryDirectory() as d3:          # crash while still writing the temp file
        p3 = os.path.join(d3, "cache.log")
        open(p3, "wb").write(old_log_bytes)
        open(p3 + ".snapshot.tmp", "wb").write(b"{not valid json, a crash landed here")
        recovered = DurableMemoize(json_echo, capacity=5, path=p3)
        assert list(recovered.cache.items()) == before_compact
print("Part 2 compaction checks OK")
```

```python
from collections import Counter

# timing: unrelated keys proceed concurrently only when func() is called outside the lock
def make_slow(delay: float):
    def slow(x):
        time.sleep(delay)
        return x * 2
    return slow


def timed_run(cls, n_keys: int, path: str) -> float:
    cache = cls(make_slow(DELAY), capacity=10, path=path)
    threads = [threading.Thread(target=cache, args=(i,)) for i in range(n_keys)]
    t0 = time.perf_counter()
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    return time.perf_counter() - t0


N_KEYS, DELAY = 6, 0.1
with tempfile.TemporaryDirectory() as d:
    naive_elapsed = timed_run(NaiveLockedMemoize, N_KEYS, os.path.join(d, "c.log"))
with tempfile.TemporaryDirectory() as d:
    fast_elapsed = timed_run(ThreadSafeMemoize, N_KEYS, os.path.join(d, "c.log"))

assert naive_elapsed > 1.8 * fast_elapsed, (naive_elapsed, fast_elapsed)  # generous margin -- see the text
print(f"Part 3 timing check OK (whole-call lock {naive_elapsed:.2f}s, single-flight {fast_elapsed:.2f}s)")

# single-flight: many threads request overlapping keys; func must run exactly once per key
KEYS = list(range(5))
call_counts: Counter = Counter()
count_lock = threading.Lock()


def counted_slow(x):
    with count_lock:
        call_counts[x] += 1
    time.sleep(0.03)
    return x * 2


with tempfile.TemporaryDirectory() as d:
    single_flight = ThreadSafeMemoize(counted_slow, capacity=10, path=os.path.join(d, "c.log"))
    n_threads = 40
    barrier = threading.Barrier(n_threads)
    results: list = [None] * n_threads
    errors: list = []

    def worker(i: int) -> None:
        key = KEYS[i % len(KEYS)]
        barrier.wait()                # every thread reaches sf(...) at effectively the same instant
        try:
            results[i] = single_flight(key)
        except Exception as exc:
            errors.append(exc)

    threads = [threading.Thread(target=worker, args=(i,)) for i in range(n_threads)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    assert not errors
    assert all(results[i] == KEYS[i % len(KEYS)] * 2 for i in range(n_threads))
    assert all(call_counts[k] == 1 for k in KEYS), call_counts     # exactly once per key, not once per call
    assert single_flight.misses == len(KEYS)
print("Part 3 single-flight check OK, invocation counts:", dict(call_counts))

# stress: many threads, overlapping keys, some calls raise; the cache stays within capacity and a
# fresh recovery afterwards matches the live cache exactly
def flaky(x):
    if x % 13 == 0:
        raise ValueError("boom")
    return x * x


with tempfile.TemporaryDirectory() as d:
    p = os.path.join(d, "c.log")
    stressed = ThreadSafeMemoize(flaky, capacity=7, path=p)
    errors = []

    def stress_worker(seed: int) -> None:
        rng = random.Random(seed)
        for _ in range(50):
            x = rng.randint(0, 25)
            try:
                assert stressed(x) == x * x
            except ValueError:
                pass
            except Exception as exc:
                errors.append(exc)

    threads = [threading.Thread(target=stress_worker, args=(s,)) for s in range(16)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    assert not errors
    assert len(stressed.cache) <= 7
    recovered = DurableMemoize(flaky, capacity=7, path=p)
    assert list(recovered.cache.items()) == list(stressed.cache.items())
print("Part 3 stress check OK")

print("all checks passed")
```

</details>

</details>
