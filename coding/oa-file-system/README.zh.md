# 在线测评：内存文件系统

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 在线测评 · 4 关 | ★★☆☆☆ | 中等 | SWE · MLE | filesystem, tree, path-normalization, quota, oop-design | 4 关 / 90 分钟 | 在线测评（OA） |
<!-- meta:end -->

## 题目

实现一个内存文件系统，作为单个类 `FileSystem`，分四关逐步搭建：只有当前一关的测试全部通过，下一关的
测试才会开始运行，且每一关都在同一个类上继续添加方法，而不是重新实现。

路径是以 `/` 开头的非空绝对字符串，由 `/` 分隔的若干*分段*（component）组成——例如 `/a/b/c` 表示 `a`
目录下 `b` 目录里的目录或文件 `c`，一路都从根目录 `/` 出发，根目录永远存在，且永远是一个目录。规范化会
把连续的 `/` 合并成一个，并去掉末尾的一个 `/`，所以 `/a//b/` 与 `/a/b` 指向同一个节点，单独一个 `/`
表示根目录。`.` 和 `..` 都不具有导航含义：如果路径不以 `/` 开头，或者规范化之后某一分段恰好等于 `.` 或
`..`，这个路径就是*非法*的。非法路径的处理方式与“当前不存在的路径”完全一样：下面每个方法都会返回它用来
表示失败的那个值。分段是不含 `/` 的任意非空字符串，比较时区分大小写；同一个分段名可以出现在两个不同的
目录下，而不代表同一个节点。

只要某个方法要在一个尚不存在的路径上创建东西——一个新文件，或者移动、复制的目标路径——它就会像 `mkdir`
一样，顺带创建这个路径缺失的所有祖先目录。只读取、删除，或者作为移动*来源*的路径则从不创建任何东西：
缺失的祖先目录只会让调用失败。凡是返回表示失败的那个值的调用，文件系统本身——以及从 Level 3 起，每个
用户记录的用量——都会保持调用前的原样：不会出现部分创建、部分移动、部分删除，或者算错用量的情况。

### Level 1 —— 文件与目录

刚创建的 `FileSystem()` 里只有根目录 `/`，别无其他。

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

示例：

```text
fs = FileSystem()
fs.mkdir("/reports/2025")                   # True  -- 创建 /reports 和 /reports/2025
fs.mkdir("/reports/2025")                   # True  -- 已经是目录：无操作
fs.write("/reports/2025/q1.txt", "draft")   # True
fs.write("/reports/2025/q1.txt", "final")   # True  -- 覆盖内容
fs.read("/reports/2025/q1.txt")             # "final"
fs.ls("/reports")                           # ["2025"]
fs.ls("/reports/2025")                      # ["q1.txt"]
fs.read("/reports/2025")                    # None  -- 是目录，不是文件
fs.write("/reports/2025", "oops")           # False -- 该处已经有一个目录
fs.ls("/reports/2025/q1.txt")               # None  -- 是文件，不是目录
fs.mkdir("/reports/2025/q1.txt")            # False -- 该处已经有一个文件
```

### Level 2 —— 移动、复制、删除

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

示例：

```text
fs.mkdir("/reports/2025")
fs.write("/reports/2025/q1.txt", "Q1 results")
fs.write("/reports/2025/q2.txt", "Q2 results here")
fs.mkdir("/archive")

fs.cp("/reports/2025", "/archive/2025")                # True -- 深拷贝
fs.write("/archive/2025/q1.txt", "changed")             # 只有副本发生变化
fs.read("/reports/2025/q1.txt")                          # "Q1 results" -- 原件未受影响
fs.size("/reports/2025/q1.txt")                           # 10
fs.size("/reports/2025")                                  # 25  -- 10 + 15，递归求和
fs.mv("/reports/2025/q2.txt", "/archive/2025/q2.txt")     # True
fs.ls("/reports/2025")                                     # ["q1.txt"]
fs.ls("/archive/2025")                                     # ["q1.txt", "q2.txt"]
fs.mv("/reports", "/reports/2025/nested")                  # False -- 目标在源的自身子树内
fs.rm("/reports/2025/q1.txt")                               # True
fs.rm("/reports")                                            # True  -- 递归删除
fs.ls("/reports")                                             # None  -- 已不存在
```

### Level 3 —— 用户与存储配额

每个用户由一个字符串*用户 id*标识。系统里总有一个隐含用户 `"admin"`，存储空间不受限制；`write` 的
`owner` 参数默认就是它，`"admin"` 本身永远不能通过注册变成普通用户，也不会被当作普通用户查到。每个
文件都有一个*所有者*（owner），即它被创建时所属的用户 id，在这个文件存在期间固定不变。文件的大小会
计入其所有者的配额，除非所有者是 `"admin"`。所有权只约束 `write`、`mv` 和 `cp`：无论文件属于谁，
`read`、`ls`、`size` 的行为都和之前完全一样。从这一关起，`mv` 和 `cp` 会保留每个被移动或复制文件
原有的所有者——不是调用者——并且和 `write` 一样，如果目标处已有文件、其所有者与新来文件的所有者不符，
调用也会失败。`cp` 还会像 `write` 一样，把每个被复制文件的大小计入其所有者的配额：如果复制一个目录会
让其中涉及的某一个所有者超出容量，整次 `cp` 调用就会失败、什么都不复制，哪怕其中一些文件本来属于还有
空间的所有者；`mv` 则永远不会因为容量而失败，因为挪动一个文件不会改变任何所有者的总量。`rm`，以及
`write`、`mv`、`cp` 执行的覆盖操作，都会把被移除的大小从其原所有者的已用量里释放出来（除非该所有者是
`"admin"`）。

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

示例：

```text
fs.add_user("alice", 20)                                 # True
fs.add_user("alice", 50)                                  # False -- 已存在
fs.write("/data/a.txt", "hello world", owner="alice")      # True  -- 大小 11，11 <= 20
fs.write("/data/b.txt", "hi", owner="alice")                # True  -- 大小 2，13 <= 20
fs.write("/data/c.txt", "0123456789", owner="alice")         # False -- 大小 10，23 > 20
fs.write("/data/a.txt", "hey", owner="bob")                   # False -- a.txt 属于 alice，不是 bob
fs.update_capacity("alice", 5)                                  # 1     -- 淘汰更大的 a.txt（11）
fs.read("/data/a.txt")                                            # None  -- 已被淘汰
fs.read("/data/b.txt")                                             # "hi"  -- 保留，仍然放得下
```

### Level 4 —— 快照

*快照*（snapshot）是当前整个状态的一份保存副本——每个目录和文件、每个文件的内容与所有者、每个用户的
容量与已用量——由调用者指定的 `snapshot_id` 标识。

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

示例：

```text
fs.mkdir("/work")
fs.write("/work/draft.txt", "v1")
fs.snapshot("before-edit")             # True
fs.write("/work/draft.txt", "v2")
fs.snapshot("before-edit")             # False -- id 已被使用
fs.read("/work/draft.txt")             # "v2"
fs.restore("before-edit")              # True
fs.read("/work/draft.txt")             # "v1"
fs.restore("nope")                     # False -- 不存在这个快照

fs.add_user("carol", 10)
fs.write("/work/notes.txt", "abcde", owner="carol")   # True
fs.snapshot("with-carol")               # True
fs.update_capacity("carol", 2)          # 1 -- 淘汰 notes.txt
fs.read("/work/notes.txt")              # None
fs.restore("with-carol")                # True
fs.read("/work/notes.txt")              # "abcde" -- 已恢复
```

## 参考解答

<details>
<summary>展开参考解答</summary>

值得事先和面试官确认的两点：`mkdir` 是否像 `mkdir -p` 那样自动创建缺失的父目录，而不是要求父目录必须
已经存在（这里采用前者）；用 `mv`/`cp` 覆盖一个已有目录时，是整体替换掉它，还是与它合并（这里采用
整体替换，因为合并需要为两边都存在的每个名字单独定义冲突规则）。

### Level 1

每个目录和文件都是内存中一棵树上的一个节点：内部类 `_Dir` 持有一个从子节点名字到子节点的 `dict`；
内部类 `_File` 持有 `content`，从 Level 3 起还有 `owner`。根目录是唯一一个 `_Dir`，它自己从不以某个
名字被存放在别处。`_split` 把一个路径转换成它的分段列表，路径非法则返回 `None`。

有两个辅助方法会贯穿此后每一关。`_precheck` 只读地沿一串分段往下走，报告是否有某个已存在的分段挡住了
继续往下走的路（路径的某个真前缀已经是文件而不是目录），以及如果没挡住，完整路径处当前是什么（什么都
没有则为 `None`）。`_materialize` 沿同样的分段往下走，沿途把缺失的目录补上；它只会在同一串分段（或其
前缀）的 `_precheck` 已经确认沿途没有文件挡路之后才被调用。正是这个先后顺序，使下面每个方法都是原子
的——失败的调用不会留下任何痕迹——背后是一个简单的不变量：一旦 `_precheck` 发现某个分段*缺失*，它之后
的每个分段也必然缺失，因为 `_materialize` 刚创建出来的目录本来就是空的。因此，真正的冲突只可能出现在
请求路径与已有内容分道扬镳的*第一个*位置，而那时还什么都没被创建。

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

`mkdir`、`write`、`read`、`ls` 都是 `path` 深度 $d$ 的 $O(d)$，`ls` 另外还要为它的 $k$ 个子项做一次
$O(k \log k)$ 的排序。

### Level 2

`mv` 和 `cp` 共用一个辅助方法 `_move_or_copy`，二者的唯一区别是 `src` 是否保留。`_on_same_chain` 用
一次检查就捕捉到 `src` 与 `dst` 的每一种非法搭配：两串分段中一个是另一个的前缀——二者相等时成立，
`dst` 落在 `src` 子树内部时成立，以及容易被忽略的那个方向：`dst` 是 `src` 的*祖先*时也成立。如果只
检查“`dst` 在 `src` 内部”这一个方向，`mv("/a/b", "/a")` 就会被放过：把结果放到 `/a` 时，会先丢弃
`/a` 原有的整棵子树，而这棵子树里恰好包含 `/a/b` 本身，从 Level 3 起，这会把实际上只是挪了位置、并未
消失的字节所占的配额白白释放掉。

放置目标节点复用了与 Level 1 完全相同的 `_precheck` 和 `_materialize`：已存在的 `dst` 必须与 `src` 同
类型，否则调用在触碰任何东西之前就被拒绝；只有在确认目标位置已经腾空之后，`mv` 才会去移除 `src`。

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

`rm` 和 `size` 相对于它们触及的子树大小 $n$ 都是 $O(n)$；`_move_or_copy` 对 `cp` 是 $O(n)$（要复制
整个子树），对 `mv` 是 $O(d)$（只需重新挂接一个节点），两种情况下若要删除被覆盖的目标，都还要另加
$O(n)$。

### Level 3

Level 3 给每个文件加上 `owner`，给每个用户加上容量；`write` 和 `_move_or_copy` 都被替换成在同一套树
遍历前面叠加所有权与配额检查的新版本。`write` 只需要检查它触及的那一个文件；`mv` 和 `cp` 一次可能
触及整棵子树，所以它们先用 `_files_by_owner` 按所有者把子树里的大小汇总出来。

这里的关键简化是：**`mv` 永远不需要做容量检查**。挪动一个文件不会改变任何所有者的总量：它的字节离开
一个已经计入所有者名下的位置，落到另一个位置，仍然只被计入一次。一次会替换整个目录的 `mv`，唯一和
容量有关的影响就是丢弃 `dst` 原来的内容，而这只会释放容量，绝不会多占用容量。`cp` 则不同，它会复制
出新的字节——源文件保留自己的一份，目标又多出一份——所以两者之中只有它才可能被配额卡住。

两者的所有权检查方式一样：覆盖一个已有文件时，总是要求该文件当前的所有者等于新来文件的所有者——这只是
对 `_precheck` 已经查到的那个条目做一次相等性比较。出于同样的原因，下面 `rm` 也需要一点自己的补充：
释放所有者配额这件记账工作，`write`、`mv`、`cp` 都是搭在一次覆盖操作上顺带完成的，而 `rm` 没有覆盖
操作可以挂靠。

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

`update_capacity` 扫描整棵树找出该用户的文件是 $O(n)$，再加上对找到的 $k$ 个文件排序的
$O(k \log k)$；Level 3 的其他方法都保持着它们所替换的 Level 1/2 方法原本的复杂度。

### Level 4

Level 4 不改动任何已有方法：`snapshot` 和 `restore` 只新增一份状态——一个保存 `(root, users)` 元组的
`dict`。已经为 `cp` 写好的 `_deepcopy` 被复用来拷贝树；`copy.deepcopy` 负责拷贝扁平的 `_users` 字典。
两个方向都要拷贝：`snapshot` 从当前的树里拷贝*出来*，这样此后的写入就够不着已保存的快照；`restore`
从保存的快照里拷贝*出来*，这样修改恢复出来的树，或者以后再次 `restore` 同一个 id，也够不着那份保存
下来的副本。

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

`snapshot` 和 `restore` 相对于树的大小都是 $O(n)$。

### 追问

- 没有提供 `chown`：一个已有文件的所有者一旦创建就固定不变，因为上面每条覆盖规则都依赖这一点。要加上
  它，就得决定它是否要把配额从旧所有者原子地转移到新所有者名下，以及是否允许超出新所有者的容量。
- `update_capacity` 的淘汰逻辑要扫描整棵树才能找出一个用户的文件。维护一个 `dict[user_id,
  set[node]]` 索引，由每个创建、移动、删除文件的方法负责保持它是最新的，就能把这次扫描变成一次直接
  查找，代价是要在所有权发生变化的每一处都维护这个索引。
- 每次快照在时间和空间上都是 $O(n)$。写时复制（copy-on-write）的树——让快照与当前树共享未改动的
  子树，只在某个节点下面第一次发生改动时才复制它——能把 `snapshot` 本身变成 $O(1)$，代价是每次写入都
  可能要复制一条祖先链，而不是原地修改。
- `size` 每次调用都要遍历一个目录的整棵子树。为每个目录缓存一个实时汇总值，由它下面的每个修改类方法
  负责保持更新，就能把它变成 $O(1)$，代价是每次写入、移动、复制、删除都要多做一些记账。
- 真实的文件系统还会记录时间戳和访问权限（每个用户的读/写/执行权限）；仅凭 `owner` 就已经足以回答
  这道题要求的每一条规则，所以这两者都不在本题范围内。

<details>
<summary>验证代码（可运行）</summary>

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
