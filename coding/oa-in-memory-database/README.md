# Online Assessment: In-Memory Database

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Online assessment · 4 levels | ★★★☆☆ | Medium | SWE · MLE · Infra Eng | oop-design, hash-table, ttl, snapshotting | 4 levels / 90 min | Online assessment |
<!-- meta:end -->

## Problem

Implement the four levels below in order: a level's tests must pass before the next level's tests
run, and every level extends the same class rather than replacing it. A *record* is identified by a
string `key` and holds a set of *fields*, each a string field name mapped to a string value; a key, a
field name, and a value are each non-empty strings, and a field name or a value never contains `(`,
`)`, or `,`. Every method returns a string; one with nothing useful to report returns `""`.

### Level 1 — Core operations

```py
class InMemoryDatabase:
    def set(self, key: str, field: str, value: str) -> str:
        """Sets field of the record key to value, creating the record if key is new and
        overwriting any value already held there. Returns ""."""

    def get(self, key: str, field: str) -> str:
        """Returns the value held in field of key, or "" if key does not exist or holds no
        such field."""

    def delete(self, key: str, field: str) -> str:
        """Removes field from key if present. Returns "true" if a field was removed, "false"
        if key did not exist or held no such field."""
```

Example:

```text
db = InMemoryDatabase()
db.set("order-1", "status", "packed")     # -> ""
db.set("order-1", "carrier", "fedex")     # -> ""
db.get("order-1", "status")               # -> "packed"
db.get("order-1", "weight")               # -> ""            -- field never set
db.get("order-9", "status")               # -> ""            -- key never created
db.delete("order-1", "carrier")           # -> "true"
db.delete("order-1", "carrier")           # -> "false"        -- already gone
db.set("order-1", "status", "shipped")    # -> ""             -- overwrites
db.get("order-1", "status")               # -> "shipped"
```

### Level 2 — Scan

```py
class InMemoryDatabase:
    def scan(self, key: str) -> str:
        """Returns every field of key formatted as "name1(value1), name2(value2), ...", with
        fields sorted lexicographically by name (Python's default string ordering); "" if key
        does not exist or holds no fields."""

    def scan_by_prefix(self, key: str, prefix: str) -> str:
        """Same as scan, restricted to the fields of key whose name starts with prefix
        (prefix="" behaves exactly like scan)."""
```

Example:

```text
db = InMemoryDatabase()
db.set("order-1", "carrier", "fedex")
db.set("order-1", "cost", "12.50")
db.set("order-1", "status", "packed")
db.scan_by_prefix("order-1", "c")   # -> "carrier(fedex), cost(12.50)"
db.scan("order-1")                  # -> "carrier(fedex), cost(12.50), status(packed)"
db.scan_by_prefix("order-9", "c")   # -> ""                              -- key never created
```

### Level 3 — Timestamps and TTL

Every call from this level on happens at an integer *timestamp*; across the whole sequence of calls
made to one instance, timestamps never decrease, whether or not they are given explicitly. Level 1's
and Level 2's five methods keep working exactly as specified above, but internally they now happen at
the current value of a clock that starts at $0$ and advances by $1$ after every such call. Each of the
five also gains a twin whose name ends in `_at` and which takes an explicit `timestamp` as its first
argument instead of reading the clock; a call to one of these sets the clock to `timestamp`, which is
never smaller than the clock's current value. Both kinds of call share this one clock, so a field
written by a plain `set` can be read by `get_at`, and one written by `set_at_with_ttl` can expire
under a later plain `get`.

```py
class InMemoryDatabase:
    def set_at(self, timestamp: int, key: str, field: str, value: str) -> str:
        """Same as set, at timestamp. Clears any TTL field previously held, even one that has
        not expired yet -- the field becomes permanent."""

    def set_at_with_ttl(self, timestamp: int, key: str, field: str, value: str, ttl: int) -> str:
        """Same as set_at, but field is visible from timestamp up to, but not including,
        timestamp + ttl (ttl is a positive integer). A later set_at_with_ttl call on the same
        field replaces this window outright, computed fresh from the new call; it does not
        extend the old one."""

    def get_at(self, timestamp: int, key: str, field: str) -> str:
        """Same as get, at timestamp: a field whose TTL window has elapsed by timestamp counts
        as absent."""

    def delete_at(self, timestamp: int, key: str, field: str) -> str:
        """Same as delete, at timestamp; a field whose TTL window has elapsed by timestamp
        counts as already absent, so this returns "false" for it."""

    def scan_at(self, timestamp: int, key: str) -> str:
        """Same as scan, at timestamp."""

    def scan_by_prefix_at(self, timestamp: int, key: str, prefix: str) -> str:
        """Same as scan_by_prefix, at timestamp."""
```

Example:

```text
db = InMemoryDatabase()
db.set_at_with_ttl(10, "order-1", "carrier", "fedex", 5)  # carrier alive on [10, 15)
db.set_at_with_ttl(12, "order-1", "carrier", "ups", 6)    # replaced: window becomes [12, 18), not extended to 21
db.set_at(13, "order-1", "cost", "12.50")                  # no ttl -> cost is permanent
db.scan_at(17, "order-1")                # -> "carrier(ups), cost(12.50)"   17 < 18: carrier still alive
db.scan_at(18, "order-1")                # -> "cost(12.50)"                 18 == 18: carrier just expired
db.delete_at(18, "order-1", "carrier")   # -> "false"    already expired -- nothing to delete
db.set_at_with_ttl(19, "order-1", "carrier", "usps", 3)    # carrier alive again, on [19, 22)
db.set_at(20, "order-1", "carrier", "dhl")                 # no ttl -> clears the TTL, now permanent
db.scan_at(22, "order-1")                # -> "carrier(dhl), cost(12.50)"   22 is the old expiry, but it was cleared
```

### Level 4 — Backup and restore

```py
class InMemoryDatabase:
    def backup(self, timestamp: int) -> str:
        """Deep-snapshots every field currently live at timestamp under Level 3's rules. A
        field with a TTL is stored in the snapshot as the *remaining* time until its expiry
        (its own expiry minus timestamp), not as an absolute expiry; a permanent field is
        stored as permanent. A second backup at the same timestamp replaces the first.
        Returns ""."""

    def restore(self, timestamp: int, timestamp_to_restore: int) -> str:
        """Finds the backup with the largest own timestamp that is <= timestamp_to_restore (at
        least one such backup is guaranteed to have been made) and makes it the database's
        entire live state as of timestamp, discarding every write and delete made since --
        including ones made by an earlier restore. Every field of the restored snapshot that
        carried a remaining-TTL delta d gets a new expiry of timestamp + d: its countdown
        resumes from the moment of this restore, not from the moment of the original backup or
        of the field's original write. No backup is ever deleted, so a later restore call may
        still reach a backup made after this one. Returns ""."""
```

Example:

```text
db = InMemoryDatabase()
db.set_at_with_ttl(0, "order-1", "carrier", "fedex", 10)  # alive on [0, 10)
db.set_at(0, "order-1", "cost", "12.50")                   # permanent
db.backup(4)                              # -> ""   snapshot: carrier has 6 remaining (10 - 4), cost permanent
db.set_at(5, "order-1", "cost", "15.00")  # the live database changes after the backup; the snapshot does not
db.delete_at(6, "order-1", "carrier")     # -> "true"   carrier was alive at 6 (6 < 10)
db.restore(100, 4)                        # -> ""   restores the ts=4 backup as of timestamp 100
db.get_at(100, "order-1", "cost")         # -> "12.50"   back to the pre-backup value
db.get_at(100, "order-1", "carrier")      # -> "fedex"    restored, with 6 ticks left, resumed from 100
db.get_at(105, "order-1", "carrier")      # -> "fedex"    100 + 6 = 106: still alive at 105
db.get_at(106, "order-1", "carrier")      # -> ""         106 is the new expiry -- exclusive again
```

## Reference solution

<details>
<summary>Show the reference solution</summary>

Worth confirming up front: what a missing key or field should return from `get` (here `""`, matching
`delete` and `scan`'s use of `""` for "nothing here"); and, once TTLs exist, whether Level 1's and
Level 2's plain methods stay in the interface or are dropped in favor of the timestamped ones (here
they stay, driven by an internal clock, since a grader that gates levels keeps re-running the earlier
levels' tests once a later level unlocks).

### Level 1

A record is a plain `dict[str, str]`; the whole database is a `dict[str, dict[str, str]]` from key to
record. `set` uses `setdefault` so a missing key gets an empty record on first use; `get` chains two
`.get` calls with `""` as the default at each step, which folds "key missing" and "field missing" into
one line; `delete` looks the field up once and only removes it if present, so a repeated call
correctly answers `"false"`.

```python
class InMemoryDatabase:
    def __init__(self) -> None:
        self._records: dict[str, dict[str, str]] = {}

    def set(self, key: str, field: str, value: str) -> str:
        self._records.setdefault(key, {})[field] = value
        return ""

    def get(self, key: str, field: str) -> str:
        return self._records.get(key, {}).get(field, "")

    def delete(self, key: str, field: str) -> str:
        fields = self._records.get(key, {})
        if field in fields:
            del fields[field]
            return "true"
        return "false"
```

Each method is $O(1)$.

### Level 2

`scan` and `scan_by_prefix` share one formatting step, `_format_fields`: sort the field names that
pass the filter — all of them for `scan`, those starting with `prefix` for the other — then join
`"name(value)"` pieces with `", "`. Filtering before sorting, rather than sorting the whole record and
then filtering, keeps `scan_by_prefix` to one pass over the record plus a sort of only the matches.

```python
def _format_fields(fields: dict[str, str], names) -> str:
    return ", ".join(f"{name}({fields[name]})" for name in names)


class InMemoryDatabase(InMemoryDatabase):
    def scan(self, key: str) -> str:
        fields = self._records.get(key, {})
        return _format_fields(fields, sorted(fields))

    def scan_by_prefix(self, key: str, prefix: str) -> str:
        fields = self._records.get(key, {})
        return _format_fields(fields, sorted(f for f in fields if f.startswith(prefix)))
```

Reopening `InMemoryDatabase` as its own subclass, rather than editing Level 1's block, is how the rest
of this solution extends it level by level. Both methods cost $O(m \log m)$ in the record's field
count $m$: Python's standard library has no sorted string index to seek `startswith` matches out of
directly.

### Level 3

Level 3 refactors the storage rather than adding to it: keeping only the current value per field stops
being enough once a field can expire, so each field becomes a `(value, expiry)` pair, with
`expiry=None` for a permanent field and an absolute timestamp otherwise. Because the record dict's
value type changes, Level 1's and Level 2's methods are overridden here, not inherited unchanged —
inheriting `get`, for instance, would return the raw `(value, expiry)` tuple instead of `value`. An
internal clock, `_clock`, gives the plain methods a timestamp to work with: `_tick` reads and advances
it for a plain call, `_bump` folds in an explicit one. `_alive` checks only the upper bound of a
field's window — the lower bound of $[t, t+ttl)$ never needs a check of its own, because every write
already happened at some earlier point in the same non-decreasing sequence of timestamps, so a query
can never arrive before the write it is reading.

```python
class InMemoryDatabase(InMemoryDatabase):
    def __init__(self) -> None:
        # _records[key][field] = (value, expiry); expiry is None (permanent) or the first
        # timestamp at which the field is gone.
        self._records: dict[str, dict[str, tuple[str, int | None]]] = {}
        self._clock = 0

    def _tick(self) -> int:
        timestamp = self._clock
        self._clock += 1
        return timestamp

    def _bump(self, timestamp: int) -> None:
        self._clock = max(self._clock, timestamp)   # NOTE: max, not "=" -- defends the ordering guarantee

    @staticmethod
    def _alive(entry: tuple[str, int | None] | None, timestamp: int) -> bool:
        return entry is not None and (entry[1] is None or timestamp < entry[1])   # NOTE: strict "<": t+ttl is dead

    def set(self, key: str, field: str, value: str) -> str:
        return self.set_at(self._tick(), key, field, value)

    def get(self, key: str, field: str) -> str:
        return self.get_at(self._tick(), key, field)

    def delete(self, key: str, field: str) -> str:
        return self.delete_at(self._tick(), key, field)

    def scan(self, key: str) -> str:
        return self.scan_at(self._tick(), key)

    def scan_by_prefix(self, key: str, prefix: str) -> str:
        return self.scan_by_prefix_at(self._tick(), key, prefix)

    def set_at(self, timestamp: int, key: str, field: str, value: str) -> str:
        self._bump(timestamp)
        self._records.setdefault(key, {})[field] = (value, None)   # NOTE: plain-setting always clears a TTL
        return ""

    def set_at_with_ttl(self, timestamp: int, key: str, field: str, value: str, ttl: int) -> str:
        self._bump(timestamp)
        self._records.setdefault(key, {})[field] = (value, timestamp + ttl)   # NOTE: a fresh window, not extended
        return ""

    def get_at(self, timestamp: int, key: str, field: str) -> str:
        self._bump(timestamp)
        entry = self._records.get(key, {}).get(field)
        return entry[0] if self._alive(entry, timestamp) else ""

    def delete_at(self, timestamp: int, key: str, field: str) -> str:
        self._bump(timestamp)
        fields = self._records.get(key, {})
        if self._alive(fields.get(field), timestamp):
            del fields[field]
            return "true"
        return "false"                                            # NOTE: an already-expired field is "false"

    def _live_fields(self, key: str, timestamp: int) -> dict[str, str]:
        fields = self._records.get(key, {})
        return {name: value for name, (value, expiry) in fields.items()
                if self._alive((value, expiry), timestamp)}

    def scan_at(self, timestamp: int, key: str) -> str:
        self._bump(timestamp)
        live = self._live_fields(key, timestamp)
        return _format_fields(live, sorted(live))

    def scan_by_prefix_at(self, timestamp: int, key: str, prefix: str) -> str:
        self._bump(timestamp)
        live = self._live_fields(key, timestamp)
        return _format_fields(live, sorted(f for f in live if f.startswith(prefix)))
```

This is lazy expiry: liveness is checked on every read instead of being swept out on a timer or on
every write. It is enough here because nothing in this class needs to know how many fields exist
without reading them first, so an eager purge would save only memory, and that memory is reclaimed
anyway the next time `set_at` or `set_at_with_ttl` overwrites the same field. An eager sweep would
need a background thread, or a check squeezed into every call, to buy back a saving that lazy expiry
gets for free.

### Level 4

`backup` takes a copy of exactly what `_alive` already says is visible at `timestamp`, reusing it
rather than re-deriving liveness; the copy stores each field's *remaining* TTL, not its absolute
expiry, which is the detail `restore` depends on. `restore` looks up the right backup with `max` over
a generator expression, rebuilds every field's expiry relative to the restore's own `timestamp`, and
replaces `_records` wholesale: a field created after the backup was taken is simply not in the
snapshot, so it is absent from the rebuilt dict with no separate deletion step.

```python
import copy


class InMemoryDatabase(InMemoryDatabase):
    def __init__(self) -> None:
        super().__init__()
        self._backups: dict[int, dict[str, dict[str, tuple[str, int | None]]]] = {}

    def backup(self, timestamp: int) -> str:
        self._bump(timestamp)
        snapshot: dict[str, dict[str, tuple[str, int | None]]] = {}
        for key, fields in self._records.items():
            live = {name: (value, None if expiry is None else expiry - timestamp)
                    for name, (value, expiry) in fields.items() if self._alive((value, expiry), timestamp)}
            if live:
                snapshot[key] = live
        self._backups[timestamp] = snapshot         # NOTE: a second backup at this timestamp replaces the first
        return ""

    def restore(self, timestamp: int, timestamp_to_restore: int) -> str:
        self._bump(timestamp)
        backup_ts = max(ts for ts in self._backups if ts <= timestamp_to_restore)
        restored: dict[str, dict[str, tuple[str, int | None]]] = {}
        for key, fields in self._backups[backup_ts].items():
            restored[key] = {name: (value, None if delta is None else timestamp + delta)   # NOTE: delta, not the
                              for name, (value, delta) in fields.items()}                   #  backup's own clock
        self._records = copy.deepcopy(restored)      # NOTE: deep copy -- later writes must not mutate the backup
        return ""
```

Both methods cost $O(\text{size of the database})$: `backup` walks every live field once, and
`restore` rebuilds the whole record dict from the chosen snapshot. Storing the remaining TTL instead
of the absolute expiry is what makes a restored field's countdown resume from `timestamp`: storing the
expiry itself would carry the backup's own clock reading forward untouched, so a field backed up with
$6$ ticks left could come back already expired, or with far more than $6$ ticks left, depending only
on how much time had passed before the restore — exactly backward from what "resume from the moment
of the restore" means.

### Follow-ups

- A look-back variant replaces `backup`/`restore` with a single historical-query method that takes a
  key, a field and a past `timestamp` and answers directly from an append-only per-field history via
  binary search on the write timestamps; it answers one field's past value in $O(\log n)$ without ever
  materializing a snapshot, but has no equivalent of `restore`'s "make this the live state again,"
  since rolling every key back would mean re-running that search over every field that ever existed.
- Making this thread-safe needs one lock held around the whole read-check-write sequence inside each
  `_at` method, not just around the dictionary access inside it: two threads racing on
  `set_at_with_ttl` for the same field could otherwise both read the old entry before either writes
  the new one.
- `backup` and `restore` here copy the whole live database; a store too large to copy on every backup
  would instead diff against the previous backup, or use a persistent, structurally-shared data
  structure, so a snapshot costs proportional to what changed rather than to the database's whole
  size.
- A size cap on a key, field, or value, as most real key-value stores impose, would need `set`,
  `set_at`, and `set_at_with_ttl` to validate and reject an oversized write, with an exact return
  value of their own for that case.

<details>
<summary>Checks (runnable)</summary>

```python
# ---- Level 1 example ----
db = InMemoryDatabase()
assert db.set("order-1", "status", "packed") == ""
assert db.set("order-1", "carrier", "fedex") == ""
assert db.get("order-1", "status") == "packed"
assert db.get("order-1", "weight") == ""
assert db.get("order-9", "status") == ""
assert db.delete("order-1", "carrier") == "true"
assert db.delete("order-1", "carrier") == "false"
assert db.set("order-1", "status", "shipped") == ""
assert db.get("order-1", "status") == "shipped"

# ---- Level 2 example ----
db = InMemoryDatabase()
db.set("order-1", "carrier", "fedex")
db.set("order-1", "cost", "12.50")
db.set("order-1", "status", "packed")
assert db.scan_by_prefix("order-1", "c") == "carrier(fedex), cost(12.50)"
assert db.scan("order-1") == "carrier(fedex), cost(12.50), status(packed)"
assert db.scan_by_prefix("order-9", "c") == ""

# ---- Level 3 example ----
db = InMemoryDatabase()
assert db.set_at_with_ttl(10, "order-1", "carrier", "fedex", 5) == ""
assert db.set_at_with_ttl(12, "order-1", "carrier", "ups", 6) == ""
assert db.set_at(13, "order-1", "cost", "12.50") == ""
assert db.scan_at(17, "order-1") == "carrier(ups), cost(12.50)"
assert db.scan_at(18, "order-1") == "cost(12.50)"
assert db.delete_at(18, "order-1", "carrier") == "false"
assert db.set_at_with_ttl(19, "order-1", "carrier", "usps", 3) == ""
assert db.set_at(20, "order-1", "carrier", "dhl") == ""
assert db.scan_at(22, "order-1") == "carrier(dhl), cost(12.50)"

# ---- Level 4 example ----
db = InMemoryDatabase()
db.set_at_with_ttl(0, "order-1", "carrier", "fedex", 10)
db.set_at(0, "order-1", "cost", "12.50")
assert db.backup(4) == ""
db.set_at(5, "order-1", "cost", "15.00")
db.delete_at(6, "order-1", "carrier")
assert db.restore(100, 4) == ""
assert db.get_at(100, "order-1", "cost") == "12.50"
assert db.get_at(100, "order-1", "carrier") == "fedex"
assert db.get_at(105, "order-1", "carrier") == "fedex"
assert db.get_at(106, "order-1", "carrier") == ""
print("all four levels' worked examples replayed exactly")

# ---- edge cases the examples do not reach ----
db = InMemoryDatabase()
assert db.delete("ghost", "field") == "false"
assert db.scan("ghost") == ""
db.set("k", "alpha", "1")
db.set("k", "beta", "2")
assert db.scan_by_prefix("k", "") == db.scan("k") == "alpha(1), beta(2)"

# plain and _at calls share one clock
db = InMemoryDatabase()
db.set("k", "a", "1")                                 # plain: happens at internal tick 0
assert db.get_at(50, "k", "a") == "1"                  # a later explicit read still sees it
assert db.set_at_with_ttl(60, "k", "b", "2", 3) == ""  # b alive on [60, 63)
for _ in range(3):
    db.get("k", "z")                                   # three plain calls, unrelated field: clock 60 -> 63
assert db.get("k", "b") == ""                          # this plain call queries at exactly 63: b just expired

# a field that never had a TTL survives arbitrarily far into the future
db = InMemoryDatabase()
db.set_at(0, "k", "permanent", "x")
assert db.get_at(10 ** 6, "k", "permanent") == "x"

# multiple backups: restore keeps every one of them reachable, including ones made after an earlier restore
db = InMemoryDatabase()
db.set_at(0, "k", "a", "v0")
db.backup(10)
db.set_at(11, "k", "a", "v1")
db.backup(20)
db.set_at(21, "k", "a", "v2")
assert db.restore(30, 15) == ""                        # -> the backup at ts=10 (the latest <= 15)
assert db.get_at(30, "k", "a") == "v0"
assert db.restore(40, 25) == ""                        # -> the backup at ts=20, still there after the earlier restore
assert db.get_at(40, "k", "a") == "v1"

# a restore discards a field created after the backup, with no explicit delete needed
db = InMemoryDatabase()
db.set_at(0, "k", "a", "v0")
db.backup(5)
db.set_at(6, "k", "brand-new", "x")
db.restore(10, 5)
assert db.get_at(10, "k", "brand-new") == ""
assert db.scan_at(10, "k") == "a(v0)"
print("edge cases OK")
```

```python
import random


class NaiveDatabase:
    """Independent reference model, straight from the statement: every (key, field) is an append-only
    list of (ts, value, ttl) events in call order, where value=None marks a delete. A query answers
    by scanning the whole list for the last event at or before the query time -- "the value is
    whatever it was last set to, alive while ts < set_ts + ttl" -- with no lazy-expiry cleverness."""

    def __init__(self) -> None:
        self.history: dict[tuple[str, str], list[tuple[int, str | None, int | None]]] = {}
        self.backups: dict[int, dict[tuple[str, str], tuple[str, int | None]]] = {}
        self.clock = 0

    def _tick(self) -> int:
        ts = self.clock
        self.clock += 1
        return ts

    def _bump(self, ts: int) -> None:
        self.clock = max(self.clock, ts)

    def _last(self, key: str, field: str, ts: int):
        chosen = None
        for event in self.history.get((key, field), []):
            if event[0] <= ts:
                chosen = event
        return chosen

    def _value(self, key: str, field: str, ts: int) -> str | None:
        chosen = self._last(key, field, ts)
        if chosen is None or chosen[1] is None:
            return None
        e_ts, value, ttl = chosen
        return None if (ttl is not None and ts >= e_ts + ttl) else value

    def _write(self, ts, key, field, value, ttl=None) -> None:
        self.history.setdefault((key, field), []).append((ts, value, ttl))

    def set_at(self, ts, key, field, value) -> str:
        self._bump(ts)
        self._write(ts, key, field, value)
        return ""

    def set_at_with_ttl(self, ts, key, field, value, ttl) -> str:
        self._bump(ts)
        self._write(ts, key, field, value, ttl)
        return ""

    def get_at(self, ts, key, field) -> str:
        self._bump(ts)
        value = self._value(key, field, ts)
        return value if value is not None else ""

    def delete_at(self, ts, key, field) -> str:
        self._bump(ts)
        existed = self._value(key, field, ts) is not None
        self._write(ts, key, field, None)
        return "true" if existed else "false"

    def _live_names(self, key, ts) -> list[str]:
        return sorted(f for (k, f) in self.history if k == key and self._value(k, f, ts) is not None)

    def scan_at(self, ts, key) -> str:
        self._bump(ts)
        names = self._live_names(key, ts)
        return ", ".join(f"{f}({self._value(key, f, ts)})" for f in names)

    def scan_by_prefix_at(self, ts, key, prefix) -> str:
        self._bump(ts)
        names = [f for f in self._live_names(key, ts) if f.startswith(prefix)]
        return ", ".join(f"{f}({self._value(key, f, ts)})" for f in names)

    def backup(self, ts) -> str:
        self._bump(ts)
        snap = {}
        for pair in self.history:
            value = self._value(*pair, ts)
            if value is not None:
                e_ts, _, ttl = self._last(*pair, ts)
                snap[pair] = (value, None if ttl is None else e_ts + ttl - ts)
        self.backups[ts] = snap
        return ""

    def restore(self, ts, ts_to_restore) -> str:
        self._bump(ts)
        best = max(b for b in self.backups if b <= ts_to_restore)
        snapshot = self.backups[best]
        for pair in list(self.history):
            if pair in snapshot:
                value, remaining = snapshot[pair]
                self._write(ts, pair[0], pair[1], value, remaining)
            else:
                self._write(ts, pair[0], pair[1], None)
        return ""

    def set(self, key, field, value) -> str:
        return self.set_at(self._tick(), key, field, value)

    def get(self, key, field) -> str:
        return self.get_at(self._tick(), key, field)

    def delete(self, key, field) -> str:
        return self.delete_at(self._tick(), key, field)

    def scan(self, key) -> str:
        return self.scan_at(self._tick(), key)

    def scan_by_prefix(self, key, prefix) -> str:
        return self.scan_by_prefix_at(self._tick(), key, prefix)


def _run_trial(seed: int, n_ops: int = 150) -> dict[str, int]:
    rng = random.Random(seed)
    real, naive = InMemoryDatabase(), NaiveDatabase()
    keys, fields, values = [f"k{i}" for i in range(3)], [f"f{i}" for i in range(4)], [f"v{i}" for i in range(5)]
    sim_clock, backup_tss = 0, []
    counts = {"plain": 0, "at": 0, "ttl": 0, "backup": 0, "restore": 0, "delete_hit": 0}

    def explicit_ts() -> int:
        nonlocal sim_clock
        sim_clock += rng.randint(0, 3)
        return sim_clock

    for _ in range(n_ops):
        pick, key, field, value = rng.random(), rng.choice(keys), rng.choice(fields), rng.choice(values)
        if pick < 0.12:
            counts["plain"] += 1
            method = rng.choice(["set", "get", "delete", "scan", "scan_by_prefix"])
            args = {"set": (key, field, value), "get": (key, field), "delete": (key, field),
                    "scan": (key,), "scan_by_prefix": (key, field[:1])}[method]
            got, want = getattr(real, method)(*args), getattr(naive, method)(*args)
            assert got == want, (seed, method, args, got, want)
            counts["delete_hit"] += method == "delete" and want == "true"
            sim_clock += 1                                    # mirrors the plain call's own tick
        elif pick < 0.55:
            counts["at"] += 1
            ts = explicit_ts()
            method = rng.choice(["set_at", "get_at", "delete_at", "scan_at", "scan_by_prefix_at"])
            args = {"set_at": (ts, key, field, value), "get_at": (ts, key, field), "delete_at": (ts, key, field),
                    "scan_at": (ts, key), "scan_by_prefix_at": (ts, key, field[:1])}[method]
            got, want = getattr(real, method)(*args), getattr(naive, method)(*args)
            assert got == want, (seed, method, args, got, want)
            counts["delete_hit"] += method == "delete_at" and want == "true"
        elif pick < 0.75:
            counts["ttl"] += 1
            ts, ttl = explicit_ts(), rng.randint(1, 6)
            assert real.set_at_with_ttl(ts, key, field, value, ttl) == naive.set_at_with_ttl(ts, key, field, value, ttl) == ""
        elif pick < 0.85:
            counts["backup"] += 1
            ts = explicit_ts()
            assert real.backup(ts) == naive.backup(ts) == ""
            backup_tss.append(ts)
        elif backup_tss:
            counts["restore"] += 1
            ts = explicit_ts()
            ts_to_restore = rng.choice(backup_tss + [rng.randint(min(backup_tss), max(backup_tss) + 2)])
            assert real.restore(ts, ts_to_restore) == naive.restore(ts, ts_to_restore) == ""
    return counts


totals = {"plain": 0, "at": 0, "ttl": 0, "backup": 0, "restore": 0, "delete_hit": 0}
for seed in range(300):
    trial = _run_trial(seed)
    totals = {k: totals[k] + trial[k] for k in totals}
assert min(totals.values()) > 50, totals   # every kind of call, including a delete that actually removed something
print(f"cross-validated 300 random sequences against an independent reference model: {totals}")
print("all checks passed")
```

</details>

</details>
