# Design Doc Review: A Batched Inference Service

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Design review | ★★★★☆ | Hard | SWE · MLE · EM | design-review, batching, load-balancing, capacity-planning, gpu-scheduling | 55 min | Technical phone screen · Onsite |
<!-- meta:end -->

## Problem

The document below proposes a batching and load-balancing layer for a pool of GPU model servers.
Each server holds one loaded model and executes requests in *batches*: a batch groups several
requests into one call to the model, so the GPU does one pass of work instead of many. A server's
batch call has a fixed contract that this review does not revisit — it is owned by a different team:

```py
def run_batch(requests: list[Request]) -> list[Response]:
    """Runs one inference batch on this server and blocks until it returns."""
```

- `1 <= len(requests) <= 50`; the call returns exactly one response per request, in the same order.
- The call takes a fixed **100 ms**, whether 1 or 50 of the 50 slots are filled.
- A server accepts one `run_batch` call at a time; a second call blocks until the first returns.
- The model itself is a black box behind this call: this review is about the system that queues,
  batches, and routes requests to it, not about anything that happens inside `run_batch`.
- Continuous or token-level batching, and any other change to this contract, are out of scope.

Spend about ten minutes reading the document, then complete the tasks below it. The document has
problems of different severity, in both what it proposes and what it never addresses, and at least
two places where the design has already been reasoned through correctly — treat both kinds as part
of the review.

> **Batching and load balancing for the inference-serving tier**
> **Status: draft, circulated for review**
>
> **Context.** Clients send inference requests over HTTP; each request names a model and carries its
> input. One pool of identical servers behind this design serves a single model. Most traffic is
> *interactive* — a person is waiting on the response — and a smaller share is *bulk*: requests
> submitted programmatically in large numbers, with no one waiting on any individual response. At
> peak the service receives **10,000 requests/second**, of which 9,000/s are interactive and 1,000/s
> are bulk.
>
> **Goals.** Sustain 10,000 requests/second at peak. Keep GPU utilization high by batching requests
> rather than running them one at a time. Give a client a way to recover a lost response without
> paying to redo work that already finished.
>
> **Non-goals.** Changing the `run_batch` contract — fixed batch size, fixed latency, one call at a
> time per server — that belongs to the runtime team. Multi-region failover. Anything about the
> model's own accuracy or sampling behavior.
>
> **Proposed design.**
>
> *Batching.* Each server owns a request queue and a batcher. A pending batch closes and is sent to
> its server's `run_batch` call as soon as it reaches 50 requests, or 30 ms after the first request in
> it arrived, whichever comes first — except that if the server is still busy with a previous batch
> when the 30 ms timer fires, the pending batch keeps absorbing arrivals rather than closing early,
> since closing it sooner would not get it to the server any faster. Interactive and bulk requests are
> not distinguished once admitted: both sit on the same queue and batch together in arrival order.
>
> *Load balancing.* A stateless load balancer assigns each new client connection to one of the
> servers by round robin; every request sent on that connection is forwarded to its assigned server's
> queue for the connection's lifetime. This keeps the load balancer itself simple — it never inspects
> a server's queue — and spreads new connections evenly across the fleet by construction.
>
> *Response cache.* Responses are cached in a fixed-size table of 250,000 entries (evicted
> least-recently-used), keyed by an exact hash of the request's input. Replaying a one-week sample of
> 3.2 million production requests, 35% of requests matched an entry already in that sample — so we
> expect the cache to absorb about a third of traffic in production.
>
> *Retries.* If a client does not receive a response within 500 ms, it should retry the request with
> the same parameters.
>
> *Model isolation.* A server is pinned to one model at a time; the router rejects a request for a
> model a server doesn't hold before that request reaches any queue, so a batch never mixes requests
> for two different models.
>
> *Observability.* The service exports the request-count and latency metrics that every service in
> the fleet already emits.
>
> **Capacity estimate.** Applying the 35% cache offset, the batching tier must serve about
> 10,000 × 0.65 = 6,500 requests/second. Each server completes a full batch of 50 requests every
> 100 ms, i.e. 500 requests/second. 6,500 / 500 = 13 servers; we will provision 15 for a small margin.
>
> **Rollout.** Deploy the new batching and load-balancing layer behind a feature flag, run it
> alongside the existing path for a week, then switch all traffic over.
>
> **Appendix: a shared pool for a large and a small model.** Some deployments serve two sizes of the
> same model from one pool of 8 GPUs: a batch for the *large* size must run across all 8 GPUs at once,
> a batch for the *small* size runs on a single GPU, and a batch of either size takes the same 100 ms.
> We propose two queues, one per model size, each batched as above. Whichever queue has the higher
> estimated arrival rate is dispatched first when both have a ready batch; when the two rates are
> close, the large-model queue goes first, since it fills a full batch sooner.

Produce:

1. A ranked list of the most significant problems with the design — both what it gets wrong and what
   it never addresses — in priority order. For each: the section or claim it responds to, the concrete
   way it fails, backed by a number, and an alternative with its trade-off. Leave out minor wording
   issues; rank by consequence, not by how many problems you can list.
2. A corrected capacity estimate for the main pool: how many servers peak traffic actually requires,
   and which of the document's own assumptions were wrong.
3. What the Observability and Rollout sections would need to add before this could ship.
4. Follow-up: a critique of the dispatch policy the appendix proposes for the shared 8-GPU pool, and a
   policy that fixes it.

## Reference solution

<details>
<summary>Show the reference solution</summary>

Worth confirming before reviewing: whether the interviewer wants the ranked list in full before any
discussion, or the top risk first with the rest raised as time allows — this write-up gives the full
ranking, as the numbered task asks. It also assumes the whole pipeline downstream of the fixed
`run_batch` contract is open for revision, since nothing else in the document is marked as pinned.

### The review, ranked

1. **"Each server completes a full batch of 50 requests every 100 ms" (Capacity estimate).** This is
   only true if every batch that reaches a server is completely full, which needs the server to
   always have at least 50 requests waiting the instant its previous batch finishes — the estimate
   assumes the one outcome that needs zero queueing margin, the opposite of what a capacity plan
   should assume. At the fill a real deployment achieves — the corrected estimate below uses 66.7%,
   itself already after 20% headroom is added on top — the same formula gives 30 servers, not 15.
   Every other number built on "500 requests/second per server" inherits this error. *Proposed
   change:* size the fleet from an achieved batch size below the maximum, with headroom on top, as in
   the corrected estimate below.

2. **"35% of requests matched an entry already in that sample" (Response cache).** This measures
   whether a request repeats *at all* within a one-week, 3.2-million-request sample — an unbounded
   lookback of up to a week. The proposed cache holds only 250,000 entries under LRU eviction; at the
   document's own 6,500 requests/second effective load, a new entry displaces the oldest one roughly
   every 250,000 / 6,500 ≈ 38 seconds. A match found only by searching up to a week back is
   overwhelmingly likely to have already fallen out of a cache that forgets anything older than 38
   seconds, unless repeats cluster inside that window — which a 35% weekly figure does not establish,
   and the document does not measure separately. If repeat gaps are spread roughly evenly across the
   week (a generous assumption, since nothing in the document suggests clustering), the same data
   implies a true hit rate near 35% × (38 s / 604,800 s) ≈ 0.002%: negligible, not a third of traffic.
   *Proposed change:* measure the hit rate by replaying the week through an actual bounded, evicting
   cache of the proposed size, not by checking for any match anywhere in the sample.

3. **"Interactive and bulk requests are not distinguished once admitted" (Batching).** Bulk traffic is
   submitted programmatically and can arrive in a burst far larger than any single interactive
   request would generate — a scheduled reprocessing job enqueuing hundreds of thousands of requests
   at once is exactly the bulk traffic Goals says this system should carry. On one shared FIFO queue,
   that burst sits ahead of every interactive request submitted after it, and nothing in this document
   bounds how long that is. *Proposed change:* two logical queues, one per traffic class, each with
   its own servers or its own share of a common pool; splitting the corrected 30-server pool this way
   costs 2 extra servers to lost pooling efficiency (32 vs. 30, worked out below) — a small price for
   making sure bulk traffic cannot delay interactive traffic at all.

4. **Nowhere in the document.** Nothing bounds how many requests can be queued, and nothing sheds load
   when demand exceeds capacity: every request that arrives is queued indefinitely, however far behind
   the servers already are. Losing half the corrected 30-server fleet to a bad rollout for two
   minutes — a realistic scenario the Rollout section does nothing to guard against — drops capacity
   to 7,500 requests/second against 10,000/second of demand; even after the fleet recovers, draining
   the 300,000-request backlog that built up takes another 60 seconds on top of the 120-second
   incident. With no cap, every one of those requests is still served, just increasingly late, instead
   of a bounded number being shed quickly so the rest keep meeting the SLA. *Proposed change:* cap the
   total admitted-but-unfinished count per queue at a small multiple of one batch cycle's worth of
   capacity; past that, reject new requests immediately, with a retry hint, shedding bulk traffic
   before interactive.

5. **"assigns each new client connection to one of the servers by round robin" (Load balancing).**
   Round robin balances *connection count*, not *request rate* — it has no way to know a new
   connection will turn out to be a chatty client sending hundreds of requests a second. If two such
   connections land on the same server, which nothing here prevents, that server alone can be asked
   for more than its 500 requests/second ceiling while the fleet-wide average still looks perfectly
   healthy. The simulation below places three connections at about 467 requests/second among twenty
   connections, on ten servers at a healthy 70% fleet-wide average utilization: the busiest server
   alone is asked for about 590 requests/second, above its ceiling by itself, and its queue grows
   without bound over the run — its p99 wait grows from about 5.5 s to about 14.8 s as the run
   lengthens from 30 s to 80 s, instead of settling. *Proposed change:* route each request
   independently rather than pinning a connection to one server; routing by current backlog (for
   example, power of two choices: sample two servers, send to whichever reports the shorter queue)
   also protects against a server that is individually slow or draining, which round robin at any
   granularity cannot see. In the same simulation, backlog-aware routing holds p99 at about 0.2 s
   regardless of run length.

6. **"it should retry the request with the same parameters" (Retries).** Nothing gives a retried
   request a way to find the original one; a slow-but-successful call and a genuinely dropped one look
   identical to the client, so both get retried, and the server runs the request again from scratch.
   Under exactly the degraded conditions items 1, 2, and 4 create — the moment retries are most
   likely — even a modest 5% retry rate turns 500 of the 10,000 requests/second into pure duplicate
   work: a full server's worth of capacity spent recomputing answers that were already produced.
   *Proposed change:* a client-supplied idempotency key, stored with the completed response for a
   short TTL; a retry carrying a seen key returns the stored result instead of running the request
   again.

7. **Nowhere in the document.** No section says what happens to the up to 50 requests inside a
   `run_batch` call on a server that stops responding mid-call — no lease, no heartbeat, no detection.
   At a plausible 4-hour mean time between crashes per server, the corrected 30-server fleet loses a
   server about every 8 minutes; each time, up to 50 in-flight requests are silently dropped, with
   nothing to notice, retry, or report the loss. *Proposed change:* a lease per dispatched batch and a
   heartbeat from the server; a lease that expires without a heartbeat requeues its requests onto a
   fresh server, using the same idempotency key from item 6 so a request the client already resent
   separately doesn't run twice.

### Corrected capacity estimate

The formula is servers = ceil(QPS × batch_time / (achieved_batch_size × slots)), plus headroom — not
the document's QPS × batch_time / max_batch_size, which is only the special case where
achieved_batch_size equals the maximum on every single dispatch.

**Demand.** Item 2 shows the cache offset is not real; demand stays at the full 10,000
requests/second.

**Achieved batch size.** The document's own 30 ms max-wait, together with waiting out a busy server
rather than closing a timed-out batch early, is short enough that once a server has any backlog at
all, the wait timer stops being the binding constraint — the pending batch simply absorbs arrivals for
as long as the server is busy, which tracks utilization directly (that part of the design needs no
note; see Calibration). Targeting 80% average fill for provisioning gives achieved_batch_size =
0.8 × 50 = 40.

**Slots.** 1 — a server runs one `run_batch` call at a time, per the fixed contract.

**Raw.** ceil(10,000 × 0.1 / (40 × 1)) = ceil(25) = 25.

**Headroom.** +20%, enough to lose a couple of servers to a rolling deploy or absorb uneven load
without breaching the fill target: 25 × 1.2 = 30.

**Realized.** At 30 servers, each carries 10,000 / 30 ≈ 333.3 requests/second, ρ ≈ 66.7%, achieved
fill ≈ 33.3 of 50 — below the 80% used to size the raw count, which is the headroom doing its job: the
same pattern applies throughout capacity planning, provisioning for the point that stays safe, not for
the average.

**Conclusion.** 30 servers — double the document's 15.

Splitting the pool into an interactive queue and a bulk queue (item 3) applies the same formula to
each share separately: interactive (9,000/s) needs ceil(900/40) = 23 raw, ×1.2 → 28; bulk (1,000/s)
needs ceil(100/40) = 3 raw, ×1.2 → 4. 28 + 4 = 32, two more than the pooled 30 — the cost of losing
some pooling efficiency to isolation, paid at both the raw and the headroom rounding step.

### Missing sections

**Observability.** Exporting the same request-count and latency metrics every other service already
emits says nothing about *this* service's own failure modes. Before this ships it needs: the
batch-fill distribution, so an under-filled batch shows up directly instead of being inferred later
from a capacity shortfall; per-server queue depth and time since a server's batch last closed, so the
imbalance in item 5 is visible live rather than only in a retrospective simulation; latency split into
queueing and execution, so a slow response can be told apart from an overloaded queue; a live cache
hit rate measured against the actual running cache, not the offline weekly-sample figure item 2
disputes; the retry and duplicate-completion rate once item 6's idempotency key exists; and the rate
of mid-batch server loss from item 7, however it ends up being detected.

**Rollout.** "Run it alongside the existing path for a week, then switch all traffic over" names no
criterion for what passing that week means, no canary percentage before all traffic moves, and no
rollback trigger beyond someone deciding to stop. Before this ships it needs: a canary that starts at
a small percentage of traffic and grows on explicit, met criteria, not a fixed week regardless of what
happens; a rollback trigger tied to an SLO, such as interactive p99 above target for some sustained
window, so reverting does not depend on someone noticing; a load test at the corrected 10,000
requests/second, not the document's 6,500, before any real traffic reaches the new path; and a stated
behavior for a request already in flight on the old path at the moment the flag flips.

### The shared 8-GPU pool

The appendix's policy compares the two queues' estimated arrival rates and, when they are close,
dispatches the large-model queue first. That is a tie-break for the moment when both a small batch
and a large batch are ready and GPUs happen to be free; it says nothing about the rest of the time,
which for a resource that needs gang scheduling — all 8 GPUs at once, not eight independent one-GPU
slots — is the part that matters. Nothing in the policy ever holds a GPU back or stops the
small-model queue from claiming one the instant it frees, so under any sustained small-model load a
freed GPU is claimed by the next ready small batch before the tie-break has anything to act on. All 8
GPUs are rarely simultaneously idle, no matter how the tie-break is written, and the large-model queue
starves — not merely waits longer.

The simulation below reproduces this directly. At a small-model load that keeps the pool a little
over 90% busy on small work alone — not an extreme figure — every large request still completes
within the 200-second run, but its p99 wait is about 194 seconds, within a few seconds of the entire
run, against 0.12 seconds for small requests served in the meantime. For any real latency target that
is starvation in practice, whether or not the queue is formally unbounded at this exact load.

**Proposed policy: atomic reservation with a bounded drain.** A fix needs two guarantees the
document's policy has neither of: a bound on how long a formed large batch can wait, and a bound on
how long any small request waits behind one. Small batches keep dispatching onto any free GPU the
instant they are ready — nothing changes for the common case. What changes is what happens once a
large batch has *formed*: instead of simply waiting for all 8 GPUs to align by chance, it starts a
clock. Once that clock reaches `aging` with the pool still not entirely free, the dispatcher freezes
admission of any *new* small batch — batches already running are left to finish their own 100 ms —
until the instant all 8 GPUs are simultaneously idle, at which point it reserves them as one atomic
unit and launches the large batch. The first guarantee follows immediately: `aging` plus at most one
more small-batch cycle. The second follows from the freeze itself: once a large batch is running, no
small request already queued waits longer than that one run plus however long its own queue took to
drain before the freeze began. At `aging` = 150 ms the simulation puts the large model's p99 wait at
about 0.87 s and the small model's at about 0.33 s, against roughly 194 s for the large model under
the document's policy.

**Utilization cost.** This fix is not free, but most of what it costs was never avoidable: holding
all 8 GPUs for a batch that needs all 8 spends about 8.1% of GPU-seconds under either policy, the
document's or the fix, since both eventually run every large batch to completion — reserving a
gang-scheduled resource costs the same no matter how the queue in front of it is managed. The fix
adds one further cost on top, the drain itself: once the pool starts waiting out the small batches
still running, a GPU that finishes early has nothing to do until the slowest one catches up, because
only then can all 8 be handed to the large batch together. At `aging` = 150 ms that idle time comes to
about 3.1% of GPU-seconds; stretching `aging` to 1 s cuts it to about 2.4% by making drains rarer and
fuller, at the cost of pushing the large model's p99 wait to about 1.7 s, roughly double. `aging` is
therefore a dial between the large model's tail latency and the fleet's utilization, not a setting to
optimize away — no value of it drives idle time to zero, since that would mean knowing in advance
which small batch will finish last and steering new work elsewhere just before it does.

### Calibration

Two places in the document need no note, and saying so is as much a part of the review as flagging
what is wrong. The 30 ms max-wait, together with waiting out a busy server rather than closing a
timed-out batch early, is exactly the right rule: once a server has any backlog, the wait timer stops
binding and the achieved batch size tracks utilization on its own, which is what makes an 80% fill
target reasonable to provision for in the first place. Model isolation — a server pinned to one model,
with the router rejecting a mismatched request before it reaches any queue — already closes off the
one way a batch could end up mixing two different models' requests; there is nothing to add there.

The `run_batch` contract itself — fixed batch size, fixed latency, one call at a time — is in the same
category for a different reason: it is pinned by a different team and stated as a non-goal, so a
review that spends its time arguing for continuous batching instead is answering a question that was
not asked, however true the argument might be.

Everything ranked above earns its place by a different test: each item is either demonstrably wrong
on its own terms (the full-batch assumption, the cache measurement) or silent about a failure mode
that a plausible, ordinary event would trigger (a rollout that removes half the fleet, a server that
crashes mid-batch) — not a hypothetical edge case or a stylistic preference. Ranking by the
consequence of being wrong, rather than by how many issues can be listed, is what keeps a ten-minute
read from turning into a pass over every sentence: items 1 and 2 alone double the required server
count and would be caught by the first load test after launch; items 6 and 7 matter, but only once
something has already gone wrong.

### Follow-ups

- A longer max-wait would raise achieved fill further, but 30 ms was already chosen so the timer
  rarely binds once there is any backlog — the remaining fill gap comes from headroom, not from a
  short timer, so lengthening it would add latency for little throughput gain.
- Not every workload makes a response cache worthless — item 2's argument is about *this* cache
  holding *this* traffic; genuinely short-lived repeats (a client retrying an identical submission
  within seconds) would show up directly in a bounded-cache replay, the measurement item 2 asks for
  instead of the one the document ran.
- An idempotency key two genuinely different requests reuse by mistake, rather than a legitimate
  retry, is indistinguishable from a retry at the store unless the stored record is checked against
  the new request's content, not just its key; hashing the request body into the stored record and
  rejecting a mismatch catches it.
- A connection that starts light and turns heavy later is invisible to round robin at creation time
  either way; fixing item 5 by routing every request independently, rather than only fixing where new
  connections land, closes this case along with the one the simulation shows.
- Once bulk traffic is split from interactive (item 3), its queue has no per-request latency target at
  all and could use a longer max-wait than 30 ms for a higher achieved fill — the same direction the
  corrected estimate already takes by giving bulk traffic fewer, more heavily loaded servers.

<details>
<summary>Estimate check (runnable)</summary>

```python
import heapq
import math
import random
import statistics as stats

# ---- recompute the document's own numbers ----
peak = 10_000
interactive, bulk = 9_000, 1_000
assert interactive + bulk == peak

B_MAX, T_BATCH, MAX_WAIT = 50, 0.1, 0.03
per_server_full = B_MAX / T_BATCH
assert per_server_full == 500.0

cache_hit_claimed = 0.35
effective_doc = peak * (1 - cache_hit_claimed)
assert effective_doc == 6_500.0

raw_doc = effective_doc / per_server_full
assert raw_doc == 13.0
provisioned_doc = 13 + 2
assert provisioned_doc == 15

# ---- the cache offset is not a hit rate ----
cache_capacity = 250_000
turnover_s = cache_capacity / effective_doc
assert round(turnover_s) == 38
week_s = 7 * 24 * 3600
assert week_s == 604_800
window_ratio = turnover_s / week_s
corrected_hit = cache_hit_claimed * window_ratio
assert corrected_hit < 0.0001                           # under 0.01%, not 35%

# ---- corrected capacity estimate ----
# servers = ceil(QPS * batch_time / (achieved_batch_size * slots)), plus headroom
target_fill = 0.80
achieved_batch = target_fill * B_MAX
assert achieved_batch == 40.0
slots = 1                                                # one run_batch call in flight per server
raw = math.ceil(peak * T_BATCH / (achieved_batch * slots))
assert raw == 25
headroom = 1.20
provisioned = math.ceil(raw * headroom)
assert provisioned == 30

realized_rate = peak / provisioned
rho = realized_rate * T_BATCH / B_MAX
assert round(realized_rate, 1) == 333.3
assert round(rho, 4) == 0.6667
assert round(rho * B_MAX, 1) == 33.3

under_provisioning_factor = provisioned / provisioned_doc
assert under_provisioning_factor == 2.0

# splitting the pool by traffic class (interactive vs. bulk) costs a little pooling efficiency
interactive_raw = math.ceil(interactive * T_BATCH / achieved_batch)
bulk_raw = math.ceil(bulk * T_BATCH / achieved_batch)
assert (interactive_raw, bulk_raw) == (23, 3)
interactive_prov = math.ceil(interactive_raw * headroom)
bulk_prov = math.ceil(bulk_raw * headroom)
assert (interactive_prov, bulk_prov) == (28, 4)
split_overhead = (interactive_prov + bulk_prov) - provisioned
assert split_overhead == 2

# ---- retries without an idempotency key ----
retry_rate_under_degradation = 0.05
duplicate_load = peak * retry_rate_under_degradation
assert duplicate_load == 500.0
assert duplicate_load / per_server_full == 1.0           # exactly one server's worth of pure waste

# ---- no admission control: a capacity dip backs up without bound ----
half_fleet = provisioned // 2
assert half_fleet == 15
degraded_capacity = half_fleet * per_server_full
deficit = peak - degraded_capacity
incident_s = 120
backlog = deficit * incident_s
assert (degraded_capacity, deficit, backlog) == (7_500.0, 2_500.0, 300_000.0)
recovered_capacity = provisioned * per_server_full
surplus = recovered_capacity - peak
drain_s = backlog / surplus
assert (recovered_capacity, surplus, drain_s) == (15_000.0, 5_000.0, 60.0)

# ---- a server dying mid-batch ----
mtbf_hours_per_server = 4
fleet_mtbf_minutes = (mtbf_hours_per_server * 60) / provisioned
assert fleet_mtbf_minutes == 8.0                          # a crash every ~8 minutes, fleet-wide
assert B_MAX == 50                                        # requests silently lost per crash

print("all quoted numbers in the review check out")


# ---- load balancing: connection-sticky round robin vs. backlog-aware routing ----
def simulate_pool(n_servers: int, streams: list[tuple[float, int | None]], duration: float,
                   seed: int, warmup: float = 5.0) -> dict:
    """streams: (rate, fixed_server) pairs. fixed_server=None routes each arrival independently by
    power of two choices on current backlog; otherwise every arrival from that stream goes to the
    fixed server, modelling a connection pinned to one server for its lifetime."""
    rng = random.Random(seed)
    arrivals: list[tuple[float, int | None]] = []
    for rate, fixed in streams:
        t = 0.0
        while t < duration:
            t += rng.expovariate(rate)
            if t < duration:
                arrivals.append((t, fixed))
    arrivals.sort(key=lambda x: x[0])
    finish = [None] * len(arrivals)

    events, seq = [], 0

    def push(time_, kind, data=None):
        nonlocal seq
        heapq.heappush(events, (time_, seq, kind, data))
        seq += 1

    servers = [dict(pending=[], gen=0, expired=False, ready=[], busy=False) for _ in range(n_servers)]

    def backlog(s):
        srv = servers[s]
        return len(srv["pending"]) + sum(len(b) for b in srv["ready"]) + (B_MAX if srv["busy"] else 0)

    def choose_server(fixed):
        if fixed is not None:
            return fixed
        i, j = rng.randrange(n_servers), rng.randrange(n_servers)
        return i if backlog(i) <= backlog(j) else j

    def close(s):
        srv = servers[s]
        srv["ready"].append(list(srv["pending"]))
        srv["pending"] = []
        srv["gen"] += 1
        srv["expired"] = False

    def dispatch(s, now):
        srv = servers[s]
        if not srv["busy"] and srv["ready"]:
            batch = srv["ready"].pop(0)
            srv["busy"] = True
            push(now + T_BATCH, "FREE", (s, batch))

    for i, (at, _) in enumerate(arrivals):
        push(at, "ARRIVE", i)

    by_time_server: dict[tuple[int, float], list[int]] = {}
    while events:
        now, _, kind, data = heapq.heappop(events)
        if kind == "ARRIVE":
            i = data
            at, fixed = arrivals[i]
            s = choose_server(fixed)
            by_time_server.setdefault((s, at), []).append(i)
            srv = servers[s]
            if not srv["pending"]:
                push(at + MAX_WAIT, "TIMER", (s, srv["gen"]))
            srv["pending"].append(at)
            if len(srv["pending"]) >= B_MAX:
                close(s)
                dispatch(s, now)
        elif kind == "TIMER":
            s, gen = data
            srv = servers[s]
            if srv["pending"] and gen == srv["gen"]:
                if not srv["busy"]:
                    close(s)
                    dispatch(s, now)
                else:
                    srv["expired"] = True
        elif kind == "FREE":
            s, batch = data
            srv = servers[s]
            for at in batch:
                for i in by_time_server[(s, at)]:
                    if finish[i] is None:
                        finish[i] = now
                        break
            srv["busy"] = False
            if srv["ready"]:
                dispatch(s, now)
            elif srv["pending"] and srv["expired"]:
                close(s)
                dispatch(s, now)

    waits = [f - a for (a, _), f in zip(arrivals, finish) if f is not None and a >= warmup]
    return dict(
        n_completed=sum(1 for f in finish if f is not None), n_total=len(arrivals),
        p50=stats.median(waits), p99=stats.quantiles(waits, n=100)[98], max=max(waits),
    )


n_servers = 10
n_heavy, n_light = 3, 17
heavy_rate = 1_400 / n_heavy                              # about 467/s each
light_rate = 2_100 / n_light                              # about 124/s each
assert round(heavy_rate) == 467 and round(light_rate) == 124
overloaded_server_rate = heavy_rate + light_rate           # a heavy connection plus the one light
assert round(overloaded_server_rate) == 590                # connection that also lands on server 0/1/2
assert overloaded_server_rate > per_server_full             # over the 500/s ceiling on its own

sticky_streams = [(heavy_rate if c < n_heavy else light_rate, c % n_servers) for c in range(n_heavy + n_light)]
aware_streams = [(heavy_rate if c < n_heavy else light_rate, None) for c in range(n_heavy + n_light)]

by_duration = {}
for duration in (30, 80):
    sticky = simulate_pool(n_servers, sticky_streams, duration, seed=7)
    aware = simulate_pool(n_servers, aware_streams, duration, seed=7)
    assert sticky["n_completed"] == sticky["n_total"] and aware["n_completed"] == aware["n_total"]
    by_duration[duration] = (sticky, aware)
sticky_30, aware_30 = by_duration[30]
sticky_80, aware_80 = by_duration[80]

assert sticky_30["p99"] < 6.0 and sticky_80["p99"] > 14.0     # sticky's tail keeps growing with run length
assert sticky_80["p99"] > 2.2 * sticky_30["p99"]               # roughly linear backlog growth: not converging
assert aware_30["p99"] < 0.21 and aware_80["p99"] < 0.21        # backlog-aware routing stays flat
assert aware_80["max"] < 0.5                                   # bounded by wait-plus-execution, not growing

print("load-balancing simulation confirms the connection-sticky overload and its fix")


# ---- the shared 8-GPU pool: the document's policy vs. atomic reservation ----
class SharedPool:
    """Small- and large-model batches sharing a pool of n_gpus GPUs. A small batch needs one GPU; a
    large batch needs all n_gpus at once. Both take batch_time once dispatched.

    policy="opportunistic" is the document's appendix: a large batch launches only at the instant
    every GPU happens to be simultaneously free, and nothing ever holds a GPU back to make that more
    likely. policy="reserve" adds an aging-triggered drain: once a formed large batch has waited
    `aging` with the pool not entirely free, no *new* small batch is admitted (already-running ones
    finish normally) until all n_gpus are free, which are then reserved atomically for the large batch.
    """

    def __init__(self, rate_small, rate_large, batch_small, batch_large, wait_small, wait_large,
                 batch_time, n_gpus, policy, aging, duration, seed):
        self.batch_time, self.n_gpus = batch_time, n_gpus
        self.policy, self.aging, self.duration = policy, aging, duration
        self.batch_small, self.batch_large = batch_small, batch_large
        self.wait_small, self.wait_large = wait_small, wait_large
        rng = random.Random(seed)

        def poisson(rate):
            out, t = [], 0.0
            while t < duration:
                t += rng.expovariate(rate)
                if t < duration:
                    out.append(t)
            return out

        self.arr_s, self.arr_l = poisson(rate_small), poisson(rate_large)
        self.fin_s, self.fin_l = [None] * len(self.arr_s), [None] * len(self.arr_l)
        self.by_s, self.by_l = {}, {}
        for i, at in enumerate(self.arr_s):
            self.by_s.setdefault(at, []).append(i)
        for i, at in enumerate(self.arr_l):
            self.by_l.setdefault(at, []).append(i)

        self.events, self.seq = [], 0
        for i, at in enumerate(self.arr_s):
            self._push(at, "ARR_S", i)
        for i, at in enumerate(self.arr_l):
            self._push(at, "ARR_L", i)

        self.pend_s, self.gen_s, self.expired_s, self.ready_s = [], 0, False, []
        self.pend_l, self.gen_l, self.ready_l = [], 0, []        # ready_l holds (ready_at, batch)
        self.free = n_gpus
        self.draining = False
        self.fills_s, self.fills_l = [], []
        self.drain_idle_seconds = 0.0
        self._track_t = self._track_free = None

    def _push(self, t, kind, data=None):
        heapq.heappush(self.events, (t, self.seq, kind, data))
        self.seq += 1

    def _close_s(self):
        self.ready_s.append(list(self.pend_s))
        self.pend_s, self.gen_s, self.expired_s = [], self.gen_s + 1, False

    def _close_l(self, now):
        self.ready_l.append((now, list(self.pend_l)))
        self.pend_l, self.gen_l = [], self.gen_l + 1

    def _dispatch_s(self, now):
        while not self.draining and self.free > 0 and self.ready_s:
            batch = self.ready_s.pop(0)
            self.free -= 1
            self.fills_s.append(len(batch))
            self._push(now + self.batch_time, "FREE_S", batch)

    def _track_idle(self, now):
        if self._track_t is not None:
            self.drain_idle_seconds += self._track_free * (now - self._track_t)
        self._track_t, self._track_free = now, self.free

    def _try_launch_large(self, now):
        if self.free == self.n_gpus and self.ready_l:
            _, batch = self.ready_l.pop(0)
            self.fills_l.append(len(batch))
            self.free = 0                                          # occupies every GPU until FREE_L
            self._push(now + self.batch_time, "FREE_L", batch)
            self.draining, self._track_t = False, None
            return True
        return False

    def _evaluate_large(self, now):
        if self.policy == "opportunistic":
            self._try_launch_large(now)
            return
        if self.draining:
            self._track_idle(now)
            self._try_launch_large(now)
            return
        if not self.ready_l or self._try_launch_large(now):
            return
        ready_at, _ = self.ready_l[0]
        if now - ready_at >= self.aging:
            self.draining, self._track_t, self._track_free = True, now, self.free

    def _resume_small(self, now):
        if not self.draining:
            if self.pend_s and self.expired_s and self.free > 0:
                self._close_s()
            self._dispatch_s(now)

    def run(self) -> "SharedPool":
        while self.events:
            now, _, kind, data = heapq.heappop(self.events)
            if kind == "ARR_S":
                at = self.arr_s[data]
                if not self.pend_s:
                    self._push(at + self.wait_small, "TIMER_S", self.gen_s)
                self.pend_s.append(at)
                if len(self.pend_s) >= self.batch_small:
                    self._close_s()
                    self._dispatch_s(now)
            elif kind == "TIMER_S":
                if self.pend_s and data == self.gen_s:
                    if not self.draining and self.free > 0:
                        self._close_s()
                        self._dispatch_s(now)
                    else:
                        self.expired_s = True
            elif kind == "ARR_L":
                at = self.arr_l[data]
                if not self.pend_l:
                    self._push(at + self.wait_large, "TIMER_L", self.gen_l)
                self.pend_l.append(at)
                if len(self.pend_l) >= self.batch_large:
                    self._close_l(now)
                    self._evaluate_large(now)
            elif kind == "TIMER_L":
                if self.pend_l and data == self.gen_l:
                    self._close_l(now)
                    self._evaluate_large(now)
            elif kind == "FREE_S":
                for at in data:
                    for i in self.by_s[at]:
                        if self.fin_s[i] is None:
                            self.fin_s[i] = now
                            break
                self.free += 1
                self._evaluate_large(now)
                self._resume_small(now)
            elif kind == "FREE_L":
                for at in data:
                    for i in self.by_l[at]:
                        if self.fin_l[i] is None:
                            self.fin_l[i] = now
                            break
                self.free = self.n_gpus
                self._evaluate_large(now)
                self._resume_small(now)
        return self

    def stats(self, warmup: float = 5.0) -> dict:
        def waits(arr, fin):
            return [f - a for a, f in zip(arr, fin) if f is not None and a >= warmup]
        w_s, w_l = waits(self.arr_s, self.fin_s), waits(self.arr_l, self.fin_l)
        gpu_seconds = self.n_gpus * (self.duration - warmup)
        return dict(
            n_s=len(self.arr_s), n_l=len(self.arr_l),
            done_s=sum(1 for f in self.fin_s if f is not None),
            done_l=sum(1 for f in self.fin_l if f is not None),
            p99_s=stats.quantiles(w_s, n=100)[98], max_s=max(w_s),
            p99_l=stats.quantiles(w_l, n=100)[98], max_l=max(w_l),
            large_gpu_pct=100 * len(self.fills_l) * self.batch_time * self.n_gpus / gpu_seconds,
            idle_drain_pct=100 * self.drain_idle_seconds / gpu_seconds,
        )


gpu_common = dict(rate_small=500, rate_large=1, batch_small=8, batch_large=2,
                   wait_small=0.02, wait_large=0.5, batch_time=0.1, n_gpus=8, duration=200, seed=11)

doc_policy = SharedPool(policy="opportunistic", aging=None, **gpu_common).run().stats()
fixed_policy = SharedPool(policy="reserve", aging=0.15, **gpu_common).run().stats()
slower_drain = SharedPool(policy="reserve", aging=1.0, **gpu_common).run().stats()

for r in (doc_policy, fixed_policy, slower_drain):
    assert r["done_s"] == r["n_s"] and r["done_l"] == r["n_l"]      # every request eventually completes

assert doc_policy["p99_l"] > 190 and doc_policy["max_l"] > 190        # nearly the whole 200 s run:
assert doc_policy["max_s"] < 0.2                                      # starvation, while small is fine

assert fixed_policy["p99_l"] < 1.0 and fixed_policy["max_l"] < 1.0    # bounded to aging + ~one batch cycle
assert fixed_policy["max_s"] < 1.0                                    # small's own wait stays sub-second

assert abs(fixed_policy["large_gpu_pct"] - doc_policy["large_gpu_pct"]) < 0.05     # the reservation cost
                                                                                    # itself is policy-independent
assert slower_drain["idle_drain_pct"] < fixed_policy["idle_drain_pct"]     # longer aging: fewer, fuller drains
assert slower_drain["p99_l"] > 1.5 * fixed_policy["p99_l"]                 # ... at a latency cost

print("8-GPU pool simulation: the document's policy starves the large model; the fix bounds both")
print("all estimation numbers check out")
```

</details>

</details>
