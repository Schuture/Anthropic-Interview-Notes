# 在线测评：云存储系统

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 在线测评 · 4 关 | ★★☆☆☆ | 中等 | SWE · MLE | oop-design, hash-map, storage-quotas, string-matching | 4 关 / 90 分钟 | 在线测评（OA） |
<!-- meta:end -->

## 题目

实现一个 `CloudStorage` 类，只按名称和大小追踪文件：文件的字节内容从不存储也不读取，只关心它有多少
字节。这个类分四关搭建，按顺序进行；每一关的测试必须先通过，才会用到下一关的方法；并且每一关都是在
前面几关定义的同一个 `CloudStorage` 类上添加方法，绝不会另起一个新类。

*文件名*（file name）是一个非空字符串，由 ASCII 字母、数字、`.`、`_` 和 `-` 组成；名称区分大小写，
所以 `"Notes.txt"` 和 `"notes.txt"` 是两个不同的文件。在任意时刻，整个存储中最多只能存在一个使用某个
名称的文件，不管它有没有主人（所有权从第 3 关开始出现）。文件的大小是一个正整数字节数，只要这个名称
还存在，它就不会变化：复制、压缩、解压（第 4 关）做的都是创建一个改了名字的新文件，而不是原地修改
一个已有的文件。*用户 ID*（user id）是一个非空字符串，字符集和文件名相同；用户 ID 和文件名是两个独立
的命名空间，所以同一个字符串可以同时用作文件名和用户 ID，互不冲突。本题中每一次调用传入的名称、ID
或大小，都已经满足上述规则。

### 第 1 关 —— 文件、复制与大小

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

例如：

```text
cs = CloudStorage()
cs.add_file("report.txt", 200)                   # -> True
cs.add_file("report.txt", 10)                    # -> False  -- "report.txt" 已经存在
cs.copy_file("report.txt", "report_backup.txt")  # -> True   -- report_backup.txt 现在存在，大小 200
cs.copy_file("missing.txt", "x.txt")             # -> False  -- source 不存在
cs.copy_file("report.txt", "report_backup.txt")  # -> False  -- destination 已经存在
cs.get_file_size("report_backup.txt")            # -> 200
cs.get_file_size("nope.txt")                     # -> None
```

### 第 2 关 —— 按前缀和后缀搜索

`find_files(prefix, suffix)` 会在存储当前的所有文件里查找——不论有没有主人——找出名称以 `prefix`
开头、以 `suffix` 结尾的文件，这里的“开头”“结尾”就是 `str.startswith` 和 `str.endswith` 的字面
含义：两个参数都可以是 `""`，此时那一侧对任何名称都算匹配；一个很短的名称也可能同时满足两侧，比如
`prefix="ab"`、`suffix="ab"` 时的 `"ab"`。每个匹配项被格式化为 `f"{name}({size})"`——例如
`"report.txt(200)"`——返回的列表先按大小降序排列，大小相同的文件再按名称升序排列。

```py
    def find_files(self, prefix: str, suffix: str) -> list[str]:
        """Returns "name(size)" for every file currently in the store whose name starts with prefix
        and ends with suffix, sorted by size descending and, among files of the same size, by name
        ascending."""
```

接着上面的存储：

```text
cs.add_file("report_draft.txt", 200)
cs.add_file("report_final.txt", 500)
cs.add_file("summary.md", 50)
cs.find_files("report", ".txt")
# ["report_final.txt(500)", "report.txt(200)", "report_backup.txt(200)", "report_draft.txt(200)"]
# 500 排最前；并列 200 的三个文件按名称升序排列 -- ASCII 中 "." 排在 "_" 前面，
# 所以 "report.txt" < "report_backup.txt" < "report_draft.txt"；"summary.md" 不以 "report" 开头
cs.find_files("summary", "")
# ["summary.md(50)"]   -- suffix="" 匹配所有名称，prefix 为 "summary" 排除了所有 report_*.txt
```

### 第 3 关 —— 用户账户与存储配额

*用户*（user）有一个*容量*（capacity）——这是一个正整数字节数，在 `add_user` 创建账户时就固定下来——
并拥有零个或多个文件；只要一个文件存在，它的大小就会一直计入其主人的容量；用户的*剩余容量*
（remaining capacity）等于它的容量减去它当前拥有的所有文件的大小之和。`add_file` 创建的文件没有
主人，永远不会计入任何人的容量。从这一关开始，每次调用传入的 `user_id` 和 `capacity` 都已经满足
这些规则。

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

从这一关开始，`copy_file` 会让 `destination` 拥有和 `source` 相同的主人（如果 `source` 没有主人，
`destination` 也没有）。当 `source` 有主人时，`destination` 的大小会计入这个主人的剩余容量：如果这个
主人的剩余容量小于 `source` 的大小，`copy_file` 现在也会失败，返回 `False`，不改变任何东西——即使
`source` 存在、`destination` 也还没被占用。

接着上面的存储：

```text
cs.add_user("alice", 500)                          # -> True
cs.add_user("alice", 100)                          # -> False  -- "alice" 已经存在
cs.add_file_by("alice", "alice_a.txt", 300)        # -> 200  -- 剩余：500 - 300
cs.add_file_by("alice", "report.txt", 10)          # -> None  -- "report.txt" 已经存在，且没有主人
cs.add_file_by("carol", "c.txt", 10)               # -> None  -- "carol" 不是用户
cs.add_file_by("alice", "alice_b.txt", 250)        # -> None  -- 250 超过了 alice 剩余的 200
cs.get_file_size("alice_a.txt")                    # -> 300   -- 不受上面三次失败调用的影响
cs.copy_file("alice_a.txt", "alice_a_copy.txt")    # -> False -- 还需要 300，但 alice 只剩 200
cs.copy_file("report_final.txt", "report_final_copy.txt")  # -> True -- source 没有主人：不检查容量
cs.add_file_by("alice", "alice_small.txt", 100)    # -> 100   -- 剩余：200 - 100
cs.copy_file("alice_small.txt", "alice_small_copy.txt")     # -> True -- 正好需要 100，alice 正好剩 100
cs.get_file_size("alice_small_copy.txt")           # -> 100
cs.add_file_by("alice", "alice_c.txt", 1)          # -> None  -- alice 现在剩 0 字节
cs.add_user("bob", 50)                             # -> True
cs.add_file_by("bob", "bob_notes.txt", 20)         # -> 30
cs.merge_user("alice", "bob")                      # -> 30    -- alice：容量 500+50，已用 500+20
cs.add_file_by("bob", "bob_notes2.txt", 5)         # -> None  -- "bob" 已经不是用户了
cs.merge_user("alice", "bob")                      # -> None  -- source "bob" 已经不存在
cs.get_file_size("bob_notes.txt")                  # -> 20    -- 文件本身还在，现在归 alice 所有
cs.merge_user("alice", "alice")                    # -> None  -- target 等于 source
cs.merge_user("dave", "alice")                     # -> None  -- "dave" 不是用户
```

### 第 4 关 —— 压缩、备份与恢复

第 4 关新增四个方法；前两个方法只会作用于 `user_id` 当前拥有的文件。记 `COMPRESSED_SUFFIX` 为固定
字符串 `".cmp"`。

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

`compress_file` 的取整规则是：把原大小除以二后向上取整（$`\lceil \text{size} / 2 \rceil`$），所以
大小为 1 的文件压缩后仍然是大小为 1，绝不会变成 0。`decompress_file` 则是精确地翻倍，不做任何取整，
所以当被压缩前的大小是奇数时，解压并不总能还原出压缩前的精确大小。`decompress_file` 的容量检查，
比较的是 `name` 当前（压缩后）的大小与 `user_id` 的剩余容量——这时 `name` 仍按当前大小计入容量——
而不是翻倍之后即将变成的大小。

`restore_user` 只比较名称，不比较大小：如果 `user_id` 在备份时刻和现在都拥有某个名称，这个文件会被
原样保留，哪怕它的大小已经变了——这种情况会在这个名称被释放（通过 `compress_file` 或
`decompress_file`）之后、又被后续某次调用给了一个不相关的文件时出现；`restore_user` 并不会察觉，
留下来的就是那个不相关的文件。一份备份只在某个用户 ID 持续指向同一个账户时才属于它：把某个用户合并掉，
或者之后用 `add_user` 把它空出来的 ID 给一个新账户，都会让 `restore_user` 找不到任何东西。恢复操作
不会消耗备份：同一份备份可以被恢复不止一次。

接着上面的存储：

```text
cs.add_user("erin", 100)                     # -> True
cs.add_file_by("erin", "photo.raw", 41)      # -> 59
cs.backup_user("erin")                       # -> True  -- 快照：{photo.raw: 41}，容量 100
cs.compress_file("erin", "photo.raw")        # -> "photo.raw.cmp"  -- 大小：ceil(41 / 2) = 21
cs.get_file_size("photo.raw")                # -> None  -- 被压缩操作释放了
cs.get_file_size("photo.raw.cmp")            # -> 21
cs.add_file_by("erin", "logo.png", 30)       # -> 49    -- 剩余：100 - 21 - 30
cs.restore_user("erin")                      # -> 59    -- 回到 {photo.raw: 41}，容量 100
cs.get_file_size("photo.raw")                # -> 41    -- 被恢复操作重新创建
cs.get_file_size("photo.raw.cmp")            # -> None  -- 不在快照里，所以被删除
cs.get_file_size("logo.png")                 # -> None  -- 备份之后才添加，所以被删除

cs.add_user("frank", 60)
cs.add_file_by("frank", "video.raw", 25)          # -> 35
cs.compress_file("frank", "video.raw")            # -> "video.raw.cmp"  -- 大小：ceil(25 / 2) = 13
cs.decompress_file("frank", "video.raw.cmp")      # -> "video.raw"  -- 大小：13 * 2 = 26，不是原来的 25
cs.get_file_size("video.raw")                     # -> 26

cs.add_user("gina", 25)
cs.add_file_by("gina", "clip.raw", 25)            # -> 0
cs.compress_file("gina", "clip.raw")              # -> "clip.raw.cmp"  -- 大小：ceil(25 / 2) = 13，剩余 12
cs.decompress_file("gina", "clip.raw.cmp")        # -> None  -- 还需要 13 字节的容量，但只剩 12
```

## 参考解答

<details>
<summary>展开参考解答</summary>

值得先确认两件事：文件内容确实不在题目范围内，只追踪大小；`find_files` 的前缀和后缀就是字面意义上的
`str.startswith` 和 `str.endswith`，不是 glob 或正则表达式。

### Level 1

存储本身是一个从名称到大小的 `dict`；每个方法都只是一次直接的查找或写入。

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

三个方法都是 $O(1)$。

### Level 2

`find_files` 一次遍历 `_files` 完成过滤，再对匹配项排序；把大小取负号就能让排序按大小降序，同时
`sorted` 保持名称升序，一个 key 就同时做到了两件事。

```python
def find_files(self, prefix: str, suffix: str) -> list[str]:
    matches = [(name, size) for name, size in self._files.items()
               if name.startswith(prefix) and name.endswith(suffix)]
    matches.sort(key=lambda item: (-item[1], item[0]))    # NOTE: size descending, then name ascending
    return [f"{name}({size})" for name, size in matches]


CloudStorage.find_files = find_files
```

过滤整个存储是 $O(n)$（$`n`$ 是文件总数），再给 $k$ 个匹配项排序又加上 $O(k \log k)$。

### Level 3

所有权和容量信息放在 `_files` 之外另外两个 dict 里：`_owner` 记录一个文件属于哪个用户，`add_file`
创建的文件在里面没有条目；`_used` 记录每个用户目前被计入了多少字节，这样剩余容量只需要一次减法。

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

`merge_user` 是把两份容量、两份已用量分别相加，而不是从文件重新算一遍；结果不可能让
`_remaining(target)` 变成负数，因为合并前每个用户自己的已用量都不超过自己的容量，两个这样的不等式
相加，得到的仍然是同一个不等式。

`add_user`、`add_file_by`、`copy_file` 都是 $O(1)$；`merge_user` 是 $O(f)$（$`f`$ 是整个存储当前的
文件数），因为它要扫描 `_owner` 的每一项才能找到 `source` 的文件。

### Level 4

`compress_file` 和 `decompress_file` 都是删掉一个文件、插入另一个，`_used` 按两者大小之差调整。
`backup_user` 精确地拷出 `restore_user` 需要的那些名称与大小；`restore_user` 把这份快照当作期望
状态，把当前的所有权当作实际状态，只按名称做一次集合差来协调两者——两边都有的名称会被原样保留，
这正是上面提到的“名称被释放又复用”这种情况得以出现的原因。

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

对任何 $s \ge 1$ 都有 $\lceil s/2 \rceil \le s$，所以 `compress_file` 只可能减少其主人的已用量，
永远不需要容量检查；`decompress_file` 会让已用量恰好增加当前（解压前）的 `size`（翻倍后的文件替换掉
了一个已经按 `size` 计入的文件），这正是和 `_remaining` 比较的量。`restore_user` 会在删除或创建任何
东西之前，把每一个 `to_create` 的名称拿去和整个存储（不只是 `user_id` 自己的文件）核对，所以一次
无法完全成功的恢复会让每个文件、每一份容量都保持原样。

`compress_file` 和 `decompress_file` 都是 $O(1)$；`backup_user` 和 `restore_user` 是 $O(f)$
（$`f`$ 是存储当前的文件数）。

### 追问

- 为什么 `merge_user`、`backup_user`、`restore_user` 都要扫描整个存储，而不是只扫描某一个用户自己
  的文件？给每个用户维护一个 `dict[str, set[str]]` 记录它拥有的名称，并在每个修改点保持更新，就能
  让这三个方法各自的复杂度只取决于该用户自己的文件数。
- 如果一个用户需要同时保留不止一份备份呢？`_backups[user_id]` 可以换成一个按调用方给定的标签建索引
  的 dict，`restore_user` 相应地多接收一个标签参数。
- 为什么 `decompress_file` 不总能精确撤销对应的 `compress_file`？向上取整会丢掉奇数大小的最低位，
  翻倍也补不回来；要做到无损，需要把这一位信息存在别处，比如压缩后文件大小的奇偶性上。
- 题目里从不存储文件内容，所以 `compress_file` 的压缩比固定是二分之一。真实场景中的实现会把压缩后
  的大小，或者根据实际字节测出的压缩比，作为参数传入，而不是假定一个常数。

<details>
<summary>验证代码（可运行）</summary>

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
