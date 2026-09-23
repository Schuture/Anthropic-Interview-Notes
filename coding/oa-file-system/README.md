# Online Assessment: In-Memory File System

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Online assessment · 4 levels | ★★☆☆☆ | Medium | SWE · MLE | filesystem, tree, path-normalization, quota, oop-design | 4 levels / 90 min | Online assessment |
<!-- meta:end -->

## Problem

Implement an in-memory file system as a single class `FileSystem`, built up over four levels: a level's
tests must pass before the next level's tests run, and each level adds methods to the same class rather
than replacing it.

A path is an absolute, non-empty string that starts with `/`, made of *components* separated by `/` — for
example `/a/b/c` names the directory or file `c` inside `b` inside `a`, all reached from the root `/`,
which always exists and is a directory. Normalization collapses consecutive `/` characters into one and
drops one trailing `/`, so `/a//b/` and `/a/b` name the same node, and `/` alone names the root. Neither
`.` nor `..` acts as a navigation shortcut: a path is *invalid* if it does not start with `/`, or if, after
normalization, one of its components is exactly `.` or `..`. An invalid path is treated exactly like a path
that does not currently exist: every method below returns whatever value it uses to signal failure.
Components are arbitrary non-empty strings that contain no `/`; they are compared case-sensitively, and the
same component name can appear under two different directories without naming the same node.

Wherever a method creates something at a path that does not yet exist — a new file, or the destination of a
move or a copy — it also creates any missing ancestor directory of that path, the same way `mkdir` does. A
method that only reads, removes, or moves *from* a path never creates anything on that side: a missing
ancestor there simply makes the call fail. Whenever a call returns the value that signals failure, the file
system — and, from Level 3 on, every user's stored usage — is left exactly as it was before the call:
nothing is partially created, moved, deleted, or billed.

### Level 1 — Files and directories

A fresh `FileSystem()` starts with nothing but the root directory `/`.

```py
class FileSystem:
    def mkdir(self, path: str) -> bool:
        """Creates a directory at path, creating every missing ancestor directory along the way
        (like mkdir -p). Returns True once the directory exists, whether it already existed or was
        just created. Returns False, without effect, for an invalid path, or if path or any
        ancestor of it already exists as a file."""

    def write(self, path: str, content: str) -> bool:
        """Creates a file at path holding content, or overwrites its content if a file is already
        there. A file's *size* is the number of characters in its content. Returns True on success.
        Returns False, without effect, for an invalid path, for path == "/", if path already names
        a directory, or if any ancestor of path already exists as a file."""

    def read(self, path: str) -> str | None:
        """Returns the content of the file at path. Returns None for an invalid path, if nothing
        exists at path, or if path names a directory."""

    def ls(self, path: str) -> list[str] | None:
        """Returns the names of path's immediate children, files and subdirectories together in one
        list, sorted lexicographically (Python's default string order: comparing Unicode code
        points, so every uppercase letter sorts before every lowercase one). Returns None for an
        invalid path, if nothing exists at path, or if path names a file."""
```

Example:

```text
fs = FileSystem()
fs.mkdir("/reports/2025")                   # True  -- creates /reports and /reports/2025
fs.mkdir("/reports/2025")                   # True  -- already a directory: no-op
fs.write("/reports/2025/q1.txt", "draft")   # True
fs.write("/reports/2025/q1.txt", "final")   # True  -- overwrites the content
fs.read("/reports/2025/q1.txt")             # "final"
fs.ls("/reports")                           # ["2025"]
fs.ls("/reports/2025")                      # ["q1.txt"]
fs.read("/reports/2025")                    # None  -- a directory, not a file
fs.write("/reports/2025", "oops")           # False -- a directory already exists there
fs.ls("/reports/2025/q1.txt")               # None  -- a file, not a directory
fs.mkdir("/reports/2025/q1.txt")            # False -- a file already exists there
```

### Level 2 — Move, copy, delete

```py
class FileSystem:
    def mv(self, src: str, dst: str) -> bool:
        """Moves src to dst -- src's entire subtree too, if it is a directory -- so that src no
        longer exists afterwards. If dst does not currently exist, src is placed there in full. If
        dst exists and is the same kind (file or directory) as src, a file overwrites a file's
        content in place, and a directory discards its whole subtree and replaces it with src's.
        Returns True on success. Returns False, without effect, for an invalid src or dst, if
        nothing exists at src, if src and dst name the same path or lie on the same ancestry chain
        (one a prefix of the other's components, so neither can move into its own subtree or over
        its own ancestor), if some ancestor of dst already exists as a file, or if dst exists but
        is not the same kind as src."""

    def cp(self, src: str, dst: str) -> bool:
        """Same as mv, but src is left untouched and the copy at dst is independent of it
        afterwards: writing to either one never changes the other."""

    def rm(self, path: str) -> bool:
        """Deletes whatever is at path; deleting a directory recursively deletes its entire
        subtree. Returns True if something was deleted. Returns False, without effect, for an
        invalid path, if nothing exists at path, or if path is the root "/" (the root can never be
        deleted)."""

    def size(self, path: str) -> int | None:
        """Returns a file's size as defined in Level 1, or, for a directory, the sum of the sizes
        of every file anywhere in its subtree (0 for an empty directory, or one that holds only
        empty files). Returns None for an invalid path or if nothing exists at path."""
```

Example:

```text
fs.mkdir("/reports/2025")
fs.write("/reports/2025/q1.txt", "Q1 results")
fs.write("/reports/2025/q2.txt", "Q2 results here")
fs.mkdir("/archive")

fs.cp("/reports/2025", "/archive/2025")                # True -- deep copy
fs.write("/archive/2025/q1.txt", "changed")             # only the copy changes
fs.read("/reports/2025/q1.txt")                          # "Q1 results" -- original untouched
fs.size("/reports/2025/q1.txt")                           # 10
fs.size("/reports/2025")                                  # 25  -- 10 + 15, recursive
fs.mv("/reports/2025/q2.txt", "/archive/2025/q2.txt")     # True
fs.ls("/reports/2025")                                     # ["q1.txt"]
fs.ls("/archive/2025")                                     # ["q1.txt", "q2.txt"]
fs.mv("/reports", "/reports/2025/nested")                  # False -- destination is inside source's own subtree
fs.rm("/reports/2025/q1.txt")                               # True
fs.rm("/reports")                                            # True  -- recursive
fs.ls("/reports")                                             # None  -- no longer exists
```

### Level 3 — Users and storage quotas

Every user is identified by a *user id*, a string. There is always an implicit user `"admin"` with
unlimited storage; `write`'s `owner` defaults to it, and `"admin"` can never be registered or looked up as
a regular user. Every file has an *owner*, the user id it was created under, fixed for that file's whole
lifetime. A file's size counts against its owner's quota unless the owner is `"admin"`. Ownership only
constrains `write`, `mv` and `cp`; `read`, `ls` and `size` behave exactly as before, for any file regardless
of its owner. From this level on, `mv` and `cp` preserve each moved or copied file's owner — the owner it
already had, never the caller — and, like `write`, fail if an existing destination file's owner does not
match the incoming file's owner. `cp` also counts every copied file's size against its owner's capacity
exactly as `write` does: if copying a directory would push any one of the several owners whose files it
contains over that owner's capacity, the entire `cp` call fails and nothing is copied, even the files
belonging to owners who did have room; `mv` never fails on capacity, since relocating a file changes no
owner's total. `rm`, and any overwrite performed by `write`, `mv` or `cp`, free the removed size from its
former owner's used total (unless that owner is `"admin"`).

```py
class FileSystem:
    def add_user(self, user_id: str, capacity: int) -> bool:
        """Registers user_id with a storage *capacity* -- the largest total size, summed over
        every file it owns anywhere in the tree, it may hold at once -- and a *used* total that
        starts at 0. Returns True on success. Returns False, without effect, if user_id is "admin"
        or already registered, or if capacity < 0."""

    def write(self, path: str, content: str, owner: str = "admin") -> bool:
        """Grows Level 1's write with three further rules. owner must be "admin" or a registered
        user id, or the call fails. If a file already exists at path, owner must equal its current
        owner, or the call fails -- a file's owner never changes across a write. If owner is not
        "admin", the call also fails whenever it would push owner's used total over its capacity
        (the size after this write, counting every other file owner already has, compared against
        capacity)."""

    def update_capacity(self, user_id: str, capacity: int) -> int | None:
        """Sets user_id's capacity. If its used total is already at or under the new capacity,
        nothing else happens and the call returns 0. Otherwise deletes that user's files, largest
        first, ties broken by comparing the two files' full paths as strings in the same order as
        ls, until the used total is at or under capacity, and returns the number of files deleted.
        Returns None, without effect, if user_id is not registered (including "admin") or
        capacity < 0."""
```

Example:

```text
fs.add_user("alice", 20)                                 # True
fs.add_user("alice", 50)                                  # False -- already exists
fs.write("/data/a.txt", "hello world", owner="alice")      # True  -- size 11, 11 <= 20
fs.write("/data/b.txt", "hi", owner="alice")                # True  -- size 2, 13 <= 20
fs.write("/data/c.txt", "0123456789", owner="alice")         # False -- size 10, 23 > 20
fs.write("/data/a.txt", "hey", owner="bob")                   # False -- a.txt is owned by alice, not bob
fs.update_capacity("alice", 5)                                  # 1     -- evicts a.txt (11), the larger file
fs.read("/data/a.txt")                                            # None  -- evicted
fs.read("/data/b.txt")                                             # "hi"  -- kept, still fits
```

### Level 4 — Snapshots

A *snapshot* is a saved copy of the entire current state — every directory and file, every file's content
and owner, and every user's capacity and used total — identified by a `snapshot_id` chosen by the caller.

```py
class FileSystem:
    def snapshot(self, snapshot_id: str) -> bool:
        """Captures the entire current state under snapshot_id. Returns True on success. Returns
        False, without effect, if snapshot_id was already used by an earlier snapshot."""

    def restore(self, snapshot_id: str) -> bool:
        """Replaces the entire current state with an independent copy of the one captured under
        snapshot_id: later changes to the file system, or a later restore of a different snapshot,
        must never affect a previously captured snapshot, and restoring the same snapshot_id twice
        must give the same result both times. Returns True on success. Returns False, without
        effect, if no snapshot was ever captured under snapshot_id."""
```

Example:

```text
fs.mkdir("/work")
fs.write("/work/draft.txt", "v1")
fs.snapshot("before-edit")             # True
fs.write("/work/draft.txt", "v2")
fs.snapshot("before-edit")             # False -- id already used
fs.read("/work/draft.txt")             # "v2"
fs.restore("before-edit")              # True
fs.read("/work/draft.txt")             # "v1"
fs.restore("nope")                     # False -- no such snapshot

fs.add_user("carol", 10)
fs.write("/work/notes.txt", "abcde", owner="carol")   # True
fs.snapshot("with-carol")               # True
fs.update_capacity("carol", 2)          # 1 -- evicts notes.txt
fs.read("/work/notes.txt")              # None
fs.restore("with-carol")                # True
fs.read("/work/notes.txt")              # "abcde" -- restored
```

## Reference solution

<details>
<summary>Show the reference solution</summary>

Worth confirming with the interviewer up front: whether `mkdir` behaves like `mkdir -p`, creating missing
parent directories, rather than requiring the parent to already exist (assumed here); and whether
overwriting an existing directory with `mv`/`cp` replaces it outright or merges into it (replacement is
assumed here, since a merge needs its own conflict rule for every name that exists on both sides).

### Level 1

Every directory and file is a node of a tree kept in memory: an internal `_Dir` holds a `dict` from child
name to child node, and an internal `_File` holds `content` and, from Level 3 on, `owner`. The root is one
`_Dir` that is never itself stored under a name. `_split` turns a path into its list of components, or
`None` if the path is invalid.

Two helpers carry every level from here on. `_precheck` walks a list of components read-only and reports
whether an existing component blocks the walk (some proper prefix of the path is already a file) and, if
not, whatever sits at the full path (`None` if nothing does). `_materialize` walks the same components,
creating any missing directory as it goes, and only ever runs after a `_precheck` has confirmed nothing
along the way is a file. This ordering makes every method below atomic — a failed call leaves no trace —
because once `_precheck` finds a component *missing*, every component after it is missing too (a directory
`_materialize` just created starts out empty), so a conflict can only surface at the *first* point the
requested path diverges from what already exists, before anything is created.

```python
from __future__ import annotations
import copy


class _File:
    __slots__ = ("content", "owner")

    def __init__(self, content: str, owner: str):
        self.content = content
        self.owner = owner


class _Dir:
    __slots__ = ("children",)

    def __init__(self):
        self.children: dict[str, "_Dir | _File"] = {}


def _split(path: str) -> list[str] | None:
    if not isinstance(path, str) or not path.startswith("/"):
        return None
    parts = [seg for seg in path.split("/") if seg != ""]   # NOTE: collapses repeated "/" and a trailing "/"
    if any(seg in (".", "..") for seg in parts):
        return None
    return parts


class FileSystem:
    def __init__(self):
        self._root = _Dir()
        self._users: dict[str, dict] = {}        # user_id -> {"capacity": int, "used": int} -- Level 3

    def _precheck(self, comps: list[str]):
        node = self._root
        for c in comps:
            if not isinstance(node, _Dir):
                return "blocked", None
            nxt = node.children.get(c)
            if nxt is None:
                return "ok", None           # NOTE: nothing under an absent directory can exist either
            node = nxt
        return "ok", node

    def _materialize(self, comps: list[str]) -> "_Dir":
        node = self._root
        for c in comps:
            if c not in node.children:
                node.children[c] = _Dir()
            node = node.children[c]
        return node

    def _resolve(self, comps: list[str]):
        status, node = self._precheck(comps)
        return node if status == "ok" else None

    def mkdir(self, path: str) -> bool:
        comps = _split(path)
        if comps is None:
            return False
        status, existing = self._precheck(comps)
        if status == "blocked" or isinstance(existing, _File):
            return False
        self._materialize(comps)
        return True

    def write(self, path: str, content: str) -> bool:
        comps = _split(path)
        if comps is None or not comps:          # NOTE: not comps -- "/" itself is never a file
            return False
        status, existing = self._precheck(comps)
        if status == "blocked" or isinstance(existing, _Dir):
            return False
        parent = self._materialize(comps[:-1])
        parent.children[comps[-1]] = _File(content, "admin")
        return True

    def read(self, path: str) -> str | None:
        comps = _split(path)
        node = self._resolve(comps) if comps is not None else None
        return node.content if isinstance(node, _File) else None

    def ls(self, path: str) -> list[str] | None:
        comps = _split(path)
        node = self._resolve(comps) if comps is not None else None
        return sorted(node.children) if isinstance(node, _Dir) else None
```

`mkdir`, `write`, `read` and `ls` are all $O(d)$ in the depth $d$ of `path`, aside from `ls`'s
$O(k \log k)$ sort of its $k$ children.

### Level 2

`mv` and `cp` share one helper, `_move_or_copy`, differing only in whether `src` survives.
`_on_same_chain` catches every illegal pairing in one check: one component list is a prefix of the other —
true when they are equal, when `dst` sits inside `src`'s subtree, and, easy to miss, when `dst` is an
*ancestor* of `src`. Skipping that last case would let `mv("/a/b", "/a")` through: placing the result at
`/a` would first discard `/a`'s old subtree, which contains `/a/b` itself, freeing quota (from Level 3 on)
for bytes that only moved, not vanished.

Placement reuses `_precheck` and `_materialize` exactly as Level 1 does: an existing `dst` must be the same
kind as `src`, or the call is rejected before anything is touched, and `mv` only removes `src` once that
destination slot is confirmed clear.

```python
def _deepcopy(node):
    if isinstance(node, _File):
        return _File(node.content, node.owner)
    new = _Dir()
    for k, v in node.children.items():
        new.children[k] = _deepcopy(v)
    return new


def _on_same_chain(a: list[str], b: list[str]) -> bool:
    """True if a is a prefix of b, or b is a prefix of a (equal counts as both)."""
    shorter, longer = (a, b) if len(a) <= len(b) else (b, a)
    return longer[:len(shorter)] == shorter


def _size_of(node) -> int:
    if isinstance(node, _File):
        return len(node.content)
    return sum(_size_of(c) for c in node.children.values())


def size(self, path: str) -> int | None:
    comps = _split(path)
    node = self._resolve(comps) if comps is not None else None
    return None if node is None else _size_of(node)


def rm(self, path: str) -> bool:
    comps = _split(path)
    if comps is None or not comps:
        return False
    parent = self._resolve(comps[:-1])
    if not isinstance(parent, _Dir) or comps[-1] not in parent.children:
        return False
    del parent.children[comps[-1]]
    return True


def _move_or_copy(self, src: str, dst: str, remove_src: bool) -> bool:
    src_comps, dst_comps = _split(src), _split(dst)
    if src_comps is None or not src_comps or dst_comps is None or not dst_comps:
        return False
    src_node = self._resolve(src_comps)
    if src_node is None or _on_same_chain(src_comps, dst_comps):
        return False                        # NOTE: covers dst == src, dst inside src, AND src inside dst
    status, existing = self._precheck(dst_comps)
    if status == "blocked" or (existing is not None and type(existing) is not type(src_node)):
        return False                        # NOTE: file/directory type mismatch at an existing destination
    new_node = src_node if remove_src else _deepcopy(src_node)
    parent = self._materialize(dst_comps[:-1])   # -- validated: safe to mutate from here on --
    if remove_src:
        src_parent = self._resolve(src_comps[:-1])
        del src_parent.children[src_comps[-1]]
    parent.children[dst_comps[-1]] = new_node
    return True


def mv(self, src: str, dst: str) -> bool:
    return self._move_or_copy(src, dst, remove_src=True)


def cp(self, src: str, dst: str) -> bool:
    return self._move_or_copy(src, dst, remove_src=False)


FileSystem.size = size
FileSystem.rm = rm
FileSystem._move_or_copy = _move_or_copy
FileSystem.mv = mv
FileSystem.cp = cp
```

`rm` and `size` are $O(n)$ in the size $n$ of the subtree they touch; `_move_or_copy` is $O(n)$ for `cp`
(it copies the subtree) and $O(d)$ for `mv` (it only relinks one node), plus $O(n)$ to delete an overwritten
destination in either case.

### Level 3

Level 3 gives every file an `owner` and every user a capacity; `write` and `_move_or_copy` are replaced by
versions that layer ownership and quota checks onto the same tree walk. `write` checks only the one file
it touches; `mv` and `cp` can touch a whole subtree, so they first total its sizes per owner with
`_files_by_owner`.

The key simplification is that **`mv` never needs a capacity check**. Relocating a file changes no owner's
total: its bytes leave one path already counted against their owner and arrive at another, still counted
exactly once. The only capacity-relevant effect a directory-replacing `mv` can have is discarding whatever
used to be at `dst`, which only frees capacity, never spends it. `cp`, by contrast, duplicates bytes — the
source keeps its copy and the destination gets a new one — so it is the only one of the two that a full
quota can block.

Ownership is checked the same way for both: overwriting a file always requires its current owner to equal
the incoming file's owner, a plain equality check on the entry `_precheck` already looked up. `rm` needs
its own small addition below: freeing an owner's quota is bookkeeping that `write`, `mv` and `cp` already
do on an overwrite, and `rm` has no overwrite to attach it to.

```python
def _files_by_owner(node) -> dict:
    totals: dict[str, int] = {}
    stack = [node]
    while stack:
        n = stack.pop()
        if isinstance(n, _File):
            totals[n.owner] = totals.get(n.owner, 0) + len(n.content)
        else:
            stack.extend(n.children.values())
    return totals


def add_user(self, user_id: str, capacity: int) -> bool:
    if user_id == "admin" or user_id in self._users or capacity < 0:
        return False
    self._users[user_id] = {"capacity": capacity, "used": 0}
    return True


def write(self, path: str, content: str, owner: str = "admin") -> bool:
    if owner != "admin" and owner not in self._users:
        return False
    comps = _split(path)
    if comps is None or not comps:
        return False
    status, existing = self._precheck(comps)
    if status == "blocked" or isinstance(existing, _Dir):
        return False
    if isinstance(existing, _File) and existing.owner != owner:
        return False                        # NOTE: an existing file can only be overwritten by its own owner
    old_size = len(existing.content) if isinstance(existing, _File) else 0
    if owner != "admin":
        u = self._users[owner]
        if u["used"] - old_size + len(content) > u["capacity"]:   # NOTE: old_size cancels when owner
            return False                                          # overwrites its own file, so shrinking never fails
    parent = self._materialize(comps[:-1])    # -- validated: safe to mutate from here on --
    if owner != "admin":
        self._users[owner]["used"] += len(content) - old_size
    parent.children[comps[-1]] = _File(content, owner)
    return True


def _move_or_copy(self, src: str, dst: str, remove_src: bool) -> bool:
    src_comps, dst_comps = _split(src), _split(dst)
    if src_comps is None or not src_comps or dst_comps is None or not dst_comps:
        return False
    src_node = self._resolve(src_comps)
    if src_node is None or _on_same_chain(src_comps, dst_comps):
        return False
    status, existing = self._precheck(dst_comps)
    if status == "blocked" or (existing is not None and type(existing) is not type(src_node)):
        return False
    if isinstance(existing, _File) and existing.owner != src_node.owner:
        return False                        # NOTE: same rule as write -- owner must match to overwrite a file
    freed = _files_by_owner(existing) if existing is not None else {}
    if not remove_src:                      # NOTE: mv skips this -- relocating already-owned bytes can only
        added = _files_by_owner(src_node)   # free capacity, never spend more of it
        for owner, amount in added.items():
            if owner == "admin":
                continue
            u = self._users[owner]
            if u["used"] - freed.get(owner, 0) + amount > u["capacity"]:
                return False
    parent = self._materialize(dst_comps[:-1])   # -- validated: safe to mutate from here on --
    for owner, amount in freed.items():
        if owner != "admin":
            self._users[owner]["used"] -= amount
    if remove_src:
        new_node = src_node
        src_parent = self._resolve(src_comps[:-1])
        del src_parent.children[src_comps[-1]]
    else:
        new_node = _deepcopy(src_node)
        for owner, amount in _files_by_owner(src_node).items():
            if owner != "admin":
                self._users[owner]["used"] += amount
    parent.children[dst_comps[-1]] = new_node
    return True


def rm(self, path: str) -> bool:
    comps = _split(path)
    if comps is None or not comps:
        return False
    parent = self._resolve(comps[:-1])
    if not isinstance(parent, _Dir) or comps[-1] not in parent.children:
        return False
    for owner, amount in _files_by_owner(parent.children[comps[-1]]).items():
        if owner != "admin":                # NOTE: rm must free capacity too, not just write/mv/cp/eviction
            self._users[owner]["used"] -= amount
    del parent.children[comps[-1]]
    return True


def update_capacity(self, user_id: str, capacity: int) -> int | None:
    if user_id not in self._users or capacity < 0:
        return None
    u = self._users[user_id]
    u["capacity"] = capacity
    if u["used"] <= capacity:
        return 0
    victims = []                            # (size, full path, parent dir, key) for every file user_id owns
    def collect(node, prefix):
        for name, child in node.children.items():
            path = f"{prefix}/{name}"
            if isinstance(child, _File):
                if child.owner == user_id:
                    victims.append((len(child.content), path, node, name))
            else:
                collect(child, path)
    collect(self._root, "")
    victims.sort(key=lambda v: (-v[0], v[1]))   # largest first; ties by path ascending
    deleted = 0
    for sz, _, parent, name in victims:
        if u["used"] <= capacity:
            break
        del parent.children[name]
        u["used"] -= sz
        deleted += 1
    return deleted


FileSystem.add_user = add_user
FileSystem.write = write
FileSystem.rm = rm
FileSystem._move_or_copy = _move_or_copy
FileSystem.update_capacity = update_capacity
```

`update_capacity` is $O(n)$ to scan the tree for the user's files plus $O(k \log k)$ to sort the $k$ found;
every other Level 3 method keeps the complexity of the Level 1/2 method it replaces.

### Level 4

Level 4 changes no existing method: `snapshot` and `restore` only add one new piece of state, a `dict` of
saved `(root, users)` pairs. `_deepcopy`, already written for `cp`, is reused for the tree; `copy.deepcopy`
handles the flat `_users` dict. Both directions copy: `snapshot` copies out of the live tree so later
writes cannot reach a saved snapshot, and `restore` copies out of the saved snapshot so mutating the
restored tree, or restoring the same id again, cannot reach back into the saved copy.

```python
def snapshot(self, snapshot_id: str) -> bool:
    if not hasattr(self, "_snapshots"):
        self._snapshots = {}
    if snapshot_id in self._snapshots:
        return False
    self._snapshots[snapshot_id] = (_deepcopy(self._root), copy.deepcopy(self._users))
    return True


def restore(self, snapshot_id: str) -> bool:
    if not hasattr(self, "_snapshots") or snapshot_id not in self._snapshots:
        return False
    root, users = self._snapshots[snapshot_id]
    self._root = _deepcopy(root)            # NOTE: copy out -- restoring twice must not share state either
    self._users = copy.deepcopy(users)
    return True


FileSystem.snapshot = snapshot
FileSystem.restore = restore
```

`snapshot` and `restore` are both $O(n)$ in the size of the tree.

### Follow-ups

- No `chown` is provided: an owner is fixed once a file is created, since every overwrite rule above
  assumes it. Adding one means deciding whether it moves quota between owners atomically, and whether it
  may exceed the new owner's capacity.
- `update_capacity`'s eviction scans the whole tree for one user's files. A `dict[user_id, set[node]]`
  index, kept current by every method that creates, moves, or deletes a file, would turn that scan into a
  direct lookup, at the cost of maintaining the index everywhere ownership changes.
- Snapshots are $O(n)$ in both time and space per call. A copy-on-write tree — sharing unchanged subtrees
  between a snapshot and the live tree, copying a node only the first time something under it changes —
  makes `snapshot` itself $O(1)$, at the cost of every write potentially copying an ancestor chain instead
  of mutating in place.
- `size` walks a directory's whole subtree on every call. Caching a running total per directory, kept
  current by every mutating method under it, makes it $O(1)$ at the cost of more bookkeeping on every
  write, move, copy, and delete.
- A real file system also tracks timestamps and per-user read/write/execute permissions; `owner` alone
  already answers every rule this problem asks for, so neither is in scope here.

<details>
<summary>Checks (runnable)</summary>

```python
import random

# --- Level 1 example, replayed exactly ---
fs = FileSystem()
assert fs.mkdir("/reports/2025")
assert fs.mkdir("/reports/2025")
assert fs.write("/reports/2025/q1.txt", "draft")
assert fs.write("/reports/2025/q1.txt", "final")
assert fs.read("/reports/2025/q1.txt") == "final"
assert fs.ls("/reports") == ["2025"]
assert fs.ls("/reports/2025") == ["q1.txt"]
assert fs.read("/reports/2025") is None
assert not fs.write("/reports/2025", "oops")
assert fs.ls("/reports/2025/q1.txt") is None
assert not fs.mkdir("/reports/2025/q1.txt")

# --- Level 2 example, replayed exactly ---
fs = FileSystem()
fs.mkdir("/reports/2025")
fs.write("/reports/2025/q1.txt", "Q1 results")
fs.write("/reports/2025/q2.txt", "Q2 results here")
fs.mkdir("/archive")
assert fs.cp("/reports/2025", "/archive/2025")
fs.write("/archive/2025/q1.txt", "changed")
assert fs.read("/reports/2025/q1.txt") == "Q1 results"
assert fs.size("/reports/2025/q1.txt") == 10
assert fs.size("/reports/2025") == 25
assert fs.mv("/reports/2025/q2.txt", "/archive/2025/q2.txt")
assert fs.ls("/reports/2025") == ["q1.txt"]
assert fs.ls("/archive/2025") == ["q1.txt", "q2.txt"]
assert not fs.mv("/reports", "/reports/2025/nested")
assert fs.rm("/reports/2025/q1.txt")
assert fs.rm("/reports")
assert fs.ls("/reports") is None

# --- Level 3 example, replayed exactly ---
fs = FileSystem()
assert fs.add_user("alice", 20)
assert not fs.add_user("alice", 50)
assert fs.write("/data/a.txt", "hello world", owner="alice")
assert fs.write("/data/b.txt", "hi", owner="alice")
assert not fs.write("/data/c.txt", "0123456789", owner="alice")
assert not fs.write("/data/a.txt", "hey", owner="bob")
assert fs.update_capacity("alice", 5) == 1
assert fs.read("/data/a.txt") is None
assert fs.read("/data/b.txt") == "hi"

# --- Level 4 example, replayed exactly ---
fs = FileSystem()
fs.mkdir("/work")
fs.write("/work/draft.txt", "v1")
assert fs.snapshot("before-edit")
fs.write("/work/draft.txt", "v2")
assert not fs.snapshot("before-edit")
assert fs.read("/work/draft.txt") == "v2"
assert fs.restore("before-edit")
assert fs.read("/work/draft.txt") == "v1"
assert not fs.restore("nope")
fs.add_user("carol", 10)
assert fs.write("/work/notes.txt", "abcde", owner="carol")
assert fs.snapshot("with-carol")
assert fs.update_capacity("carol", 2) == 1
assert fs.read("/work/notes.txt") is None
assert fs.restore("with-carol")
assert fs.read("/work/notes.txt") == "abcde"
print("level examples replayed")

# --- path normalization and invalid paths ---
fs = FileSystem()
assert fs.mkdir("/a//b/c")
assert fs.ls("/a/b") == ["c"]              # repeated slashes normalize like a single one
assert fs.mkdir("/x/y/")
assert fs.ls("/x") == ["y"]                 # a trailing slash normalizes away
for bad in ["", "a/b", "/.", "/..", "/a/./b", "/a/../b", "/a/b/.."]:
    assert not fs.mkdir(bad)
    assert not fs.write(bad, "x")
    assert fs.read(bad) is None
    assert fs.ls(bad) is None

# --- the root is special ---
assert fs.ls("/") is not None
assert not fs.write("/", "x")
assert not fs.rm("/")

# --- a failed cp leaves no orphaned directory and bills nothing; mv never fails on capacity ---
fs = FileSystem()
fs.add_user("dee", 5)
fs.write("/keep.txt", "abcde", owner="dee")           # exactly fills the 5-byte quota
assert not fs.cp("/keep.txt", "/new/deep/copy.txt")    # would need 5 more, 10 > 5
assert fs.ls("/new") is None                            # no orphaned "/new" left behind by the failed cp
assert fs._users["dee"]["used"] == 5
assert fs.mv("/keep.txt", "/moved/keep.txt")            # pure relocation, no capacity check possible
assert fs.read("/moved/keep.txt") == "abcde"
assert fs._users["dee"]["used"] == 5                    # a move never changes an owner's used total

# --- moving/copying onto one of your own ancestors is rejected, not just into your own subtree ---
fs = FileSystem()
fs.add_user("finn", 1000)
fs.mkdir("/p/q")
fs.write("/p/q/f.txt", "x" * 10, owner="finn")
fs.write("/p/g.txt", "y" * 5, owner="finn")
assert not fs.mv("/p/q", "/p")
assert not fs.cp("/p/q", "/p")
assert fs._users["finn"]["used"] == 15                  # unaffected by the rejected attempts

# --- more Level 3 edge cases: unknown users, negative capacity, eviction tie-break ---
fs = FileSystem()
assert not fs.add_user("admin", 10)
assert not fs.add_user("gia", -1)
assert fs.update_capacity("nobody", 5) is None
assert fs.update_capacity("admin", 5) is None            # "admin" is never a registered user id
assert not fs.write("/x.txt", "hi", owner="nobody")

fs.add_user("gia", 100)
fs.write("/z/bb.txt", "x" * 10, owner="gia")
fs.write("/z/aa.txt", "y" * 10, owner="gia")             # ties bb.txt on size; "aa" sorts first
fs.write("/z/cc.txt", "z" * 5, owner="gia")
assert fs.update_capacity("gia", 5) == 2                  # evicts aa.txt, then bb.txt
assert fs.read("/z/aa.txt") is None and fs.read("/z/bb.txt") is None
assert fs.read("/z/cc.txt") == "z" * 5

# --- a restored snapshot is independent of everything that happens after it is taken or restored ---
fs = FileSystem()
fs.write("/f.txt", "one")
fs.snapshot("s")
fs.write("/f.txt", "two")
fs.restore("s")
fs.write("/f.txt", "three")
fs.restore("s")
assert fs.read("/f.txt") == "one"
print("edge cases passed")


# --- brute force: every path is a flat dict key, independent of the tree above ---
class _RefFS:
    def __init__(self):
        self.entries: dict[tuple, dict] = {(): {"type": "dir"}}
        self.users: dict[str, dict] = {}
        self.snapshots: dict[str, tuple] = {}

    def _norm(self, path):
        if not isinstance(path, str) or not path.startswith("/"):
            return None
        parts = [s for s in path.split("/") if s]
        return None if any(s in (".", "..") for s in parts) else tuple(parts)

    def _ok_as_dir(self, comps) -> bool:
        return all(self.entries.get(comps[:i], {"type": "dir"})["type"] == "dir"
                   for i in range(1, len(comps) + 1))

    def _materialize_dirs(self, comps) -> None:
        for i in range(1, len(comps) + 1):
            self.entries.setdefault(comps[:i], {"type": "dir"})

    def _subtree(self, comps) -> dict:
        return {p: e for p, e in self.entries.items()
                if p == comps or (len(p) > len(comps) and p[:len(comps)] == comps)}

    def mkdir(self, path) -> bool:
        comps = self._norm(path)
        if comps is None or not self._ok_as_dir(comps):
            return False
        self._materialize_dirs(comps)
        return True

    def read(self, path):
        comps = self._norm(path)
        e = None if comps is None else self.entries.get(comps)
        return e["content"] if e is not None and e["type"] == "file" else None

    def ls(self, path):
        comps = self._norm(path)
        e = None if comps is None else self.entries.get(comps)
        if e is None or e["type"] != "dir":
            return None
        return sorted(p[-1] for p in self.entries if len(p) == len(comps) + 1 and p[:len(comps)] == comps)

    def write(self, path, content, owner="admin") -> bool:
        if owner != "admin" and owner not in self.users:
            return False
        comps = self._norm(path)
        if not comps or not self._ok_as_dir(comps[:-1]):
            return False
        existing = self.entries.get(comps)
        if existing is not None and (existing["type"] == "dir" or existing["owner"] != owner):
            return False
        old_size = len(existing["content"]) if existing is not None else 0
        if owner != "admin":
            u = self.users[owner]
            if u["used"] - old_size + len(content) > u["capacity"]:
                return False
            u["used"] += len(content) - old_size
        self._materialize_dirs(comps[:-1])
        self.entries[comps] = {"type": "file", "content": content, "owner": owner}
        return True

    def size(self, path):
        comps = self._norm(path)
        if comps is None or comps not in self.entries:
            return None
        e = self.entries[comps]
        if e["type"] == "file":
            return len(e["content"])
        return sum(len(v["content"]) for p, v in self.entries.items()
                   if v["type"] == "file" and len(p) > len(comps) and p[:len(comps)] == comps)

    def rm(self, path) -> bool:
        comps = self._norm(path)
        if not comps or comps not in self.entries:
            return False
        for p, e in self._subtree(comps).items():
            if e["type"] == "file" and e["owner"] != "admin":
                self.users[e["owner"]]["used"] -= len(e["content"])
            del self.entries[p]
        return True

    def _move_or_copy(self, src, dst, remove_src) -> bool:
        s, d = self._norm(src), self._norm(dst)
        if not s or not d or s not in self.entries:
            return False
        shorter, longer = (s, d) if len(s) <= len(d) else (d, s)
        if longer[:len(shorter)] == shorter or not self._ok_as_dir(d[:-1]):
            return False
        src_entry, existing = self.entries[s], self.entries.get(d)
        if existing is not None and (existing["type"] != src_entry["type"]
                                     or (existing["type"] == "file" and existing["owner"] != src_entry["owner"])):
            return False
        src_subtree, dst_subtree = self._subtree(s), (self._subtree(d) if existing is not None else {})
        freed, added = {}, {}
        for e in dst_subtree.values():
            if e["type"] == "file" and e["owner"] != "admin":
                freed[e["owner"]] = freed.get(e["owner"], 0) + len(e["content"])
        for e in src_subtree.values():
            if e["type"] == "file" and e["owner"] != "admin":
                added[e["owner"]] = added.get(e["owner"], 0) + len(e["content"])
        if not remove_src:
            for owner, amount in added.items():
                u = self.users[owner]
                if u["used"] - freed.get(owner, 0) + amount > u["capacity"]:
                    return False
        for p in dst_subtree:
            del self.entries[p]
        for owner, amount in freed.items():
            self.users[owner]["used"] -= amount
        self._materialize_dirs(d[:-1])
        for p, e in src_subtree.items():
            self.entries[d + p[len(s):]] = dict(e)
        if remove_src:
            for p in src_subtree:
                del self.entries[p]
        else:
            for owner, amount in added.items():
                self.users[owner]["used"] += amount
        return True

    def mv(self, src, dst):
        return self._move_or_copy(src, dst, True)

    def cp(self, src, dst):
        return self._move_or_copy(src, dst, False)

    def add_user(self, user_id, capacity) -> bool:
        if user_id == "admin" or user_id in self.users or capacity < 0:
            return False
        self.users[user_id] = {"capacity": capacity, "used": 0}
        return True

    def update_capacity(self, user_id, capacity):
        if user_id not in self.users or capacity < 0:
            return None
        u = self.users[user_id]
        u["capacity"] = capacity
        if u["used"] <= capacity:
            return 0
        victims = sorted((("/" + "/".join(p), len(e["content"])) for p, e in self.entries.items()
                          if e["type"] == "file" and e["owner"] == user_id),
                         key=lambda pair: (-pair[1], pair[0]))
        deleted = 0
        for full_path, sz in victims:
            if u["used"] <= capacity:
                break
            del self.entries[self._norm(full_path)]
            u["used"] -= sz
            deleted += 1
        return deleted

    def snapshot(self, snapshot_id) -> bool:
        if snapshot_id in self.snapshots:
            return False
        self.snapshots[snapshot_id] = (copy.deepcopy(self.entries), copy.deepcopy(self.users))
        return True

    def restore(self, snapshot_id) -> bool:
        if snapshot_id not in self.snapshots:
            return False
        entries, users = self.snapshots[snapshot_id]
        self.entries, self.users = copy.deepcopy(entries), copy.deepcopy(users)
        return True


def _rand_path(rng, names, max_depth=3) -> str:
    return "/" + "/".join(rng.choice(names) for _ in range(rng.randint(1, max_depth)))


def _cross_check(seed: int, n_ops: int = 300) -> None:
    rng = random.Random(seed)
    names = ["a", "b", "c", "d"]
    fs, ref = FileSystem(), _RefFS()
    for _ in range(n_ops):
        kind = rng.choice(["mkdir", "write", "read", "ls", "mv", "cp", "rm", "size",
                           "add_user", "update_capacity", "snapshot", "restore"])
        if kind == "mkdir":
            p = _rand_path(rng, names)
            assert fs.mkdir(p) == ref.mkdir(p)
        elif kind == "write":
            p, c = _rand_path(rng, names), rng.choice(["", "x", "hello", "0123456789"])
            owner = rng.choice(["admin"] + list(ref.users))
            assert fs.write(p, c, owner=owner) == ref.write(p, c, owner=owner)
        elif kind == "read":
            p = _rand_path(rng, names)
            assert fs.read(p) == ref.read(p)
        elif kind == "ls":
            p = _rand_path(rng, names, max_depth=2) if rng.random() < 0.3 else "/"
            assert fs.ls(p) == ref.ls(p)
        elif kind == "mv":
            s, d = _rand_path(rng, names), _rand_path(rng, names)
            assert fs.mv(s, d) == ref.mv(s, d)
        elif kind == "cp":
            s, d = _rand_path(rng, names), _rand_path(rng, names)
            assert fs.cp(s, d) == ref.cp(s, d)
        elif kind == "rm":
            p = _rand_path(rng, names)
            assert fs.rm(p) == ref.rm(p)
        elif kind == "size":
            p = _rand_path(rng, names)
            assert fs.size(p) == ref.size(p)
        elif kind == "add_user":
            u, cap = rng.choice(["alice", "bob"]), rng.choice([-1, 0, 5, 10, 30])
            assert fs.add_user(u, cap) == ref.add_user(u, cap)
        elif kind == "update_capacity":
            u, cap = rng.choice(["alice", "bob", "nope"]), rng.choice([-1, 0, 3, 8, 20])
            assert fs.update_capacity(u, cap) == ref.update_capacity(u, cap)
        elif kind == "snapshot":
            sid = rng.choice(["s1", "s2", "s3"])
            assert fs.snapshot(sid) == ref.snapshot(sid)
        else:
            sid = rng.choice(["s1", "s2", "s3", "missing"])
            assert fs.restore(sid) == ref.restore(sid)
        for u in ref.users:                               # every user's bookkeeping stays in sync too
            assert fs._users[u] == ref.users[u]


for seed in range(150):
    _cross_check(seed)
print("all checks passed")
```

</details>

</details>
