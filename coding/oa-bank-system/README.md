# Online Assessment: Banking System

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Online assessment · 4 levels | ★★★☆☆ | Medium | SWE · Fellows | oop-design, ledger, state-machine, timestamps, merge | 4 levels / 90 min | Online assessment |
<!-- meta:end -->

## Problem

Implement the four levels below in order: a level's tests must pass before the next level's tests
run, and every level extends the same class rather than replacing it. Every method's first argument
is `timestamp`, a non-negative integer; across the whole sequence of calls made to one instance,
timestamps never decrease. An `account_id` is a non-empty string, and every `amount` is a positive
integer. An `account_id` identifies at most one *live* account at a time: `create_account` can make it
live, `merge_accounts` (Level 4) can make it stop being live, and no other call changes this.

### Level 1 — Accounts, deposits and withdrawals

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

Example:

```text
create_account(0, "A")     # -> True
create_account(1, "A")     # -> False    -- "A" is already live
deposit(2, "A", 100)       # -> 100
withdraw(3, "A", 40)       # -> 60
withdraw(4, "A", 1000)     # -> None     -- only 60 is available
deposit(5, "B", 10)        # -> None     -- "B" was never created
withdraw(6, "B", 1)        # -> None
```

### Level 2 — Ranking by outgoing spend

Every live account tracks an *outgoing total*: the sum of every `amount` that a successful `withdraw`
call has taken from it (a call that returns `None` leaves it unchanged). A freshly created account
starts at an outgoing total of 0.

```py
class Bank:
    def top_spenders(self, timestamp: int, n: int) -> list[str]:
        """Returns the n live accounts with the largest outgoing total, ranked highest first;
        accounts tied on outgoing total are ordered by account_id ascending. Each entry is the
        string f"{account_id}({outgoing_total})", e.g. "A(120)". Returns every live account, in
        the same order, if fewer than n are live. n is a positive integer."""
```

Example:

```text
create_account(0, "A"); create_account(0, "B"); create_account(0, "C")
deposit(1, "A", 100); deposit(2, "B", 100); deposit(3, "C", 100)
withdraw(4, "A", 30)              # -> 70     A's outgoing total is now 30
withdraw(5, "B", 30)              # -> 70     B's outgoing total is now 30, tied with A
top_spenders(6, 2)                # -> ["A(30)", "B(30)"]          tied on 30: "A" before "B"
top_spenders(6, 5)                # -> ["A(30)", "B(30)", "C(0)"]  only 3 accounts exist
withdraw(7, "A", 1000)            # -> None   fails: A's outgoing total stays 30
top_spenders(8, 1)                # -> ["A(30)"]
```

### Level 3 — Two-step transfers

From this level on, wherever this problem says an operation is allowed only if `amount` is at most an
account's balance, it means the account's *available balance*: its balance minus the sum of `amount`
over every transfer, started by `transfer` below, that is still pending with that account as its
source. Deposits, and the balance itself, are unaffected by this; only what `withdraw` and `transfer`
may take is. A transfer created at timestamp $t_0$ is *pending* for a call timestamped $t$ while
$t_0 \le t \le t_0 + 1{,}000$, both ends inclusive, and *expired* for every later $t$; an expired
transfer's hold stops counting against its source's available balance, the same as a completed one's,
but nothing about the transfer itself changes retroactively — expiry is not a completion.

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

Example:

```text
create_account(0, "A"); create_account(0, "B"); create_account(0, "C")
deposit(1, "A", 500)                          # -> 500
transfer(10, "A", "B", 200)                   # -> "transfer1"   holds 200; window closes at 10+1000=1010
transfer(15, "A", "C", 250)                   # -> "transfer2"   holds 250; window closes at 15+1000=1015
withdraw(20, "A", 100)                        # -> None          available = 500-200-250 = 50 < 100
accept_transfer(1010, "B", "transfer1")       # -> True          last acceptable instant: 1010 <= 1010
                                               #    A debited 200 -> balance 300; B credited 200 -> balance 200
accept_transfer(1010, "B", "transfer1")       # -> False         already completed
accept_transfer(1011, "A", "transfer2")       # -> False         A is not transfer2's target ("C" is)
accept_transfer(1016, "C", "transfer2")       # -> False         1016 > 1015: one past the window, expired
withdraw(1300, "A", 300)                      # -> 0             transfer2's hold no longer counts: available = 300
```

### Level 4 — Merging accounts

`merge_accounts(timestamp, a, b)` merges `b` into `a`. Returns `False`, without effect, if `a`
equals `b`, or if `a` or `b` is not live. Otherwise it has all of the following effects, and returns
`True`:

- `a`'s balance increases by `b`'s balance.
- `a`'s outgoing total increases by `b`'s outgoing total.
- Every transfer still pending with `b` as its `source_id` has `a` as its `source_id` from now on (its
  hold counts against `a`, not `b`); every transfer still pending with `b` as its `target_id` has `a`
  as its `target_id` from now on (`a`, not `b`, may complete it). A transfer whose `source_id` and
  `target_id` are both `a` after this stays pending and may be completed like any other; completing it
  debits and credits the same account, so it only releases the hold.
- `b` stops being live.

`account_id` may identify a sequence of accounts over time, one *generation* at a time: `create_account`
starts a new generation of `account_id` whenever `account_id` is not currently live, including right
after `merge_accounts` has just made its previous generation stop being live. A generation's balance
starts at 0 and is entirely its own: `merge_accounts` is the only way one generation's balance can
affect another's, and it affects only whichever generation of `a` is live at the moment of the merge,
never a later generation that reuses the id `b`.

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

Example:

```text
create_account(0, "A"); create_account(0, "B")
deposit(1, "A", 100)                           # -> 100
deposit(2, "B", 50)                            # -> 50
transfer(3, "B", "A", 20)                      # -> "transfer1"   B holds 20, pending, target "A"
merge_accounts(5, "A", "B")                    # -> True          A: balance 100+50=150; transfer1's source
                                                #    becomes "A" (target was already "A"); "B" stops being live
get_balance(6, "B", 4)                         # -> None          "B" is not live, at any time_at
get_balance(6, "A", 4)                         # -> 100           A's own balance just before the merge
get_balance(6, "A", 5)                         # -> 150           the merge itself, at its own timestamp
accept_transfer(10, "A", "transfer1")          # -> True          source and target are both "A": debits then
                                                #    credits the same account, net 0; balance stays 150
get_balance(11, "A", 10)                       # -> 150
create_account(12, "B")                        # -> True          a fresh generation of "B"
get_balance(13, "B", 4)                        # -> None          4 predates this generation's own creation (12);
                                                #    the merged-away generation's balance at 4 (which was 50)
                                                #    does not carry over
get_balance(13, "B", 12)                       # -> 0
deposit(14, "B", 9)                            # -> 9
get_balance(15, "B", 14)                       # -> 9
merge_accounts(16, "A", "A")                   # -> False         a and b are the same account
merge_accounts(17, "A", "nope")                # -> False         "nope" is not live
```

## Reference solution

<details>
<summary>Show the reference solution</summary>

Worth confirming up front, because reasonable designs differ on them: whether a hold changes the
balance figure itself or only what a later withdrawal or transfer may take (here, only the latter —
a hold is bookkeeping alongside the balance, not a change to it); and whether an account's outgoing
total should count a transfer at the moment it starts or only once it is accepted (here, at the moment
it starts, win or lose, since that is the only point every transfer is guaranteed to reach).

### Level 1

An account is a small object carrying its balance and, from the timestamp it was created, the list of
`(timestamp, balance)` pairs `get_balance` will need from Level 4 on; recording this from Level 1 costs
nothing and needs no change later. `_apply` is the one place a balance actually changes, so every later
level's writes go through it too. `_available`, used by `withdraw` here and by `transfer` from Level 3
on, already has its final form: until `transfer` exists, no account ever gets an entry in `_holds`, so
it simply returns the balance.

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

Each method is $O(1)$, aside from `_available`'s scan of `account_id`'s own transfers, which only ever
holds entries from Level 3 on.

### Level 2

`top_spenders` is one sort over the live accounts, by a key that negates the outgoing total (so the
largest sorts first) and keeps `account_id` itself as the tie-break, ascending, since it is not negated.

```python
def top_spenders(self, timestamp: int, n: int) -> list[str]:
    ranked = sorted(self._live.items(), key=lambda item: (-item[1].outgoing_total, item[0]))
    return [f"{account_id}({account.outgoing_total})" for account_id, account in ranked[:n]]


Bank.top_spenders = top_spenders
```

$O(k \log k)$ in the number of live accounts $k$; slicing to $n$ after a full sort costs nothing extra
worth a separate data structure at this scale.

### Level 3

A `_Transfer` records its source, target, amount, creation timestamp and whether it has been accepted;
nothing else needs to be stored, because "expired" is never written down anywhere — `_available` and
`accept_transfer` both recompute it from `created_at` and their own call's `timestamp` every time they
need it. `transfer` validates against `_available` (already correct, from Level 1) and files the new
transfer under both its source (for `_available`) and its target (for `merge_accounts`, in Level 4).

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

`transfer` and `accept_transfer` are both $O(1)$ beyond the `_available` scan; `_available` is $O(h)$
in the number of transfers ever started from that account, since a completed or expired one is left in
`_holds` rather than removed (cheap here, since an OA-scale call sequence never makes $h$ large enough
for the scan to matter; a long-lived account would instead prune `_holds` opportunistically).

### Level 4

`merge_accounts` moves three things from `b` to `a`: the balance (one `_apply`, dated to the merge
itself), the outgoing total (a plain add), and every pending transfer that names `b`, in both
directions — `_holds[b]` (where `b` is the source) and `_incoming[b]` (where `b` is the target) are
each folded into `a`'s own list, and the `_Transfer` objects themselves are repointed so that a later
`accept_transfer` or a further merge sees `a`, not the now-dead `b`. Deleting `b` from `_live` is what
makes it stop being live; the next `create_account` call for `b`, whenever it comes, builds a brand
new `_Account`, so its `events` list starts fresh at its own creation timestamp — it shares nothing
with the generation that was merged away, which is exactly what makes `get_balance` correct for a
reused id. `get_balance` looks up the live account, if any, and binary-searches its `events` for the
last entry at or before `time_at`.

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

`merge_accounts` is $O(p)$ in the number of transfers it reassigns; `get_balance` is $O(\log e)$ in the
number of balance-changing events its account's current generation has recorded.

### Follow-ups

- Undoing a merge: `merge_accounts` folds `b`'s balance into `a` as a single credit and deletes `b`
  outright, so nothing records which of `a`'s transfers came from `b`, or what `a`'s balance was the
  instant before. A reversible merge would need to keep that snapshot until it is safe to discard.
- Concurrent calls: every method above reads and writes `_live`, `_holds` and `_incoming` without a
  lock, so two threads calling `transfer` on the same source at once could both read the same
  `_available` figure before either commits its hold. A per-account lock held for the duration of
  `withdraw`, `transfer` and `accept_transfer` serializes exactly the calls that touch shared state,
  without blocking calls on unrelated accounts.
- An account with a very long transfer history: `_available` scans every transfer that account has
  ever started, including ones long since accepted or expired. Dropping a transfer's id from `_holds`
  the moment it is accepted, and lazily whenever `_available` notices one has expired, keeps the scan
  bounded by the number of transfers currently pending instead of ever started.
- A configurable expiry window: nothing about the solution depends on the exact figure $1{,}000$, so
  `TRANSFER_WINDOW` could become a constructor argument, or a per-transfer one passed to `transfer`
  itself, without touching the expiry logic.

<details>
<summary>Checks (runnable)</summary>

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
