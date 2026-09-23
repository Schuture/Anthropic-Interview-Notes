# Reviewing and Improving Pull Requests with an AI Coding Assistant

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Code review · AI-assisted, Python | ★★★☆☆ | Medium | SWE · Infra Eng | code-review, pull-requests, race-condition, deserialization, testing | 4 PRs / 55 min | Technical phone screen · Onsite |
<!-- meta:end -->

## Problem

You are working in a browser-based editor with an integrated terminal, open on a small Python repository.
A separate code-hosting page lists four open pull requests against this repository; each is shown below as
a unified *diff* (a text description of the lines a change adds and removes, in `diff` format) together
with the description its author wrote. An AI coding assistant is available in the terminal and may be used
for any part of the task, but every review comment and every line of code you submit is your responsibility:
being unable to explain why a comment or a change is correct is treated the same as not having made it.

The repository has one module, `jobqueue.py`:

```python
import heapq
import itertools
import json
import threading


class Job:
    def __init__(self, id, payload, priority=0, status="pending", attempts=0, max_attempts=3):
        self.id = id
        self.payload = payload
        self.priority = priority
        self.status = status                # "pending" | "running" | "done" | "failed"
        self.attempts = attempts
        self.max_attempts = max_attempts

    def to_dict(self):
        return dict(id=self.id, payload=self.payload, priority=self.priority,
                    status=self.status, attempts=self.attempts, max_attempts=self.max_attempts)


class JobQueue:
    """A priority job queue, safe to share between several worker threads in one process. Ready jobs
    are handed out by priority (higher first); jobs of equal priority are handed out in submission order."""

    def __init__(self):
        self._lock = threading.Lock()
        self._jobs = {}                     # id -> Job, every job ever submitted
        self._heap = []                     # (-priority, seq, id), pending jobs only
        self._counter = itertools.count()

    def submit(self, job_id, payload, priority=0, max_attempts=3):
        with self._lock:
            if job_id in self._jobs:
                raise ValueError(f"duplicate job id {job_id!r}")
            job = Job(job_id, payload, priority, "pending", 0, max_attempts)
            self._jobs[job_id] = job
            heapq.heappush(self._heap, (-priority, next(self._counter), job_id))
        return job

    def pop_ready(self):
        """Return the highest-priority pending job, marking it "running" so no other caller can
        receive the same job. Returns None if no job is pending."""
        with self._lock:
            while self._heap:
                _, _, job_id = heapq.heappop(self._heap)
                job = self._jobs[job_id]
                if job.status == "pending":
                    job.status = "running"
                    return job
            return None

    def complete(self, job_id):
        with self._lock:
            self._jobs[job_id].status = "done"

    def fail(self, job_id):
        """Record a failed attempt at a running job. Reschedules it as pending if attempts remain
        (it becomes poppable again immediately), else marks it permanently "failed"."""
        with self._lock:
            job = self._jobs[job_id]
            job.attempts += 1
            if job.attempts < job.max_attempts:
                job.status = "pending"
                heapq.heappush(self._heap, (-job.priority, next(self._counter), job_id))
            else:
                job.status = "failed"

    def get(self, job_id):
        return self._jobs[job_id]

    def counts(self):
        """Number of jobs currently in each status, for monitoring."""
        with self._lock:
            out = {"pending": 0, "running": 0, "done": 0, "failed": 0}
            for job in self._jobs.values():
                out[job.status] += 1
            return out

    def save(self, path):
        with self._lock:
            rows = [job.to_dict() for job in self._jobs.values()]
        with open(path, "w") as f:
            json.dump(rows, f)

    @classmethod
    def load(cls, path):
        queue = cls()
        with open(path) as f:
            rows = json.load(f)
        for row in rows:
            job = Job(**row)
            queue._jobs[job.id] = job
            if job.status == "pending":
                heapq.heappush(queue._heap, (-job.priority, next(queue._counter), job.id))
        return queue
```

The repository also has `test_jobqueue.py`, with `unittest` tests for `submit` (rejects a duplicate id),
`pop_ready` (priority order, then submission order among equal priorities), `fail` (retries until
`max_attempts` is reached, then marks the job `"failed"`), and `save`/`load` (a round trip through a file
reproduces every job's fields, and only jobs left `"pending"` are poppable again afterwards). The
repository's `README` names its test command: `python -m unittest -v`.

**PR #1 — Add `submit_many` for batch job submission** (opened by `grace-oyelaran`)

> We seed the queue with a backlog of a few hundred jobs at startup, replayed from the previous day's
> snapshot, and today that means every caller writes its own loop over `submit`. This adds `submit_many`,
> which takes the same `(job_id, payload, priority)` triples callers already build and submits them in one
> call, saving that loop and taking the lock once per batch instead of once per job. No change to `submit`
> or to any other method.

```diff
diff --git a/jobqueue.py b/jobqueue.py
--- a/jobqueue.py
+++ b/jobqueue.py
@@ class JobQueue:
             heapq.heappush(self._heap, (-priority, next(self._counter), job_id))
         return job

+    def submit_many(self, items):
+        """items: an iterable of (job_id, payload, priority) triples. Submits each in order and
+        returns the list of created Job objects."""
+        return [self.submit(job_id, payload, priority) for job_id, payload, priority in items]
+
     def pop_ready(self):
         """Return the highest-priority pending job, marking it "running" so no other caller can
diff --git a/test_jobqueue.py b/test_jobqueue.py
--- a/test_jobqueue.py
+++ b/test_jobqueue.py
@@ class TestJobQueue(unittest.TestCase):
     def test_submit_rejects_duplicate_id(self):
         q = JobQueue()
         q.submit("a", {})
         with self.assertRaises(ValueError):
             q.submit("a", {})
+
+    def test_submit_many_creates_all_jobs_in_order(self):
+        q = JobQueue()
+        jobs = q.submit_many([("a", {}, 0), ("b", {}, 5), ("c", {}, 0)])
+        self.assertEqual([j.id for j in jobs], ["a", "b", "c"])
+        self.assertEqual(q.pop_ready().id, "b")
```

**PR #2 — Reduce lock contention in `submit()`** (opened by `marcus-kaelin`)

> Under load, `submit` shows up in profiling as the most contended lock: every call takes `self._lock` for
> the whole method, even though the duplicate-id check never touches shared mutable state as long as the id
> turns out to be free. This moves the check ahead of the lock, so the common case — a fresh id — does
> less work while holding the lock. Behavior is unchanged: a duplicate id still raises `ValueError`.

```diff
diff --git a/jobqueue.py b/jobqueue.py
--- a/jobqueue.py
+++ b/jobqueue.py
@@ class JobQueue:
     def submit(self, job_id, payload, priority=0, max_attempts=3):
-        with self._lock:
-            if job_id in self._jobs:
-                raise ValueError(f"duplicate job id {job_id!r}")
+        if job_id in self._jobs:
+            raise ValueError(f"duplicate job id {job_id!r}")
+        with self._lock:
             job = Job(job_id, payload, priority, "pending", 0, max_attempts)
             self._jobs[job_id] = job
             heapq.heappush(self._heap, (-priority, next(self._counter), job_id))
         return job
```

**PR #3 — Persist queue state with pickle instead of JSON** (opened by `owen-t`)

> A couple of internal callers want to stash richer objects in a job's `payload` — namedtuples,
> dataclasses, even a small numpy array for one use case — and JSON can't represent those, so `save`
> silently can't be used for them today. Pickle handles anything picklable with no schema work on our side,
> and it is a drop-in replacement: same file, same call sites, same round trip. Swapped both `save` and
> `load` over; nothing else in the file changes.

```diff
diff --git a/jobqueue.py b/jobqueue.py
--- a/jobqueue.py
+++ b/jobqueue.py
@@
 import heapq
 import itertools
-import json
+import pickle
 import threading
@@ class JobQueue:
     def save(self, path):
         with self._lock:
             rows = [job.to_dict() for job in self._jobs.values()]
-        with open(path, "w") as f:
-            json.dump(rows, f)
+        with open(path, "wb") as f:
+            pickle.dump(rows, f)

     @classmethod
     def load(cls, path):
         queue = cls()
-        with open(path) as f:
-            rows = json.load(f)
+        with open(path, "rb") as f:
+            rows = pickle.load(f)
         for row in rows:
             job = Job(**row)
             queue._jobs[job.id] = job
```

**PR #4 — Pluggable scheduling policies** (opened by `felix-wu`)

> Priority-then-FIFO will not stay the only scheduling rule we need — the on-call rotation for the
> batch-import team has already asked for round-robin across tenants, so one noisy tenant cannot starve the
> others. Rather than hard-code a second rule into `pop_ready` later, this introduces a `SchedulingPolicy`
> interface and a small registry, and moves the existing behavior into a `PriorityPolicy` so nothing changes
> by default. Also added a general-purpose `metadata` field on `Job`, since a per-tenant policy will need
> somewhere to keep its own bookkeeping per job.

```diff
diff --git a/jobqueue.py b/jobqueue.py
--- a/jobqueue.py
+++ b/jobqueue.py
@@
+from abc import ABC, abstractmethod
+
 import heapq
 import itertools
 import json
 import threading


+class SchedulingPolicy(ABC):
+    """Decides which pending job pop_ready() should hand out next."""
+
+    @abstractmethod
+    def select(self, heap, jobs):
+        """heap: the queue's internal (-priority, seq, job_id) list. jobs: id -> Job.
+        Returns a ready Job, or None if none is ready."""
+
+
+class PriorityPolicy(SchedulingPolicy):
+    def select(self, heap, jobs):
+        while heap:
+            _, _, job_id = heapq.heappop(heap)
+            job = jobs[job_id]
+            if job.status == "pending":
+                return job
+        return None
+
+
+_POLICIES = {"priority": PriorityPolicy}
+
+
+def _resolve_policy(policy):
+    if isinstance(policy, SchedulingPolicy):
+        return policy
+    return _POLICIES[policy or "priority"]()
+
+
 class Job:
-    def __init__(self, id, payload, priority=0, status="pending", attempts=0, max_attempts=3):
+    def __init__(self, id, payload, priority=0, status="pending", attempts=0, max_attempts=3, metadata={}):
         self.id = id
         self.payload = payload
         self.priority = priority
         self.status = status
         self.attempts = attempts
         self.max_attempts = max_attempts
+        self.metadata = metadata
@@ class JobQueue:
-    def __init__(self):
+    def __init__(self, policy=None):
         self._lock = threading.Lock()
         self._jobs = {}
         self._heap = []
         self._counter = itertools.count()
+        self._policy = _resolve_policy(policy)
@@ class JobQueue:
     def pop_ready(self):
         """Return the highest-priority pending job, marking it "running" so no other caller can
         receive the same job. Returns None if no job is pending."""
         with self._lock:
-            while self._heap:
-                _, _, job_id = heapq.heappop(self._heap)
-                job = self._jobs[job_id]
-                if job.status == "pending":
-                    job.status = "running"
-                    return job
-            return None
+            job = self._policy.select(self._heap, self._jobs)
+            if job is not None:
+                job.status = "running"
+            return job
```

### Part 1 — Review the four pull requests

For each pull request, in the order given, write:

1. The product or engineering problem it is trying to solve, in one or two sentences.
2. A summary of its implementation approach and which parts of `jobqueue.py` (or its tests) it touches.
3. Review comments covering correctness, edge cases, maintainability, tests, performance, and security or
   compatibility, each pointing at a specific line or a specific behavior; state which tests are missing
   for the new or changed behavior.
4. Whether the pull request is mergeable as it stands; if not, exactly what blocks it.

### Part 2 — Improve one pull request

Choose exactly one of the four pull requests to improve; it need not be the one with the most obvious
problems. State which one and why, including why each of the other three was not chosen. Then:

1. Modify `jobqueue.py` (and `test_jobqueue.py`) directly so that the chosen pull request's remaining
   problems are fixed.
2. Add, update, or fix tests for every piece of new or changed behavior. Do not delete a test unless the
   behavior it checked no longer exists.
3. Run the repository's test command, `python -m unittest -v`, and report its result.
4. Rewrite the pull request's description: a summary of the change, the design trade-offs made, the test
   results, and any known limitations that remain.

## Reference solution

<details>
<summary>Show the reference solution</summary>

Confirm with the interviewer, if it is not already settled, that the written review comments are the
graded deliverable of Part 1 (not a transcript of the assistant's chat), and that "mergeable" means
mergeable into the codebase as it stands today, not mergeable after some unstated amount of future work.

### Working with the assistant

The time budget is unforgiving — four reviews and one hands-on fix — so what the assistant is asked to do
changes across the round.

For Part 1, the assistant is useful for a mechanical first pass on each diff: "list every line this diff
adds or removes and, for each, what observable behavior changes" produces a draft to edit, not a comment to
paste. It is not useful for the verdict itself: asked in general terms whether PR #3 "looks OK," it tends to
answer in general terms too — pickle is well documented, the diff is small. The question that actually
reaches the defect is specific to this code: "`load()` opens a file at a caller-supplied path — list every
place in this repository, present or plausible, that calls `save` or `load` with a path a value from outside
the process could influence." That is answerable from the file, line by line; "does this look safe" is not.
The same applies to PR #2: instead of asking whether moving the check outside the lock is safe, ask for one
interleaving of two threads, line by line, that reaches the duplicate-id check with both seeing `job_id not
in self._jobs`. If the assistant cannot produce one, the claim that the PR is safe carries no weight; if it
can, the interleaving is the review comment, almost verbatim.

Diffs the assistant proposes for Part 2 get read the way a colleague's PR would, not skimmed for a green
checkmark. Asked to make PR #1's batch atomic, a first draft that calls `self.submit(...)` in a loop from
inside `with self._lock:` is a plausible answer, since it reuses existing, tested code — and it would
deadlock on the first multi-item batch, because `threading.Lock` is not reentrant and `submit` takes the
same lock again before the outer `with` exits. Running the new tests once catches this regardless of whether
it was spotted by inspection first. A change is worth submitting only once its author can say in one
sentence why each line is needed, independent of the assistant's own explanation for it — if that sentence
cannot be produced, the line is dropped or sent back.

With roughly 55 minutes for four reviews plus one implementation, about 5 minutes per review — writing down
a verdict already reached while reading, not composing one inside the assistant — leaves the rest for Part 2.
That budget is itself why PR #1 is the one improved below: a correct fix to PR #2 or PR #3 would need to be
proven, not just rewritten, and proving either one (a forced-interleaving test, or a real redesign of the
persistence format) does not fit in what is left once four reviews are written.

### Part 1

**PR #1 — `submit_many`.** Solves batch submission at startup, replacing each caller's own loop over
`submit` with one call. The implementation is a one-line list comprehension that calls `self.submit` once
per item and collects the results; no other method changes.

- The description's performance claim is not what the diff does: `submit_many` calls `self.submit`, and
  `submit` still takes `self._lock` on every call, so the batch acquires the lock once per item, exactly as
  the loop it replaces did. The comment worth leaving is not "this is wrong" — it still works — but that
  the description overstates what changed, and a reader who trusts the description over the diff would
  believe a benefit that is not there.
- No test for an empty batch (`submit_many([])`; harmless here, just unstated), and none for a duplicate id
  *within* a batch or between the batch and a job already in the queue. On that duplicate, `self.submit`
  raises `ValueError` on the *n*-th item, but the first *n* - 1 have already been submitted and stay in the
  queue — the call is not atomic, and nothing here says whether that is intended.
- Mergeable: not as it stands. The implementation needs no change if partial submission on a duplicate id is
  the intended behavior, but that intent has to be written down and tested, and the two missing edge cases
  above need tests either way; the misleading performance claim should come out of the description.

**PR #2 — lock contention in `submit`.** Solves (or aims to solve) lock contention under a submission burst,
by moving the duplicate-id check ahead of `with self._lock:` so the common case, a fresh id, spends less
time holding the lock. Touches only `submit`; no test file changes.

- `if job_id in self._jobs` and the insertion that follows it are no longer one atomic step. Two threads
  calling `submit("x", ...)` at the same moment can both evaluate the check before either has inserted `"x"`
  into `self._jobs`; both see `"x" not in self._jobs`, both proceed, and both succeed — neither raises
  `ValueError`, although the PR's own description says "a duplicate id still raises `ValueError`." Whichever
  of the two `self._jobs["x"] = job` assignments runs last wins; the other caller's `Job` object is silently
  discarded even though its `submit` call returned normally. Any caller that relies on the exception to
  detect a collision (for instance, retried submission of the same job id, a common way to make submission
  idempotent) loses that guarantee precisely under the concurrent load this PR is optimizing for.
- No new test covers concurrent submission of the same id — the one behavior this change touches.
- The performance motivation itself is asserted, not measured: the description cites profiling but the PR
  carries no benchmark, and the check moved outside the lock is a single dict lookup, not an expensive
  operation, so the win it is trading correctness for is not established.
- Mergeable: no. This is a regression on the one property (rejecting a duplicate id) the module explicitly
  contracts for, it is silent (no exception, no log line), and the fix is not a smaller version of this
  diff — it is reverting to checking and inserting under one uninterrupted hold of `self._lock`.

**PR #3 — pickle persistence.** Solves representing non-JSON-safe payloads in a saved job. Touches only
`save` and `load`, replacing `json.dump`/`json.load` with `pickle.dump`/`pickle.load` and switching the file
mode from text to binary; call sites are unchanged, which is exactly why this reads as a safe, drop-in
refactor at a glance.

- `pickle.load` executes code while deserializing: a crafted file can call arbitrary Python through an
  object's `__reduce__` the moment `load` reads it. `json.load` has no such path — unparseable input raises
  `json.JSONDecodeError`, or `UnicodeDecodeError` if the bytes are not even valid text, and nothing else
  runs. Whether this is exploitable depends on whether a `path`
  ever reaches `load` that the process itself did not just write with `save` — a restored volume, an
  uploaded crash dump, a path derived from a job id — and the PR does not discuss that boundary at all, let
  alone defend it.
- Old snapshots written by the current `save` are plain JSON text; the new `load` opens them in binary mode
  and calls `pickle.load` on JSON bytes, which fails rather than reading them, with no migration and no
  mention in the description that the format change breaks existing files.
- No new test for `save`/`load` at all; in particular nothing exercises the richer payload types (namedtuple,
  dataclass, numpy array) the description gives as the motivation, so the one thing this PR claims to newly
  support is untested.
- Mergeable: no, and not fixable by adjusting this diff. If richer payloads are a real requirement, the
  answer is a payload encoding that cannot execute code on load — for example JSON plus a small, explicit
  registry of extra types the loader is allowed to reconstruct — not a general-purpose deserializer pointed
  at a file path.

**PR #4 — pluggable scheduling policies.** Solves (for a use case that does not exist in this repository
yet) letting `pop_ready` use a scheduling rule other than priority-then-FIFO. Touches the top of the file
(new `SchedulingPolicy` interface and registry), `Job.__init__` (new `metadata` field), `JobQueue.__init__`
(new `policy` parameter), and `pop_ready` (delegates to the policy).

- `Job.__init__(..., metadata={})` gives `metadata` a mutable default argument. Every `Job` created without
  an explicit `metadata=` shares the *same* dict object, so writing to one job's `metadata` silently changes
  every other job's. This is a concrete, provable bug independent of any opinion about the design, and it
  ships even though no policy in this PR reads or writes `metadata`.
- Only one concrete policy, `PriorityPolicy`, exists; the interface, the registry, and the new constructor
  parameter have no second implementation to be exercised against, so nothing here can yet show the
  abstraction is the right shape for the round-robin use case the description motivates it with.
- `SchedulingPolicy.select`'s docstring does not say whether an implementation may assume `self._lock` is
  already held when it runs (`pop_ready` calls it from inside the lock); `PriorityPolicy` needs no locking of
  its own, so the question does not surface here, but a future policy author has no way to answer it.
- `_resolve_policy` looks up an unknown string policy name in `_POLICIES` with plain indexing, so
  `JobQueue(policy="round_robin")` before that policy exists fails with a bare `KeyError`, not a clear error
  at the call site. No test exercises `policy=`, `SchedulingPolicy`, `PriorityPolicy`, or `metadata` at all.
- Mergeable: no. The mutable-default bug is a hard blocker by itself; past that, this is a scope question for
  a conversation with the author, not a line-by-line fix — ship `PriorityPolicy`'s behavior unchanged today
  and defer the plugin surface until a second policy actually needs it, or land the interface alone with a
  real second implementation in the same PR so it is reviewable.

### Part 2

PR #1 is the one whose remaining work is bounded and fully verifiable inside what is left of the time box.
PR #2 and PR #3 are not partial fixes away from mergeable — PR #2 needs its concurrency claim actually
proven, not just re-asserted, and PR #3 needs a different persistence design, not a smaller diff. PR #4's
mutable-default bug is a one-line fix, but the PR's real problem is scope, which is a decision for the
author to make, not one to make unilaterally on their behalf. PR #1's implementation is already correct for
its stated (non-atomic) behavior; what is missing is deciding and testing the one thing the description
left silent — what happens on a duplicate id inside a batch — and, since the description's own performance
claim is worth honoring rather than deleting, actually making the batch take the lock once.

The fix validates every id before inserting any job, under one unbroken hold of `self._lock`, so a batch
either succeeds completely or leaves the queue exactly as it found it:

```python
def submit_many(self, items):
    """items: an iterable of (job_id, payload, priority) triples. Submits the whole batch atomically:
    if any job_id is already taken -- by a job already in the queue, or by an earlier item in this same
    batch -- the call raises ValueError and no job in the batch is created. Returns the list of created
    Job objects, in the given order, on success."""
    items = list(items)
    with self._lock:
        seen = set()
        for job_id, payload, priority in items:
            if job_id in self._jobs or job_id in seen:
                raise ValueError(f"duplicate job id {job_id!r}")
            seen.add(job_id)
        created = []
        # NOTE: inserts directly rather than calling self.submit(), which would deadlock here --
        # self._lock is a plain threading.Lock, not reentrant, and submit() takes it too.
        for job_id, payload, priority in items:
            job = Job(job_id, payload, priority, "pending", 0, 3)
            self._jobs[job_id] = job
            heapq.heappush(self._heap, (-priority, next(self._counter), job_id))
            created.append(job)
        return created


JobQueue.submit_many = submit_many
```

Validating first and inserting second, both under the same lock acquisition, is what makes this safe under
concurrent callers: nothing can observe the queue between the check and the insert, the same property PR #2
got wrong for a single `submit`. Materializing `items` into a list up front means a batch built from a
generator can be scanned twice (once to validate, once to insert) without the generator being exhausted
after the first pass.

Four tests replace PR #1's one happy-path test: the happy path, the empty batch, a duplicate within the
batch, and a duplicate against a job already in the queue — the two cases the original PR left untested,
plus a check that both failure cases leave the queue completely unchanged:

```python
import unittest


class TestSubmitMany(unittest.TestCase):
    def test_happy_path_submits_all_in_order(self):
        q = JobQueue()
        jobs = q.submit_many([("a", {}, 1), ("b", {}, 5), ("c", {}, 0)])
        self.assertEqual([j.id for j in jobs], ["a", "b", "c"])
        self.assertEqual(q.pop_ready().id, "b")          # highest priority first, batch order aside

    def test_empty_batch_returns_empty_list(self):
        q = JobQueue()
        self.assertEqual(q.submit_many([]), [])
        self.assertEqual(q.counts(), {"pending": 0, "running": 0, "done": 0, "failed": 0})

    def test_duplicate_within_batch_submits_nothing(self):
        q = JobQueue()
        with self.assertRaises(ValueError):
            q.submit_many([("a", {}, 0), ("b", {}, 0), ("a", {}, 0)])
        self.assertEqual(q.counts(), {"pending": 0, "running": 0, "done": 0, "failed": 0})

    def test_duplicate_against_existing_job_submits_nothing_new(self):
        q = JobQueue()
        q.submit("a", {})
        with self.assertRaises(ValueError):
            q.submit_many([("b", {}, 0), ("a", {}, 0)])
        self.assertEqual(q.counts(), {"pending": 1, "running": 0, "done": 0, "failed": 0})
```

Running the repository's test command over this suite together with the existing `TestJobQueue` tests
described in the Problem section (reconstructed below so the checks are self-contained) gives:

```text
python -m unittest -v
test_completed_job_reports_done (TestJobQueue) ... ok
test_fail_retries_then_marks_failed (TestJobQueue) ... ok
test_pop_ready_priority_then_submission_order (TestJobQueue) ... ok
test_save_load_roundtrip (TestJobQueue) ... ok
test_submit_rejects_duplicate_id (TestJobQueue) ... ok
test_duplicate_against_existing_job_submits_nothing_new (TestSubmitMany) ... ok
test_duplicate_within_batch_submits_nothing (TestSubmitMany) ... ok
test_empty_batch_returns_empty_list (TestSubmitMany) ... ok
test_happy_path_submits_all_in_order (TestSubmitMany) ... ok

----------------------------------------------------------------------
Ran 9 tests in 0.002s

OK
```

**Rewritten PR #1 description.**

> **Summary.** `submit_many` lets a caller submit a batch of `(job_id, payload, priority)` triples in one
> call instead of looping over `submit`. Unlike that loop, the batch is atomic: it takes `self._lock` once
> for the whole call, validates every id first, and either creates every job in the batch or, if any id
> collides with an existing job or with an earlier item in the same batch, creates none of them and raises
> `ValueError`.
>
> **Trade-offs.** Atomicity costs one extra pass over `items` to validate before inserting, and requires
> materializing `items` into a list even if the caller passed a generator, so the whole batch is held in
> memory at once. Neither cost is significant at the batch sizes this is for (startup backlog replay, a few
> hundred jobs); a caller seeding a very large one-off backlog should chunk it rather than pass one huge
> batch.
>
> **Test results.** `python -m unittest -v` passes, 9 tests / 0 failures, shown above; both new duplicate
> cases additionally assert the queue is left unchanged.
>
> **Known limitations.** Atomicity is validated only within one process; two processes sharing state purely
> through `save`/`load` files could still each accept the same id before either has written its file. The
> error message on a collision does not say whether the clash was against an existing job or another item in
> the same batch — fine for today's only caller (logging the batch and retrying), worth revisiting if a
> caller needs to distinguish the two.

### Follow-ups

- A crash leaves some jobs `"running"` forever: `load` only re-queues `"pending"` jobs, by design, since a
  `"running"` job might genuinely still be in flight in another process. A supervisor that knows no worker
  survived the crash would need a separate `requeue_stuck(older_than)` call, not a change to `load` itself.
- `pop_ready` returning `None` immediately makes an idle worker either busy-poll or sleep between calls; the
  alternative, blocking inside `pop_ready` until a job is ready, trades that for a condition variable that
  every `submit` and `fail` must remember to notify.
- `self._lock` only protects threads inside one process; two processes both calling `save` against the same
  path, or one saving while another loads, can still interleave at the filesystem level and need their own
  coordination (a lock file, or a single process that owns the state file) that is out of scope for this
  module.

<details>
<summary>Checks (runnable)</summary>

```python
import io
import json
import os
import pickle
import tempfile
import threading
import unittest


# --- run the Part 2 test suite: the repository's test command, python -m unittest -v ---
class TestJobQueue(unittest.TestCase):
    def test_submit_rejects_duplicate_id(self):
        q = JobQueue()
        q.submit("a", {})
        with self.assertRaises(ValueError):
            q.submit("a", {})

    def test_pop_ready_priority_then_submission_order(self):
        q = JobQueue()
        q.submit("low", {}, priority=0)
        q.submit("high", {}, priority=5)
        q.submit("also-low", {}, priority=0)
        self.assertEqual(q.pop_ready().id, "high")
        self.assertEqual(q.pop_ready().id, "low")         # equal priority: earlier submission first
        self.assertEqual(q.pop_ready().id, "also-low")
        self.assertIsNone(q.pop_ready())

    def test_completed_job_reports_done(self):
        q = JobQueue()
        q.submit("a", {})
        q.complete(q.pop_ready().id)
        self.assertEqual(q.get("a").status, "done")

    def test_fail_retries_then_marks_failed(self):
        q = JobQueue()
        q.submit("a", {}, max_attempts=2)
        q.fail(q.pop_ready().id)
        self.assertEqual(q.get("a").status, "pending")    # one attempt left
        q.fail(q.pop_ready().id)
        self.assertEqual(q.get("a").status, "failed")      # attempts exhausted

    def test_save_load_roundtrip(self):
        q = JobQueue()
        q.submit("a", {"x": 1}, priority=2)
        q.pop_ready()                                       # "a" is now "running"
        q.submit("b", {"y": 2}, priority=1)
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, "state.json")
            q.save(path)
            restored = JobQueue.load(path)
        self.assertEqual(restored.get("a").status, "running")
        self.assertEqual(restored.get("a").payload, {"x": 1})
        self.assertEqual(restored.get("b").status, "pending")
        self.assertEqual(restored.pop_ready().id, "b")      # "a" is running, not re-queued


suite = unittest.TestSuite([
    unittest.defaultTestLoader.loadTestsFromTestCase(TestJobQueue),
    unittest.defaultTestLoader.loadTestsFromTestCase(TestSubmitMany),   # defined in Part 2, above
])
result = unittest.TextTestRunner(verbosity=0, stream=io.StringIO()).run(suite)
assert result.wasSuccessful() and result.testsRun == 9
print(f"python -m unittest -v: {result.testsRun} tests, "
      f"{len(result.failures) + len(result.errors)} failures -- OK")


# --- PR #2: reconstruct the lock-scope change and force the exact bad interleaving ---
class RacyJobQueue(JobQueue):
    """PR #2's submit(): the duplicate-id check runs before the lock is taken. `_checkpoint` runs
    once, right after that check, purely to force a deterministic interleaving below -- production
    code never passes one."""

    def submit(self, job_id, payload, priority=0, max_attempts=3, _checkpoint=lambda: None):
        if job_id in self._jobs:
            raise ValueError(f"duplicate job id {job_id!r}")
        _checkpoint()
        with self._lock:
            job = Job(job_id, payload, priority, "pending", 0, max_attempts)
            self._jobs[job_id] = job
            heapq.heappush(self._heap, (-priority, next(self._counter), job_id))
        return job


def demonstrate_forced_race():
    """Thread A pauses right after its duplicate check, with job "x" still absent; thread B then
    runs its own submit("x", ...) to completion; only then does thread A resume and insert."""
    queue = RacyJobQueue()
    a_at_checkpoint, b_finished = threading.Event(), threading.Event()
    results = {}

    def checkpoint():
        a_at_checkpoint.set()
        assert b_finished.wait(timeout=5), "thread b did not finish in time"

    def run_a():
        try:
            results["a"] = queue.submit("x", {"from": "a"}, _checkpoint=checkpoint)
        except ValueError as e:
            results["a"] = e

    def run_b():
        assert a_at_checkpoint.wait(timeout=5), "thread a never reached its checkpoint"
        try:
            results["b"] = queue.submit("x", {"from": "b"})
        except ValueError as e:
            results["b"] = e
        b_finished.set()

    ta, tb = threading.Thread(target=run_a), threading.Thread(target=run_b)
    ta.start(), tb.start()
    ta.join(timeout=5), tb.join(timeout=5)
    return queue, results


racy_queue, racy_results = demonstrate_forced_race()
assert not isinstance(racy_results.get("a"), Exception) and not isinstance(racy_results.get("b"), Exception), \
    f"PR #2: both callers should wrongly believe they succeeded, got {racy_results}"
assert racy_queue.counts()["pending"] == 1, "PR #2: two successful submits should collapse into one job"
first, second = racy_queue.pop_ready(), racy_queue.pop_ready()
assert first is not None and first.id == "x" and second is None, \
    "PR #2: one of the two submitted payloads should have vanished without a trace"
print("PR #2 (racy submit): two threads both 'succeeded', but only one job was ever dispatched")


def concurrent_double_submit(queue):
    results = {}

    def call(key):
        try:
            results[key] = queue.submit("x", {"from": key})
        except ValueError as e:
            results[key] = e

    ta, tb = threading.Thread(target=call, args=("a",)), threading.Thread(target=call, args=("b",))
    ta.start(), tb.start()
    ta.join(timeout=5), tb.join(timeout=5)
    return results


for trial in range(50):                                      # no forced interleaving needed: the
    fixed_queue = JobQueue()                                  # lock makes every interleaving correct
    outcome = concurrent_double_submit(fixed_queue)
    wins = sum(not isinstance(v, Exception) for v in outcome.values())
    fails = sum(isinstance(v, ValueError) for v in outcome.values())
    assert wins == 1 and fails == 1, f"trial {trial}: fixed submit() let both or neither succeed -- {outcome}"
    assert fixed_queue.counts()["pending"] == 1
print("PR #2 (fixed submit): 50 concurrent trials, always exactly one winner and one ValueError")


# --- PR #3: reconstruct the pickle switch and prove it executes an attacker-controlled file ---
class PickleJobQueue(JobQueue):
    """PR #3's save/load: pickle instead of json, otherwise identical to JobQueue."""

    def save(self, path):
        with self._lock:
            rows = [job.to_dict() for job in self._jobs.values()]
        with open(path, "wb") as f:
            pickle.dump(rows, f)

    @classmethod
    def load(cls, path):
        queue = cls()
        with open(path, "rb") as f:
            rows = pickle.load(f)
        for row in rows:
            job = Job(**row)
            queue._jobs[job.id] = job
            if job.status == "pending":
                heapq.heappush(queue._heap, (-job.priority, next(queue._counter), job.id))
        return queue


_rce_marker = []


def _record(tag):     # a module-level function so pickle can reference it by name, like any real one
    _rce_marker.append(tag)
    return []          # PickleJobQueue.load then iterates an empty "rows": no crash, no trace either


class MaliciousState:
    """What an attacker-controlled "saved state" file can contain: __reduce__ makes unpickling it
    call an arbitrary function with arbitrary arguments, before any application code runs."""

    def __reduce__(self):
        return (_record, ("pwned",))


with tempfile.TemporaryDirectory() as d:
    path = os.path.join(d, "state")
    with open(path, "wb") as f:
        pickle.dump(MaliciousState(), f)

    PickleJobQueue.load(path)                                  # merely loading the file runs _record
    assert _rce_marker == ["pwned"], "PR #3: unpickling should have executed the payload"

    try:
        JobQueue.load(path)                                    # the json-based load on the same bytes
        raised = None
    except Exception as e:
        raised = e
    assert raised is not None, "PR #3 control: pickle bytes are not valid JSON and must fail to parse"
    assert _rce_marker == ["pwned"], "json.load must never execute the file's content, however it fails"
print(f"PR #3 (pickle load): crafted file executed code on load; "
      f"the json-based load only raised {type(raised).__name__}")


# --- PR #4: reconstruct the mutable-default bug on Job.metadata ---
class BuggyJob:
    """PR #4's change to Job.__init__: metadata={} is a mutable default argument."""

    def __init__(self, id, metadata={}):
        self.id = id
        self.metadata = metadata


buggy_a, buggy_b = BuggyJob("a"), BuggyJob("b")
buggy_a.metadata["owner"] = "tenant-1"
assert buggy_b.metadata == {"owner": "tenant-1"}, "PR #4: every Job should wrongly share one dict"


class FixedJob:                       # metadata=None, defaulted inside __init__: the standard fix
    def __init__(self, id, metadata=None):
        self.id = id
        self.metadata = metadata if metadata is not None else {}


fixed_a, fixed_b = FixedJob("a"), FixedJob("b")
fixed_a.metadata["owner"] = "tenant-1"
assert fixed_b.metadata == {}, "a fresh dict per job must not leak between jobs"
print("PR #4 (mutable default): confirmed the shared-dict leak, and that metadata=None does not have it")

print("all checks passed")
```

</details>

</details>
