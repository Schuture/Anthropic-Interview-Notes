<div align="center">

# Anthropic Interview Notes

**Practice problems modelled on Anthropic's online assessments and its coding, system design, research and
behavioral interviews, plus a guide to the hiring process.<br>
Full problem statements, worked solutions, and code that CI runs on every push.**

English · [中文](README.zh.md)

[![Check](https://github.com/Schuture/Anthropic-Interview-Notes/actions/workflows/check.yml/badge.svg)](https://github.com/Schuture/Anthropic-Interview-Notes/actions/workflows/check.yml)
![Problems](https://img.shields.io/badge/problems-38-blue)
![Languages](https://img.shields.io/badge/languages-English%20%7C%20%E4%B8%AD%E6%96%87-blue)
[![Text: CC BY-NC 4.0](https://img.shields.io/badge/text-CC%20BY--NC%204.0-lightgrey)](LICENSE)
[![Code: MIT](https://img.shields.io/badge/code-MIT-green)](LICENSE-CODE)
[![GitHub stars](https://img.shields.io/github/stars/Schuture/Anthropic-Interview-Notes?style=social)](https://github.com/Schuture/Anthropic-Interview-Notes/stargazers)

</div>

> [!NOTE]
> **Unofficial.** This project is not affiliated with, endorsed by or sponsored by Anthropic. The problems are
> reconstructed from second-hand accounts of past interviews and written up from scratch: statements, examples,
> solutions and code are all original. Interviews differ between teams and change over time, so treat each page
> as practice on the kind of problem you may meet, not as the exact question. If you believe something here
> should not be public, [open an issue](https://github.com/Schuture/Anthropic-Interview-Notes/issues) and it will be taken down.

## What is in it

| Section | Pages | What they cover |
| --- | ---: | --- |
| [Hiring process guide](PROCESS.md) | 1 | How to get an interview, every stage from the online assessment to the offer, timelines, levels and posted salary ranges, the Fellows Program, and the rules during interviews |
| [Coding](#coding-25) | 25 | Nine online assessments (multi-level "build a small system" problems), practical Python problems with concurrency, I/O and durability follow-ups, AI-assisted pull-request review, agent loops with tool use, research-track screens (attention, GRPO, data batching, configuration, prompting) and performance modelling |
| [System design](#system-design-6) | 6 | Batched LLM inference serving, as a blank-page design and as a design-document review; model-weight distribution; one-to-one chat; a prompt playground; a telemetry pipeline |
| [Behavioral](#behavioral-4) | 4 | Recruiter screen, the culture round on AI safety and values, hiring manager, project deep dive |
| [Take-home](#take-home-3) | 3 | Kernel optimisation on a simulated vector machine, a four-hour exploratory ML study, sample-wise double descent |

What sets the pages apart:

- **Complete statements.** Each problem is written out in full, with definitions, function signatures and worked
  examples, so you can attempt it without guessing what was meant. Most problems come in three to five parts or
  levels that build on each other, the way the round does.
- **Solutions you can check.** Every coding, system-design and take-home solution ends with a collapsed block of
  runnable checks: asserts on the examples and, wherever one can be written, a brute force built independently from
  the statement; for system design, the estimates recomputed in code. CI runs the code of all pages on every push.
- **Pitfalls where they happen.** Instead of a separate list of mistakes, `# NOTE:` comments sit on the line the
  mistake would be made.
- **The process, not only the questions.** The [hiring process guide](PROCESS.md) separates what Anthropic
  publishes (linked) from what candidates report, stage by stage.
- **Study tracks.** The [roadmap](ROADMAP.md) orders the pages for software engineering (SWE / Infra), research
  (RS / RE), performance engineering and the Anthropic Fellows Program.
- **English and Chinese.** Every page exists in both languages with identical code.

## How to use it

### 1. Read the process guide

Start with the [hiring process guide](PROCESS.md): which stages your track has, what each one asks, how long it
takes. The recruiter screen already asks about AI safety, so the behavioral pages are worth starting early.

### 2. Pick a track and a pace

Work down the track for your role in the [roadmap](ROADMAP.md). ★ shows how often a question comes up, from
★★★★★ (again and again) to ★☆☆☆☆ (rarely), and each track is ordered so that the top of the list pays off first.

| Time you have | [SWE / Infra](ROADMAP.md#software-engineering-swe--infra) | [Research (RS / RE)](ROADMAP.md#research-rs--re) | [Performance](ROADMAP.md#performance-engineering) | [Fellows](ROADMAP.md#anthropic-fellows) |
| --- | --- | --- | --- | --- |
| About a week | stages 1–3 | stages 1–2, and the take-home your team uses | stages 1–2 | stage 1 |
| Two to four weeks | stages 1–7 | the whole track | the whole track | the whole track |
| More than that | add stage 8 and the research screens close to your team | add stages 2–3 of the SWE track | add the SWE system-design stages | add the research screens |

### 3. Practise one problem

**Coding.** Read only the **Problem** section. Before writing code, note what you would ask the interviewer
(edge cases, tie-breaking, input sizes), then compare with the first lines of the reference solution, which list
the points worth confirming. Give yourself a time limit and take the parts in order, treating each new part as a
change of requirements: extend your code instead of starting over. Run it on the examples, then open the reference
solution and compare approach, complexity and the `# NOTE:` comments. The checks call the functions named in the
statement, so if you keep the same names and signatures you can usually run them against your own code.

**Online assessments.** Set a 90-minute timer and implement the levels in order without looking ahead to the
solution, as the real assessment unlocks one level at a time. Then replay the examples and run the checks, which
compare random operation sequences against a slow model written from the statement.

**System design.** Give yourself about 55 minutes and work through the prompt out loud or in a document:
requirements, estimates, API and data model, architecture, then two or three deep dives. For the design-document
review, spend ten minutes reading the document and write your review as ranked comments. Then compare with the
reference solution; the estimates are computed in the checks block, so you can change an input and rerun them.

**Behavioral.** Each page lists what the round asks, what each question probes and how a strong answer is
structured, and ends with an outline to fill in with your own stories. Rehearse them aloud, and keep your
filled-in version in `my/`, which git ignores.

**Take-home.** Read the assignment and plan the time boxes before opening the solution, which works the assignment
at laptop scale and says what the write-up or presentation should contain.

### 4. Read a page

Every page starts with a table (type, priority, difficulty, roles, topics and, where known, format and round)
and has exactly two sections:

| Section | Coding | System design | Behavioral | Take-home |
| --- | --- | --- | --- | --- |
| **Problem** | definitions, then one block per part or level: task, signature, example | the system, its users, the numbers given, the scope | the round and its questions | the assignment, its rules and deliverables |
| **Reference solution** (collapsed) | per part: idea, derivation, code, with pitfalls as `# NOTE:` comments; then follow-ups and the checks | requirements, data model and API, architecture, deep dives, follow-ups, estimate check | what is probed, how to structure the answer, an outline to fill in | the plan, the code and its results, the interpretation, the write-up |

Roles: **RS** research scientist · **RE** research engineer · **MLE** machine-learning engineer ·
**SWE** software engineer · **Infra Eng** infrastructure engineer · **Perf Eng** performance engineer ·
**Data Eng** data engineer · **EM** engineering manager · **Fellows** Anthropic Fellows Program.

### 5. Run the code

```bash
git clone https://github.com/Schuture/Anthropic-Interview-Notes.git
cd Anthropic-Interview-Notes
pip install -r requirements.txt                              # NumPy, SciPy, scikit-learn, PyTorch, Pillow
python scripts/run_snippets.py coding/file-deduplication     # one page
python scripts/run_snippets.py --all                         # every page
```

A page's ```` ```python ```` blocks run top to bottom as one script; ```` ```py ```` blocks are illustrative
(bare signatures, code that contains planted bugs) and are not executed. CI uses Python 3.11.

## Problems

Sorted by priority within each section. A dash under Difficulty means it has not been rated.

<!-- index:begin -->
### Coding (25)

| # | Problem | Priority | Difficulty | Roles | Topics |
| ---: | --- | --- | --- | --- | --- |
| 1 | [File Deduplication](coding/file-deduplication/README.md) | ★★★★★ | Medium | SWE · MLE · Infra Eng | filesystem, hashing, io, concurrency |
| 2 | [Image Processing Pipeline](coding/image-processing-pipeline/README.md) | ★★★★★ | Medium | SWE · MLE · Infra Eng | image-processing, pillow, multiprocessing, concurrency, caching |
| 3 | [Call Stacks from Profiler Samples](coding/stack-trace-reconstruction/README.md) | ★★★★☆ | Medium | SWE · MLE · Infra Eng | call-stack, longest-common-prefix, streaming, debounce, trie, recursion |
| 4 | [Concurrent Web Crawler](coding/concurrent-web-crawler/README.md) | ★★★★☆ | Medium | SWE · MLE · Infra Eng | bfs, threading, asyncio, url-normalization, thread-safety |
| 5 | [Building an Agent Loop with Tool Use](coding/llm-agent-tool-loop/README.md) | ★★★★☆ | Medium | SWE · RE · RS · MLE | tool-use, agent-loop, llm-api, debugging, prompt-engineering |
| 6 | [Memoising LRU Cache: Bug Fix and Durability](coding/lru-cache-bugfix-durability/README.md) | ★★★★☆ | Medium | SWE · MLE | caching, hashing, persistence, concurrency, serialization |
| 7 | [Distributed Mode, Median and Sort](coding/distributed-mode-median/README.md) | ★★★☆☆ | Medium | SWE · MLE · Infra Eng | distributed-systems, message-passing, bandwidth, hashing, sorting |
| 8 | [Reviewing and Improving Pull Requests with an AI Coding Assistant](coding/agentic-pr-review/README.md) | ★★★☆☆ | Medium | SWE · Infra Eng | code-review, pull-requests, race-condition, deserialization, testing |
| 9 | [Longest-Match Tokenizer](coding/longest-match-tokenizer/README.md) | ★★★☆☆ | Medium | SWE · MLE | greedy, string-processing, trie, tokenization |
| 10 | [Online Assessment: In-Memory Database](coding/oa-in-memory-database/README.md) | ★★★☆☆ | Medium | SWE · MLE · Infra Eng | oop-design, hash-table, ttl, snapshotting |
| 11 | [Online Assessment: Recipe Manager](coding/oa-recipe-manager/README.md) | ★★★☆☆ | Medium | SWE · MLE | oop-design, case-folding, sorting, permissions, versioning |
| 12 | [Online Assessment: Banking System](coding/oa-bank-system/README.md) | ★★★☆☆ | Medium | SWE · Fellows | oop-design, ledger, state-machine, timestamps, merge |
| 13 | [Weighted Data Batcher with Checkpointing](coding/weighted-data-batcher/README.md) | ★★★☆☆ | Hard | MLE · RE · RS | sampling, checkpointing, iterators, data-loading |
| 14 | [Boot Program Simulator: Find and Fix the Loop](coding/bootloader-loop-fix/README.md) | ★★☆☆☆ | Medium | SWE | simulation, cycle-detection, graphs, parsing |
| 15 | [Online Assessment: Task Manager with Expiry and Snapshots](coding/oa-task-manager/README.md) | ★★☆☆☆ | Medium | SWE · MLE | oop-design, hash-table, ttl, time-travel, bisect |
| 16 | [Online Assessment: In-Memory File System](coding/oa-file-system/README.md) | ★★☆☆☆ | Medium | SWE · MLE | filesystem, tree, path-normalization, quota, oop-design |
| 17 | [Online Assessment: DNS Resolver](coding/oa-dns-resolver/README.md) | ★★☆☆☆ | Medium | Fellows | dns, recursion, caching, ttl, concurrency |
| 18 | [Online Assessment: Office Access, Promotions and Pay](coding/oa-worker-grant-management/README.md) | ★★☆☆☆ | Medium | SWE · MLE | oop-design, interval-merge, timestamps, payroll, ranking |
| 19 | [Debugging a GRPO Training Loop](coding/grpo-debug/README.md) | ★★☆☆☆ | Hard | RE · RS · MLE | grpo, ppo-clipping, policy-gradient, advantage-normalization, reinforcement-learning |
| 20 | [Multi-Head Attention with einsum](coding/qkv-attention-einsum/README.md) | ★★☆☆☆ | Hard | MLE · RE · RS | attention, einsum, multi-head, masking, numerical-stability, debugging |
| 21 | [Prompting and Engineering with LLMs: Structured Output and Evaluation](coding/prompting-with-llms/README.md) | ★★☆☆☆ | Medium | RE · RS · MLE | prompt-design, structured-output, parsing, evaluation, error-analysis |
| 22 | [ML Experiment Configuration System](coding/ml-config-system/README.md) | ★★☆☆☆ | Hard | RE · RS · MLE | deep-merge, cycle-detection, cli-overrides, schema-validation, reproducibility |
| 23 | [Performance Modelling: Matmuls, Roofline and Sharding](coding/matmul-performance-modeling/README.md) | ★★☆☆☆ | Hard | Perf Eng · MLE · Infra Eng | roofline, arithmetic-intensity, gpu-performance, parallelism, ffn-sharding |
| 24 | [Online Assessment: Cloud File Storage](coding/oa-cloud-storage/README.md) | ★★☆☆☆ | Medium | SWE · MLE | oop-design, hash-map, storage-quotas, string-matching |
| 25 | [Online Assessment: Completing a Small LLM Inference Engine](coding/oa-inference-engine/README.md) | ★☆☆☆☆ | Hard | Fellows | llm-serving, scheduling, kv-cache, prefix-caching, oop-design |

### System design (6)

| # | Problem | Priority | Difficulty | Roles | Topics |
| ---: | --- | --- | --- | --- | --- |
| 1 | [LLM Inference API with Batched GPU Serving](system-design/inference-api-batching/README.md) | ★★★★★ | Hard | SWE · MLE · Infra Eng | batching, kv-cache, gpu-scheduling, admission-control, capacity-planning |
| 2 | [Design Doc Review: A Batched Inference Service](system-design/design-doc-review/README.md) | ★★★★☆ | Hard | SWE · MLE · EM | design-review, batching, load-balancing, capacity-planning, gpu-scheduling |
| 3 | [Distributing Model Weights to a GPU Fleet](system-design/model-weight-distribution/README.md) | ★★★★☆ | Hard | MLE · Infra Eng · SWE | bandwidth, peer-to-peer, chunking, versioning, readiness |
| 4 | [One-to-One Chat System](system-design/one-on-one-chat/README.md) | ★★★★☆ | Medium | SWE · MLE | websocket, presence, offline-delivery, message-ordering, kafka-vs-redis |
| 5 | [Prompt Playground (Product and Data Design)](system-design/prompt-playground/README.md) | ★★★★☆ | Medium | SWE · MLE | schema-design, versioning, permissions, streaming, concurrency-control, product-design |
| 6 | [Telemetry Pipeline with Schema Evolution](system-design/telemetry-data-infrastructure/README.md) | ★★☆☆☆ | Hard | SWE · Data Eng · Infra Eng | ingestion, schema-evolution, stream-processing, partitioning, access-control, pii |

### Behavioral (4)

| # | Problem | Priority | Difficulty | Roles | Topics |
| ---: | --- | --- | --- | --- | --- |
| 1 | [Culture Round: AI Safety, Values and Judgment](behavioral/culture-ai-safety/README.md) | ★★★★★ | — | All | ai-safety, values, culture, critical-thinking, dilemmas |
| 2 | [Recruiter Screen: Motivation, Mission and Logistics](behavioral/recruiter-screen/README.md) | ★★★★★ | — | All | why-anthropic, ai-safety, mission, background, compensation |
| 3 | [Hiring Manager Round: Impact, Collaboration and Growth](behavioral/hiring-manager/README.md) | ★★★★★ | — | All | impact, ownership, collaboration, cross-functional, mentorship, roadmap-influence |
| 4 | [Project Deep Dive: Presentation and Technical Q&A](behavioral/project-deep-dive/README.md) | ★★★★★ | — | All | presentation, project-deep-dive, trade-offs, metrics, retrospective |

### Take-home (3)

| # | Problem | Priority | Difficulty | Roles | Topics |
| ---: | --- | --- | --- | --- | --- |
| 1 | [Take-Home: Optimising a Kernel on a Simulated Vector Machine](take-home/performance-kernel-optimization/README.md) | ★★☆☆☆ | Hard | Perf Eng | vectorization, loop-unrolling, instruction-scheduling, software-pipelining, simulation |
| 2 | [Take-Home: A Four-Hour Exploratory ML Study](take-home/ml-take-home/README.md) | ★★☆☆☆ | Hard | RE · RS · MLE · Fellows | label-noise, model-capacity, early-stopping, generalization, numpy |
| 3 | [Take-Home: Sample-Wise Double Descent](take-home/interp-double-descent/README.md) | ★☆☆☆☆ | Hard | RE · RS | double-descent, linear-regression, bias-variance, ridge-regression, random-matrix-theory |
<!-- index:end -->

## Contributing

Corrections, new variants and translations are welcome. Found a wrong answer, a missing edge case, an outdated
statement in the process guide or a sentence that does not make sense?
[Open an issue](https://github.com/Schuture/Anthropic-Interview-Notes/issues). For pull requests, see
[CONTRIBUTING.md](CONTRIBUTING.md).

## License

The text (prose, tables and diagrams) is licensed under [CC BY-NC 4.0](LICENSE); the code, both in the pages and
under `scripts/`, under the [MIT License](LICENSE-CODE).
