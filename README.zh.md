<div align="center">

# Anthropic 面试题笔记

**按 Anthropic 在线测评以及编程、系统设计、研究方向和行为面整理的练习题，外加一份招聘与面试流程指南。<br>
完整的题面、详细的参考解答，以及每次推送都由 CI 自动运行的代码。**

[English](README.md) · 中文

[![Check](https://github.com/Schuture/Anthropic-Interview-Notes/actions/workflows/check.yml/badge.svg)](https://github.com/Schuture/Anthropic-Interview-Notes/actions/workflows/check.yml)
![Problems](https://img.shields.io/badge/problems-38-blue)
![Languages](https://img.shields.io/badge/languages-English%20%7C%20%E4%B8%AD%E6%96%87-blue)
[![Text: CC BY-NC 4.0](https://img.shields.io/badge/text-CC%20BY--NC%204.0-lightgrey)](LICENSE)
[![Code: MIT](https://img.shields.io/badge/code-MIT-green)](LICENSE-CODE)
[![GitHub stars](https://img.shields.io/github/stars/Schuture/Anthropic-Interview-Notes?style=social)](https://github.com/Schuture/Anthropic-Interview-Notes/stargazers)

</div>

> [!NOTE]
> **非官方项目。** 本项目与 Anthropic 没有任何关联，未获其认可或赞助。题目根据二手的面试经历重建，题面、示例、解答和代码
> 全部重新撰写。不同团队的面试不一样，题目也会随时间变化，所以请把每一页当作对同类题目的练习，而不是原题。
> 如果你认为某些内容不应公开，请[提一个 issue](https://github.com/Schuture/Anthropic-Interview-Notes/issues)，相关内容会被下架。

## 内容概览

| 分类 | 页数 | 覆盖内容 |
| --- | ---: | --- |
| [招聘与面试流程](PROCESS.zh.md) | 1 | 怎样拿到面试，从在线测评到 offer 的每一轮，时间线，职级与公开的薪资区间，Fellows 项目，以及面试中的规则 |
| [编程](#编程-25) | 25 | 9 道在线测评（多关卡的“搭一个小系统”题），带并发、I/O 和持久化追问的实用 Python 题，借助 AI 助手评审 PR，带工具调用的智能体循环，研究方向的技术初筛（注意力、GRPO、数据批采样、配置系统、提示工程），以及性能建模 |
| [系统设计](#系统设计-6) | 6 | 批处理的 LLM 推理服务（白纸设计与设计文档评审两种形式）、模型权重分发、一对一聊天、提示词 Playground、遥测数据管道 |
| [行为面](#行为面-4) | 4 | HR 初筛、关于 AI 安全与价值观的文化面、用人经理面、项目深挖 |
| [Take-home 作业](#take-home-作业-3) | 3 | 在模拟向量机上优化计算内核、四小时探索性机器学习研究、样本维度的双下降 |

这些页面的特点：

- **完整的题面。** 每道题都写全了定义、函数签名和逐步演算的示例，拿来就能做，不用猜题意。大多数题目分成三到五个
  逐步递进的部分或关卡，和真实面试的节奏一致。
- **可以验证的解答。** 每道编程题、系统设计题和 take-home 题的解答末尾都有一个折叠的、可运行的检查块：针对示例的断言，
  以及只依据题面独立写出的暴力解（能写的都写了）；系统设计题是用代码重算的估算。每次推送，CI 都会运行全部页面的代码。
- **易错点写在出错的地方。** 不单列一张易错清单，而是在最容易写错的那一行加 `# NOTE:` 注释。
- **不只有题目，还有流程。** [招聘与面试流程](PROCESS.zh.md)逐轮区分了 Anthropic 官方公开的信息（附链接）和候选人的反馈。
- **学习路线。** [学习路线](ROADMAP.zh.md)按软件工程（SWE / Infra）、研究方向（RS / RE）、性能工程和 Anthropic Fellows
  四条路线给页面排好了顺序。
- **中英双语。** 每一页都有中英文两个版本，代码完全一致。

## 怎么用

### 1. 先读流程指南

从[招聘与面试流程](PROCESS.zh.md)开始：你的路线有哪几轮，每一轮问什么，要花多长时间。HR 初筛就会问到 AI 安全，
所以行为面的几页值得尽早开始准备。

### 2. 选路线，定节奏

从[学习路线](ROADMAP.zh.md)里和你岗位对应的那条往下做。★ 表示题目出现的频率，从 ★★★★★（反复出现）到 ★☆☆☆☆（少见），
每条路线都按收益排好了序，越靠前越值得先做。

| 可用时间 | [SWE / Infra](ROADMAP.zh.md#软件工程swe--infra) | [研究方向（RS / RE）](ROADMAP.zh.md#研究方向rs--re) | [性能工程](ROADMAP.zh.md#性能工程) | [Fellows](ROADMAP.zh.md#anthropic-fellows) |
| --- | --- | --- | --- | --- |
| 一周左右 | 第 1–3 阶段 | 第 1–2 阶段，加上你所在团队用的 take-home | 第 1–2 阶段 | 第 1 阶段 |
| 两到四周 | 第 1–7 阶段 | 整条路线 | 整条路线 | 整条路线 |
| 更长 | 加上第 8 阶段，以及和你团队相关的研究方向初筛 | 加上 SWE 路线的第 2–3 阶段 | 加上 SWE 路线的系统设计阶段 | 加上研究方向的技术初筛 |

### 3. 练一道题

**编程题。** 先只读**题目**一节。动手写代码之前，记下你会向面试官确认的问题（边界情况、并列时怎么处理、输入规模），
再和参考解答开头的几行对照，那里列出了值得确认的点。给自己设一个时限，按顺序做各个部分，把每个新部分当作一次需求变更：
在已有代码上扩展，而不是推倒重来。先在示例上跑通，再展开参考解答，对照思路、复杂度和 `# NOTE:` 注释。
检查代码调用的是题面里给出的函数名，所以只要保持相同的函数名和签名，一般就能拿它来测你自己的实现。

**在线测评。** 设一个 90 分钟的计时器，按顺序实现各关，不要提前看解答，因为真实测评也是一关一关解锁的。
做完后重放示例并运行检查代码，它会把随机操作序列和一个照题面直接写出的慢速模型逐一对比。

**系统设计题。** 给自己 55 分钟左右，出声讲或在文档里写：需求、估算、API 与数据模型、架构，然后选两三个点深入。
设计文档评审题先用十分钟读文档，再把评审意见按重要性排好写下来。最后和参考解答对照；估算都在检查块里用代码算过，
可以改一个输入再跑一遍。

**行为面。** 每页列出这一轮问什么、每个问题考察什么、好的回答怎样组织，末尾有一份提纲，用你自己的经历填写。
大声练几遍，填好的版本放在 `my/` 目录里（git 会忽略它）。

**Take-home 作业。** 先读题并规划好时间分配，再展开解答；解答在笔记本电脑规模上完整做了一遍作业，
并说明书面报告或演示应该包含什么。

### 4. 一页的结构

每页开头是一张表（题型、优先级、难度、岗位、考点，已知的话还有形式和轮次），正文只有两节：

| 章节 | 编程题 | 系统设计题 | 行为面 | Take-home 作业 |
| --- | --- | --- | --- | --- |
| **题目** | 定义，然后每个部分或关卡一块：任务、函数签名、示例 | 要设计的系统、用户、给定的数字、范围 | 这一轮的形式和问题 | 作业内容、规则和交付物 |
| **参考解答**（折叠） | 按部分写思路、推导、代码，易错点是代码里的 `# NOTE:` 注释；最后是追问和检查代码 | 需求、数据模型与 API、架构、深入话题、追问、估算核对 | 考察点、回答的结构、待填写的提纲 | 计划、代码及其结果、结果解读、报告内容 |

岗位缩写：**RS** 研究科学家 · **RE** 研究工程师 · **MLE** 机器学习工程师 · **SWE** 软件工程师 ·
**Infra Eng** 基础设施工程师 · **Perf Eng** 性能工程师 · **Data Eng** 数据工程师 · **EM** 工程经理 ·
**Fellows** Anthropic Fellows 项目。

### 5. 运行代码

```bash
git clone https://github.com/Schuture/Anthropic-Interview-Notes.git
cd Anthropic-Interview-Notes
pip install -r requirements.txt                              # NumPy、SciPy、scikit-learn、PyTorch、Pillow
python scripts/run_snippets.py coding/file-deduplication     # 一页
python scripts/run_snippets.py --all                         # 全部页面
```

一页里的 ```` ```python ```` 代码块按顺序作为一个脚本运行；```` ```py ```` 代码块只作示意（单独的函数签名、含有预埋 bug 的代码），
不会执行。CI 使用 Python 3.11。

## 题目

每个分类内按优先级排序。难度一栏为 — 表示未评定。

<!-- index:begin -->
### 编程 (25)

| # | 题目 | 优先级 | 难度 | 岗位 | 考点 |
| ---: | --- | --- | --- | --- | --- |
| 1 | [重复文件查找](coding/file-deduplication/README.zh.md) | ★★★★★ | 中等 | SWE · MLE · Infra Eng | filesystem, hashing, io, concurrency |
| 2 | [图像处理流水线](coding/image-processing-pipeline/README.zh.md) | ★★★★★ | 中等 | SWE · MLE · Infra Eng | image-processing, pillow, multiprocessing, concurrency, caching |
| 3 | [从采样还原调用栈事件](coding/stack-trace-reconstruction/README.zh.md) | ★★★★☆ | 中等 | SWE · MLE · Infra Eng | call-stack, longest-common-prefix, streaming, debounce, trie, recursion |
| 4 | [并发网页爬虫](coding/concurrent-web-crawler/README.zh.md) | ★★★★☆ | 中等 | SWE · MLE · Infra Eng | bfs, threading, asyncio, url-normalization, thread-safety |
| 5 | [用工具调用搭建智能体循环](coding/llm-agent-tool-loop/README.zh.md) | ★★★★☆ | 中等 | SWE · RE · RS · MLE | tool-use, agent-loop, llm-api, debugging, prompt-engineering |
| 6 | [LRU 记忆化缓存：修 bug 与持久化](coding/lru-cache-bugfix-durability/README.zh.md) | ★★★★☆ | 中等 | SWE · MLE | caching, hashing, persistence, concurrency, serialization |
| 7 | [分布式众数、中位数与排序](coding/distributed-mode-median/README.zh.md) | ★★★☆☆ | 中等 | SWE · MLE · Infra Eng | distributed-systems, message-passing, bandwidth, hashing, sorting |
| 8 | [借助 AI 编程助手评审并改进 PR](coding/agentic-pr-review/README.zh.md) | ★★★☆☆ | 中等 | SWE · Infra Eng | code-review, pull-requests, race-condition, deserialization, testing |
| 9 | [最长匹配分词器](coding/longest-match-tokenizer/README.zh.md) | ★★★☆☆ | 中等 | SWE · MLE | greedy, string-processing, trie, tokenization |
| 10 | [在线测评：内存数据库](coding/oa-in-memory-database/README.zh.md) | ★★★☆☆ | 中等 | SWE · MLE · Infra Eng | oop-design, hash-table, ttl, snapshotting |
| 11 | [在线测评：菜谱管理系统](coding/oa-recipe-manager/README.zh.md) | ★★★☆☆ | 中等 | SWE · MLE | oop-design, case-folding, sorting, permissions, versioning |
| 12 | [在线测评：银行系统](coding/oa-bank-system/README.zh.md) | ★★★☆☆ | 中等 | SWE · Fellows | oop-design, ledger, state-machine, timestamps, merge |
| 13 | [带断点续跑的加权数据批采样器](coding/weighted-data-batcher/README.zh.md) | ★★★☆☆ | 困难 | MLE · RE · RS | sampling, checkpointing, iterators, data-loading |
| 14 | [启动程序模拟：找到并修复死循环](coding/bootloader-loop-fix/README.zh.md) | ★★☆☆☆ | 中等 | SWE | simulation, cycle-detection, graphs, parsing |
| 15 | [在线测评：带过期与快照的任务管理](coding/oa-task-manager/README.zh.md) | ★★☆☆☆ | 中等 | SWE · MLE | oop-design, hash-table, ttl, time-travel, bisect |
| 16 | [在线测评：内存文件系统](coding/oa-file-system/README.zh.md) | ★★☆☆☆ | 中等 | SWE · MLE | filesystem, tree, path-normalization, quota, oop-design |
| 17 | [在线测评：DNS 解析器](coding/oa-dns-resolver/README.zh.md) | ★★☆☆☆ | 中等 | Fellows | dns, recursion, caching, ttl, concurrency |
| 18 | [在线测评：员工考勤、晋升与薪酬](coding/oa-worker-grant-management/README.zh.md) | ★★☆☆☆ | 中等 | SWE · MLE | oop-design, interval-merge, timestamps, payroll, ranking |
| 19 | [调试 GRPO 训练循环](coding/grpo-debug/README.zh.md) | ★★☆☆☆ | 困难 | RE · RS · MLE | grpo, ppo-clipping, policy-gradient, advantage-normalization, reinforcement-learning |
| 20 | [用 einsum 实现多头注意力](coding/qkv-attention-einsum/README.zh.md) | ★★☆☆☆ | 困难 | MLE · RE · RS | attention, einsum, multi-head, masking, numerical-stability, debugging |
| 21 | [LLM 提示与工程：结构化输出与评测](coding/prompting-with-llms/README.zh.md) | ★★☆☆☆ | 中等 | RE · RS · MLE | prompt-design, structured-output, parsing, evaluation, error-analysis |
| 22 | [机器学习实验配置系统](coding/ml-config-system/README.zh.md) | ★★☆☆☆ | 困难 | RE · RS · MLE | deep-merge, cycle-detection, cli-overrides, schema-validation, reproducibility |
| 23 | [性能建模：矩阵乘法、Roofline 与切分](coding/matmul-performance-modeling/README.zh.md) | ★★☆☆☆ | 困难 | Perf Eng · MLE · Infra Eng | roofline, arithmetic-intensity, gpu-performance, parallelism, ffn-sharding |
| 24 | [在线测评：云存储系统](coding/oa-cloud-storage/README.zh.md) | ★★☆☆☆ | 中等 | SWE · MLE | oop-design, hash-map, storage-quotas, string-matching |
| 25 | [在线测评：补全一个小型 LLM 推理引擎](coding/oa-inference-engine/README.zh.md) | ★☆☆☆☆ | 困难 | Fellows | llm-serving, scheduling, kv-cache, prefix-caching, oop-design |

### 系统设计 (6)

| # | 题目 | 优先级 | 难度 | 岗位 | 考点 |
| ---: | --- | --- | --- | --- | --- |
| 1 | [带批处理的 LLM 推理 API](system-design/inference-api-batching/README.zh.md) | ★★★★★ | 困难 | SWE · MLE · Infra Eng | batching, kv-cache, gpu-scheduling, admission-control, capacity-planning |
| 2 | [设计文档评审：批处理推理服务](system-design/design-doc-review/README.zh.md) | ★★★★☆ | 困难 | SWE · MLE · EM | design-review, batching, load-balancing, capacity-planning, gpu-scheduling |
| 3 | [向 GPU 集群分发模型权重](system-design/model-weight-distribution/README.zh.md) | ★★★★☆ | 困难 | MLE · Infra Eng · SWE | bandwidth, peer-to-peer, chunking, versioning, readiness |
| 4 | [一对一聊天系统](system-design/one-on-one-chat/README.zh.md) | ★★★★☆ | 中等 | SWE · MLE | websocket, presence, offline-delivery, message-ordering, kafka-vs-redis |
| 5 | [提示词 Playground（产品与数据设计）](system-design/prompt-playground/README.zh.md) | ★★★★☆ | 中等 | SWE · MLE | schema-design, versioning, permissions, streaming, concurrency-control, product-design |
| 6 | [带 schema 演进的遥测数据管道](system-design/telemetry-data-infrastructure/README.zh.md) | ★★☆☆☆ | 困难 | SWE · Data Eng · Infra Eng | ingestion, schema-evolution, stream-processing, partitioning, access-control, pii |

### 行为面 (4)

| # | 题目 | 优先级 | 难度 | 岗位 | 考点 |
| ---: | --- | --- | --- | --- | --- |
| 1 | [文化面：AI 安全、价值观与判断](behavioral/culture-ai-safety/README.zh.md) | ★★★★★ | — | 全部 | ai-safety, values, culture, critical-thinking, dilemmas |
| 2 | [HR 初筛：动机、使命与流程事项](behavioral/recruiter-screen/README.zh.md) | ★★★★★ | — | 全部 | why-anthropic, ai-safety, mission, background, compensation |
| 3 | [用人经理面：影响力、协作与成长](behavioral/hiring-manager/README.zh.md) | ★★★★★ | — | 全部 | impact, ownership, collaboration, cross-functional, mentorship, roadmap-influence |
| 4 | [项目深挖：演讲与技术问答](behavioral/project-deep-dive/README.zh.md) | ★★★★★ | — | 全部 | presentation, project-deep-dive, trade-offs, metrics, retrospective |

### Take-home 作业 (3)

| # | 题目 | 优先级 | 难度 | 岗位 | 考点 |
| ---: | --- | --- | --- | --- | --- |
| 1 | [Take-home：在模拟向量机上优化计算内核](take-home/performance-kernel-optimization/README.zh.md) | ★★☆☆☆ | 困难 | Perf Eng | vectorization, loop-unrolling, instruction-scheduling, software-pipelining, simulation |
| 2 | [Take-home：四小时探索性机器学习研究](take-home/ml-take-home/README.zh.md) | ★★☆☆☆ | 困难 | RE · RS · MLE · Fellows | label-noise, model-capacity, early-stopping, generalization, numpy |
| 3 | [Take-home：样本维度的双下降](take-home/interp-double-descent/README.zh.md) | ★☆☆☆☆ | 困难 | RE · RS | double-descent, linear-regression, bias-variance, ridge-regression, random-matrix-theory |
<!-- index:end -->

## 参与贡献

欢迎指正错误、补充变体和翻译。发现了错误的答案、遗漏的边界情况、流程指南里过时的说法，或者读不通的句子？
[提一个 issue](https://github.com/Schuture/Anthropic-Interview-Notes/issues)。提交 pull request 前请先看
[CONTRIBUTING.zh.md](CONTRIBUTING.zh.md)。

## 许可证

正文（文字、表格和图示）采用 [CC BY-NC 4.0](LICENSE) 许可；代码（包括页面中的代码和 `scripts/` 下的脚本）采用
[MIT 许可证](LICENSE-CODE)。
