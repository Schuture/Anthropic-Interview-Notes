# Online Assessment: Cloud File Storage

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Online assessment · 4 levels | ★★☆☆☆ | Medium | SWE · MLE | oop-design, hash-map, storage-quotas, string-matching | 4 levels / 90 min | Online assessment |
<!-- meta:end -->

## Problem

Implement a `CloudStorage` class that tracks files by name and size only: the bytes of a file are
never stored or read, only how many of them it has. The class is built up over four levels, in order;
each level's tests must pass before the next level's methods are exercised, and every level adds
methods to the same `CloudStorage` class the levels before it defined, never a new one.

A *file name* is a nonempty string of ASCII letters, digits, `.`, `_` and `-`; names are
case-sensitive, so `"Notes.txt"` and `"notes.txt"` name different files. At most one file may exist
under a given name at any moment, across the whole store, regardless of who — if anyone — owns it
(ownership starts at Level 3). A file's size is a positive integer number of bytes that never changes
while that name exists: copying, compressing and decompressing (Level 4) each create a differently
named file instead of changing an existing one in place. A *user id* is a nonempty string built from
the same characters as a file name; user ids and file names are separate namespaces, so the same
string may serve as both without conflict. Every call in this problem passes a name, id or size that
already satisfies these rules.

### Level 1 — Files, copies, sizes

```py
class CloudStorage:
    def add_file(self, name: str, size: int) -> bool:
        """Creates a file called name with this size. Fails, returning False and changing nothing, if
        a file called name already exists; otherwise creates it and returns True."""

    def copy_file(self, source: str, destination: str) -> bool:
        """Creates a file called destination with the same size as source; source is unaffected and
        continues to exist as before. Fails, returning False and changing nothing, if source does not
        currently exist or if destination already exists; otherwise creates destination and returns
        True."""

    def get_file_size(self, name: str) -> int | None:
        """Returns the current size of name, or None if no file called name currently exists."""
```

For example:

```text
cs = CloudStorage()
cs.add_file("report.txt", 200)                   # -> True
cs.add_file("report.txt", 10)                    # -> False  -- "report.txt" already exists
cs.copy_file("report.txt", "report_backup.txt")  # -> True   -- report_backup.txt now exists, size 200
cs.copy_file("missing.txt", "x.txt")             # -> False  -- source does not exist
cs.copy_file("report.txt", "report_backup.txt")  # -> False  -- destination already exists
cs.get_file_size("report_backup.txt")            # -> 200
cs.get_file_size("nope.txt")                     # -> None
```

### Level 2 — Search by prefix and suffix

`find_files(prefix, suffix)` searches every file currently in the store, owned or not, for those
whose name starts with `prefix` and ends with `suffix`, in the sense of `str.startswith` and
`str.endswith`: either argument may be `""`, which matches every name on that side, and a short name
can satisfy both at once, as `"ab"` does for `prefix="ab"` and `suffix="ab"`. Each match is formatted
as `f"{name}({size})"` — for instance `"report.txt(200)"` — and the returned list is sorted by size
descending, then by name ascending among files of the same size.

```py
    def find_files(self, prefix: str, suffix: str) -> list[str]:
        """Returns "name(size)" for every file currently in the store whose name starts with prefix
        and ends with suffix, sorted by size descending and, among files of the same size, by name
        ascending."""
```

Continuing the store above:

```text
cs.add_file("report_draft.txt", 200)
cs.add_file("report_final.txt", 500)
cs.add_file("summary.md", 50)
cs.find_files("report", ".txt")
# ["report_final.txt(500)", "report.txt(200)", "report_backup.txt(200)", "report_draft.txt(200)"]
# size 500 first; the three files tied at 200 follow in name order -- "." sorts before "_" in ASCII,
# so "report.txt" < "report_backup.txt" < "report_draft.txt"; "summary.md" does not start with "report"
cs.find_files("summary", "")
# ["summary.md(50)"]   -- suffix="" matches every name, and prefix "summary" rules out every report_*.txt
```

### Level 3 — User accounts and storage quotas

A *user* has a *capacity*, a positive integer number of bytes fixed when `add_user` creates the
account, and owns zero or more files; a file's size counts against its owner's capacity for as long
as the file exists, and a user's *remaining capacity* is its capacity minus the sizes of every file it
currently owns. A file created by `add_file` has no owner and never counts against anyone's capacity.
Every call to a method introduced from here on passes a `user_id` and a `capacity` that already
satisfy these rules.

```py
    def add_user(self, user_id: str, capacity: int) -> bool:
        """Registers user_id with this capacity and no files. Fails, returning False, if user_id is
        already a user; otherwise returns True."""

    def add_file_by(self, user_id: str, name: str, size: int) -> int | None:
        """Creates a file called name with this size, owned by user_id. Fails, returns None, changing
        nothing, if user_id is not currently a user, if a file called name already exists (owned or
        not), or if size exceeds user_id's remaining capacity; otherwise creates the file and returns
        user_id's remaining capacity after this addition."""

    def merge_user(self, target: str, source: str) -> int | None:
        """Moves every file source currently owns to target (names and sizes unchanged) and adds
        source's capacity to target's. Fails, returns None, changing nothing, if target equals source
        or if target or source is not currently a user. Otherwise source is removed as a user --
        afterwards, user id source refers to no one, as if add_user had never been called for it,
        until a later add_user gives that id a fresh account -- and the call returns target's
        remaining capacity after the merge."""
```

From this level on, `copy_file` gives `destination` the same owner as `source` (no owner, if `source`
has none). When `source` has an owner, `destination`'s size counts against that owner's remaining
capacity: `copy_file` now also fails, returns `False`, changing nothing, if the owner's remaining
capacity is less than `source`'s size, even when `source` exists and `destination` is free.

Continuing the store above:

```text
cs.add_user("alice", 500)                          # -> True
cs.add_user("alice", 100)                          # -> False  -- "alice" already exists
cs.add_file_by("alice", "alice_a.txt", 300)        # -> 200  -- remaining: 500 - 300
cs.add_file_by("alice", "report.txt", 10)          # -> None  -- "report.txt" already exists, unowned
cs.add_file_by("carol", "c.txt", 10)               # -> None  -- "carol" is not a user
cs.add_file_by("alice", "alice_b.txt", 250)        # -> None  -- 250 exceeds alice's remaining 200
cs.get_file_size("alice_a.txt")                    # -> 300   -- unaffected by the three failed calls above
cs.copy_file("alice_a.txt", "alice_a_copy.txt")    # -> False -- would need 300 more, alice has 200 left
cs.copy_file("report_final.txt", "report_final_copy.txt")  # -> True -- unowned source: no capacity check
cs.add_file_by("alice", "alice_small.txt", 100)    # -> 100   -- remaining: 200 - 100
cs.copy_file("alice_small.txt", "alice_small_copy.txt")     # -> True -- needs exactly 100, alice has exactly 100
cs.get_file_size("alice_small_copy.txt")           # -> 100
cs.add_file_by("alice", "alice_c.txt", 1)          # -> None  -- alice now has 0 bytes left
cs.add_user("bob", 50)                             # -> True
cs.add_file_by("bob", "bob_notes.txt", 20)         # -> 30
cs.merge_user("alice", "bob")                      # -> 30    -- alice: capacity 500+50, used 500+20
cs.add_file_by("bob", "bob_notes2.txt", 5)         # -> None  -- "bob" is no longer a user
cs.merge_user("alice", "bob")                      # -> None  -- source "bob" no longer exists
cs.get_file_size("bob_notes.txt")                  # -> 20    -- the file still exists, now owned by alice
cs.merge_user("alice", "alice")                    # -> None  -- target equals source
cs.merge_user("dave", "alice")                     # -> None  -- "dave" is not a user
```

### Level 4 — Compression, backup and restore

Level 4 adds four methods; the first two only ever act on a file `user_id` currently owns. Let
`COMPRESSED_SUFFIX` denote the fixed string `".cmp"`.

```py
    def compress_file(self, user_id: str, name: str) -> str | None:
        """Replaces name, owned by user_id, with a smaller file: removes name and creates a file
        called name + COMPRESSED_SUFFIX, still owned by user_id, whose size is name's previous size
        divided by two and rounded up. Fails, returns None, changing nothing, if user_id is not
        currently a user, if name does not currently exist or is not currently owned by user_id, or
        if name + COMPRESSED_SUFFIX already names an existing file; otherwise returns the new file's
        name. May be called on a name that already ends with COMPRESSED_SUFFIX, which compresses it
        again."""

    def decompress_file(self, user_id: str, name: str) -> str | None:
        """The reverse of compress_file: removes name and creates a file called name with its
        trailing COMPRESSED_SUFFIX removed, still owned by user_id, whose size is exactly double
        name's previous size. Fails, returns None, changing nothing, if user_id is not currently a
        user, if name does not currently exist or is not currently owned by user_id, if name does
        not currently end with COMPRESSED_SUFFIX, if the name with COMPRESSED_SUFFIX removed already
        names an existing file, or if name's current size exceeds user_id's remaining capacity;
        otherwise returns the new file's name."""

    def backup_user(self, user_id: str) -> bool:
        """Records, as user_id's one backup, the name and size of every file user_id currently owns
        and user_id's current capacity, replacing any earlier backup. Fails, returns False, if
        user_id is not currently a user; otherwise returns True."""

    def restore_user(self, user_id: str) -> int | None:
        """Resets user_id's files and capacity to its most recent backup_user(user_id): a name
        user_id owns now but the backup does not list is deleted; a name the backup lists that
        user_id does not currently own is re-created at its backed-up size; a name owned at both
        times is left exactly as it is now. Fails, returns None, changing nothing, if user_id is not
        a user, has never been backed up, or a name to be created would collide with a file this
        restore does not itself remove; otherwise resets user_id's capacity to the backed-up value
        and returns the resulting remaining capacity."""
```

The rounding in `compress_file` is the previous size divided by two, rounded up
($\lceil \text{size} / 2 \rceil$), so a file of size 1 compresses to a file of size 1, never to size
0. `decompress_file` doubles exactly, with no rounding, so it does not always recover the exact size
from before a matching `compress_file` when that size was odd. `decompress_file`'s capacity check
compares `name`'s current, compressed size against `user_id`'s remaining capacity computed with
`name` still counted at that size — not against the doubled size the call is about to produce.

`restore_user` compares by name only, not by size: a name `user_id` owned both at backup time and now
is left untouched even if its size has changed, which can happen in the unusual case where that name
was freed — by `compress_file` or `decompress_file` — and then given to an unrelated file by a later
call; `restore_user` does not notice, and the unrelated file is what remains. A backup belongs to a
user id only for as long as that id keeps referring to the same account: merging a user away, or
later giving its freed id a new account with `add_user`, leaves nothing for `restore_user` to find.
Restoring does not consume the backup: the same backup can be restored more than once.

Continuing the store above:

```text
cs.add_user("erin", 100)                     # -> True
cs.add_file_by("erin", "photo.raw", 41)      # -> 59
cs.backup_user("erin")                       # -> True  -- snapshot: {photo.raw: 41}, capacity 100
cs.compress_file("erin", "photo.raw")        # -> "photo.raw.cmp"  -- size: ceil(41 / 2) = 21
cs.get_file_size("photo.raw")                # -> None  -- freed by the compression
cs.get_file_size("photo.raw.cmp")            # -> 21
cs.add_file_by("erin", "logo.png", 30)       # -> 49    -- remaining: 100 - 21 - 30
cs.restore_user("erin")                      # -> 59    -- back to {photo.raw: 41}, capacity 100
cs.get_file_size("photo.raw")                # -> 41    -- re-created by the restore
cs.get_file_size("photo.raw.cmp")            # -> None  -- not in the snapshot, so deleted
cs.get_file_size("logo.png")                 # -> None  -- added after the backup, so deleted

cs.add_user("frank", 60)
cs.add_file_by("frank", "video.raw", 25)          # -> 35
cs.compress_file("frank", "video.raw")            # -> "video.raw.cmp"  -- size: ceil(25 / 2) = 13
cs.decompress_file("frank", "video.raw.cmp")      # -> "video.raw"  -- size: 13 * 2 = 26, not the original 25
cs.get_file_size("video.raw")                     # -> 26

cs.add_user("gina", 25)
cs.add_file_by("gina", "clip.raw", 25)            # -> 0
cs.compress_file("gina", "clip.raw")              # -> "clip.raw.cmp"  -- size: ceil(25 / 2) = 13, remaining 12
cs.decompress_file("gina", "clip.raw.cmp")        # -> None  -- needs 13 more bytes of capacity, only 12 remain
```

## Reference solution

<details>
<summary>Show the reference solution</summary>

Worth confirming up front: that file content is genuinely out of scope, so a size is the only thing
tracked, and that `find_files`'s prefix and suffix are meant literally as `str.startswith` and
`str.endswith`, not a glob or a regular expression.

### Level 1

The store is one `dict` from name to size; every method is a direct lookup or write.

```python
class CloudStorage:
    def __init__(self):
        self._files: dict[str, int] = {}

    def add_file(self, name: str, size: int) -> bool:
        if name in self._files:
            return False
        self._files[name] = size
        return True

    def copy_file(self, source: str, destination: str) -> bool:
        if source not in self._files or destination in self._files:
            return False
        self._files[destination] = self._files[source]
        return True

    def get_file_size(self, name: str) -> int | None:
        return self._files.get(name)
```

All three run in $O(1)$.

### Level 2

`find_files` filters `_files` in one pass and sorts the matches; negating the size sorts it
descending while `sorted` keeps names ascending, so a single key does both at once.

```python
def find_files(self, prefix: str, suffix: str) -> list[str]:
    matches = [(name, size) for name, size in self._files.items()
               if name.startswith(prefix) and name.endswith(suffix)]
    matches.sort(key=lambda item: (-item[1], item[0]))    # NOTE: size descending, then name ascending
    return [f"{name}({size})" for name, size in matches]


CloudStorage.find_files = find_files
```

Filtering the whole store is $O(n)$ for $n$ files, and sorting the $k$ matches adds $O(k \log k)$.

### Level 3

Ownership and capacity live in two more dicts alongside `_files`: `_owner` names the user who owns a
file, absent for a file `add_file` created, and `_used` tracks each user's total bytes charged so
far, so a remaining capacity is one subtraction away.

```python
_init_l2 = CloudStorage.__init__


def __init__(self):
    _init_l2(self)
    self._owner: dict[str, str] = {}      # file name -> owning user id (absent: unowned)
    self._capacity: dict[str, int] = {}   # user id -> total capacity
    self._used: dict[str, int] = {}       # user id -> bytes currently charged to it


def _remaining(self, user_id: str) -> int:
    return self._capacity[user_id] - self._used[user_id]


def add_user(self, user_id: str, capacity: int) -> bool:
    if user_id in self._capacity:
        return False
    self._capacity[user_id] = capacity
    self._used[user_id] = 0
    return True


def add_file_by(self, user_id: str, name: str, size: int) -> int | None:
    if user_id not in self._capacity or name in self._files:
        return None
    if size > self._remaining(user_id):        # NOTE: capacity checked before any mutation
        return None
    self._files[name] = size
    self._owner[name] = user_id
    self._used[user_id] += size
    return self._remaining(user_id)


def copy_file(self, source: str, destination: str) -> bool:
    if source not in self._files or destination in self._files:
        return False
    size = self._files[source]
    owner = self._owner.get(source)
    if owner is not None and size > self._remaining(owner):    # NOTE: a copy counts against SOURCE's owner
        return False
    self._files[destination] = size
    if owner is not None:
        self._owner[destination] = owner
        self._used[owner] += size
    return True


def merge_user(self, target: str, source: str) -> int | None:
    if target == source or target not in self._capacity or source not in self._capacity:
        return None                              # NOTE: forbids a self-merge that would double one's own capacity
    for name, owner in self._owner.items():
        if owner == source:
            self._owner[name] = target
    self._capacity[target] += self._capacity[source]
    self._used[target] += self._used[source]
    del self._capacity[source]
    del self._used[source]
    return self._remaining(target)


CloudStorage.__init__ = __init__
CloudStorage._remaining = _remaining
CloudStorage.add_user = add_user
CloudStorage.add_file_by = add_file_by
CloudStorage.copy_file = copy_file
CloudStorage.merge_user = merge_user
```

`merge_user` adds the two capacities and the two used-totals separately, rather than recomputing
either from the files themselves; the result can never make `_remaining(target)` negative, because
each user's own used total never exceeded their own capacity before the merge, and the sum of two
such inequalities is the same inequality for the sums.

`add_user`, `add_file_by` and `copy_file` are $O(1)$; `merge_user` is $O(f)$ in the number of files
the whole store currently holds, since it scans every entry of `_owner` to find `source`'s files.

### Level 4

`compress_file` and `decompress_file` both remove one file and insert another, adjusting `_used` by
the difference between the two sizes. `backup_user` copies out exactly the pairs `restore_user` will
need; `restore_user` treats that snapshot as the desired state and the current ownership as the
actual one, and reconciles the two with a set difference on names alone — a name present on both
sides is left untouched, which is what makes the freed-and-reused-name case described above possible.

```python
COMPRESSED_SUFFIX = ".cmp"

_init_l3 = CloudStorage.__init__


def __init__(self):
    _init_l3(self)
    self._backups: dict[str, tuple[int, dict[str, int]]] = {}    # user id -> (capacity, {name: size})


def compress_file(self, user_id: str, name: str) -> str | None:
    if user_id not in self._capacity or self._owner.get(name) != user_id:   # NOTE: also covers "no such file"
        return None
    new_name = name + COMPRESSED_SUFFIX
    if new_name in self._files:
        return None                              # NOTE: compressed-name collision
    size = self._files.pop(name)
    del self._owner[name]
    self._used[user_id] -= size
    new_size = -(-size // 2)                     # NOTE: ceiling division -- never rounds size 1 down to 0
    self._files[new_name] = new_size
    self._owner[new_name] = user_id
    self._used[user_id] += new_size              # NOTE: never exceeds capacity: new_size <= size always
    return new_name


def decompress_file(self, user_id: str, name: str) -> str | None:
    if user_id not in self._capacity or self._owner.get(name) != user_id:
        return None
    if not name.endswith(COMPRESSED_SUFFIX):
        return None
    new_name = name[:-len(COMPRESSED_SUFFIX)]
    if new_name in self._files:
        return None
    size = self._files[name]
    if size > self._remaining(user_id):          # NOTE: name itself still counts here -- see the level text
        return None
    del self._files[name]
    del self._owner[name]
    self._used[user_id] -= size
    new_size = size * 2
    self._files[new_name] = new_size
    self._owner[new_name] = user_id
    self._used[user_id] += new_size
    return new_name


def backup_user(self, user_id: str) -> bool:
    if user_id not in self._capacity:
        return False
    owned = {name: size for name, size in self._files.items() if self._owner.get(name) == user_id}
    self._backups[user_id] = (self._capacity[user_id], owned)     # NOTE: replaces any earlier backup
    return True


def restore_user(self, user_id: str) -> int | None:
    if user_id not in self._capacity or user_id not in self._backups:
        return None
    snap_capacity, snap_files = self._backups[user_id]
    current = {name for name, owner in self._owner.items() if owner == user_id}
    to_delete = current - snap_files.keys()       # owned now, not in the snapshot
    to_create = snap_files.keys() - current        # in the snapshot, not owned now
    if any(name in self._files for name in to_create):    # NOTE: every collision checked before any mutation
        return None
    for name in to_delete:
        del self._files[name]
        del self._owner[name]
    for name in to_create:
        self._files[name] = snap_files[name]
        self._owner[name] = user_id
    self._capacity[user_id] = snap_capacity
    # NOTE: recomputed from what user_id actually owns now, NOT sum(snap_files.values()) -- a name kept
    # untouched because it was owned at backup time and still is now is not re-verified against its
    # backed-up size, so the snapshot's own total can be stale (see the level text).
    self._used[user_id] = sum(size for name, size in self._files.items() if self._owner.get(name) == user_id)
    return self._remaining(user_id)


CloudStorage.__init__ = __init__
CloudStorage.compress_file = compress_file
CloudStorage.decompress_file = decompress_file
CloudStorage.backup_user = backup_user
CloudStorage.restore_user = restore_user
```

$\lceil s/2 \rceil \le s$ for every $s \ge 1$, so `compress_file` can only reduce what its owner has
used and never needs a capacity check; `decompress_file` increases it by exactly the current,
pre-decompression `size` (the doubled file replaces one already counted at `size`), which is exactly
what is compared against `_remaining`. `restore_user` checks every `to_create` name against the whole
store, not only against `user_id`'s own files, and does so before deleting or creating anything, so a
restore that cannot fully succeed leaves every file and every capacity exactly as it was.

`compress_file` and `decompress_file` are $O(1)$; `backup_user` and `restore_user` are $O(f)$ in the
number of files the store currently holds.

### Follow-ups

- Why do `merge_user`, `backup_user` and `restore_user` each scan every file in the store instead of
  just the ones one user owns? A per-user `dict[str, set[str]]` of owned names, kept current at every
  mutation site, would make each of them linear in that user's own files instead.
- What if a user needed more than one backup at a time? `_backups[user_id]` would become a dict keyed
  by a caller-supplied label, and `restore_user` would take that label as a second argument.
- Why does `decompress_file` not always undo a matching `compress_file` exactly? Ceiling division
  loses the low bit of an odd size, and doubling cannot bring it back; a lossless scheme would need to
  store that bit somewhere, such as in the compressed file's own size being odd or even.
- File content is never stored, so `compress_file`'s ratio is fixed at exactly one half. A production
  version would take the compressed size, or a ratio measured from the actual bytes, as an argument
  instead of assuming a constant.

<details>
<summary>Checks (runnable)</summary>

```python
# ---- replay every level's worked example exactly ----
cs = CloudStorage()
assert cs.add_file("report.txt", 200) is True
assert cs.add_file("report.txt", 10) is False
assert cs.copy_file("report.txt", "report_backup.txt") is True
assert cs.copy_file("missing.txt", "x.txt") is False
assert cs.copy_file("report.txt", "report_backup.txt") is False
assert cs.get_file_size("report_backup.txt") == 200
assert cs.get_file_size("nope.txt") is None

assert cs.add_file("report_draft.txt", 200) is True
assert cs.add_file("report_final.txt", 500) is True
assert cs.add_file("summary.md", 50) is True
assert cs.find_files("report", ".txt") == [
    "report_final.txt(500)", "report.txt(200)", "report_backup.txt(200)", "report_draft.txt(200)",
]
assert cs.find_files("summary", "") == ["summary.md(50)"]

assert cs.add_user("alice", 500) is True
assert cs.add_user("alice", 100) is False
assert cs.add_file_by("alice", "alice_a.txt", 300) == 200
assert cs.add_file_by("alice", "report.txt", 10) is None
assert cs.add_file_by("carol", "c.txt", 10) is None
assert cs.add_file_by("alice", "alice_b.txt", 250) is None
assert cs.get_file_size("alice_a.txt") == 300
assert cs.copy_file("alice_a.txt", "alice_a_copy.txt") is False
assert cs.copy_file("report_final.txt", "report_final_copy.txt") is True
assert cs.add_file_by("alice", "alice_small.txt", 100) == 100
assert cs.copy_file("alice_small.txt", "alice_small_copy.txt") is True
assert cs.get_file_size("alice_small_copy.txt") == 100
assert cs.add_file_by("alice", "alice_c.txt", 1) is None
assert cs.add_user("bob", 50) is True
assert cs.add_file_by("bob", "bob_notes.txt", 20) == 30
assert cs.merge_user("alice", "bob") == 30
assert cs.add_file_by("bob", "bob_notes2.txt", 5) is None
assert cs.merge_user("alice", "bob") is None
assert cs.get_file_size("bob_notes.txt") == 20
assert cs.merge_user("alice", "alice") is None
assert cs.merge_user("dave", "alice") is None

assert cs.add_user("erin", 100) is True
assert cs.add_file_by("erin", "photo.raw", 41) == 59
assert cs.backup_user("erin") is True
assert cs.compress_file("erin", "photo.raw") == "photo.raw.cmp"
assert cs.get_file_size("photo.raw") is None
assert cs.get_file_size("photo.raw.cmp") == 21
assert cs.add_file_by("erin", "logo.png", 30) == 49
assert cs.restore_user("erin") == 59
assert cs.get_file_size("photo.raw") == 41
assert cs.get_file_size("photo.raw.cmp") is None
assert cs.get_file_size("logo.png") is None

assert cs.add_user("frank", 60) is True
assert cs.add_file_by("frank", "video.raw", 25) == 35
assert cs.compress_file("frank", "video.raw") == "video.raw.cmp"
assert cs.decompress_file("frank", "video.raw.cmp") == "video.raw"
assert cs.get_file_size("video.raw") == 26

assert cs.add_user("gina", 25) is True
assert cs.add_file_by("gina", "clip.raw", 25) == 0
assert cs.compress_file("gina", "clip.raw") == "clip.raw.cmp"
assert cs.decompress_file("gina", "clip.raw.cmp") is None
print("level examples OK")

# ---- edge cases the examples do not reach ----
cs2 = CloudStorage()
cs2.add_user("u", 100)
cs2.add_file_by("u", "a", 10)

cs2.add_file("system.log", 5)                        # unowned
assert cs2.compress_file("u", "system.log") is None    # not owned by u
assert cs2.compress_file("u", "missing") is None       # does not exist
cs2.add_user("v", 100)
cs2.add_file_by("v", "vfile", 4)
assert cs2.compress_file("u", "vfile") is None          # owned by v, not u

cs2.add_file("a" + COMPRESSED_SUFFIX, 999)              # occupies the name compress_file("u", "a") would need
assert cs2.compress_file("u", "a") is None
assert cs2.get_file_size("a") == 10                     # the failed call changed nothing

cs2.add_file_by("u", "tiny", 1)
assert cs2.compress_file("u", "tiny") == "tiny.cmp"
assert cs2.get_file_size("tiny.cmp") == 1               # ceil(1 / 2) == 1, never 0

cs2.add_file_by("u", "b", 9)
assert cs2.compress_file("u", "b") == "b.cmp"
assert cs2.get_file_size("b.cmp") == 5                  # ceil(9 / 2)
assert cs2.compress_file("u", "b.cmp") == "b.cmp.cmp"   # compressing an already-compressed name chains the suffix
assert cs2.get_file_size("b.cmp.cmp") == 3              # ceil(5 / 2)

cs2.add_file_by("u", "plain", 6)
assert cs2.decompress_file("u", "plain") is None         # does not end with COMPRESSED_SUFFIX
assert cs2.decompress_file("u", "missing.cmp") is None   # does not exist
assert cs2.decompress_file("v", "b.cmp.cmp") is None      # owned by u, not v

cs2.add_file("b.cmp", 1)                                 # a fresh, unrelated file happens to reuse the freed name
assert cs2.decompress_file("u", "b.cmp.cmp") is None      # "b.cmp" already exists -- collision, not decompressed
assert cs2.get_file_size("b.cmp.cmp") == 3                # unchanged

print("compress/decompress edge cases OK")

# backup / restore edge cases
cs3 = CloudStorage()
cs3.add_user("w", 50)
assert cs3.restore_user("w") is None                      # never backed up
cs3.add_file_by("w", "f1", 10)
assert cs3.backup_user("w") is True                        # snapshot #1: {f1: 10}, capacity 50
cs3.add_file_by("w", "f2", 5)
assert cs3.backup_user("w") is True                         # snapshot #2 replaces #1: {f1: 10, f2: 5}, capacity 50
cs3.add_file_by("w", "f3", 1)
assert cs3.restore_user("w") == 35                           # back to snapshot #2: capacity 50, used 10 + 5
assert cs3.get_file_size("f1") == 10 and cs3.get_file_size("f2") == 5 and cs3.get_file_size("f3") is None
assert cs3.restore_user("w") == 35                           # restoring again from the same, un-consumed backup

cs3.add_user("x", 10)
assert cs3.merge_user("w", "x") == 45                        # x owns nothing: only capacity moves, 35 + 10
assert cs3.add_user("x", 20) is True                          # "x" is free again -- a brand new account
assert cs3.restore_user("x") is None                          # the new "x" has no backup of its own

cs3.add_file_by("w", "f4", 1)
assert cs3.backup_user("w") is True
cs3.add_file_by("w", "f4b", 2)                                 # occupies no snapshot name; will just be deleted
cs3.compress_file("w", "f1")                                    # frees "f1", creates "f1.cmp"
cs3.add_user("y", 5)
cs3.add_file_by("y", "f1", 3)                                    # someone else claims the freed name "f1"
assert cs3.restore_user("w") is None                              # restoring "f1" would collide with y's file
assert cs3.get_file_size("f1.cmp") is not None and cs3.get_file_size("f4b") == 2  # unchanged: the restore was aborted

cs4 = CloudStorage()
cs4.add_user("z", 20)
cs4.add_file_by("z", "doc", 10)
cs4.backup_user("z")                          # snapshot: {doc: 10}, capacity 20
cs4.compress_file("z", "doc")                  # frees "doc"
cs4.add_file_by("z", "doc", 3)                  # an unrelated file reuses the freed name, at a different size
assert cs4.restore_user("z") == 17               # capacity 20, used 3 -- "doc" is z's both at backup and now
assert cs4.get_file_size("doc") == 3              # left exactly as it is now, not reset to the backed-up size 10
assert cs4.get_file_size("doc.cmp") is None        # deleted: not in the snapshot
print("backup/restore edge cases OK")
```

```python
# ---- random cross-validation against an independent, unoptimized model ----
import random


class SlowCloudStorage:
    """A direct model of the statement: every file is a [name, size, owner] record in one list,
    scanned from scratch by every method. Shares no code or data structure with CloudStorage."""

    def __init__(self):
        self.files = []
        self.users = {}
        self.backups = {}

    def _find(self, name):
        for record in self.files:
            if record[0] == name:
                return record
        return None

    def _remaining(self, user_id):
        used = sum(size for _, size, owner in self.files if owner == user_id)
        return self.users[user_id] - used

    def add_file(self, name, size):
        if self._find(name) is not None:
            return False
        self.files.append([name, size, None])
        return True

    def copy_file(self, source, destination):
        src = self._find(source)
        if src is None or self._find(destination) is not None:
            return False
        owner = src[2]
        if owner is not None and src[1] > self._remaining(owner):
            return False
        self.files.append([destination, src[1], owner])
        return True

    def get_file_size(self, name):
        record = self._find(name)
        return record[1] if record else None

    def find_files(self, prefix, suffix):
        matches = [(n, s) for n, s, _ in self.files if n.startswith(prefix) and n.endswith(suffix)]
        matches.sort(key=lambda ns: (-ns[1], ns[0]))
        return [f"{n}({s})" for n, s in matches]

    def add_user(self, user_id, capacity):
        if user_id in self.users:
            return False
        self.users[user_id] = capacity
        return True

    def add_file_by(self, user_id, name, size):
        if user_id not in self.users or self._find(name) is not None:
            return None
        if size > self._remaining(user_id):
            return None
        self.files.append([name, size, user_id])
        return self._remaining(user_id)

    def merge_user(self, target, source):
        if target == source or target not in self.users or source not in self.users:
            return None
        for record in self.files:
            if record[2] == source:
                record[2] = target
        self.users[target] += self.users[source]
        del self.users[source]
        return self._remaining(target)

    def compress_file(self, user_id, name):
        record = self._find(name)
        if user_id not in self.users or record is None or record[2] != user_id:
            return None
        new_name = name + COMPRESSED_SUFFIX
        if self._find(new_name) is not None:
            return None
        self.files.remove(record)
        new_size = -(-record[1] // 2)
        self.files.append([new_name, new_size, user_id])
        return new_name

    def decompress_file(self, user_id, name):
        record = self._find(name)
        if user_id not in self.users or record is None or record[2] != user_id:
            return None
        if not name.endswith(COMPRESSED_SUFFIX):
            return None
        new_name = name[:-len(COMPRESSED_SUFFIX)]
        if self._find(new_name) is not None:
            return None
        if record[1] > self._remaining(user_id):
            return None
        self.files.remove(record)
        self.files.append([new_name, record[1] * 2, user_id])
        return new_name

    def backup_user(self, user_id):
        if user_id not in self.users:
            return False
        owned = {n: s for n, s, o in self.files if o == user_id}
        self.backups[user_id] = (self.users[user_id], owned)
        return True

    def restore_user(self, user_id):
        if user_id not in self.users or user_id not in self.backups:
            return None
        snap_capacity, snap_files = self.backups[user_id]
        current = {n for n, s, o in self.files if o == user_id}
        to_create = set(snap_files) - current
        if any(self._find(n) is not None for n in to_create):
            return None
        self.files = [r for r in self.files if not (r[2] == user_id and r[0] not in snap_files)]
        for n in sorted(to_create):                     # NOTE: sorted -- iteration order must not leak into state
            self.files.append([n, snap_files[n], user_id])
        self.users[user_id] = snap_capacity
        return self._remaining(user_id)


NAMES = ["f0", "f1", "f2", "f3", "f0" + COMPRESSED_SUFFIX, "f1" + COMPRESSED_SUFFIX]
USERS = ["u0", "u1", "u2"]
METHODS = ["add_file", "copy_file", "get_file_size", "find_files", "add_user", "add_file_by",
           "merge_user", "compress_file", "decompress_file", "backup_user", "restore_user"]


def random_call(rng):
    method = rng.choice(METHODS)
    if method == "add_file":
        return method, (rng.choice(NAMES), rng.randint(1, 20))
    if method == "copy_file":
        return method, (rng.choice(NAMES), rng.choice(NAMES))
    if method == "get_file_size":
        return method, (rng.choice(NAMES),)
    if method == "find_files":
        sample = rng.choice(NAMES)
        cut = rng.randint(0, len(sample))
        side = rng.random()
        prefix = sample[:cut] if side < 0.6 else ""
        suffix = sample[cut:] if side >= 0.3 else ""
        return method, (prefix, suffix)
    if method == "add_user":
        return method, (rng.choice(USERS), rng.randint(1, 30))
    if method == "add_file_by":
        return method, (rng.choice(USERS), rng.choice(NAMES), rng.randint(1, 20))
    if method == "merge_user":
        return method, (rng.choice(USERS), rng.choice(USERS))
    if method in ("compress_file", "decompress_file"):
        return method, (rng.choice(USERS), rng.choice(NAMES))
    if method in ("backup_user", "restore_user"):
        return method, (rng.choice(USERS),)
    raise AssertionError(method)


def run_trial(seed, n_ops=300):
    rng = random.Random(seed)
    fast, slow = CloudStorage(), SlowCloudStorage()
    outcomes = {"True": 0, "False": 0, "None": 0, "other": 0}
    for _ in range(n_ops):
        method, args = random_call(rng)
        got, want = getattr(fast, method)(*args), getattr(slow, method)(*args)
        assert got == want, (seed, method, args, got, want)
        key = "True" if got is True else "False" if got is False else "None" if got is None else "other"
        outcomes[key] += 1
    return outcomes


totals = {"True": 0, "False": 0, "None": 0, "other": 0}
for seed in range(400):
    result = run_trial(seed)
    for k in totals:
        totals[k] += result[k]
assert totals["True"] > 500 and totals["False"] > 500 and totals["None"] > 500 and totals["other"] > 500
print(f"cross-validated {400 * 300} random calls against an independent model: {totals}")
print("all checks passed")
```

</details>

</details>
