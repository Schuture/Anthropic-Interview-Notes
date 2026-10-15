# 并发网页爬虫

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 并发，Python | ★★★★☆ | 中等 | SWE · MLE · Infra Eng | bfs, threading, asyncio, url-normalization, thread-safety | 3 个部分 / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

一次从 `start_url` 出发的*爬取*（crawl），会借助下面给定的辅助接口，访问所有从它可达、且主机名
（hostname）与 `start_url` 相同的页面：

```py
class HtmlParser:
    def get_urls(self, url: str) -> list[str]:
        """Returns the absolute URLs of every link found on the page at `url`."""
        ...
```

`get_urls` 是 I/O 密集型调用，可能耗时很久才返回；从多个线程同时调用它是安全的。它也可能因为某个页面加载
失败而抛出异常：一次调用抛出异常的页面仍然算作已爬取——毕竟有别的页面链接到它，爬虫也确实尝试抓取过它，
所以它仍然属于结果——但它不会贡献任何新链接，爬取会带着已经发现的其它页面继续进行。`start_url`，以及
`get_urls` 返回的每一个 URL，都是绝对 URL，带有明确的协议（scheme）和主机名；它们都不是相对路径，因此
永远不需要做相对 URL 解析。

去掉*片段*（fragment，即 `#` 及其后面的部分）之后，两个字符串相同的 URL 就规范化为同一个字符串，也就是
同一个页面；规范化除此之外不改变 URL 的任何其它部分。`http://a.com/x#top` 和 `http://a.com/x#bottom`
都规范化为 `http://a.com/x`，算作同一个页面，只抓取一次。当且仅当 `urlparse(link).hostname` 与
`urlparse(start_url).hostname` 相等时（按普通字符串比较——`urlparse` 已经把 `hostname` 转成小写），
一个链接才*属于爬取范围*：子域名是不同的主机（`shop.a.com` 不是 `a.com`），完全无关的域名当然也是；但
链接的协议和端口（port）都不参与这个比较——一个 `https` 链接，或者显式带端口的链接，只要主机名相同，依然
属于爬取范围。不过协议和端口不会被规范化去掉，所以它们仍然影响 URL 的身份：`http://a.com/x` 和
`https://a.com/x` 是两个不同的规范化 URL，都属于爬取范围，各自独立抓取。链接图中可能存在环、同一个链接
在一个页面内或跨多个页面重复出现，以及自链接；每一个规范化后、属于爬取范围的 URL，无论被多少个页面链接、
链接了多少次，都最多只被抓取一次——即最多调用一次 `get_urls`。

爬取函数把它抓取过的每一个规范化 URL，按字典序（升序，即普通的字符串顺序）排成一个列表返回，其中包含
`start_url` 本身。

### Part 1 —— 单线程爬取

```py
def crawl(start_url: str, html_parser: HtmlParser) -> list[str]: ...
```

`crawl` 可以按任意顺序访问页面，只要保证一个页面在链接到它的关系被发现之后才会被抓取（广度优先是最自然
的选择，下面的推演用的也是它）；由于返回结果是排好序的列表，页面被访问的先后顺序不会体现在输出里。

```text
start_url = "http://a.com/index"

http://a.com/index    -> http://a.com/about#team, http://b.com/x,
                          http://a.com/index, https://a.com/index
http://a.com/about    -> http://a.com/index#top, http://a.com/products
https://a.com/index   -> http://a.com/products
http://a.com/products -> http://a.com/products

crawl(start_url, parser):
  visit http://a.com/index                # 起始页面
    -> http://a.com/about#team   规范化为 .../about，  新页面，                   入队
    -> http://b.com/x            主机名 "b.com" != "a.com"，                     跳过
    -> http://a.com/index        已经访问过（一个自链接），                       跳过
    -> https://a.com/index       主机名仍是 "a.com"（协议不参与比较）；
                                  这是一个新的规范化 URL，因为规范化
                                  不会改动协议，                                 入队
  visit http://a.com/about
    -> http://a.com/index#top    规范化为 .../index，已经访问过，                 跳过
    -> http://a.com/products     新页面，                                       入队
  visit https://a.com/index
    -> http://a.com/products     已经访问过（刚刚被上一步入队），                  跳过
  visit http://a.com/products
    -> http://a.com/products     指向自身，已经访问过（单页面的环），             跳过

crawl(start_url, parser) == [
    "http://a.com/about", "http://a.com/index",
    "http://a.com/products", "https://a.com/index",
]
```

### Part 2 —— 多线程并发爬取

实现 `crawl_concurrent`，契约与 `crawl` 相同，但用最多 `max_workers` 个同时运行的线程去重叠 `get_urls`
的延迟，而不是一次只抓取一个页面。

```py
def crawl_concurrent(start_url: str, html_parser: HtmlParser, max_workers: int = 8) -> list[str]: ...
```

无论 `max_workers` 取什么值，返回的列表都必须与 `crawl` 在同一张图上的结果完全一致，并且 `get_urls`
仍然必须对每个被爬取的 URL 恰好调用一次：并发只能改变答案到达的速度，绝不能改变抓取了哪些页面、或者每个
页面被抓取了几次。在上面的图上，`crawl_concurrent(start_url, parser, max_workers=2)` 依然返回同样的
四个 URL，总共恰好调用四次 `get_urls`，无论这四次调用在线程间如何交错。任何时刻，正在运行的 `get_urls`
调用都不能超过 `max_workers` 个。

### Part 3 —— 有并发上限的 asyncio 爬取

实现 `crawl_async`，一个契约与 `crawl` 相同的 `async def` 协程函数，用 `asyncio` 保证同时在途的
`get_urls` 调用最多有 `concurrency` 个。`get_urls` 本身是一个普通的阻塞函数，不是协程，所以不能直接
`await` 它；必须用某种不会在调用期间阻塞事件循环的方式去驱动它。

```py
async def crawl_async(start_url: str, html_parser: HtmlParser, concurrency: int = 8) -> list[str]: ...
```

无论 `concurrency` 取什么值，`crawl_async` 都必须返回与 `crawl` 在同一张图上完全相同的结果，同样对每
个被爬取的 URL 恰好调用一次 `get_urls`，并且同时运行的调用永远不超过 `concurrency` 个——比如
`concurrency = 2`，且有十个页面被 `start_url` 直接链接，那么这十次 `get_urls` 调用里最多只有两次在
同时运行，无论已经为其中多少个创建了任务。

## 参考解答

<details>
<summary>展开参考解答</summary>

动手之前有两点值得跟面试官确认，因为不同面试官对这个接口的表述并不完全一样：链接的协议或端口是否应该
影响它是否属于爬取范围（这里的答案是不影响——只比较主机名，但协议和端口仍然会让两个 URL 在抓取意义上
是不同的），以及 `get_urls` 是否可能失败（这里的答案是可能——一次抓取失败的页面仍然算作已爬取，只是不
会贡献任何新链接）。

### Part 1

`normalize` 负责去掉片段；`same_host` 比较 `urlparse(...).hostname` 字符串。一次朴素的广度优先搜索
维护一个规范化 URL 的 `visited` 集合（用起始 URL 作为种子），以及一个待抓取 URL 的 FIFO 队列；被抓取
页面返回的每一个链接，只要同时满足属于爬取范围、是新页面，就立刻入队。`get_urls` 抛出的异常只在唯一的
调用处被捕获，所以某一个页面的失败永远不会阻止队列里其余部分继续排空。

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

记 $V$ 为从 `start_url` 可达、互不相同且属于爬取范围的规范化 URL 个数（含 `start_url` 本身），$`L`$ 为
所有 `get_urls` 调用返回的链接总数。`crawl` 恰好调用 `get_urls` $V$ 次，并用哈希集合对每个链接做
$O(1)$ 的处理，所以除了那 $V$ 次 `get_urls` 调用本身耗费的时间之外，总开销是 $O(V + L)$——而那部分时
间，正是 Part 2 要重叠掉的。

### Part 2

队列变成一个 `queue.Queue`，由固定数量（`max_workers` 个）的线程共享，每个线程都跑同一个循环：取出
一个 URL、抓取它、把每一个新发现的链接重新塞回队列。真正需要小心的一步，是把“检查一个 URL 是否是新的，
如果是就据为己有”变成一个不可分割的原子操作：如果两个线程在同一时刻都看到某个尚未访问过的 URL，它们不
能都认为自己就是负责抓取它的那一个，否则 `get_urls` 就会在它身上被调用两次。围绕这一次检查加添加的操作
加一把锁就够了，而且这把锁只需要守护这个很小的集合操作本身，从不需要守护 `get_urls` 调用——所以没有任
何线程会在阻塞于 I/O 的同时还持有这把锁。

终止条件不能是“队列看起来空了就停”：一个暂时空了的队列，仍然可能有一个页面正在处理中、它的链接还没有
被塞回去。`queue.Queue` 用一个内部计数器精确地跟踪这件事——每次 `put` 加一，每次与之匹配的 `task_done`
减一；`Queue.join()` 会阻塞（不轮询、不空转），直到这个计数器归零——那一刻恰好意味着，每一个曾经入队
的 URL 都已经处理完毕，包括它接下来又入队的那些。

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

`max_workers` 个线程只启动一次，在整次爬取过程中反复使用，所以无论图最终有多大，线程池的规模都是有界
的；`frontier.join()` 返回，正是“每一个可达页面都已经被抓取”这件事得到证明的时刻，所以关闭用的哨兵值
只有在确凿无疑地没有更多工作时才会被发出，而且每个线程都会在函数返回之前被 join。

### Part 3

`crawl_async` 保持和 Part 2 一样的结构——发现一个页面的链接、筛选并认领其中新的那些、递归地访问它们——
只是用协程而不是线程池来表达。由于 `get_urls` 是一个阻塞调用，它通过 `loop.run_in_executor` 交给一个
`ThreadPoolExecutor` 去运行，并在调用外层加一个 `asyncio.Semaphore` 来限制同时在途的抓取数量；线程池
恰好开 `concurrency` 个线程，这样信号量放行的每一次调用都能立刻有线程可用，不需要再等。这里 `visited`
上的检查加添加操作不需要加锁：asyncio 只会在 `await` 处切换到
另一个协程，而这里的成员检查和紧接着的添加操作之间没有任何 `await`，所以不会有别的协程能够插进来运行——
单线程的事件循环让这一对操作天然就是原子的。

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

这里的终止根本不需要计数器或队列：`crawl_async` 只有在 `await visit(start)` 返回之后才会返回，而
`visit` 只有在它创建的每一个任务都被 `asyncio.gather` 等到返回之后才会返回，对它创建的那些任务，这个
条件又递归地成立——所以最外层的 `await` 能够完成之前，整张可达的图必然已经被访问过。这和 Part 2 里的
`Queue.join()` 本质相同，只不过是由 `await` 的结构、而不是显式计数器构建出来的。

### 追问

- **线程还是进程。** `get_urls` 是 I/O 密集型的：等待网络的时间发生在 Python 解释器之外，所以某次调用
  阻塞时，GIL 对其它线程是空闲的，这正是 Part 2 里的线程能够重叠执行的原因。进程池同样可行，但它额外
  的代价——创建子进程、在进程之间序列化参数和结果——在这里没有任何收益，因为没有需要摊到多个核心上的
  CPU 密集型解析工作；只有当抓取到的页面内容本身需要繁重的、纯 Python 的处理时，进程池才能真正物有
  所值。
- **`asyncio.gather` 与 `as_completed`，以及手工管理的信号量。** `gather`（上面用的就是它）会等待一
  组固定的可等待对象，把它们的结果一起返回，或者在其中任何一个抛出异常时立刻抛出；`as_completed` 按
  完成顺序依次产出结果，适合调用方想先对最快完成的页面做出反应、而不是等整批都完成的场景；像上面那样
  手工管理一个受信号量约束的并发池，则是当工作集合并非一开始就已知、而是在探索过程中不断增长时所需要
  的做法——这正是爬取的情形。
- **robots.txt 与单个 host 的限速。** 真正的爬虫会在第一次请求某个 host 之前先抓取并缓存它的
  `robots.txt`，并且在上面那个全局并发上限之外（而不是取代它）再对每个 host 做请求限速（按主机名分桶
  的令牌桶是常见做法）——否则一次跨多个 host 的爬取仍然可能把其中某一个 host 打垮。
- **超时与重试。** 一次永远不返回的 `get_urls` 调用不应该拖住整个爬取；给它包一层超时，再加上有限次
  数、带退避的重试，就能把“永远挂起”变成“最终被当作一次抓取失败的页面处理”，也就是前面为抛出异常定义
  的同一种结局。
- **多机爬取。** 按主机名的哈希把待抓取队列分片到多台机器上，这样同一个 host 下的每个页面永远由同一
  台机器抓取，每个 host 各自的状态（限速、`robots.txt`）也就永远不需要跨机器共享；把进程内的 `visited`
  集合换成一个共享存储（一个分布式哈希集合，或者一张以规范化 URL 为键的表），每台机器在抓取页面之前
  都对它做原子的检查与更新；如果某台机器认领了一个 URL 却没有汇报完成就消失了，这个 URL 要被重新放回
  队列，这样一台机器崩溃也不会丢失工作。
- **深度或页面数上限。** 用距 `start_url` 的跳数、或者用总页面数的上限来约束爬取，可以防止在一个极
  大或者链接结构病态的网站上无限爬下去；这两种限制都只需要在 Part 1 已经用到的 BFS 分层结构上做一点
  小小的补充。

<details>
<summary>验证代码（可运行）</summary>

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
