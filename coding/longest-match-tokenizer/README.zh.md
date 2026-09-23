# 最长匹配分词器

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 字符串处理 | ★★★☆☆ | 中等 | SWE · MLE | greedy, string-processing, trie, tokenization | 3 个部分 / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

*词表*（vocabulary）`vocab` 是一个 `dict[str, int]`，键是互不相同的非空字符串，称为*词条*
（token）；它的值是任意整数 id，值之间不要求互不相同。`unk_id` 是与 `vocab` 分开传入的一个整数，
且不等于 `vocab` 中的任何一个值，所以下面每个函数返回的 id 都不含糊：它要么是 `vocab` 中的某个值
（真正匹配上了），要么是 `unk_id`（没有匹配上），两者不会混淆。`text` 是一个普通的 Python 字符串，
其中可能包含 `vocab` 任何词条都不包含的字符。

### Part 1 —— 贪心最长匹配

从左到右对 `text` 分词。在当前位置，如果 `text` 从这个位置开始恰好以 `vocab` 的某个键开头，就说这个
键在该位置*匹配*（match）。如果至少有一个键匹配，就取其中最长的那个——两个不同的键如果都在同一位置
匹配且长度相同，那它们必然是同一个字符串，所以最长匹配一旦存在就必然唯一——把它的 id 追加到输出中，
并把位置向前移动这个键的长度。如果没有任何键匹配，哪怕是该位置的单个字符也不匹配，就把 `unk_id`
追加到输出中，位置只向前移动一个字符。如此重复，直到位置到达 `text` 末尾；`text` 为空时，`tokenize`
返回 `[]`。

```py
def tokenize(text: str, vocab: dict[str, int], unk_id: int) -> list[int]:
    """Greedy longest match from the left. One id per matched token, unk_id for each
    character that matches no key of vocab."""
```

例如：

```text
vocab = {"a": 1, "ab": 2, "abc": 3, "b": 4, "bcd": 5}, unk_id = 0

tokenize("abcxbcdy", vocab, 0)
# 位置 0："a"、"ab"、"abc" 都在此处匹配，最长的是 "abc" -> 输出 3，移动到位置 3
# 位置 3："x" 不匹配任何键                              -> 输出 0，移动到位置 4
# 位置 4："b"、"bcd" 都在此处匹配，最长的是 "bcd"        -> 输出 5，移动到位置 7
# 位置 7："y" 不匹配任何键                              -> 输出 0，移动到位置 8（= len(text)）
# -> [3, 0, 5, 0]
```

### Part 2 —— 有界与合并

设 $L$ 为 `vocab` 中最长键的长度，`vocab` 为空时记 $L$ 为 $0$；记 $n$ 为 `len(text)`。`tokenize_merged`
返回和 `tokenize` 相同的 token 序列，只是连续两个或更多个 `unk_id` 组成的极大子序列会合并成一个
`unk_id`；其他重复出现的 id——两个相邻的匹配 token 恰好携带同一个 id——保持原样，不做合并。在每个
位置，`tokenize_merged` 至多只能尝试 $L$ 种候选长度，绝不能像 `tokenize` 那样从剩余字符数一路尝试到
$1$：这样的完整扫描一旦 `text` 很长而 `vocab` 里的键都很短，就太慢了。

```py
def tokenize_merged(text: str, vocab: dict[str, int], unk_id: int) -> list[int]:
    """Same token sequence as tokenize(text, vocab, unk_id), except that a maximal run of
    unk_id collapses to one unk_id. Tries at most L candidate lengths per position, L = the
    longest key of vocab, instead of tokenize's full n - i."""
```

例如：

```text
vocab = {"a": 1, "ab": 2, "abc": 3, "b": 4, "bcd": 5}, unk_id = 0

tokenize("abczzbcd", vocab, 0)         -> [3, 0, 0, 5]   # "abc"、"z"、"z"、"bcd"
tokenize_merged("abczzbcd", vocab, 0)  -> [3, 0, 5]      # 两个连续的 UNK 合并成了一个
```

### Part 3 —— 可复用的 trie

现在 `vocab` 很大，并且 `tokenize_merged` 要在共享同一个 `vocab` 的许多段独立文本上分别调用——比如
一批文档，每篇调用一次——所以为匹配准备 `vocab` 的开销应该只付出一次，而不是每次调用都重来一遍。
实现 `TrieTokenizer`：它的构造函数只构建一次基于 `vocab` 的 trie，`tokenize` 对调用它的任意 `text`
都返回与 `tokenize_merged(text, vocab, unk_id)` 完全相同的结果，计算方式是遍历这棵 trie，而不是
重新扫描 `vocab`。

```py
class TrieTokenizer:
    def __init__(self, vocab: dict[str, int], unk_id: int) -> None:
        """Builds a trie from vocab once, for reuse across many calls to tokenize."""

    def tokenize(self, text: str) -> list[int]:
        """Returns exactly what tokenize_merged(text, vocab, unk_id) would, computed by
        walking the trie built in __init__."""
```

例如：

```text
vocab = {"a": 1, "ab": 2, "abc": 3, "b": 4, "bcd": 5}, unk_id = 0
tk = TrieTokenizer(vocab, 0)

tk.tokenize("abcxbcdy")  -> [3, 0, 5, 0]   # 与上面 tokenize 例子的结果相同
tk.tokenize("abczzbcd")  -> [3, 0, 5]      # 与上面 tokenize_merged 例子的结果相同
```

## 参考解答

<details>
<summary>展开参考解答</summary>

编码前有两点值得和面试官确认：最长匹配是否可能出现并列（不会——两个长度相同、又都在同一位置匹配的键
必然是同一个字符串，也就是同一个 dict 键），以及 `unk_id` 是否可能和 `vocab` 中已有的某个值重合
（这里假定不会，这样返回的 id 来源才不会含糊）。

### Part 1

在每个位置，从剩余字符数开始，依次尝试长度递减到 `1` 的每一种候选长度，取第一个使 `text` 对应切片
成为 `vocab` 某个键的长度——因为是从最长开始尝试，第一次命中自然就是最长匹配。`for`/`else` 正好表达
了这一点：只有循环没有 `break` 就结束时，`else` 才会执行。

```python
def tokenize(text: str, vocab: dict[str, int], unk_id: int) -> list[int]:
    ids = []
    i, n = 0, len(text)
    while i < n:
        for length in range(n - i, 0, -1):        # longest possible down to a single character
            piece = text[i:i + length]
            if piece in vocab:
                ids.append(vocab[piece])
                i += length
                break
        else:
            ids.append(unk_id)                     # NOTE: no key matched, not even text[i] alone
            i += 1
    return ids
```

位置在 `while` 循环每一轮至少前进一格，所以循环至多执行 $n$ 次；每一轮尝试至多 $n - i$ 种候选长度，
最坏情况下为 $O(n^2)$ 时间——也就是一段和 `vocab` 完全匹配不上的长 `text`。

### Part 2

每次调用只计算一次 `max_len`，对 `vocab` 中的 $k$ 个键来说是 $O(k)$；之后把候选长度的上限从
`n - i` 改成 `max_len`，就把内层循环限制到了 $L$。合并在同一趟里完成：只有当上一个追加的 id 还不是
`unk_id` 时才追加新的 `unk_id`，因此输出里任何一段连续的 `unk_id` 长度都不会超过 `1`。

```python
def tokenize_merged(text: str, vocab: dict[str, int], unk_id: int) -> list[int]:
    max_len = max((len(token) for token in vocab), default=0)   # L: longest key, 0 if vocab is empty
    ids = []
    i, n = 0, len(text)
    while i < n:
        for length in range(min(max_len, n - i), 0, -1):        # NOTE: bounded by L, not by n - i
            piece = text[i:i + length]
            if piece in vocab:
                ids.append(vocab[piece])
                i += length
                break
        else:
            if not (ids and ids[-1] == unk_id):                 # NOTE: merge into the previous UNK
                ids.append(unk_id)
            i += 1
    return ids
```

现在内层循环每个位置至多尝试 $L$ 种候选长度，整个运行过程一共 $O(n \cdot L)$ 次字典查找。但每次查找的
代价不止 $O(1)$：`text[i:i + length]` 得先复制出 `length` 个字符，`vocab` 才能对结果做哈希，所以长度
为 $\ell$ 的一次查找代价是 $O(\ell)$，同一个位置上所有查找加起来是 $O(L^2)$——最坏情况下总时间是
$O(n \cdot L^2)$，而不是 $O(n \cdot L)$。Part 3 去掉的正是这一份按查找付费的开销。

贪心最长匹配并不能让 token 数量最少。取 `vocab = {"aaaa": 1, "aaa": 2, "a": 3}`，
`text = "aaaaaa"`（六个连续的 `"a"`），贪心规则会先匹配 `"aaaa"`——它是在位置 `0` 处匹配的最长
键——剩下两个字符只能一次一个地用 `"a"` 匹配：`tokenize("aaaaaa", vocab, unk_id)` 得到
`[1, 3, 3]`，三个 token，`tokenize_merged` 结果相同，因为其中没有一个是 `unk_id`。换成连续匹配
两次 `"aaa"`，同样覆盖这六个字符，却只用两个 token，`[2, 2]`——比贪心的三个更少。（同样形状的反例
也会让贪心找零失败：面额为 `1`、`3`、`4` 时，凑 `6` 的贪心解是 `4 + 1 + 1`，而不是最优的
`3 + 3`。）

### Part 3

每个 trie 节点只需要保存它的子节点（用到达它们的那一个字符作为键），以及一个可选的 id，标记
`vocab` 中某个键恰好在这个节点结束。构建 trie 时把每个键走一遍，逐字符前进，只在子节点不存在时才
新建：耗时 $O(V)$，其中 $V$ 是 `vocab` 里所有键的总长度。在某个位置做匹配时，从根节点出发，只要下
一个字符有对应的子节点就继续往下走，并单独记录沿途经过的、本身就是某个键结尾的最后一个节点的
id——也就是这一路走下来找到的最长匹配，每经过这样一个节点就更新一次，这和 `tokenize_merged` 在尝试
越来越短的候选长度时不断更新自己的最长匹配是同一回事。

```python
class _TrieNode:
    __slots__ = ("children", "token_id")

    def __init__(self):
        self.children: dict[str, "_TrieNode"] = {}
        self.token_id: int | None = None    # None until some key of vocab ends exactly here


class TrieTokenizer:
    def __init__(self, vocab: dict[str, int], unk_id: int):
        self._unk_id = unk_id
        self._root = _TrieNode()
        for token, token_id in vocab.items():
            node = self._root
            for ch in token:
                node = node.children.setdefault(ch, _TrieNode())
            node.token_id = token_id

    def tokenize(self, text: str) -> list[int]:
        ids = []
        i, n = 0, len(text)
        while i < n:
            node, j, best = self._root, i, None    # best: (end index, id) of the last terminal node
            while j < n and text[j] in node.children:
                node = node.children[text[j]]
                j += 1
                if node.token_id is not None:
                    best = (j, node.token_id)
            if best is None:
                if not (ids and ids[-1] == self._unk_id):
                    ids.append(self._unk_id)
                i += 1
            else:
                end, token_id = best
                ids.append(token_id)
                i = end
        return ids
```

`tokenize` 从 $n$ 个起始位置中的每一个出发，至多沿 trie 走 $L$ 条边就必须停下——要么是 `text` 走
完了，要么没有子节点可以继续。每一步只是一次以单个字符为键的字典查找，是 $O(1)$ 操作，不像
`tokenize_merged` 里每个候选长度都要重新切出、哈希一段长度为 $\ell$ 的子串，代价是 $O(\ell)$：这才
把 `tokenize_merged` 的 $O(n \cdot L^2)$ 变成了真正的 $O(n \cdot L)$，而不仅仅是常数因子更小。trie
在此之外真正带来的好处，是把 `__init__` 里 $O(V)$ 的构建开销分摊到用同一个 `vocab` 分词的所有文本
上，这一点由下面的验证代码直接测量。

### 追问

- 只要词条长度超过一两个字符，有界循环的 $O(n \cdot L^2)$ 就已经比 trie 的 $O(n \cdot L)$ 慢；在此
  之外，trie 一次性的 $O(V)$ 构建开销，还要等 `vocab` 被多次调用复用之后才能赚回来。只有词条本身很
  短、`vocab` 又只会被调用一次的场合，有界循环不做任何预处理也差不多快。
- `detokenize(tokens, vocab)` 需要把 `vocab` 按 id 反查一遍，一次性建成
  `{v: k for k, v in vocab.items()}`；一个等于 `unk_id` 的 id 没有字符串可以反查，因为它替代的
  字符已经丢失了。要做到无损的往返，需要在 id 之外，为每个 `unk_id` 额外保留它对应的字符区间
  `(start, end)`。
- 如果字节是逐个到达而不是整段 `text` 一次给出，分词器就不能一找到匹配就立刻提交：在
  `vocab = {"ab": 1, "abc": 2}` 上，目前只看到 `"ab"` 时必须先等着，而不是提交 `1`，因为下一个字节
  仍有可能把它延伸成 `"abc"`。Part 3 的 trie 遍历可以直接搬过来用——让当前节点在两次调用之间保持存
  活，而不是每次都从根节点重新开始，只有当来了一个字节、当前节点没有对应的子节点，或者输入结束时，
  才提交这次匹配。
- 一次流式返回的计费要把输入和输出 token 分开定价，`price_in * n_input + price_out * n_output`；
  `n_output` 是实际流回的 token 数，需要随着每个分片到达时逐个累加（服务方的流式接口通常是增量给出
  用量，或者只在最后一个事件里给出）；如果每来一个分片就把目前为止的完整回复重新分词一遍来算
  `n_output`，耗时相对分片数是平方级的，而不是线性的。
- 有一类找 bug 的变体会给出成对的 `tokenize`/`detokenize` 函数，其中 `tokenize` 每次只读入一个字符
  去扩展当前候选串，一旦它第一次匹配上 `vocab` 中的某个键就立刻提交——追加它的 id 并重置——而不是
  尽量扩展到最长匹配。在 `vocab = {"a": 1, "ab": 2}`、`text = "ab"` 上，它会在第一个字符处就提交
  `"a"`，根本没有机会尝试扩展成 `"ab"`；而且对于一直到文本结束都没能匹配上的候选串，代码也没有兜底
  处理，于是在剩下的 `"b"` 上失败。修复方法就是 Part 1 的规则：只要还可能匹配更长的键就继续扩展，
  直到不能再扩展了才提交。

<details>
<summary>验证代码（可运行）</summary>

```python
import random
import time


def reference_tokenize(text, vocab, unk_id):
    """Independent restatement of Part 1's rule: at each position, try every possible
    match length from longest to shortest and take the first (hence longest) key of
    vocab that matches."""
    ids = []
    i, n = 0, len(text)
    while i < n:
        match = None
        for length in range(n - i, 0, -1):
            piece = text[i:i + length]
            if piece in vocab:
                match = (length, vocab[piece])
                break
        if match is None:
            ids.append(unk_id)
            i += 1
        else:
            length, token_id = match
            ids.append(token_id)
            i += length
    return ids


def merge_unk_runs(ids, unk_id):
    merged = []
    for token_id in ids:
        if token_id == unk_id and merged and merged[-1] == unk_id:
            continue
        merged.append(token_id)
    return merged


# --- the examples of the statement ---
vocab_ex = {"a": 1, "ab": 2, "abc": 3, "b": 4, "bcd": 5}
assert tokenize("abcxbcdy", vocab_ex, 0) == [3, 0, 5, 0]
assert tokenize("abczzbcd", vocab_ex, 0) == [3, 0, 0, 5]
assert tokenize_merged("abczzbcd", vocab_ex, 0) == [3, 0, 5]
tk_ex = TrieTokenizer(vocab_ex, 0)
assert tk_ex.tokenize("abcxbcdy") == [3, 0, 5, 0]
assert tk_ex.tokenize("abczzbcd") == [3, 0, 5]

# --- the token-count counterexample ---
vocab_cx = {"aaaa": 1, "aaa": 2, "a": 3}
assert tokenize("aaaaaa", vocab_cx, -1) == [1, 3, 3]
assert tokenize_merged("aaaaaa", vocab_cx, -1) == [1, 3, 3]

# --- edge cases the examples do not reach: empty text, and a vocab with no usable entries ---
assert tokenize("", vocab_ex, 0) == []
assert tokenize_merged("", vocab_ex, 0) == []
assert TrieTokenizer(vocab_ex, 0).tokenize("") == []
assert tokenize("xyz", {}, -1) == [-1, -1, -1]
assert tokenize_merged("xyz", {}, -1) == [-1]
assert TrieTokenizer({}, -1).tokenize("xyz") == [-1]


def random_vocab(rng, n_words, alphabet, max_len):
    words = set()
    while len(words) < n_words:
        words.add("".join(rng.choice(alphabet) for _ in range(rng.randint(1, max_len))))
    long_words = sorted(w for w in words if len(w) >= 2)     # NOTE: sorted -- order must not depend on hashing
    if long_words:
        base = rng.choice(long_words)                        # force a prefix-of-each-other case
        words.add(base[:rng.randint(1, len(base) - 1)])
    return {word: idx for idx, word in enumerate(sorted(words))}    # NOTE: sorted -- deterministic ids


def random_text(rng, vocab, alphabet, target_len):
    words = list(vocab)                # dict order is insertion order (from sorted words), not hash order
    out, total = [], 0
    while total < target_len:
        piece = rng.choice(words) if words and rng.random() < 0.6 else rng.choice(alphabet)
        out.append(piece)
        total += len(piece)
    return "".join(out)


saw_multi_unk_merge = False
saw_repeated_real_id = False   # a repeated id that is NOT unk_id -- must NOT be merged
for seed in range(300):
    rng = random.Random(seed)
    alphabet = "abc" if seed % 2 else "abcd"    # small alphabets make UNK runs and prefix collisions frequent
    vocab = random_vocab(rng, rng.randint(1, 8), alphabet, 4)
    unk_id = -1
    for _ in range(3):
        text = random_text(rng, vocab, alphabet, rng.randint(0, 40))
        expected_raw = reference_tokenize(text, vocab, unk_id)
        expected_merged = merge_unk_runs(expected_raw, unk_id)
        assert tokenize(text, vocab, unk_id) == expected_raw, (seed, text, vocab)
        assert tokenize_merged(text, vocab, unk_id) == expected_merged, (seed, text, vocab)
        assert TrieTokenizer(vocab, unk_id).tokenize(text) == expected_merged, (seed, text, vocab)
        if len(expected_raw) != len(expected_merged):
            saw_multi_unk_merge = True
        if any(a == b != unk_id for a, b in zip(expected_merged, expected_merged[1:])):
            saw_repeated_real_id = True
assert saw_multi_unk_merge and saw_repeated_real_id   # both rules were actually exercised, not just stated

# --- Part 3: one TrieTokenizer reused across many texts beats rebuilding it for every text ---
rng = random.Random(12345)
big_vocab = random_vocab(rng, 5000, "abcdefgh", 7)
texts = [random_text(rng, big_vocab, "abcdefgh", 300) for _ in range(80)]

start = time.perf_counter()
reused = TrieTokenizer(big_vocab, -1)
for text in texts:
    reused.tokenize(text)
reused_seconds = time.perf_counter() - start

start = time.perf_counter()
for text in texts:
    TrieTokenizer(big_vocab, -1).tokenize(text)      # rebuilds the trie before every single text
rebuilt_seconds = time.perf_counter() - start

assert rebuilt_seconds > 8 * reused_seconds, (rebuilt_seconds, reused_seconds)   # measured: 40x or more

print("all checks passed")
```

</details>

</details>
