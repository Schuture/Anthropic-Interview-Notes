# 设计文档评审：批处理推理服务

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 设计评审 | ★★★★☆ | 困难 | SWE · MLE · EM | design-review, batching, load-balancing, capacity-planning, gpu-scheduling | 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

下面这份文档针对一个由 GPU 模型服务器组成的池，提出了一套批处理与负载均衡层。每台服务器加载一个模型，并以*批次*的形式执行请求：一个批次把若干个请求合并成对模型的一次调用，这样 GPU 就只做一轮计算，而不是分别做许多轮。服务器的批次调用接口有一份固定的约定，这份评审不会重新讨论它——它属于另一个团队：

```py
def run_batch(requests: list[Request]) -> list[Response]:
    """Runs one inference batch on this server and blocks until it returns."""
```

- `1 <= len(requests) <= 50`；调用会为每个请求恰好返回一个响应，顺序与请求一致。
- 无论 50 个槽位中实际填了 1 个还是 50 个，这次调用都固定耗时 **100 ms**。
- 一台服务器同一时刻只接受一次 `run_batch` 调用；第二次调用会阻塞，直到第一次返回为止。
- 模型本身在这次调用背后是一个黑盒：这份评审关心的是把请求排队、组批、路由给它的系统，而不是 `run_batch` 内部发生的任何事情。
- 连续批处理或 token 级批处理，以及对这份约定的任何其他改动，都不在本次评审范围内。

用大约十分钟读完这份文档，然后完成文档下方的任务。文档里既有它提出的方案中的问题，也有它从未触及、因而被遗漏的问题，二者的严重程度各不相同；文档中至少有两处，设计已经被正确地考虑过——把这两类情况都当作评审的一部分。

> **推理服务层的批处理与负载均衡**
> **状态：草稿，已分发评审**
>
> **背景**。客户端通过 HTTP 发送推理请求，每个请求指定一个模型并携带其输入。本设计背后是一个由同构服务器组成的池，服务单个模型。大部分流量属于*交互式*（interactive）——有真人在等待响应——另有一小部分属于*批量*（bulk）：由程序批量提交，数量较大，没有人在等待其中任何一个请求各自的响应。峰值时该服务每秒收到 **10,000** 个请求，其中 9,000/s 为交互式请求，1,000/s 为批量请求。
>
> **目标**。峰值时维持每秒 10,000 个请求。通过批处理请求而非逐个运行来保持较高的 GPU 利用率。为客户端提供一种在响应丢失后恢复结果的方式，而不必为已经完成的工作重新付出一次代价。
>
> **非目标**。改动 `run_batch` 接口约定——固定批次大小、固定延迟、每台服务器同一时刻只处理一次调用——这属于运行时团队的职责范围。跨区域故障切换。与模型自身准确率或采样行为有关的任何内容。
>
> **设计方案**。
>
> *批处理*。每台服务器拥有自己的请求队列和一个批处理器。一个正在组装的批次，只要达到 50 个请求，或者自其中第一个请求到达起过了 30 ms，以先到者为准，就会关闭并发往该服务器的 `run_batch` 调用——但如果 30 ms 计时器触发时服务器仍在忙于处理上一个批次，正在组装的批次就继续吸收新到达的请求而不提前关闭，因为提前关闭并不会让它更快地被送到服务器执行。请求一旦被接纳，交互式与批量请求就不再区分：两者位于同一个队列上，按到达顺序一起组批。
>
> *负载均衡*。一个无状态的负载均衡器，通过轮询（round robin）把每个新的客户端连接分配给某一台服务器；此后该连接发送的每个请求，在这条连接的整个生命周期内都被转发到它所分配到的那台服务器的队列。这让负载均衡器本身保持简单——它从不检查任何服务器的队列——并且从构造上保证新连接被均匀地分散到整个机群。
>
> *响应缓存*。响应被缓存在一张固定大小、容量为 250,000 条目的表中（按最近最少使用，即 LRU，淘汰），键是请求输入的精确哈希。回放一份为期一周、包含 3,200,000 条生产请求的样本时，35% 的请求命中了同一份样本中已经出现过的某个条目——因此我们预计该缓存在生产环境中能吸收大约三分之一的流量。
>
> *重试*。如果客户端在 500 ms 内没有收到响应，就应该使用相同的参数重试该请求。
>
> *模型隔离*。一台服务器在同一时刻只固定服务一个模型；路由器会在请求到达任何队列之前，拒绝一个服务器未持有对应模型的请求，因此一个批次里永远不会混入两个不同模型的请求。
>
> *可观测性*。该服务导出请求计数和延迟指标，这些是机群中每个服务都已经在导出的指标。
>
> **容量估算**。扣除 35% 的缓存命中之后，批处理层实际需要承担大约 10,000 × 0.65 = 6,500 个请求/秒。每台服务器每 100 ms 完成一个装满 50 个请求的批次，也就是 500 个请求/秒。6,500 / 500 = 13 台服务器；我们将预置 15 台，留一点余量。
>
> **上线**。把新的批处理与负载均衡层部署在一个功能开关（feature flag）之后，与现有路径并行运行一周，然后把全部流量切换过去。
>
> **附录：大模型与小模型共享的池**。部分部署会从一个 8 卡 GPU 池中同时服务同一模型的两种规格：*大*规格的一个批次必须一次性用满全部 8 块 GPU，*小*规格的一个批次只用一块 GPU，两种规格的一个批次耗时相同，都是 100 ms。我们提出两条队列，每种规格一条，各自按上面的方式组批。当两条队列都有已就绪的批次时，估计到达率更高的那条队列优先派发；当两者的到达率接近时，大模型队列优先，因为它更快能装满一个完整批次。

请给出：

1. 按优先级排列的、这份设计中最重要问题的清单——既包括它做错的地方，也包括它从未触及的地方。每一条都要说明：它针对文档的哪个章节或哪个论断、具体是怎样失败的并附带一个数字支撑、以及一个替代方案及其取舍。略去次要的措辞问题；按后果的严重程度排序，而不是按能列出多少条问题排序。
2. 针对主服务器池的修正后容量估算：峰值流量实际需要多少台服务器，以及文档自身的哪些假设是错的。
3. “可观测性”与“上线”两个部分在上线前还需要补充什么。
4. 追问：对附录中为共享 8 卡 GPU 池提出的派发策略的批评，以及一个修正该策略的方案。

## 参考解答

<details>
<summary>展开参考解答</summary>

评审之前值得先确认一点：面试官是想要先看到完整的排序列表再讨论，还是想先听最重要的风险、其余的视时间而定——这份解答按编号任务的要求给出完整排序。解答还假设，除了固定的 `run_batch` 约定之外，下游的整条流水线都可以修改，因为文档里没有标出任何其他部分是不可更改的。

### 按优先级排序的评审意见

1. **“每台服务器每 100 ms 完成一个装满 50 个请求的批次”（容量估算）**。这句话只有在到达服务器的每一个批次都被完全装满时才成立，而要做到这一点，服务器需要在上一个批次刚结束的那一刻，就总有至少 50 个请求已经在等待——这个估算假设的正是唯一一种排队余量为零的结果，而这恰恰与容量规划应有的假设相反。按照真实部署能达到的填充率——下面修正后的估算用的是 66.7%，而这已经是在加上 20% 余量之后的数字——同样的公式给出的是 30 台服务器，而不是 15 台。所有建立在“每台服务器 500 个请求/秒”之上的其他数字，都继承了这一个错误。*提出的修改：* 按照低于上限的实际达到批次大小来确定机群规模，并在此基础上留出余量，具体做法见下面修正后的估算。

2. **“35% 的请求命中了同一份样本中已经出现过的某个条目”（响应缓存）**。这个数字衡量的是，在一份为期一周、3,200,000 个请求的样本*内部*，一个请求是否出现过重复——也就是最长可以回看一整周。而提议中的缓存在 LRU 淘汰下只有 250,000 个条目；按文档自己给出的 6,500 个请求/秒有效负载计算，一个新条目大约每 250,000 / 6,500 ≈ 38 秒就会淘汰掉最旧的一个。一个要回看长达一周才能找到的匹配，几乎必然早已从一个只记得 38 秒以内内容的缓存里被淘汰出去了——除非重复请求本身就聚集在这 38 秒的窗口以内，而 35% 这个一周尺度的数字并不能说明这一点，文档也没有单独测量过。如果重复请求之间的间隔在这一周里大致均匀分布（考虑到文档没有任何迹象表明存在聚集，这已经是一个宽松的假设），同样的数据就意味着真实命中率大约是 35% × (38 秒 / 604,800 秒) ≈ 0.002%：可以忽略不计，远不是三分之一的流量。*提出的修改：* 用提议规格的、真实存在容量上限并会淘汰条目的缓存回放这一周的数据来测量命中率，而不是检查样本中任意位置是否存在匹配。

3. **“请求一旦被接纳，交互式与批量请求就不再区分”（批处理）**。批量流量由程序提交，可能以远大于任何单个交互式请求所能产生的规模成批到达——一个定时的重新处理作业一次性排入数十万个请求，正是“目标”里说这个系统应当承载的那种批量流量。在一个共享的先进先出队列上，这样一次突发会排在它之后提交的每一个交互式请求前面，而文档里没有任何机制限定这会持续多久。*提出的修改：* 按流量类别拆成两条逻辑队列，各自拥有独立的服务器，或者共享同一个池中各自的份额；把修正后的 30 台服务器池按这种方式拆分，会因为损失一部分池化效率而多付出 2 台服务器（32 台对 30 台，下面给出具体推导）——为了保证批量流量完全不会拖慢交互式流量，这是一个很小的代价。

4. **文档中完全没有提及**。没有任何机制限制可以排队的请求数量，需求超过容量时也没有任何机制卸放负载：每一个到达的请求都会被无限期排队，无论服务器已经落后了多少。修正后 30 台服务器的机群，因一次糟糕的上线而损失一半、持续两分钟——这是“上线”部分完全没有防范的一个现实场景——容量会跌到 7,500 个请求/秒，而需求是 10,000 个/秒；即使机群恢复之后，排空这期间积压的 300,000 个请求，还需要在这 120 秒的事故之外再多花 60 秒。在没有上限的情况下，这些请求最终都会被处理，只是越来越迟，而不是快速卸放一个有限的数量，让其余请求继续满足 SLA。*提出的修改：* 把每条队列“已接纳但未完成”的总数，限制在大约一个批次周期容量的一个小倍数以内；超过这个上限就立即拒绝新请求，并给出重试提示，卸放时优先卸放批量流量而不是交互式流量。

5. **“通过轮询把每个新的客户端连接分配给某一台服务器”（负载均衡）**。轮询均衡的是*连接数*，不是*请求速率*——它没有办法知道一个新连接日后会是一个每秒发送几百个请求的话痨客户端。如果两个这样的连接恰好落到了同一台服务器上——这里没有任何机制阻止这种情况——那台服务器自己承受的请求量就可能超过它 500 个请求/秒的上限，而与此同时整个机群的平均利用率看起来仍然完全健康。下面的仿真在二十个连接里放了三个速率约为 467 个请求/秒的连接，在十台服务器、机群平均利用率健康的 70% 的场景下：最繁忙的那台服务器单独承受了大约 590 个请求/秒，单凭自己就超过了上限，而且它的队列在整次运行中无限增长——运行时长从 30 秒延长到 80 秒时，它的 p99 等待时间从约 5.5 秒涨到约 14.8 秒，而不是趋于稳定。*提出的修改：* 独立地路由每一个请求，而不是把一条连接固定到一台服务器；按当前积压量路由（例如二选一，即 power of two choices：随机抽取两台服务器，发给其中排队较短的一台）还能防范某台服务器单独变慢或正在排空的情况，而无论粒度多细的轮询都看不到这一点。在同一个仿真里，按积压量路由让 p99 稳定在约 0.2 秒，与运行时长无关。

6. **“就应该使用相同的参数重试该请求”（重试）**。没有任何机制能让一次重试找到原来的那次请求；一次慢但最终成功的调用，和一次真正丢失的调用，在客户端看来完全一样，于是两者都会被重试，服务器也就会把请求从头再跑一遍。恰恰是在第 1、2、4 条造成的那种劣化状态下——也正是重试最容易发生的时刻——哪怕只是 5% 这样不算高的重试率，也会把 10,000 个请求/秒里的 500 个变成纯粹的重复劳动：相当于一整台服务器的容量，被用来重新计算已经算出过的答案。*提出的修改：* 由客户端提供一个幂等键（idempotency key），随完成的响应一起保存一段较短的 TTL；一次携带已见过的键的重试，会直接返回保存的结果，而不是重新运行一次请求。

7. **文档中完全没有提及**。没有任何一节说明，当一台服务器在一次 `run_batch` 调用执行到一半时停止响应，调用内那最多 50 个请求会怎样——没有租约（lease），没有心跳（heartbeat），也没有任何检测机制。按每台服务器平均 4 小时发生一次崩溃这个合理的估计，修正后 30 台服务器的机群大约每 8 分钟就会失去一台服务器；每次发生时，多达 50 个正在处理中的请求会被悄无声息地丢弃，没有任何机制察觉、重试或报告这次丢失。*提出的修改：* 为每个已派发的批次配一个租约，并要求服务器发送心跳；一个到期却没有收到心跳的租约，把它对应的请求重新排队到一台新的服务器上，并复用第 6 条中的同一个幂等键，这样客户端已经另行重发过的请求就不会被跑两遍。

### 修正后的容量估算

正确的公式是 servers = ceil(QPS × batch_time / (achieved_batch_size × slots))，再加上余量——而不是文档里的 QPS × batch_time / max_batch_size，后者只是 achieved_batch_size 在每一次派发中都等于上限这一种特殊情形。

**需求**。第 2 条已经说明缓存带来的抵扣并不真实；需求维持在完整的 10,000 个请求/秒。

**达到的批次大小**。文档自己定的 30 ms 最长等待时间，再加上“等到服务器空出来而不是提前关闭已超时的批次”这条规则，本身就足够短：一旦服务器有了任何积压，等待计时器就不再是限制因素——正在组装的批次会一直吸收新到达的请求，直到服务器空出来为止，这直接跟随利用率变化（设计的这一部分不需要修改；见“校准”）。按 80% 的平均填充率来做容量规划，得到 achieved_batch_size = 0.8 × 50 = 40。

**槽位数**。1——按固定约定，一台服务器同一时刻只运行一次 `run_batch` 调用。

**原始值**。ceil(10,000 × 0.1 / (40 × 1)) = ceil(25) = 25。

**余量**。+20%，足以在滚动发布损失几台服务器，或者负载不均匀时，仍不至于突破填充率目标：25 × 1.2 = 30。

**实际结果**。在 30 台服务器的规模下，每台承担 10,000 / 30 ≈ 333.3 个请求/秒，ρ ≈ 66.7%，实际填充率约为 50 中的 33.3——低于用来确定原始值时所用的 80%，这正是余量在起作用：整个容量规划中都是同样的模式，预置的目标是保持安全的那个点，而不是平均值。

**结论**。30 台服务器——是文档给出的 15 台的两倍。

把服务器池拆分成交互式队列和批量队列（第 3 条）时，对每一份份额分别套用同样的公式：交互式（9,000/s）需要 ceil(900/40) = 23 台原始值，×1.2 → 28 台；批量（1,000/s）需要 ceil(100/40) = 3 台原始值，×1.2 → 4 台。28 + 4 = 32，比合并的 30 台多 2 台——这是为了隔离而损失一部分池化效率所付出的代价，在原始值和余量取整这两步都各付出了一点。

### 缺失的章节

**可观测性**。导出跟机群里其他每个服务一样的请求计数和延迟指标，说明不了*这个*服务自己的失效模式。上线前还需要补充：批次填充率的分布，这样一个填充不足的批次能直接被看到，而不是事后才从容量不足反推出来；每台服务器的队列深度，以及距离它上一次关闭批次过了多久，这样第 5 条里的不均衡能被实时看到，而不只是在事后的仿真里才能看到；把延迟拆分成排队时间和执行时间，这样才能区分一个响应慢是因为响应本身慢，还是因为队列过载；针对实际运行中的缓存实时测量出的命中率，而不是第 2 条已经质疑过的离线周样本数字；一旦第 6 条的幂等键存在，还需要重试率和重复完成率；以及第 7 条里批次执行到一半服务器丢失的发生率，无论最终用什么方式检测到它。

**上线**。“与现有路径并行运行一周，然后把全部流量切换过去”，这句话既没有说明“通过”这一周意味着什么标准，也没有给出全部流量切换之前的金丝雀（canary）比例，除了“有人决定停下来”之外也没有任何回滚触发条件。上线前还需要补充：一个从小比例流量开始、只有在明确、可核实的标准被满足时才逐步扩大的金丝雀发布，而不是不论发生什么都固定跑满一周；一个与 SLO 挂钩的回滚触发条件，例如交互式 p99 在某个持续时间窗口内高于目标值，这样回滚就不必依赖有人恰好注意到；在任何真实流量进入新路径之前，按修正后的 10,000 个请求/秒——而不是文档里的 6,500——做一次负载测试；以及在功能开关切换的那一刻，旧路径上仍在处理中的请求应该如何处理，需要有明确规定。

### 共享的 8 卡 GPU 池

附录中的策略比较两条队列各自估计的到达率，当两者接近时优先派发大模型队列。这只是“小批次和大批次同时就绪、且恰好有 GPU 空闲”这一瞬间的平局判定规则；它对其余的时间完全没有说明，而对一种需要团体调度（gang scheduling）——一次性用满全部 8 块 GPU，而不是 8 个各自独立的单卡槽位——的资源来说，其余的时间才是真正要紧的部分。这个策略里没有任何机制把 GPU 预留下来，也没有任何机制阻止小模型队列在一块 GPU 空出来的瞬间就把它占用，因此只要小模型负载持续存在，一块刚释放的 GPU 就会在平局判定规则有机会起作用之前，被下一个就绪的小批次抢走。无论平局判定规则怎么写，全部 8 块 GPU 同时空闲的情况都很少出现，大模型队列因而陷入饥饿（starvation）——不只是等得更久。

下面的仿真直接重现了这一点。在一个仅凭小模型工作就能让整个池维持 90% 以上繁忙的负载下——这并不是一个极端的数字——每一个大模型请求仍然能在这次 200 秒的运行里完成，但它的 p99 等待时间约为 194 秒，只比整次运行短了几秒；与此同时被处理的小请求，等待时间只有 0.12 秒。对任何实际的延迟目标而言，这在实践中就是饥饿，不管在这个精确的负载下队列在形式上是否有界。

**提出的策略：带有界排空的原子式预留**。一个修复方案需要两条文档的策略都没有的保证：一是形成后的大批次最多等多久，二是任何小请求排在大批次后面最多等多久。小批次仍然一就绪就派发到任意一块空闲 GPU 上——常见情形不变。真正改变的是大批次*一旦组成*之后会发生什么：它不再只是被动等待 8 块 GPU 恰好同时凑齐，而是启动一个计时器。一旦这个计时器到了 `aging`、而池仍未完全空闲，调度器就冻结对任何*新*小批次的接纳——已经在跑的批次仍然跑完各自的 100 ms——直到全部 8 块 GPU 同时空闲的那一刻，把它们作为一个整体原子式地预留下来，启动大批次。第一条保证由此直接成立：`aging` 加上至多再一个小批次周期。第二条保证来自这次冻结本身：一旦大批次开始运行，任何已经排队的小请求，等待都不会超过这一次大批次运行，加上冻结开始前它自己队列排空所需的时间。在 `aging` = 150 ms 时，仿真给出大模型的 p99 等待时间约为 0.87 秒，小模型约为 0.33 秒，而文档的策略会让大模型停留在约 194 秒。

**利用率代价**。这个修复方案不是没有代价，但代价的大头本来就无法避免：为一个需要全部 8 块的批次预留全部 8 块 GPU，无论文档的策略还是修复后的策略，都要花掉约 8.1% 的 GPU-秒——两者最终都会把每一个大批次跑完，而预留一个团体调度资源的代价，不会因为前面排队的方式不同而变化。修复方案在此之上还多付出一份代价，就是排空本身：一旦池开始等在跑的小批次收尾，一块提前跑完的 GPU 无事可做，只能等最慢的那一块追上来，因为只有到那时，全部 8 块才能一起交给大批次。`aging` = 150 ms 时，这份闲置时间约占 GPU-秒的 3.1%；把 `aging` 拉长到 1 秒，靠让排空更少、更满，能把它降到约 2.4%，代价是把大模型的 p99 等待时间推高到约 1.7 秒，大致翻倍。`aging` 因此是在大模型的尾延迟和机群利用率之间做取舍的一个旋钮，而不是一个可以调到最优、一劳永逸的设置——无论取什么值都无法把闲置时间压到零，因为那需要提前知道哪个小批次会最后一个跑完，并在它跑完前一刻就把新工作引去别处。

### 校准

文档中有两处不需要任何修改意见，说明这一点，和指出哪里有问题一样，都是评审的一部分。30 ms 的最长等待时间，加上“等到服务器空出来而不是提前关闭已超时的批次”这条规则，正是恰当的做法：一旦服务器有了任何积压，等待计时器就不再是限制因素，达到的批次大小会自行跟随利用率变化，这正是一开始就能按 80% 填充率目标来做容量规划的原因。模型隔离——一台服务器固定服务一个模型，路由器在请求到达任何队列之前就拒绝不匹配的请求——已经堵住了一个批次混入两个不同模型请求的唯一途径；这里没有什么需要补充的。

`run_batch` 这份约定本身——固定批次大小、固定延迟、同一时刻只处理一次调用——出于另一个原因也属于同一类：它由另一个团队固定下来，并且在“非目标”里明确写明，因此一份把时间花在论证应该改用连续批处理上的评审，回答的是一个没有人问过的问题，无论这个论证本身有多正确。

上面排序中的每一条，都是靠另一个标准才能入选的：它要么在自身的论述范围内就能被证明是错的（满批假设、缓存的测量方式），要么是对一个由某个可信、寻常的事件就会触发的失效模式保持沉默（一次拿掉半个机群的上线、一台在批次执行到一半时崩溃的服务器）——而不是一个假设性的边界情形或者措辞上的偏好。按“错了会有什么后果”排序，而不是按“能列出多少条”排序，才能让十分钟的阅读不至于变成逐句地挑毛病：仅第 1、2 两条就会让所需服务器数量翻倍，并且会在上线后的第一次负载测试里就被发现；第 6、7 两条固然重要，但只有在已经出了问题之后才会真正体现出来。

### 追问

- 更长的最长等待时间会进一步提高达到的填充率，但选择 30 ms 本来就是为了让计时器在有任何积压之后很少起作用——剩下的填充率差距来自余量，而不是来自过短的计时器，所以延长它只会增加延迟，换来的吞吐提升很有限。
- 不是所有工作负载都会让响应缓存变得毫无价值——第 2 条的论证针对的是*这个*缓存承载*这种*流量的情形；真正短时间内的重复（比如客户端在几秒内重试同一次提交）会在一个有界缓存的回放中直接体现出来，这正是第 2 条要求的测量方式，而不是文档实际做的那种测量。
- 两个真正不同的请求意外复用了同一个幂等键，与一次合法的重试，在存储层是无法区分的，除非把保存的记录拿去和新请求的内容做比对，而不只是比对键本身；把请求体哈希进保存的记录中，并在不匹配时拒绝，就能发现这种情况。
- 一条一开始流量很轻、之后才变重的连接，无论怎样都不会被创建时刻的轮询看到；把第 5 条的修复方式改成独立路由每一个请求，而不只是修复新连接落到哪里，就能连带解决这种情况和仿真展示的那种情况。
- 一旦批量流量从交互式流量中拆分出来（第 3 条），它的队列就完全没有单个请求级别的延迟目标，可以用比 30 ms 更长的最长等待时间来换取更高的填充率——这与修正后的估算已经采取的方向一致：给批量流量分配数量更少、负载更重的服务器。

<details>
<summary>估算核对（可运行）</summary>

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

