# File Deduplication

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Coding · filesystem, hashing | ★★★★★ | Medium | SWE · MLE · Infra Eng | filesystem, hashing, io, concurrency | 3 parts / 55 min | Technical phone screen · Onsite |
<!-- meta:end -->

## Problem

Write `find_duplicate_files`, which walks a directory tree and reports every group of files whose
*content* is identical.

`root_path` is the path to a directory. Walk it recursively — files can be nested arbitrarily deep —
and partition every file under it into groups by content: two files belong to the same group exactly
when they hold the same sequence of bytes, compared byte for byte rather than as text, so no encoding
ever enters into the comparison. Each returned group is a list of at least two paths; a file with no
duplicate anywhere in the tree (a *singleton*) is excluded from the result entirely, and a tree with no
duplicate files at all produces an empty list, not `None` and not a list of one-element groups.

The order of the result is fixed, so that two correct implementations produce exactly the same output:
within a group, paths are sorted lexicographically ascending, and the groups themselves are sorted by
their own (already-sorted) first path, also lexicographically ascending. A path may be returned exactly
as `os.walk` produces it (joined with `os.path.join` under `root_path`); nothing needs to be made
absolute or otherwise normalised.

Three situations each have a precise rule:

- *Empty files.* A file with zero bytes is, by the byte-for-byte definition above, identical to every
  other empty file: if the tree contains two or more empty files, they form one duplicate group
  together, regardless of their names or locations.
- *Symbolic links.* A symbolic link is never followed and never reported: it contributes no path to any
  group, whether it points at a file already in the tree, at a directory, or at nothing (a *dangling*
  link). This holds even when the link's target is itself part of a duplicate group — the link still
  never appears in the output. A directory reached only through a symbolic link is, for the same reason,
  not walked into: only the tree of real directories under `root_path` is visited.
- *Unreadable files.* Opening a file can fail — most commonly a permissions error, but a file can also
  be removed between being listed and being read. Such a file is skipped: it never appears in any group,
  and a permissions error or similar on one file never raises out of `find_duplicate_files` or stops the
  rest of the walk from grouping correctly.

```text
root/
  a.txt             "hello world"
  notes/
    b.txt           "hello world"
    c.bin           11 bytes, not "hello world" but the same length
  notes/log/
    d.txt           "goodbye"
    e.txt           "goodbye"
  empty1.txt        ""
  notes/empty2.txt  ""

find_duplicate_files("root") == [
    ["root/a.txt", "root/notes/b.txt"],                    # both 11 bytes, "hello world"
    ["root/empty1.txt", "root/notes/empty2.txt"],          # both empty
    ["root/notes/log/d.txt", "root/notes/log/e.txt"],      # both 7 bytes, "goodbye"
]
# root/notes/c.bin is a singleton: same size as a.txt and b.txt, but different content -- excluded
```

### Part 1 — Correct grouping

```py
def find_duplicate_files(root_path: str) -> list[list[str]]: ...
```

This part has no efficiency requirement: any implementation that returns the groups defined above, in
the order defined above, is acceptable, including one that reads every file in full regardless of size.

### Part 2 — Large trees

`root_path` may now hold many files, some of them large, and most sharing their size with no one else.
Implement `find_duplicate_files_fast`, with the exact contract of `find_duplicate_files` above, subject
to one added requirement: on a tree where every file has a distinct size, the function must produce its
answer without reading the content of a single file — size alone already proves every file unique, and
that fact (`os.path.getsize`, not the file's bytes) is all the answer may cost. More generally, a file
may be read in full only once some other file has been shown to share both its size and a cheap sample
of its content — its first $k$ bytes, for a fixed, small $k$ you choose (4,096 is a reasonable
default).

```py
def find_duplicate_files_fast(root_path: str) -> list[list[str]]: ...   # same contract as Part 1
```

### Part 3 — Parallel hashing, same answer

Implement `find_duplicate_files_concurrent`, with the same contract again, that computes file hashes
concurrently instead of one file at a time.

```py
def find_duplicate_files_concurrent(root_path: str, max_workers: int | None = None,
                                     use_processes: bool = False) -> list[list[str]]: ...
```

`max_workers` is passed straight through to the underlying worker pool (`None` uses its default);
`use_processes` selects a process pool instead of the default thread pool. Whichever pool is used and
however many workers it has, the returned groups — including their order — must be exactly what
`find_duplicate_files_fast` returns for the same tree: concurrency is allowed to change only how fast
the answer arrives, never the answer itself.

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two points are worth confirming before coding: whether a directory reached only through a symbolic link
should be walked (here it is not — see Symbolic links above), and what the function returns when the
tree has no duplicates at all (here, an empty list, never `None` and never a list padded with
singletons).

### Part 1

A generator that walks the tree once and filters out symbolic links and non-regular entries keeps that
policy in one place, so every later part can reuse it unchanged. `os.walk` does not follow a symbolic
link to a directory on its own (`followlinks` defaults to `False`), so such a directory is already never
descended into; a symbolic link to a file, though, still shows up in `os.walk`'s list of files and has to
be filtered out explicitly with `os.path.islink`. Hashing streams every file through SHA-256 in
fixed-size chunks rather than a single `f.read()`, so a multi-gigabyte file never has to fit in memory at
once; an unreadable file raises inside that `open` or that `read`, which is caught and turned into a skip
rather than a crash. Two files land in the same group exactly when their full-content digests match,
which handles empty files without any special case at all — every empty file hashes to the same
constant digest, so they collide automatically.

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

Every file in the tree is opened and hashed in full here, which is correct but wasteful: a `stat` call,
or even a handful of bytes, is often already enough to prove two files differ, and this reads every byte
of every one of them regardless.

### Part 2

Bucketing by size costs nothing beyond the walk `find_duplicate_files` already does: `os.path.getsize`
reads filesystem metadata the operating system already has, never a file's content. Within a bucket of
$m$ files sharing one size $s$, Part 1's approach reads $ms$ bytes no matter how many of them turn out to
be duplicates. Sampling only the first $k$ bytes of each (`PREFIX_SIZE` below) costs $mk$ bytes and, for
two files whose content differs anywhere in that first $k$-byte window — true of nearly any two
unrelated files once $k$ is a few kilobytes — already proves them distinct, with no further reading.
Only files whose sampled prefix collides go on to a full read; call that count $d$ (for a genuine
duplicate group, $d$ is exactly its size; a coincidental prefix collision between otherwise different
files is possible but, at four kilobytes, astronomically unlikely outside adversarially crafted input).
Total bytes read is about $mk + ds$, against $ms$ for hashing everything: once $s \gg k$ and duplicates
are rare ($d \ll m$), that is roughly an $s / k$-fold reduction for every file that turns out unique — a
1 GiB file sampled at 4,096 bytes is ruled out, once it can be, after reading about
$2^{30} / 2^{12} = 2^{18} = 262{,}144$ times less of it than a full hash would.

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

If even a cryptographic collision at the full-hash tier is unacceptable — guarding against a file
deliberately crafted to match another's SHA-256 digest, rather than an accident — add a fourth, final
step: for each candidate group sharing `(size, full_hash)`, compare its files byte for byte
(`filecmp.cmp(a, b, shallow=False)`, or an explicit chunked comparison) before accepting the group. That
costs one more full read of files that are already, for all ordinary purposes, certain duplicates, so it
is worth paying only when acting on a false merge would itself be costly, such as deleting one of the
"duplicate" copies outright.

### Part 3

Reading a file blocks on the operating system while the data comes back from storage, and Python
releases the GIL for the duration of a blocking system call, so several threads that are each waiting on
their own read overlap almost perfectly — why `find_duplicate_files_concurrent` defaults to a thread
pool. A process pool sidesteps the GIL entirely by using separate interpreters, at the cost of spawning
workers and pickling arguments and results between them; that cost is worth paying once the work is
genuinely CPU-bound with no I/O left to overlap, such as a slow, pure-Python hash function. Which regime a
given machine and file set falls into is a question for a benchmark on the real target hardware, not for
reasoning about it in the abstract. Because a disk's speed varies by machine and a benchmark on a shared
or virtualised one can be noisy, the checks below isolate the concurrency mechanism itself — whether
blocked workers really do let others proceed — from any particular disk's throughput, by substituting a
fixed, artificial delay for the read and confirming wall time drops by close to the expected factor.

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

`find_duplicate_files_concurrent` keeps the same three tiers as Part 2, but hashes every candidate of a
given tier in one batch through the pool instead of one path at a time, so the parallelism spans the
whole tree within each tier rather than being confined to one size or one prefix bucket at a time. Since
`pool.map` returns results in the same order as its input regardless of which worker finishes first, and
the grouping step sorts its output at the very end regardless of insertion order, the result never
depends on timing — it is exactly `find_duplicate_files_fast`'s result, computed with more workers.

### Follow-ups

- **Hash choice.** SHA-256's accidental-collision probability, about $2^{-256}$, stays negligible even
  summed over far more files than a dedup tool will ever see, so trusting a `(size, hash)` match is safe
  in practice; MD5 is faster, but a deliberately constructed collision is already known and cheap to
  produce, which rules it out the moment the file contents cannot be trusted (a shared upload directory,
  say). BLAKE3 keeps that same resistance to a deliberate collision while running several times faster,
  using a tree structure that lets it use multiple cores and SIMD instructions on a single file; a
  non-cryptographic hash such as xxHash goes faster still by giving up that resistance entirely, a fine
  trade when the only adversary is chance.
- **A file changing during the scan.** A hash is not a snapshot: content that changes between the prefix
  read and the full read, or between one candidate's read and another's, can produce a wrong grouping.
  Guarding against it precisely needs OS-level support (a copy-on-write snapshot, or a lock the writer
  also respects); short of that, comparing a file's size and modification time, captured at listing time,
  against the same metadata read again just before trusting its hash catches most cases, at the cost of a
  re-scan for the files that moved.
- **All files the same size and huge.** If every file lands in one size bucket, every one of them must be
  hashed; the way to cut wasted work is to compare candidates chunk by chunk in lock-step and abandon a
  pair the moment one chunk differs, rather than hashing files that are going to turn out unequal all the
  way to the end.
- **Distributing across machines.** Shard the tree by size bucket (or by a hash of the size, if a few
  sizes dominate), send each shard to a worker, deduplicate within it exactly as above, and merge the
  resulting groups at the end — no worker ever needs to compare a file against one outside its own
  shard, since a duplicate always shares its size. A shard containing one very large size bucket becomes a
  straggler; splitting an oversized bucket further, by prefix hash, rebalances it.
- **Real-time detection.** Maintain a persistent index from `(size, full_hash)` to the paths sharing it,
  and use a filesystem-notification API (`inotify` on Linux, `fsnotify` elsewhere) to hash a file once it
  finishes being written rather than rescanning the tree; a write-completion event, not a write-start
  event, is what should trigger the hash, since the file is still changing until then.

<details>
<summary>Checks (runnable)</summary>

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
