# 重复文件查找

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 文件系统，哈希 | ★★★★★ | 中等 | SWE · MLE · Infra Eng | filesystem, hashing, io, concurrency | 3 个部分 / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

实现 `find_duplicate_files`：遍历一棵目录树，找出其中每一组*内容*完全相同的文件。

`root_path` 是一个目录的路径。递归遍历这棵树——文件可以嵌套在任意深度的子目录里——按内容把树下的每个文件分组：当且仅当两个文件的字节序列完全相同时，才算同一组；比较是逐字节进行的，而不是按文本比较，因此编码方式不会影响结果。返回的每一组至少包含两个路径；整棵树里找不到重复的文件——即*单例*（singleton）——完全不会出现在结果里；如果整棵树里根本没有重复文件，结果是空列表，既不是 `None`，也不是由单元素组构成的列表。

结果的顺序是固定的，这样两个正确的实现才能给出完全一致的输出：组内的路径按字典序升序排列；各组之间则按各自（已经排好序的）第一个路径的字典序升序排列。路径可以按 `os.walk` 产生的原样返回（用 `os.path.join` 拼接在 `root_path` 之下）；不需要做成绝对路径，也不需要做其他规范化处理。

以下三种情况各自有一条精确的规则：

- *空文件*（empty file）。按上面逐字节的定义，一个零字节的文件和任何其他空文件都是相同的：如果树里存在两个或更多空文件，无论它们叫什么名字、位于哪个目录，都属于同一个重复组。
- *符号链接*（symbolic link）。符号链接永远不会被跟随，也永远不会出现在结果里：不管它指向树里已有的某个文件、指向某个目录，还是什么都不指向（*悬空链接*，dangling link），它都不会给任何分组贡献一个路径。即使链接指向的目标本身属于某个重复组，链接自己也依然不会出现在输出里。出于同样的原因，只能通过符号链接到达的目录也不会被遍历：只有 `root_path` 下真实的目录树会被访问。
- *不可读的文件*（unreadable file）。打开文件可能失败——最常见的是权限错误，但文件也可能在被列出之后、被读取之前就被删除了。这样的文件会被跳过：它不会出现在任何分组里，一个文件上的权限错误或类似问题，既不会让 `find_duplicate_files` 抛出异常，也不会妨碍遍历的其余部分被正确分组。

```text
root/
  a.txt             "hello world"
  notes/
    b.txt           "hello world"
    c.bin           11 个字节，不是 "hello world"，但长度相同
  notes/log/
    d.txt           "goodbye"
    e.txt           "goodbye"
  empty1.txt        ""
  notes/empty2.txt  ""

find_duplicate_files("root") == [
    ["root/a.txt", "root/notes/b.txt"],                    # 都是 11 字节，内容 "hello world"
    ["root/empty1.txt", "root/notes/empty2.txt"],          # 都是空文件
    ["root/notes/log/d.txt", "root/notes/log/e.txt"],      # 都是 7 字节，内容 "goodbye"
]
# root/notes/c.bin 是单例：和 a.txt、b.txt 大小相同，但内容不同——被排除在外
```

### Part 1 —— 正确分组

```py
def find_duplicate_files(root_path: str) -> list[list[str]]: ...
```

这一部分没有效率上的要求：只要实现返回的分组符合上面的定义、顺序也符合上面的定义，就是正确的，哪怕它不论文件大小都把每个文件完整读入内存也可以。

### Part 2 —— 大规模目录树

现在 `root_path` 下可能有很多文件，其中一些很大，而且大多数文件的大小在整棵树里是独一无二的。实现 `find_duplicate_files_fast`，接口和约定与上面的 `find_duplicate_files` 完全一致，但多一条要求：如果树里每个文件的大小都互不相同，函数必须能在不读取任何一个文件内容的情况下给出答案——大小本身已经足以证明每个文件都是唯一的，而这个事实（`os.path.getsize`，而不是文件的字节内容）是这次调用唯一允许付出的代价。更一般地说，只有当另一个文件被证明与它大小相同、并且内容的一小段样本也相同时——取它开头的 $k$ 个字节，$k$ 是你选定的一个固定的小值（4,096 是一个合理的默认值）——这个文件才可以被完整读取。

```py
def find_duplicate_files_fast(root_path: str) -> list[list[str]]: ...   # same contract as Part 1
```

### Part 3 —— 并行哈希，结果不变

实现 `find_duplicate_files_concurrent`，接口和约定不变，但改为并发地计算文件哈希，而不是一次只算一个。

```py
def find_duplicate_files_concurrent(root_path: str, max_workers: int | None = None,
                                     use_processes: bool = False) -> list[list[str]]: ...
```

`max_workers` 直接传给底层的工作池（`None` 表示使用其默认值）；`use_processes` 决定用进程池还是默认的线程池。不管用的是哪种池、开了多少个 worker，返回的分组——包括顺序——都必须和 `find_duplicate_files_fast` 对同一棵树返回的结果完全一致：并发只能改变得到答案的速度，绝不能改变答案本身。

## 参考解答

<details>
<summary>展开参考解答</summary>

动手之前有两点值得先确认：只能通过符号链接到达的目录要不要遍历（这里的答案是不要——见上文“符号链接”一条），以及树里完全没有重复文件时函数应该返回什么（这里的答案是空列表，绝不是 `None`，也绝不是用单例凑出来的列表）。

### Part 1

用一个生成器遍历整棵树、并过滤掉符号链接和非常规文件，把这条策略集中在一个地方，后面每一部分都可以原样复用它。`os.walk` 本身不会跟随指向目录的符号链接（`followlinks` 默认为 `False`），所以这样的目录天然就不会被继续遍历；但指向文件的符号链接仍然会出现在 `os.walk` 给出的文件列表里，需要用 `os.path.islink` 显式过滤掉。哈希计算把每个文件按固定大小的分块流式地喂给 SHA-256，而不是一次性 `f.read()`，这样再大的文件也不需要一次性装进内存；不可读的文件会在 `open` 或 `read` 内部抛出异常，这里捕获它并把它变成跳过，而不是让整个遍历崩溃。两个文件当且仅当完整内容的摘要相同才归入同一组，这也顺带处理了空文件，不需要任何特殊情况——每个空文件都会得到同一个固定的摘要值，自然而然地碰到一起。

```python
import hashlib
import os
from collections import defaultdict

CHUNK_SIZE = 1 << 20   # 1 MiB per streamed read, so a huge file is never loaded into memory at once


def _iter_files(root_path: str):
    """Yields every regular, non-symlink file under root_path. A symbolic link -- to a file, to a
    directory, or dangling -- is never followed and never yielded, so a directory reached only through
    one is not walked into either."""
    for dirpath, _dirnames, filenames in os.walk(root_path):   # followlinks=False by default
        for name in filenames:
            path = os.path.join(dirpath, name)
            if os.path.islink(path) or not os.path.isfile(path):
                continue    # NOTE: islink() first -- isfile() alone would follow the symlink to decide
            yield path


def _hash_file(path: str) -> bytes | None:
    """Streams path through SHA-256 in fixed-size chunks. Returns None if it cannot be read."""
    hasher = hashlib.sha256()
    try:
        with open(path, "rb") as f:
            while chunk := f.read(CHUNK_SIZE):
                hasher.update(chunk)
    except OSError:
        return None      # NOTE: permission error or similar -- skip this file, never crash the walk
    return hasher.digest()


def find_duplicate_files(root_path: str) -> list[list[str]]:
    by_hash = defaultdict(list)
    for path in _iter_files(root_path):
        digest = _hash_file(path)
        if digest is not None:
            by_hash[digest].append(path)
    groups = [sorted(paths) for paths in by_hash.values() if len(paths) >= 2]
    groups.sort(key=lambda g: g[0])   # NOTE: groups ordered by their own (already-sorted) first path
    return groups
```

这里对树里的每个文件都完整地打开并哈希了一遍，结果是对的，但很浪费：往往只需要一次 `stat` 调用，甚至只需要几个字节，就足以证明两个文件不同，而这个做法无论如何都会把每个文件的每个字节都读一遍。

### Part 2

按大小分桶不需要在 `find_duplicate_files` 已经做的遍历之外多付出任何代价：`os.path.getsize` 读取的是操作系统本来就有的文件系统元数据，从不读取文件内容。在一个大小为 $s$、有 $m$ 个文件的桶里，Part 1 的做法无论其中有多少文件最终是重复的，都要读 $ms$ 字节。只采样每个文件开头的 $k$ 个字节（下面代码里的 `PREFIX_SIZE`）只需要读 $mk$ 字节，而只要两个文件的内容在开头这 $k$ 字节的窗口内任何一处不同——一旦 $k$ 有几千字节，这对几乎任何两个不相关的文件都成立——就已经足以证明它们不同，不需要再往下读。只有采样前缀发生碰撞的文件才会进入完整读取；把这个数量记作 $d$（对于真正的重复组，$d$ 恰好就是这个组的大小；两个内容其实不同的文件碰巧前缀相同也是可能的，但在四千字节这个量级上，除非是刻意构造的对抗性输入，否则这种概率低到可以忽略）。总的读取字节数大约是 $mk + ds$，而把所有文件都完整哈希一遍则要读 $ms$ 字节：一旦 $s \gg k$ 且重复很少见（$d \ll m$），对每一个最终被证明是唯一的文件来说，这大约是 $s / k$ 倍的削减——对一个用 4,096 字节采样的 1 GiB 文件，只要它能被更早地排除，读取的字节数大约只有完整哈希的 $2^{30} / 2^{12} = 2^{18} = 262{,}144$ 分之一。

```python
PREFIX_SIZE = 4096   # bytes sampled for the cheap pre-filter -- a filesystem block is a natural size


def _hash_prefix(path: str) -> bytes | None:
    """Hashes only the first PREFIX_SIZE bytes of path (the whole file, if it is shorter). Returns
    None if it cannot be read."""
    try:
        with open(path, "rb") as f:
            data = f.read(PREFIX_SIZE)
    except OSError:
        return None
    return hashlib.sha256(data).digest()


def find_duplicate_files_fast(root_path: str) -> list[list[str]]:
    by_size = defaultdict(list)
    for path in _iter_files(root_path):
        try:
            size = os.path.getsize(path)          # NOTE: metadata the OS already has -- no content read
        except OSError:
            continue                               # gone, or unreadable metadata: treat as unreadable
        by_size[size].append(path)

    by_prefix = defaultdict(list)
    for size, paths in by_size.items():
        if len(paths) < 2:
            continue                               # NOTE: tier 1 -- a size nothing else shares can't collide
        for path in paths:
            digest = _hash_prefix(path)
            if digest is not None:
                by_prefix[(size, digest)].append(path)

    groups = []
    for (_size, _prefix_digest), candidates in by_prefix.items():
        if len(candidates) < 2:
            continue                               # NOTE: tier 2 -- distinct prefixes already rule these out
        by_full = defaultdict(list)
        for path in candidates:
            digest = _hash_file(path)              # tier 3 -- only survivors pay for a full read
            if digest is not None:
                by_full[digest].append(path)
        groups.extend(paths for paths in by_full.values() if len(paths) >= 2)

    groups = [sorted(g) for g in groups]
    groups.sort(key=lambda g: g[0])
    return groups
```

如果就连完整哈希这一层出现密码学意义上的碰撞也不能接受——防的是有人故意构造出一个 SHA-256 摘要与另一个文件相同的文件，而不是巧合——可以再加第四层、也是最后一层：对每一组共享 `(size, full_hash)` 的候选文件，在接受这一组之前先逐字节比较它们的内容（`filecmp.cmp(a, b, shallow=False)`，或者手写一个分块比较）。这会让已经在一切实际意义上确定重复的文件再多付出一次完整读取的代价，所以只有当误判合并本身代价高昂时才值得——比如据此直接删除某个重复副本。

### Part 3

读取文件时会在操作系统那里阻塞，等待数据从存储设备返回，而 Python 在阻塞的系统调用期间会释放 GIL，所以多个各自等待自己那次读取的线程几乎可以完美地重叠起来——这就是 `find_duplicate_files_concurrent` 默认使用线程池的原因。进程池彻底绕开了 GIL，用的是相互独立的解释器，代价是创建 worker 进程、在进程之间序列化参数和结果；这份代价只有在工作负载确实是 CPU 密集型、已经没有 I/O 等待可以重叠时才值得付出，比如一个很慢的纯 Python 哈希实现。具体一台机器和一批文件属于哪种情形，要靠在真实目标硬件上实测来回答，而不是凭空推理。由于磁盘速度因机器而异，在共享或虚拟化的机器上做基准测试还容易受到干扰，下面的验证代码把并发机制本身——被阻塞的 worker 是否真的能让别的 worker 继续——和某一块具体磁盘的吞吐量分开来看，做法是用一个固定的、人为设置的延迟代替真正的读取，然后确认墙钟时间确实按预期的倍数下降。

```python
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor


def _hash_many(paths, hash_fn, max_workers, pool_cls) -> dict[str, bytes]:
    """Applies hash_fn to every path using a worker pool; returns {path: digest}, silently omitting
    paths that could not be read."""
    if not paths:
        return {}
    with pool_cls(max_workers=max_workers) as pool:
        digests = pool.map(hash_fn, paths)   # NOTE: map() preserves input order, so the zip below is safe
        return {path: digest for path, digest in zip(paths, digests) if digest is not None}


def find_duplicate_files_concurrent(root_path: str, max_workers: int | None = None,
                                     use_processes: bool = False) -> list[list[str]]:
    pool_cls = ProcessPoolExecutor if use_processes else ThreadPoolExecutor

    by_size = defaultdict(list)
    for path in _iter_files(root_path):
        try:
            by_size[os.path.getsize(path)].append(path)
        except OSError:
            continue
    tier1 = [(size, p) for size, paths in by_size.items() if len(paths) >= 2 for p in paths]

    prefixes = _hash_many([p for _, p in tier1], _hash_prefix, max_workers, pool_cls)
    by_prefix = defaultdict(list)
    for size, path in tier1:
        if path in prefixes:
            by_prefix[(size, prefixes[path])].append(path)

    tier3 = [(size, p) for (size, _pfx), paths in by_prefix.items() if len(paths) >= 2 for p in paths]
    fulls = _hash_many([p for _, p in tier3], _hash_file, max_workers, pool_cls)
    by_full = defaultdict(list)
    for size, path in tier3:
        if path in fulls:
            by_full[(size, fulls[path])].append(path)

    groups = [sorted(paths) for paths in by_full.values() if len(paths) >= 2]
    groups.sort(key=lambda g: g[0])
    return groups
```

`find_duplicate_files_concurrent` 保留了和 Part 2 相同的三层结构，但把每一层里的候选文件一次性整批交给池去哈希，而不是一次只处理一个路径，这样并行的范围覆盖了整棵树在这一层的全部候选，而不是局限在某一个大小桶或某一个前缀桶里。由于 `pool.map` 返回结果的顺序和输入顺序一致，与哪个 worker 先完成无关，而分组这一步无论输入顺序如何最后都会重新排序，所以结果完全不依赖时序——它就是 `find_duplicate_files_fast` 的结果，只是用更多 worker 算出来的。

### 追问

- **哈希的选择。** SHA-256 意外碰撞的概率大约是 $2^{-256}$，哪怕加总一个去重工具一辈子会遇到的文件数量，这个概率依然可以忽略，所以信任 `(size, hash)` 相同即判定重复，在实践中是安全的；MD5 更快，但构造一次碰撞已经是已知且成本很低的事情，一旦文件内容不可信（比如一个共享的上传目录），MD5 就不能再用了。BLAKE3 保留了对“蓄意构造的碰撞”的抵抗能力，同时借助树状结构让单个文件内部也能用上多核和 SIMD 指令，速度比 SHA-256 快上好几倍；像 xxHash 这样的非密码学哈希则连这层抵抗能力也一并放弃，换取更高的吞吐量——只要对手只是偶然性，这笔交易就划算。
- **扫描过程中文件发生变化。** 哈希不是快照：内容在前缀读取和完整读取之间、或者在一个候选文件的读取和另一个候选文件的读取之间发生了变化，都可能得到错误的分组。要精确地防住这一点，需要操作系统层面的支持（写时复制快照，或者一把写入方也会遵守的锁）；退而求其次，可以在列出文件时记下它的大小和修改时间，在真正信任其哈希之前再读一次同样的元数据比较一下，这能挡住大多数情况，代价是要为那些确实变化过的文件重新扫描。
- **所有文件大小相同且都很大。** 如果所有文件都落进同一个大小桶，那就必须把它们全部哈希一遍；省掉多余工作的办法是让候选文件按块同步比较，只要有一块不同就立刻放弃这一对，而不是把注定不相等的文件从头哈希到尾。
- **跨机器分布式处理。** 按大小桶（如果个别大小的文件特别多，也可以按大小的哈希）把树分片，把每一片发给一个 worker，在片内按上面的方法各自去重，最后再合并各片的分组结果——因为重复文件必然大小相同，所以任何一个 worker 都不需要拿自己分片之外的文件来比较。如果某一片恰好包含一个特别大的大小桶，就会成为拖后腿的那个；再按前缀哈希把这个超大的桶进一步拆分，就能重新平衡负载。
- **实时检测。** 维护一个从 `(size, full_hash)` 到共享该值的路径集合的持久索引，用文件系统通知接口（Linux 上是 `inotify`，其他系统上是 `fsnotify`）在文件写完之后再去计算哈希，而不是重新扫描整棵树；触发哈希计算的应该是“写入完成”事件，而不是“写入开始”事件，因为在写完之前文件内容还在变化。

<details>
<summary>验证代码（可运行）</summary>

```python
import filecmp
import random
import tempfile
import time
from unittest import mock

# --- the worked example in the Problem section ---
with tempfile.TemporaryDirectory() as root:
    os.makedirs(os.path.join(root, "notes", "log"))
    def _write(rel, content):
        p = os.path.join(root, rel)
        with open(p, "w") as f:
            f.write(content)
        return p
    a = _write("a.txt", "hello world")
    b = _write("notes/b.txt", "hello world")
    c = os.path.join(root, "notes", "c.bin")
    with open(c, "wb") as f:
        f.write(bytes([1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]))   # 11 bytes, same length as "hello world", different content
    d = _write("notes/log/d.txt", "goodbye")
    e = _write("notes/log/e.txt", "goodbye")
    e1 = _write("empty1.txt", "")
    e2 = _write("notes/empty2.txt", "")
    link = os.path.join(root, "link_to_a.txt")
    os.symlink(a, link)
    dangling = os.path.join(root, "dangling.txt")
    os.symlink(os.path.join(root, "does_not_exist"), dangling)

    expected = sorted([sorted([a, b]), sorted([d, e]), sorted([e1, e2])], key=lambda g: g[0])
    for fn in (find_duplicate_files, find_duplicate_files_fast, find_duplicate_files_concurrent):
        got = fn(root)
        assert got == expected, (fn.__name__, got, expected)
    assert find_duplicate_files_concurrent(root, use_processes=True) == expected
    assert find_duplicate_files(os.path.join(root, "notes", "log")) == [[d, e]]   # a sub-tree on its own

# a tree with no duplicates at all returns an empty list, not None and not a list of singletons
with tempfile.TemporaryDirectory() as root:
    for i in range(5):
        with open(os.path.join(root, f"unique{i}.txt"), "w") as f:
            f.write(f"content number {i}")
    for fn in (find_duplicate_files, find_duplicate_files_fast, find_duplicate_files_concurrent):
        assert fn(root) == []

# an empty directory, and a single lone empty file: still no duplicates
with tempfile.TemporaryDirectory() as root:
    assert find_duplicate_files(root) == []
    open(os.path.join(root, "alone.txt"), "w").close()
    assert find_duplicate_files(root) == []

# --- an unreadable file (simulated so the check is valid whether or not it runs as root) ---
def _deny(denied_path):
    real_open = open
    def opener(path, mode="r", *args, **kwargs):
        if os.path.abspath(path) == denied_path:
            raise PermissionError(f"simulated: {path} is not readable")
        return real_open(path, mode, *args, **kwargs)
    return opener

with tempfile.TemporaryDirectory() as root:
    p1 = os.path.join(root, "a.txt")
    p2 = os.path.join(root, "b.txt")
    secret = os.path.join(root, "secret.txt")
    for p, content in [(p1, "duplicated"), (p2, "duplicated"), (secret, "duplicated-but-unreadable")]:
        with open(p, "w") as f:
            f.write(content)
    expected = [sorted([p1, p2])]
    with mock.patch("builtins.open", _deny(os.path.abspath(secret))):
        for fn in (find_duplicate_files, find_duplicate_files_fast, find_duplicate_files_concurrent):
            assert fn(root) == expected, fn.__name__   # secret.txt excluded; the other pair still found

# --- Part 2: quantify the bytes read, with an independent counting open() ---
class _CountingReader:
    def __init__(self, f, counter):
        self._f, self._counter = f, counter
    def read(self, n=-1):
        data = self._f.read(n)
        self._counter[0] += len(data)
        return data
    def __enter__(self):
        return self
    def __exit__(self, *exc_info):
        self._f.close()
        return False


def _measure_bytes_read(fn, root):
    counter = [0]
    real_open = open
    def counting_open(path, mode="rb"):
        return _CountingReader(real_open(path, mode), counter)
    with mock.patch("builtins.open", counting_open):
        result = fn(root)
    return result, counter[0]


assert (1 << 30) // PREFIX_SIZE == 262_144   # the 1 GiB / 4,096-byte figure quoted above

FILE_SIZE = 150_000   # well above PREFIX_SIZE, so the prefix tier reads only a fraction of each file
with tempfile.TemporaryDirectory() as root:
    rng = random.Random(1)
    total_bytes = 0
    for i in range(90):                          # 90 same-size files, content independent and distinct
        content = rng.randbytes(FILE_SIZE)
        with open(os.path.join(root, f"f{i}.bin"), "wb") as f:
            f.write(content)
        total_bytes += FILE_SIZE
    dup_content = rng.randbytes(FILE_SIZE)
    for name in ("dupA.bin", "dupB.bin"):         # one true duplicate pair, same size as the rest
        with open(os.path.join(root, name), "wb") as f:
            f.write(dup_content)
        total_bytes += FILE_SIZE

    naive_groups, naive_bytes = _measure_bytes_read(find_duplicate_files, root)
    fast_groups, fast_bytes = _measure_bytes_read(find_duplicate_files_fast, root)
    assert naive_groups == fast_groups and len(naive_groups) == 1
    assert naive_bytes == total_bytes                 # Part 1 always reads every byte of every file
    assert fast_bytes < naive_bytes // 10             # Part 2 reads far fewer bytes on the very same tree

# --- an independent brute force: compare every pair of files' actual bytes, merge into groups ---
def _brute_force_duplicates(root_path):
    paths = []
    for dirpath, _dirnames, filenames in os.walk(root_path):
        for name in filenames:
            p = os.path.join(dirpath, name)
            if not os.path.islink(p) and os.path.isfile(p):
                paths.append(p)
    readable = []
    for p in paths:
        try:
            with open(p, "rb"):
                pass
        except OSError:
            continue
        readable.append(p)
    parent = {p: p for p in readable}
    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x
    def union(a, b):
        ra, rb = find(a), find(b)
        if ra != rb:
            parent[ra] = rb
    for i in range(len(readable)):
        for j in range(i + 1, len(readable)):
            if filecmp.cmp(readable[i], readable[j], shallow=False):
                union(readable[i], readable[j])
    grouped = defaultdict(list)
    for p in readable:
        grouped[find(p)].append(p)
    groups = [sorted(g) for g in grouped.values() if len(g) >= 2]
    groups.sort(key=lambda g: g[0])
    return groups


def _random_tree(rng, root, n_files=18, n_dirs=4, alphabet_size=6, max_len=25):
    dirs = [root]
    for i in range(n_dirs):
        d = os.path.join(rng.choice(dirs), f"d{i}")
        os.makedirs(d, exist_ok=True)
        dirs.append(d)
    contents = [bytes(rng.randrange(256) for _ in range(rng.randint(0, max_len))) for _ in range(alphabet_size)]
    paths = []
    for i in range(n_files):
        path = os.path.join(rng.choice(dirs), f"f{i}.bin")
        with open(path, "wb") as f:
            f.write(rng.choice(contents))         # a small alphabet forces plenty of real duplicates
        paths.append(path)
    for i in range(rng.randint(0, 3)):             # a few symlinks to real files
        os.symlink(rng.choice(paths), os.path.join(rng.choice(dirs), f"link{i}"))
    if rng.random() < 0.5:                         # sometimes also a dangling one
        os.symlink(os.path.join(root, "nowhere"), os.path.join(rng.choice(dirs), "dangling"))
    return paths


with tempfile.TemporaryDirectory() as base:
    for seed in range(150):
        root = os.path.join(base, f"t{seed}")
        os.makedirs(root)
        _random_tree(random.Random(seed), root)
        expected = _brute_force_duplicates(root)
        got_sequential = find_duplicate_files(root)
        got_fast = find_duplicate_files_fast(root)
        got_concurrent = find_duplicate_files_concurrent(root, max_workers=3)
        assert got_sequential == expected, seed
        assert got_fast == expected, seed
        assert got_concurrent == expected, seed
        if seed < 6:                               # a handful also cross-checked with a process pool
            assert find_duplicate_files_concurrent(root, use_processes=True) == expected, seed

# --- Part 3: the concurrent version, timed against the sequential one under simulated slow storage ---
# A real disk's speed is not under this check's control, so it substitutes a fixed, artificial delay for
# every read instead: what is being measured is whether blocked workers let others proceed, which is
# exactly the property a thread pool is supposed to provide, not the throughput of any particular disk.
def _slow_open(delay):
    real_open = open
    def opener(path, mode="rb"):
        time.sleep(delay)
        return real_open(path, mode)
    return opener


with tempfile.TemporaryDirectory() as root:
    shared_content = os.urandom(500)
    for i in range(40):
        with open(os.path.join(root, f"f{i}.bin"), "wb") as f:
            f.write(shared_content)                # all 40 collide, so all 40 reach the full-hash tier

    with mock.patch("builtins.open", _slow_open(0.01)):
        t0 = time.perf_counter()
        sequential_result = find_duplicate_files_fast(root)
        sequential_seconds = time.perf_counter() - t0

    with mock.patch("builtins.open", _slow_open(0.01)):
        t0 = time.perf_counter()
        concurrent_result = find_duplicate_files_concurrent(root)
        concurrent_seconds = time.perf_counter() - t0

    assert concurrent_result == sequential_result and len(sequential_result) == 1
    assert sequential_seconds > 3 * concurrent_seconds, (sequential_seconds, concurrent_seconds)  # measured: ~5.5x

print("all checks passed")
```

</details>

</details>
