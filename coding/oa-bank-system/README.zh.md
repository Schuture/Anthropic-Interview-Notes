# 在线测评：银行系统

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 在线测评 · 4 关 | ★★★☆☆ | 中等 | SWE · Fellows | oop-design, ledger, state-machine, timestamps, merge | 4 关 / 90 分钟 | 在线测评（OA） |
<!-- meta:end -->

## 题目

按顺序实现下面四关：只有当前一关的测试全部通过，下一关的测试才会开始运行，且每一关都在同一个类上继续
扩展，而不是另起炉灶。每个方法的第一个参数都是 `timestamp`，一个非负整数；对同一个实例发起的整个调用
序列而言，时间戳不会变小。`account_id` 是一个非空字符串，每个 `amount` 都是正整数。一个 `account_id`
在任意时刻至多标识一个*存活*（live）账户：`create_account` 可以让它变为存活，`merge_accounts`
（Level 4）可以让它不再存活，除此之外没有别的调用会改变这一点。

### Level 1 —— 账户、存款与取款

```py
class Bank:
    def create_account(self, timestamp: int, account_id: str) -> bool:
        """Opens a new account with balance 0. Returns True, or False, without effect, if
        account_id is already live."""

    def deposit(self, timestamp: int, account_id: str, amount: int) -> int | None:
        """Adds amount to account_id's balance. Returns the new balance, or None, without
        effect, if account_id is not live."""

    def withdraw(self, timestamp: int, account_id: str, amount: int) -> int | None:
        """Subtracts amount from account_id's balance. Returns the new balance. Returns None,
        without effect, if account_id is not live, or if amount is greater than its balance."""
```

示例：

```text
create_account(0, "A")     # -> True
create_account(1, "A")     # -> False    -- "A" 已经是存活账户
deposit(2, "A", 100)       # -> 100
withdraw(3, "A", 40)       # -> 60
withdraw(4, "A", 1000)     # -> None     -- 只有 60 可用
deposit(5, "B", 10)        # -> None     -- "B" 从未被创建过
withdraw(6, "B", 1)        # -> None
```

### Level 2 —— 按转出总额排名

每个存活账户都维护一个*转出总额*（outgoing total）：每次成功的 `withdraw` 调用从该账户取走的
`amount` 之和（返回 `None` 的调用不会改变它）。新创建的账户转出总额从 0 开始。

```py
class Bank:
    def top_spenders(self, timestamp: int, n: int) -> list[str]:
        """Returns the n live accounts with the largest outgoing total, ranked highest first;
        accounts tied on outgoing total are ordered by account_id ascending. Each entry is the
        string f"{account_id}({outgoing_total})", e.g. "A(120)". Returns every live account, in
        the same order, if fewer than n are live. n is a positive integer."""
```

示例：

```text
create_account(0, "A"); create_account(0, "B"); create_account(0, "C")
deposit(1, "A", 100); deposit(2, "B", 100); deposit(3, "C", 100)
withdraw(4, "A", 30)              # -> 70     A 的转出总额现在是 30
withdraw(5, "B", 30)              # -> 70     B 的转出总额现在是 30，与 A 打平
top_spenders(6, 2)                # -> ["A(30)", "B(30)"]          30 分打平："A" 排在 "B" 前面
top_spenders(6, 5)                # -> ["A(30)", "B(30)", "C(0)"]  只有 3 个账户存在
withdraw(7, "A", 1000)            # -> None   失败：A 的转出总额保持 30
top_spenders(8, 1)                # -> ["A(30)"]
```

### Level 3 —— 两步转账

从这一关开始，本题中凡是说某个操作仅当 `amount` 不超过账户余额时才被允许的地方，指的都是该账户的
*可用余额*（available balance）：账户余额减去下面 `transfer` 发起的、仍以该账户为来源方且处于待处理
状态的每一笔转账的 `amount` 之和。存款以及余额本身不受此影响；受影响的只是 `withdraw` 和 `transfer`
能够取用的额度。一笔在时间戳 $t_0$ 创建的转账，对于时间戳为 $t$ 的调用而言，在
$t_0 \le t \le t_0 + 1{,}000$（两端都取得到）期间处于*待处理*（pending）状态，在此之后的任何 $t$
都视为*已过期*（expired）；已过期转账的保留额度会像已完成的转账一样，不再计入其来源方的可用余额，
但转账本身不会因此被追溯性地改变——过期本身不是一种完成。

```py
class Bank:
    def transfer(self, timestamp: int, source_id: str, target_id: str, amount: int) -> str | None:
        """Starts a transfer of amount out of source_id, addressed to target_id, and returns its
        transfer_id: the string "transfer1", "transfer2", ... in the order transfers are created
        (one counter shared by every account, advanced only when transfer succeeds). The
        transfer is pending until accept_transfer completes or expires it; neither account's
        balance changes yet. Returns None, without effect, if any of: source_id equals
        target_id; source_id or target_id is not live; amount is greater than source_id's
        available balance."""

    def accept_transfer(self, timestamp: int, account_id: str, transfer_id: str) -> bool:
        """Completes a pending transfer: source_id's balance decreases by amount and target_id's
        balance increases by amount, in one step. Only target_id may complete a transfer, and
        only while it is pending. Returns True on success. Returns False, without effect, if
        any of: transfer_id was never returned by transfer; that transfer has already been
        completed; account_id is not its target_id; the transfer is expired."""
```

示例：

```text
create_account(0, "A"); create_account(0, "B"); create_account(0, "C")
deposit(1, "A", 500)                          # -> 500
transfer(10, "A", "B", 200)                   # -> "transfer1"   保留 200；窗口在 10+1000=1010 关闭
transfer(15, "A", "C", 250)                   # -> "transfer2"   保留 250；窗口在 15+1000=1015 关闭
withdraw(20, "A", 100)                        # -> None          可用余额 = 500-200-250 = 50 < 100
accept_transfer(1010, "B", "transfer1")       # -> True          最后可接受的时刻：1010 <= 1010
                                               #    A 被扣 200 -> 余额 300；B 被加 200 -> 余额 200
accept_transfer(1010, "B", "transfer1")       # -> False         已经完成过了
accept_transfer(1011, "A", "transfer2")       # -> False         A 不是 transfer2 的接收方（"C" 才是）
accept_transfer(1016, "C", "transfer2")       # -> False         1016 > 1015：刚好超出窗口一点，已过期
withdraw(1300, "A", 300)                      # -> 0             transfer2 的保留额度已不再计入：可用余额 = 300
```

### Level 4 —— 合并账户

`merge_accounts(timestamp, a, b)` 把 `b` 合并进 `a`。如果 `a` 等于 `b`，或者 `a`、`b` 中有一个不是
存活账户，返回 `False`，且不产生任何效果。否则会产生以下全部效果，并返回 `True`：

- `a` 的余额增加 `b` 的余额。
- `a` 的转出总额增加 `b` 的转出总额。
- 每一笔仍以 `b` 为 `source_id` 的待处理转账，从此以后 `source_id` 变为 `a`（其保留额度计入 `a`，
  而不是 `b`）；每一笔仍以 `b` 为 `target_id` 的待处理转账，从此以后 `target_id` 变为 `a`（可以由
  `a`、而不是 `b` 去完成它）。如果这样一来某笔转账的 `source_id` 和 `target_id` 都变成了 `a`，它
  仍然保持待处理状态，也可以像其他转账一样被完成；完成它时借记和贷记的是同一个账户，所以只是释放
  了保留额度而已。
- `b` 不再是存活账户。

`account_id` 可以在不同时间先后标识一串不同的账户，每次算作一个*代*（generation）：只要
`account_id` 当前不是存活状态，`create_account` 就会为它开启新的一代，包括 `merge_accounts` 刚刚
让它的上一代不再存活之后。一个代的余额从 0 开始，完全独立：`merge_accounts` 是唯一能让一个代的余额
影响另一个代的方式，而且它只影响合并那一刻正存活的那一代 `a`，绝不会影响之后重新使用 `b` 这个 id
的某一代。

```py
class Bank:
    def get_balance(self, timestamp: int, account_id: str, time_at: int) -> int | None:
        """Returns the balance that account_id's current generation had at time_at: its balance
        immediately after the last deposit, withdrawal, transfer completion or merge credit on
        that generation timestamped at most time_at (a completed transfer counts at the
        timestamp accept_transfer completed it, not the one transfer created it; a merge counts
        at the timestamp merge_accounts was called). Returns None if account_id is not live, or
        if time_at is before the current generation's own create_account call. time_at is a
        non-negative integer with no ordering constraint of its own -- unlike every method's
        leading timestamp, it may repeat or go backward from one call to the next."""
```

示例：

```text
create_account(0, "A"); create_account(0, "B")
deposit(1, "A", 100)                           # -> 100
deposit(2, "B", 50)                            # -> 50
transfer(3, "B", "A", 20)                      # -> "transfer1"   B 保留 20，待处理，接收方是 "A"
merge_accounts(5, "A", "B")                    # -> True          A：余额 100+50=150；transfer1 的 source
                                                #    变为 "A"（target 原本就是 "A"）；"B" 不再存活
get_balance(6, "B", 4)                         # -> None          "B" 不是存活账户，任何 time_at 都一样
get_balance(6, "A", 4)                         # -> 100           合并前一刻 A 自己的余额
get_balance(6, "A", 5)                         # -> 150           合并本身发生的那个时间戳
accept_transfer(10, "A", "transfer1")          # -> True          source 和 target 都是 "A"：先借记再贷记
                                                #    同一个账户，净变化为 0；余额仍是 150
get_balance(11, "A", 10)                       # -> 150
create_account(12, "B")                        # -> True          "B" 的全新一代
get_balance(13, "B", 4)                        # -> None          4 早于这一代自己的创建时间（12）；
                                                #    被合并注销的那一代在 4 时刻的余额（当时是 50）
                                                #    并不会延续过来
get_balance(13, "B", 12)                       # -> 0
deposit(14, "B", 9)                            # -> 9
get_balance(15, "B", 14)                       # -> 9
merge_accounts(16, "A", "A")                   # -> False         a 和 b 是同一个账户
merge_accounts(17, "A", "nope")                # -> False         "nope" 不是存活账户
```

## 参考解答

<details>
<summary>展开参考解答</summary>

值得事先确认的两点，因为合理的设计在这两点上可能不同：保留额度究竟会改变余额这个数字本身，还是只
影响之后的取款或转账能取用多少（这里选择后者——保留额度是余额之外单独记的账，不会改变余额本身）；
以及账户的转出总额应该在转账发起的那一刻就计入，还是要等它被接收之后才计入（这里选择在发起的那一刻
就计入，不论成败，因为那是每一笔转账都保证会经过的唯一节点）。

### Level 1

账户是一个很小的对象，携带着自己的余额，以及从创建时刻起的 `(timestamp, balance)` 序列——这是
Level 4 开始 `get_balance` 会用到的东西；从 Level 1 起就记录它不需要额外成本，之后也不用再改。
`_apply` 是余额真正发生变化的唯一地方，所以之后每一关的写入操作也都经过它。`_available` 在这里被
`withdraw` 使用，从 Level 3 起也被 `transfer` 使用，它从一开始就已经是最终形态：在 `transfer` 出现
之前，任何账户都不会在 `_holds` 里有记录，所以它就只是简单地返回余额。

```python
class _Account:
    def __init__(self, created_at: int):
        self.balance = 0
        self.outgoing_total = 0
        self.events: list[tuple[int, int]] = [(created_at, 0)]   # (timestamp, balance) after each change


class Bank:
    TRANSFER_WINDOW = 1_000

    def __init__(self):
        self._live: dict[str, _Account] = {}
        self._transfers: dict[str, "_Transfer"] = {}
        self._holds: dict[str, list[str]] = {}      # account_id -> ids of transfers it is the source of
        self._incoming: dict[str, list[str]] = {}    # account_id -> ids of transfers it is the target of
        self._transfer_seq = 0

    def create_account(self, timestamp: int, account_id: str) -> bool:
        if account_id in self._live:
            return False
        self._live[account_id] = _Account(timestamp)   # NOTE: a fresh object -- see Level 4
        return True

    def deposit(self, timestamp: int, account_id: str, amount: int) -> int | None:
        account = self._live.get(account_id)
        return None if account is None else self._apply(account, timestamp, amount)

    def withdraw(self, timestamp: int, account_id: str, amount: int) -> int | None:
        account = self._live.get(account_id)
        if account is None or amount > self._available(account_id, timestamp):
            return None
        account.outgoing_total += amount
        return self._apply(account, timestamp, -amount)

    def _apply(self, account: _Account, timestamp: int, delta: int) -> int:
        account.balance += delta
        account.events.append((timestamp, account.balance))
        return account.balance

    def _available(self, account_id: str, timestamp: int) -> int:
        account = self._live[account_id]
        held = sum(self._transfers[tid].amount for tid in self._holds.get(account_id, [])
                   if not self._transfers[tid].accepted
                   and timestamp <= self._transfers[tid].created_at + self.TRANSFER_WINDOW)
        return account.balance - held   # NOTE: a hold never touches balance -- only what may be taken from it
```

每个方法都是 $O(1)$，除了 `_available` 要扫描 `account_id` 自己名下的转账——这份记录也只有从
Level 3 起才会有内容。

### Level 2

`top_spenders` 就是对所有存活账户做一次排序：排序键对转出总额取负（这样最大的排在最前面），并保留
`account_id` 本身作为并列时的次级排序键——因为它没有被取负，所以是升序。

```python
def top_spenders(self, timestamp: int, n: int) -> list[str]:
    ranked = sorted(self._live.items(), key=lambda item: (-item[1].outgoing_total, item[0]))
    return [f"{account_id}({account.outgoing_total})" for account_id, account in ranked[:n]]


Bank.top_spenders = top_spenders
```

存活账户数为 $k$ 时开销是 $O(k \log k)$；排序完之后再截取前 $n$ 个几乎不增加额外开销，在这种规模下
不值得为此单独设计一种数据结构。

### Level 3

`_Transfer` 记录了它的来源方、接收方、金额、创建时的时间戳，以及是否已被接收；不需要再存别的东西，
因为“已过期”这件事从来不会被写下来——`_available` 和 `accept_transfer` 每次需要用到它时，都是根据
`created_at` 和自己这次调用的 `timestamp` 现算的。`transfer` 用 `_available`（从 Level 1 起就已经是
正确的）来做校验，并把新转账同时登记在它的来源方名下（供 `_available` 使用）和接收方名下（供
Level 4 的 `merge_accounts` 使用）。

```python
class _Transfer:
    def __init__(self, transfer_id: str, source_id: str, target_id: str, amount: int, created_at: int):
        self.id = transfer_id
        self.source_id = source_id
        self.target_id = target_id
        self.amount = amount
        self.created_at = created_at
        self.accepted = False


def transfer(self, timestamp: int, source_id: str, target_id: str, amount: int) -> str | None:
    if (source_id == target_id or source_id not in self._live or target_id not in self._live
            or amount > self._available(source_id, timestamp)):
        return None
    self._transfer_seq += 1
    transfer_id = f"transfer{self._transfer_seq}"
    self._transfers[transfer_id] = _Transfer(transfer_id, source_id, target_id, amount, timestamp)
    self._holds.setdefault(source_id, []).append(transfer_id)
    self._incoming.setdefault(target_id, []).append(transfer_id)
    return transfer_id


def accept_transfer(self, timestamp: int, account_id: str, transfer_id: str) -> bool:
    t = self._transfers.get(transfer_id)
    if (t is None or t.accepted or t.target_id != account_id
            or timestamp > t.created_at + self.TRANSFER_WINDOW):
        return False                     # NOTE: expiry is a plain comparison, re-checked on every call --
    t.accepted = True                    # nothing is swept or scheduled ahead of time
    self._apply(self._live[t.source_id], timestamp, -t.amount)
    self._apply(self._live[t.target_id], timestamp, t.amount)
    return True


Bank.transfer = transfer
Bank.accept_transfer = accept_transfer
```

除了 `_available` 的扫描之外，`transfer` 和 `accept_transfer` 都是 $O(1)$；`_available` 的开销是
$O(h)$，$h$ 是该账户名下曾经发起过的转账数量，因为已完成或已过期的转账仍然留在 `_holds` 里，并没有
被移除（在这种规模下这样做很划算，因为在 OA 规模的调用序列里 $h$ 永远不会大到让这次扫描成为瓶颈；
一个长期存在的账户则应该顺手清理 `_holds`）。

### Level 4

`merge_accounts` 把三样东西从 `b` 搬到 `a`：余额（一次 `_apply`，记在合并本身发生的时间戳上）、
转出总额（直接相加），以及每一笔涉及 `b` 的待处理转账，且是两个方向都要处理——`_holds[b]`（`b`
作为来源方的那些）和 `_incoming[b]`（`b` 作为接收方的那些）分别被并入 `a` 自己的列表，`_Transfer`
对象本身也被改指向，这样之后的 `accept_transfer` 或者再一次合并看到的就是 `a`，而不是已经不存在的
`b`。把 `b` 从 `_live` 里删掉，正是让它不再存活的原因；之后无论什么时候再为 `b` 调用
`create_account`，都会构造一个全新的 `_Account`，它的 `events` 列表从自己的创建时间戳
重新开始——和被合并注销的那一代没有任何共享，这正是 `get_balance` 对重新使用的 id 能给出正确结果
的原因所在。`get_balance` 查找存活账户（如果存在的话），再在它的 `events` 里二分查找最后一条时间戳
不超过 `time_at` 的记录。

```python
import bisect


def merge_accounts(self, timestamp: int, a: str, b: str) -> bool:
    if a == b or a not in self._live or b not in self._live:
        return False
    acc_a, acc_b = self._live[a], self._live[b]
    self._apply(acc_a, timestamp, acc_b.balance)
    acc_a.outgoing_total += acc_b.outgoing_total
    for tid in self._holds.pop(b, []):              # NOTE: both directions -- b as source ...
        self._transfers[tid].source_id = a
        self._holds.setdefault(a, []).append(tid)
    for tid in self._incoming.pop(b, []):            # ... and b as target
        self._transfers[tid].target_id = a
        self._incoming.setdefault(a, []).append(tid)
    del self._live[b]                                # NOTE: a later create_account(b) builds a fresh _Account --
    return True                                       # nothing here keeps that new object linked to this one


def get_balance(self, timestamp: int, account_id: str, time_at: int) -> int | None:
    account = self._live.get(account_id)
    if account is None:
        return None
    i = bisect.bisect_right(account.events, time_at, key=lambda e: e[0]) - 1
    return account.events[i][1] if i >= 0 else None


Bank.merge_accounts = merge_accounts
Bank.get_balance = get_balance
```

`merge_accounts` 的开销是 $O(p)$，$p$ 是它需要改指向的转账数量；`get_balance` 的开销是
$O(\log e)$，$e$ 是该账户当前这一代已经记录下来的余额变化事件数量。

### 追问

- 撤销一次合并：`merge_accounts` 把 `b` 的余额作为一笔贷记并入 `a`，然后直接删除 `b`，所以没有任何
  记录能说明 `a` 名下哪些转账来自 `b`，也没有记录 `a` 在合并前一刻的余额。要让合并可撤销，就需要
  一直保留这样一份快照，直到确认可以安全丢弃为止。
- 并发调用：上面的每个方法都在没有加锁的情况下读写 `_live`、`_holds` 和 `_incoming`，所以两个线程
  同时对同一个来源方调用 `transfer`，可能会在任何一方真正提交保留额度之前，都读到同一个
  `_available` 数值。给 `withdraw`、`transfer`、`accept_transfer` 的整个执行过程加上按账户区分的
  锁，恰好能把真正会触碰共享状态的那些调用串行化，同时不会阻塞与之无关的账户上的调用。
- 转账历史很长的账户：`_available` 会扫描该账户曾经发起过的每一笔转账，包括早就已完成或已过期的
  那些。一笔转账被接收的那一刻就把它的 id 从 `_holds` 里去掉，并且在 `_available` 顺带发现某笔
  转账已经过期时也把它去掉，这样扫描的开销就只取决于当前待处理的转账数量，而不是历史上发起过的
  全部数量。
- 可配置的过期窗口：这个解法没有任何部分依赖 $1{,}000$ 这个具体数值，所以 `TRANSFER_WINDOW` 完全
  可以改成构造函数的参数，或者作为参数传给 `transfer` 本身，按每笔转账单独设置，而不需要改动过期
  判断的逻辑。

<details>
<summary>验证代码（可运行）</summary>

```python
# ---- Level 1 example ----
bank = Bank()
assert bank.create_account(0, "A") is True
assert bank.create_account(1, "A") is False
assert bank.deposit(2, "A", 100) == 100
assert bank.withdraw(3, "A", 40) == 60
assert bank.withdraw(4, "A", 1000) is None
assert bank.deposit(5, "B", 10) is None
assert bank.withdraw(6, "B", 1) is None

# ---- Level 2 example ----
bank = Bank()
for aid in ("A", "B", "C"):
    bank.create_account(0, aid)
bank.deposit(1, "A", 100)
bank.deposit(2, "B", 100)
bank.deposit(3, "C", 100)
assert bank.withdraw(4, "A", 30) == 70
assert bank.withdraw(5, "B", 30) == 70
assert bank.top_spenders(6, 2) == ["A(30)", "B(30)"]
assert bank.top_spenders(6, 5) == ["A(30)", "B(30)", "C(0)"]
assert bank.withdraw(7, "A", 1000) is None
assert bank.top_spenders(8, 1) == ["A(30)"]

# ---- Level 3 example ----
bank = Bank()
for aid in ("A", "B", "C"):
    bank.create_account(0, aid)
bank.deposit(1, "A", 500)
assert bank.transfer(10, "A", "B", 200) == "transfer1"
assert bank.transfer(15, "A", "C", 250) == "transfer2"
assert bank.withdraw(20, "A", 100) is None
assert bank.accept_transfer(1010, "B", "transfer1") is True
assert bank.accept_transfer(1010, "B", "transfer1") is False
assert bank.accept_transfer(1011, "A", "transfer2") is False
assert bank.accept_transfer(1016, "C", "transfer2") is False
assert bank.withdraw(1300, "A", 300) == 0

# ---- Level 4 example ----
bank = Bank()
bank.create_account(0, "A")
bank.create_account(0, "B")
assert bank.deposit(1, "A", 100) == 100
assert bank.deposit(2, "B", 50) == 50
assert bank.transfer(3, "B", "A", 20) == "transfer1"
assert bank.merge_accounts(5, "A", "B") is True
assert bank.get_balance(6, "B", 4) is None
assert bank.get_balance(6, "A", 4) == 100
assert bank.get_balance(6, "A", 5) == 150
assert bank.accept_transfer(10, "A", "transfer1") is True
assert bank.get_balance(11, "A", 10) == 150
assert bank.create_account(12, "B") is True
assert bank.get_balance(13, "B", 4) is None
assert bank.get_balance(13, "B", 12) == 0
assert bank.deposit(14, "B", 9) == 9
assert bank.get_balance(15, "B", 14) == 9
assert bank.merge_accounts(16, "A", "A") is False
assert bank.merge_accounts(17, "A", "nope") is False
print("all four levels' worked examples replayed exactly")

# ---- edge cases the examples do not reach ----
# a transfer pending TO an outside account, and one pending FROM an outside account, both survive
# a merge and still resolve to the surviving account afterwards
bank = Bank()
for aid in ("A", "B", "C", "D"):
    bank.create_account(0, aid)
bank.deposit(1, "B", 100)
bank.deposit(1, "D", 100)
assert bank.transfer(2, "B", "C", 40) == "transfer1"   # B (about to be merged away) -> outside C
assert bank.transfer(3, "D", "B", 25) == "transfer2"   # outside D -> B (about to be merged away)
assert bank.merge_accounts(4, "A", "B") is True         # A absorbs B; A itself had no funds or transfers
assert bank.accept_transfer(5, "C", "transfer1") is True     # C still completes it; A (not B) is debited
assert bank.get_balance(6, "A", 5) == 100 - 40
assert bank.accept_transfer(7, "A", "transfer2") is True     # A (not B) now completes the incoming transfer
assert bank.get_balance(8, "A", 7) == 100 - 40 + 25

# chained merges combine balances and outgoing totals transitively
bank = Bank()
for aid in ("A", "B", "C"):
    bank.create_account(0, aid)
bank.deposit(1, "A", 10)
bank.deposit(1, "B", 20)
bank.deposit(1, "C", 30)
bank.withdraw(2, "B", 5)
bank.withdraw(2, "C", 7)
assert bank.merge_accounts(3, "A", "B") is True
assert bank.merge_accounts(4, "A", "C") is True
assert bank.get_balance(5, "A", 4) == 10 + (20 - 5) + (30 - 7)
assert bank.top_spenders(5, 1) == [f"A({5 + 7})"]

# get_balance for a time_at at or after every recorded event returns the latest balance
bank = Bank()
bank.create_account(0, "A")
bank.deposit(1, "A", 3)
assert bank.get_balance(2, "A", 10 ** 6) == 3
print("edge cases OK")
```

```python
# ---- independent reference model, straight from the statement ----
# One flat dict of live accounts and one flat dict of every transfer ever created; no bisect, no
# per-account hold index -- every query re-scans whatever it needs from scratch.
import random


class _SlowBank:
    def __init__(self):
        self.accounts: dict[str, dict] = {}   # account_id -> {balance, outgoing, events}
        self.transfers: dict[str, dict] = {}
        self.seq = 0

    def create_account(self, ts, aid):
        if aid in self.accounts:
            return False
        self.accounts[aid] = {"balance": 0, "outgoing": 0, "events": [(ts, 0)]}
        return True

    def _record(self, aid, ts, new_balance):
        self.accounts[aid]["events"].append((ts, new_balance))

    def deposit(self, ts, aid, amount):
        if aid not in self.accounts:
            return None
        acc = self.accounts[aid]
        acc["balance"] += amount
        self._record(aid, ts, acc["balance"])
        return acc["balance"]

    def _held(self, aid, ts):
        return sum(t["amount"] for t in self.transfers.values()
                   if t["source"] == aid and not t["accepted"] and ts <= t["created_at"] + 1_000)

    def withdraw(self, ts, aid, amount):
        if aid not in self.accounts or amount > self.accounts[aid]["balance"] - self._held(aid, ts):
            return None
        acc = self.accounts[aid]
        acc["balance"] -= amount
        acc["outgoing"] += amount
        self._record(aid, ts, acc["balance"])
        return acc["balance"]

    def top_spenders(self, ts, n):
        ranked = sorted(self.accounts.items(), key=lambda item: (-item[1]["outgoing"], item[0]))
        return [f"{aid}({acc['outgoing']})" for aid, acc in ranked[:n]]

    def transfer(self, ts, source, target, amount):
        if source == target or source not in self.accounts or target not in self.accounts:
            return None
        if amount > self.accounts[source]["balance"] - self._held(source, ts):
            return None
        self.seq += 1
        tid = f"transfer{self.seq}"
        self.transfers[tid] = {"source": source, "target": target, "amount": amount,
                                "created_at": ts, "accepted": False}
        return tid

    def accept_transfer(self, ts, aid, tid):
        t = self.transfers.get(tid)
        if t is None or t["accepted"] or t["target"] != aid or ts > t["created_at"] + 1_000:
            return False
        t["accepted"] = True
        self.accounts[t["source"]]["balance"] -= t["amount"]
        self._record(t["source"], ts, self.accounts[t["source"]]["balance"])
        self.accounts[t["target"]]["balance"] += t["amount"]
        self._record(t["target"], ts, self.accounts[t["target"]]["balance"])
        return True

    def merge_accounts(self, ts, a, b):
        if a == b or a not in self.accounts or b not in self.accounts:
            return False
        acc_a, acc_b = self.accounts[a], self.accounts[b]
        acc_a["balance"] += acc_b["balance"]
        self._record(a, ts, acc_a["balance"])
        acc_a["outgoing"] += acc_b["outgoing"]
        for t in self.transfers.values():
            if t["source"] == b:
                t["source"] = a
            if t["target"] == b:
                t["target"] = a
        del self.accounts[b]
        return True

    def get_balance(self, ts, aid, time_at):
        if aid not in self.accounts:
            return None
        value = None
        for evt_ts, bal in self.accounts[aid]["events"]:
            if evt_ts <= time_at:
                value = bal
            else:
                break
        return value


def _apply_op(bank, op):
    return getattr(bank, op[0])(*op[1:])


def _random_ops(rng, n, ids):
    ops, ts, transfer_ids_seen = [], 0, 0
    for _ in range(n):
        ts += rng.choice([0, 0, 1, 2, 5, 900, 1400])   # occasional big jumps to cross the expiry window
        aid, other = rng.choice(ids), rng.choice(ids)
        kind = rng.choices(
            ["create", "deposit", "withdraw", "top", "transfer", "accept", "merge", "balance"],
            weights=[3, 4, 4, 2, 4, 4, 2, 3])[0]
        if kind == "create":
            ops.append(("create_account", ts, aid))
        elif kind == "deposit":
            ops.append(("deposit", ts, aid, rng.randint(1, 50)))
        elif kind == "withdraw":
            ops.append(("withdraw", ts, aid, rng.randint(1, 60)))
        elif kind == "top":
            ops.append(("top_spenders", ts, rng.randint(1, len(ids) + 1)))
        elif kind == "transfer":
            ops.append(("transfer", ts, aid, other, rng.randint(1, 40)))
            transfer_ids_seen += 1
        elif kind == "accept":
            tid = f"transfer{rng.randint(1, max(transfer_ids_seen, 1))}"
            ops.append(("accept_transfer", ts, aid, tid))
        elif kind == "merge":
            ops.append(("merge_accounts", ts, aid, other))
        else:
            ops.append(("get_balance", ts, aid, rng.randint(0, max(ts, 1))))
    return ops


def _agree(seed, n_ops):
    rng = random.Random(seed)
    ids = ["A", "B", "C", "D"]
    fast, slow = Bank(), _SlowBank()
    for op in _random_ops(rng, n_ops, ids):
        got, want = _apply_op(fast, op), _apply_op(slow, op)
        assert got == want, (seed, op, got, want)


for seed in range(600):
    _agree(seed, n_ops=20 + seed % 40)
print("cross-validated 600 random operation sequences against an independent reference model")
print("all checks passed")
```

</details>

</details>
