# Concurrent Web Crawler

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Coding · concurrency, Python | ★★★★☆ | Medium | SWE · MLE · Infra Eng | bfs, threading, asyncio, url-normalization, thread-safety | 3 parts / 55 min | Technical phone screen · Onsite |
<!-- meta:end -->

## Problem

A *crawl* starting at `start_url` visits every page reachable from it whose hostname matches
`start_url`'s, using a supplied helper:

```py
class HtmlParser:
    def get_urls(self, url: str) -> list[str]:
        """Returns the absolute URLs of every link found on the page at `url`."""
        ...
```

`get_urls` is I/O-bound and may take a long time to return; it is safe to call from several threads at
once. It can also raise an exception, for a page that fails to load: a page whose call raises is still
counted as crawled (something else linked to it, and the crawler did attempt to fetch it, so it belongs
in the result), but it contributes no further links, and the crawl continues with whatever else has
already been discovered. `start_url`, and every URL `get_urls` returns, is absolute, with an explicit
scheme and hostname; none of them is a relative path, so no relative-URL resolution is ever needed.

Two URLs normalise to the same string, and are therefore the same page, once their *fragment* — the `#`
and everything after it — is removed; normalisation changes nothing else about a URL. `http://a.com/x#top`
and `http://a.com/x#bottom` both normalise to `http://a.com/x` and count as one page, fetched once. A link
is *in scope* for the crawl exactly when `urlparse(link).hostname` equals `urlparse(start_url).hostname`,
compared as plain strings (`hostname` is already lower-cased by `urlparse`): a subdomain is a different
host (`shop.a.com` is not `a.com`), and so is an unrelated domain, but a link's scheme and port play no
part in this comparison — an `https` link, or one naming an explicit port, is still in scope as long as
its hostname matches. Scheme and port are not stripped by normalisation, though, so they still affect a
URL's identity: `http://a.com/x` and `https://a.com/x` are two different normalised URLs, both in scope,
fetched independently. The link graph may contain cycles, the same link repeated on one page or across
several, and self-links; each normalised, in-scope URL is fetched — `get_urls` called on it — at most
once, however many pages link to it or however many times.

A crawl function returns every normalised URL it fetched, as a list sorted lexicographically (ascending,
ordinary string order), including `start_url` itself.

### Part 1 — Single-threaded crawl

```py
def crawl(start_url: str, html_parser: HtmlParser) -> list[str]: ...
```

`crawl` may visit pages in any order that only ever fetches a page once a link to it has been discovered
(breadth-first is the natural choice, and the one traced below); since the result is a sorted list, the
order pages are visited in never shows up in the output.

```text
start_url = "http://a.com/index"

http://a.com/index    -> http://a.com/about#team, http://b.com/x,
                          http://a.com/index, https://a.com/index
http://a.com/about    -> http://a.com/index#top, http://a.com/products
https://a.com/index   -> http://a.com/products
http://a.com/products -> http://a.com/products

crawl(start_url, parser):
  visit http://a.com/index                # the start page
    -> http://a.com/about#team   normalises to .../about,   new,               queued
    -> http://b.com/x            hostname "b.com" != "a.com",                  skipped
    -> http://a.com/index        already visited (a self-link),                skipped
    -> https://a.com/index       hostname still "a.com" (scheme is not
                                  compared); a NEW normalised URL, since
                                  normalisation does not touch the scheme,      queued
  visit http://a.com/about
    -> http://a.com/index#top    normalises to .../index, already visited,     skipped
    -> http://a.com/products     new,                                          queued
  visit https://a.com/index
    -> http://a.com/products     already visited (queued just above),         skipped
  visit http://a.com/products
    -> http://a.com/products     itself, already visited (a 1-page cycle),    skipped

crawl(start_url, parser) == [
    "http://a.com/about", "http://a.com/index",
    "http://a.com/products", "https://a.com/index",
]
```

### Part 2 — Concurrent crawl with threads

Implement `crawl_concurrent`, with the same contract as `crawl`, that overlaps the latency of `get_urls`
across up to `max_workers` threads running at once instead of fetching one page at a time.

```py
def crawl_concurrent(start_url: str, html_parser: HtmlParser, max_workers: int = 8) -> list[str]: ...
```

For any `max_workers`, the returned list must be exactly what `crawl` returns for the same graph, and
`get_urls` must still be called exactly once per crawled URL: concurrency is only allowed to change how
fast the answer arrives, never which pages are fetched or how many times each one is. On the graph above,
`crawl_concurrent(start_url, parser, max_workers=2)` still returns the same four URLs and calls `get_urls`
exactly four times in total, however those four calls happen to interleave across threads. At no point may
more than `max_workers` calls to `get_urls` be running at the same time.

### Part 3 — Bounded asyncio crawl

Implement `crawl_async`, an `async def` coroutine function with the same contract as `crawl`, that uses
`asyncio` so that at most `concurrency` calls to `get_urls` are ever in flight together. `get_urls` itself
is an ordinary, blocking function, not a coroutine, so it cannot simply be awaited; it has to be driven in
a way that does not block the event loop for the duration of a call.

```py
async def crawl_async(start_url: str, html_parser: HtmlParser, concurrency: int = 8) -> list[str]: ...
```

For any `concurrency`, `crawl_async` must return exactly what `crawl` returns for the same graph, again
calling `get_urls` exactly once per crawled URL and never running more than `concurrency` calls to it at
once — with `concurrency = 2` and ten pages linked directly from `start_url`, for instance, at most two of
those ten calls to `get_urls` are ever running together, no matter how many of the ten already have a task
created for them.

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two points are worth confirming before coding, since interviewers phrase this interface differently from
one another: whether a link's scheme or port should affect whether it is in scope (here, no — only the
hostname is compared, though scheme and port still make two URLs distinct for fetching purposes), and
whether `get_urls` can fail (here, yes — a page whose fetch fails is still counted as crawled but yields
no further links).

### Part 1

`normalize` strips the fragment; `same_host` compares `urlparse(...).hostname` strings. A plain
breadth-first search keeps a `visited` set of normalised URLs, seeded with the start URL, and a FIFO queue
of URLs still to fetch; every link a fetched page returns is queued the instant it is both in scope and
new. `get_urls` raising is caught around the single call site, so one page's failure never stops the rest
of the queue from draining.

```python
from typing import Protocol
from urllib.parse import urlparse
from collections import deque


class HtmlParser(Protocol):
    def get_urls(self, url: str) -> list[str]: ...


def normalize(url: str) -> str:
    return url.split("#", 1)[0]        # NOTE: split on the FIRST '#' -- a fragment cannot contain one itself


def same_host(url: str, host: str) -> bool:
    return urlparse(url).hostname == host


def crawl(start_url: str, html_parser: HtmlParser) -> list[str]:
    host = urlparse(start_url).hostname
    start = normalize(start_url)
    visited = {start}
    queue = deque([start])
    while queue:
        url = queue.popleft()
        try:
            links = html_parser.get_urls(url)
        except Exception:
            continue            # NOTE: `url` is already in `visited` -- it stays in the result, it just
                                  #  contributes no further links
        for link in links:
            norm = normalize(link)
            if same_host(norm, host) and norm not in visited:
                visited.add(norm)
                queue.append(norm)
    return sorted(visited)
```

Let $V$ be the number of distinct, in-scope normalised URLs reachable from `start_url` (including it) and
$L$ the total number of links returned across every `get_urls` call made. `crawl` calls `get_urls` exactly
$V$ times and does $O(1)$ work per link with the hash set, so $O(V + L)$ beyond the time those $V$ calls to
`get_urls` themselves take — and that time is exactly what Part 2 overlaps.

### Part 2

The queue becomes a `queue.Queue` shared by a fixed pool of `max_workers` threads, each running the same
loop: pull a URL, fetch it, and push every newly-discovered link back onto the queue. The one step that
needs care is turning "check whether a URL is new, and if so claim it" into a single atomic operation: two
threads that both see a not-yet-visited URL at the same instant must not both decide they are the one to
crawl it, or `get_urls` would run on it twice. A lock around the combined check-and-add is enough, and it
only ever guards that tiny set operation, never the `get_urls` call itself, so no thread holds it while
blocked on I/O.

Termination cannot be "stop once the queue looks empty": a queue that is momentarily empty may still have a
page in flight whose links have not been pushed yet. `queue.Queue` tracks exactly this with an internal
counter of unfinished items, incremented by every `put` and decremented by every matching `task_done`;
`Queue.join()` blocks, with no polling or spinning, until that counter reaches zero — which is precisely
the point where every URL ever queued has finished being processed, including whatever it went on to queue
in turn.

```python
import threading
from queue import Queue


def crawl_concurrent(start_url: str, html_parser: HtmlParser, max_workers: int = 8) -> list[str]:
    host = urlparse(start_url).hostname
    start = normalize(start_url)

    visited = {start}
    visited_lock = threading.Lock()
    frontier: Queue = Queue()
    frontier.put(start)

    def worker() -> None:
        while True:
            url = frontier.get()
            if url is None:                     # NOTE: shutdown sentinel -- see below
                frontier.task_done()
                return
            try:
                links = html_parser.get_urls(url)
            except Exception:
                links = []                       # NOTE: `url` stays in `visited`; see Part 1
            for link in links:
                norm = normalize(link)
                if not same_host(norm, host):
                    continue
                with visited_lock:                # NOTE: check-and-add as one atomic step -- whichever
                    if norm in visited:              #  thread wins the race is the only one that queues
                        continue                      #  (and later fetches) this URL
                    visited.add(norm)
                frontier.put(norm)                # NOTE: every child is queued BEFORE task_done() below,
            frontier.task_done()                  #  so the unfinished-item count can never touch zero
                                                    #  while a child of this page is still unaccounted for

    threads = [threading.Thread(target=worker) for _ in range(max_workers)]
    for t in threads:
        t.start()

    frontier.join()              # NOTE: blocks with no busy-waiting until every put() is matched by a
                                   #  task_done() -- correct regardless of how deep or wide the graph is
    for _ in threads:
        frontier.put(None)       # one shutdown sentinel per worker; by now every worker is idle,
    for t in threads:             #  blocked in frontier.get(), so each receives exactly one
        t.join()

    return sorted(visited)
```

`max_workers` threads are started once and reused for the whole crawl, so the pool is bounded regardless of
how large the graph turns out to be; `frontier.join()` returning is what proves every reachable page has
been fetched, so the shutdown sentinels are only sent once there is provably no more work, and every thread
is joined before the function returns.

### Part 3

`crawl_async` keeps the same shape as Part 2 — discover a page's links, filter and claim the new ones,
recurse into them — expressed as a coroutine instead of a thread pool. Since `get_urls` is a blocking call,
it is run through a `ThreadPoolExecutor` via `loop.run_in_executor`, with an `asyncio.Semaphore` acquired
around that call to cap how many fetches are outstanding at once; the executor is sized to exactly
`concurrency` threads so it can run every call the semaphore admits without ever leaving one waiting for a
free thread. The check-and-add on `visited` needs no lock here: asyncio only ever switches to another
coroutine at an `await`, and the membership test and the add that follows it contain none, so no other
coroutine can run between them — the single-threaded event loop makes that pair atomic for free.

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor


async def crawl_async(start_url: str, html_parser: HtmlParser, concurrency: int = 8) -> list[str]:
    host = urlparse(start_url).hostname
    start = normalize(start_url)
    visited = {start}
    semaphore = asyncio.Semaphore(concurrency)
    loop = asyncio.get_running_loop()
    executor = ThreadPoolExecutor(max_workers=concurrency)

    async def fetch(url: str) -> list[str]:
        async with semaphore:                           # NOTE: caps calls actually running, not tasks created
            try:
                return await loop.run_in_executor(executor, html_parser.get_urls, url)
            except Exception:
                return []                                 # NOTE: `url` stays in `visited`; see Part 1

    async def visit(url: str) -> None:
        links = await fetch(url)
        children = []
        for link in links:
            norm = normalize(link)
            if not same_host(norm, host):
                continue
            if norm not in visited:                     # NOTE: no `await` between this check and the add
                visited.add(norm)                          #  below -- see the paragraph above
                children.append(asyncio.create_task(visit(norm)))
        if children:
            await asyncio.gather(*children)

    try:
        await visit(start)
    finally:
        executor.shutdown(wait=True)                     # NOTE: every pool the page opens is shut down
    return sorted(visited)
```

Termination needs no counter or queue at all here: `crawl_async` returns only once `await visit(start)`
does, and `visit` only returns once `asyncio.gather` on every task it created has returned, which in turn
holds for each of those, recursively — the whole reachable graph has necessarily been visited before the
outermost `await` can complete. This mirrors `Queue.join()` in Part 2, just built from the structure of
`await` instead of an explicit counter.

### Follow-ups

- **Threads or processes.** `get_urls` is I/O-bound: the wait for the network happens outside the Python
  interpreter, so the GIL is free for other threads while any one call is blocked, which is exactly what
  lets Part 2's threads overlap. A process pool would still work, but its extra cost — spawning workers,
  pickling arguments and results between processes — buys nothing here, since there is no CPU-bound parsing
  work to spread across cores; it would earn its keep only if the page content itself needed heavy, pure-Python
  processing once fetched.
- **`asyncio.gather` vs. `as_completed` vs. a manual semaphore.** `gather` (used above) waits for a fixed
  set of awaitables and hands back all their results together, or raises as soon as one of them does;
  `as_completed` yields results one at a time as they finish, useful when the caller wants to react to the
  fastest pages first rather than wait for a whole batch; a manually managed semaphore-bounded pool, as
  above, is what is needed once the set of work is not known up front but grows while it is being explored,
  which is exactly the case for a crawl.
- **robots.txt and per-host politeness.** A real crawler fetches and caches each host's `robots.txt` before
  its first request there, and rate-limits requests per host (a token bucket keyed by hostname is a common
  choice) in addition to, not instead of, the global concurrency cap above — otherwise a crawl over many
  hosts can still hammer any one of them.
- **Timeouts and retries.** A call to `get_urls` that never returns should not stall the whole crawl;
  wrapping it with a timeout and a bounded number of retries with backoff turns "hung forever" into
  "eventually treated like a page whose fetch failed," the same outcome already defined above for a raised
  exception.
- **Multi-machine crawling.** Shard the frontier across machines by a hash of the hostname, so that every
  page on a given host is always fetched by the same machine and per-host state (rate limits, `robots.txt`)
  never needs to be shared; replace the in-process `visited` set with a shared store (a distributed hash
  set, or a table keyed by normalised URL) that every machine checks and updates atomically before fetching
  a page, and requeue a URL a worker had claimed but not finished if that worker disappears without
  reporting completion, so a crashed machine loses no work.
- **Depth or page limits.** Bounding the crawl by the number of hops from `start_url`, or by a maximum total
  page count, keeps a crawl over a very large or pathologically linked site from running unbounded; either
  is a small addition to the BFS layer structure already used in Part 1.

<details>
<summary>Checks (runnable)</summary>

```python
import asyncio
import random
import threading
import time


class FakeHtmlParser:
    """An in-memory HtmlParser over a fixed link graph {url: [linked urls]}. Safe to call from several
    threads: `calls` counts how many times each URL was fetched and `peak_in_flight` records the largest
    number of calls that were ever running at once, both updated under a lock; the simulated latency and
    the simulated failures happen outside the lock, so concurrent callers are never serialised by it."""

    def __init__(self, graph, delay=0.0, raise_on=()):
        self._graph = graph
        self._delay = delay
        self._raise_on = set(raise_on)
        self._lock = threading.Lock()
        self.calls = {}
        self.in_flight = 0
        self.peak_in_flight = 0

    def get_urls(self, url):
        with self._lock:
            self.calls[url] = self.calls.get(url, 0) + 1
            self.in_flight += 1
            self.peak_in_flight = max(self.peak_in_flight, self.in_flight)
        try:
            if self._delay:
                time.sleep(self._delay)
            if url in self._raise_on:
                raise RuntimeError(f"simulated fetch failure for {url}")
            return list(self._graph.get(url, []))
        finally:
            with self._lock:
                self.in_flight -= 1


def check_result(parser, result, expected):
    assert result == expected, (result, expected)
    assert set(parser.calls) == set(expected), (parser.calls, expected)
    assert all(count == 1 for count in parser.calls.values()), parser.calls   # exactly once each


# --- the worked example in the Problem section, on all three implementations ---
example_graph = {
    "http://a.com/index": ["http://a.com/about#team", "http://b.com/x",
                            "http://a.com/index", "https://a.com/index"],
    "http://a.com/about": ["http://a.com/index#top", "http://a.com/products"],
    "https://a.com/index": ["http://a.com/products"],
    "http://a.com/products": ["http://a.com/products"],
}
example_expected = ["http://a.com/about", "http://a.com/index",
                     "http://a.com/products", "https://a.com/index"]

parser = FakeHtmlParser(example_graph)
check_result(parser, crawl("http://a.com/index", parser), example_expected)
for max_workers in (1, 2, 8):
    parser = FakeHtmlParser(example_graph)
    check_result(parser, crawl_concurrent("http://a.com/index", parser, max_workers=max_workers),
                 example_expected)
    assert parser.peak_in_flight <= max_workers
for concurrency in (1, 2, 8):
    parser = FakeHtmlParser(example_graph)
    got = asyncio.run(crawl_async("http://a.com/index", parser, concurrency=concurrency))
    check_result(parser, got, example_expected)
    assert parser.peak_in_flight <= concurrency

# --- same hostname, different scheme or port: in scope, but each a distinct normalised URL ---
port_graph = {
    "http://p.com/root": ["http://p.com:8080/admin", "http://p.com/root#frag", "http://p.com/root"],
    "http://p.com:8080/admin": ["http://p.com/root"],
}
port_expected = ["http://p.com/root", "http://p.com:8080/admin"]
for call in [
    lambda p: crawl("http://p.com/root", p),
    lambda p: crawl_concurrent("http://p.com/root", p, max_workers=4),
    lambda p: asyncio.run(crawl_async("http://p.com/root", p, concurrency=4)),
]:
    parser = FakeHtmlParser(port_graph)
    check_result(parser, call(parser), port_expected)

# --- a page whose fetch raises: it is still crawled, but its own links are never learned, so anything
# reachable only through it is never found, while a page reachable another way still is ---
fail_graph = {
    "http://x.com/a": ["http://x.com/b", "http://x.com/c"],
    "http://x.com/b": ["http://x.com/d", "http://x.com/e"],    # b's fetch raises -- d and e are not
    "http://x.com/c": ["http://x.com/d"],                       # learned about through b
    "http://x.com/d": [],
    "http://x.com/e": [],
}
fail_expected = ["http://x.com/a", "http://x.com/b", "http://x.com/c", "http://x.com/d"]
for call in [
    lambda p: crawl("http://x.com/a", p),
    lambda p: crawl_concurrent("http://x.com/a", p, max_workers=4),
    lambda p: asyncio.run(crawl_async("http://x.com/a", p, concurrency=4)),
]:
    parser = FakeHtmlParser(fail_graph, raise_on={"http://x.com/b"})
    check_result(parser, call(parser), fail_expected)   # "e" is reachable only through b's failed fetch


# --- an independent reachability computation, straight from the definition: its own fragment-stripping
# and its own hostname extraction (no urlparse, no same_host/normalize), a stack instead of a queue, and
# no call into crawl / crawl_concurrent / crawl_async ---
def _brute_hostname(url):
    rest = url.split("://", 1)[1]
    for i, ch in enumerate(rest):
        if ch in "/:":
            return rest[:i].lower()
    return rest.lower()


def brute_force_crawl(start_url, graph):
    host = _brute_hostname(start_url)
    start = start_url.split("#", 1)[0]
    seen = {start}
    stack = [start]
    while stack:
        for link in graph.get(stack.pop(), []):
            v = link.split("#", 1)[0]
            if _brute_hostname(v) == host and v not in seen:
                seen.add(v)
                stack.append(v)
    return sorted(seen)


assert brute_force_crawl("http://a.com/index", example_graph) == example_expected
assert brute_force_crawl("http://p.com/root", port_graph) == port_expected


def random_graph(rng, n_hosts=3, pages_per_host=5):
    hosts = [f"h{i}.test" for i in range(n_hosts)]
    pages = [f"http://{h}/p{i}" for h in hosts for i in range(pages_per_host)]
    graph = {}
    for u in pages:
        links = [rng.choice(pages) for _ in range(rng.randint(0, 4))]
        links = [v + (f"#f{rng.randint(0, 2)}" if rng.random() < 0.3 else "") for v in links]
        if rng.random() < 0.2:
            links.append(u)               # an explicit self-link, on top of whatever else was picked
        graph[u] = links
    return rng.choice(pages), graph


# many small random graphs against the brute force: cycles, duplicates, self-links and cross-host links
# arise naturally from the random choices, and every implementation agrees for a subset of the seeds
for seed in range(150):
    rng = random.Random(seed)
    start, graph = random_graph(rng)
    expected = brute_force_crawl(start, graph)
    assert crawl(start, FakeHtmlParser(graph)) == expected, seed
    if seed % 5 == 0:
        parser = FakeHtmlParser(graph)
        check_result(parser, crawl_concurrent(start, parser, max_workers=rng.choice([1, 2, 4])), expected)
        parser = FakeHtmlParser(graph)
        got = asyncio.run(crawl_async(start, parser, concurrency=rng.choice([1, 2, 4])))
        check_result(parser, got, expected)

# --- the concurrency cap is respected AND actually used (not merely correct-but-accidentally-serial) ---
star_host = "s.test"
star_graph = {f"http://{star_host}/root": [f"http://{star_host}/leaf{i}" for i in range(8)]}
for i in range(8):
    star_graph[f"http://{star_host}/leaf{i}"] = []

parser = FakeHtmlParser(star_graph, delay=0.05)
check_result(parser, crawl_concurrent(f"http://{star_host}/root", parser, max_workers=3),
             sorted(star_graph))
assert 1 < parser.peak_in_flight <= 3, parser.peak_in_flight

parser = FakeHtmlParser(star_graph, delay=0.05)
got = asyncio.run(crawl_async(f"http://{star_host}/root", parser, concurrency=3))
check_result(parser, got, sorted(star_graph))
assert 1 < parser.peak_in_flight <= 3, parser.peak_in_flight

# --- speed-up against simulated latency: what is measured is whether blocked callers let others
# proceed, not any real network's throughput, so a generous margin is used against a loaded machine ---
wide_host = "w.test"
wide_graph = {f"http://{wide_host}/root": [f"http://{wide_host}/leaf{i}" for i in range(32)]}
for i in range(32):
    wide_graph[f"http://{wide_host}/leaf{i}"] = []
wide_start = f"http://{wide_host}/root"
DELAY = 0.03

t0 = time.perf_counter()
crawl(wide_start, FakeHtmlParser(wide_graph, delay=DELAY))
sequential_seconds = time.perf_counter() - t0

t0 = time.perf_counter()
crawl_concurrent(wide_start, FakeHtmlParser(wide_graph, delay=DELAY), max_workers=8)
concurrent_seconds = time.perf_counter() - t0

t0 = time.perf_counter()
asyncio.run(crawl_async(wide_start, FakeHtmlParser(wide_graph, delay=DELAY), concurrency=8))
async_seconds = time.perf_counter() - t0

assert sequential_seconds > 3 * concurrent_seconds, (sequential_seconds, concurrent_seconds)  # measured: ~6x
assert sequential_seconds > 3 * async_seconds, (sequential_seconds, async_seconds)             # measured: ~6x

print("all checks passed")
```

</details>

</details>
