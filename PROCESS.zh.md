# Anthropic 招聘与面试流程

[English](PROCESS.md) · 中文

> [!NOTE]
> **非官方整理，最后更新于 2026 年 9 月**。本指南综合了两类信息。带来源链接的陈述来自 Anthropic 官方页面
> （招聘官网、职位发布、政策文件、工程博客），核实于 2026-09-22。标题为**候选人反馈**的段落，是对大致在
> 2026 年 4 月到 9 月之间面试过的人所写公开文章的总结；这些内容并非官方信息，也不附来源链接，只出现
> 过一次的说法会特别标出。各团队的实际流程并不完全一致，流程本身也在不断变化：以你的招聘联系人的
> 说明和职位发布页面为准，它们的优先级始终高于本页。

## 概览

| 阶段 | 内容 | 时长 | 适用对象 |
| --- | --- | --- | --- |
| [申请](#获得面试机会) | 在线投递、内部推荐，或由招聘联系人主动联系 | — | 所有人 |
| [在线测评（OA）](#在线测评) | CodeSignal 上一道分多关的“搭建一个小系统”题 | 90 分钟 | 许多软件岗位申请者会遇到，但也常被跳过 |
| [HR 初筛（recruiter screen）](#hr-初筛) | 求职动机、AI 安全、背景经历、流程事项 | 15–30 分钟 | 所有人 |
| [用人经理初筛（HM）](#用人经理初筛) | 过往工作经历，以及和团队的匹配度 | 40–60 分钟 | 部分流程 |
| [技术电面（technical screen）](#技术电面) | 一场编程或系统设计（system design）面试，有时会背靠背安排两场 | 每场 55 分钟 | 大多数流程 |
| [Take-home](#take-home-作业) | 性能工程与研究方向 | 2 小时到 72 小时 | 部分方向 |
| [虚拟现场面（VO）](#虚拟现场面) | 四到五场面试（有一个研究方向的流程有八场），分一到三天进行 | 每场 55 分钟 | 走到这一步的所有人 |
| [背景推荐人与决定](#背景推荐人定级团队匹配与-offer) | 背景推荐人（reference）、招聘委员会、职级（level）、team match（团队匹配）、offer | 几天到几周 | 现场面之后 |

从第一次接触到给出结果：报告中最快的案例大约两周，多数情况下一到两个月，如果中途赶上团队暂停招聘或
公司整体放假，还会更久（详见[时间线](#时间线)）。

## 获得面试机会

### Anthropic 官方看重什么

- “We care about what you can do, not where you learned to do it.”（我们看重你能做什么，而不是你在哪里
  学的。）技术团队里大约一半人此前没有机器学习经验，也有大约一半拥有博士学位，“but plenty of brilliant
  colleagues never went to college.”（但也有不少优秀的同事根本没上过大学。）（[招聘页](https://www.anthropic.com/careers)）
- “If you've done interesting independent research, written a thoughtful blog post, or contributed to open
  source, put that at the top of your resume.”（如果你做过有意思的独立研究、写过认真的博客文章，或者给
  开源项目贡献过代码，把这些放在简历最前面。）（[招聘页](https://www.anthropic.com/careers)）
- 职位发布页面写道：“We encourage you to apply even if you do not believe you meet every single
  qualification.”（即使你觉得自己没有满足每一条要求，我们也鼓励你申请。）官方给出的最低学历要求是学士
  学位，“or an equivalent combination of education, training, and/or experience”（或者具备与之相当的
  教育、培训和/或经验组合）。（[示例招聘启事](https://job-boards.greenhouse.io/anthropic/jobs/5283063008)）

### 在哪里申请、如何申请

所有在招职位都列在[招聘官网](https://www.anthropic.com/careers/jobs)上，可以按团队和办公地点筛选：
截至 2026 年 9 月大约覆盖二十个地点，包括旧金山、纽约、西雅图、波士顿、华盛顿特区、伦敦、都柏林、
苏黎世、巴黎、慕尼黑、东京、首尔、新加坡、悉尼和班加罗尔，此外还有一些允许远程但需要出差的美国岗位。
Anthropic 既没有公布内部推荐计划，也没有公布可申请职位数量的上限。

**候选人反馈：**

- **在线投递是有效的**。2026 年有多个流程都是从一次“裸投”开始的，其中包括一个 Staff 级别的流程，还有
  一个通过了现场面。第一次收到回复的时间从第二天（晚上投的申请，第二天就收到了测评邀请）到两个月后
  都有。
- **内推能加快进度，但不能决定结果**。大多数被内推的候选人几天内就收到了回复，有些甚至跳过了在线测评，
  或者连技术电面都跳过了；但也有一位被内推的候选人没有通过 HR 初筛，另有两位没有通过第一轮技术面试。
- **招聘联系人主动联系**：有好几个流程是这样开始的，既有资深工程师，也有一位研究方向的候选人；还有一位
  研究工程师是在一次 AI 安全相关的活动上认识了人之后被内推的。
- **同时申请多个职位**（有一份报告中超过十个）并不会影响收到回复。有一次某个职位在招聘联系人处理到该
  候选人之前就关闭了，招聘联系人转而给候选人推荐了其他职级的在招职位。
- **测评之后，简历还可能再被筛一次**。有候选人测评满分，之后好几天都没有下文；还有一位候选人在和
  招聘联系人通完电话后得知，团队看了简历没有看上。

### 在申请过程中使用 AI

Anthropic 的官方政策[《招聘过程中如何与 Claude 协作》](https://www.anthropic.com/candidate-ai-guidance)
（2025 年 7 月 10 日更新）写道：

- 申请材料：“Please create your first draft yourself, then use Claude to refine it.”（先自己写出初稿，
  再用 Claude 来润色。）
- Take-home 作业：“Complete these without Claude unless we indicate otherwise.”（除非另有说明，完成时
  不要使用 Claude。）
- 实时面试：“This is all you–no AI assistance unless we indicate otherwise.”（这部分完全靠你
  自己——除非另有说明，不能借助任何 AI。）
- 准备阶段：可以用 Claude 来了解公司、练习回答、准备问题。

不过部分职位发布页面上写的规定更严格——“please do not use AI assistants during the application
process”（申请过程中请不要使用 AI 助手）（[示例](https://job-boards.greenhouse.io/anthropic/jobs/4018206008)）
——所以要以你申请的那个职位发布页面上的说明为准。也有少数环节明确说了可以用：性能工程方向的 take-home
作业明确允许使用 AI 工具；另据候选人转述，AI 辅助编程面试的邀请里也写明可以使用 Claude Code CLI（详见
[面试规则](#面试规则)）。

### 地点、签证与混合办公

- “Currently, we expect all staff to be in one of our offices at least 25% of the time. However, some
  roles may require more time in our offices.”（目前我们要求所有员工至少 25% 的时间在某个办公室办公，
  不过部分岗位可能需要更多的在办公室时间。）（[招聘启事](https://job-boards.greenhouse.io/anthropic/jobs/5283063008)）
- “Most staff are in the Bay Area and come to the office regularly. Some live further away and come in
  for one week a month.”（大多数员工都住在湾区，会定期来办公室；也有一些员工住得比较远，每个月来办公室
  待一周。）（[招聘 FAQ](https://www.anthropic.com/careers)）
- “Yes. We sponsor visas and green cards for eligible roles.”（是的，符合条件的岗位我们会提供签证和
  绿卡担保。）（[招聘 FAQ](https://www.anthropic.com/careers)）职位发布页面还补充说，并非每个岗位和每位
  候选人都能获得担保，“but if we make you an offer, we will make every reasonable effort to get you a
  visa, and we retain an immigration lawyer to help with this.”（但如果我们给你发了 offer，我们会尽
  一切合理努力帮你拿到签证，并且会请移民律师协助处理。）（[招聘启事](https://job-boards.greenhouse.io/anthropic/jobs/5283063008)）
- “All interviews are conducted over Google Meet, and we can accommodate a variety of timezones.”（所有
  面试都通过 Google Meet 进行，我们可以适应不同的时区。）（[招聘页](https://www.anthropic.com/careers)）
  整个流程没有任何一个环节是线下面对面进行的。

### 实习、应届生与 Fellows 项目

- “We don't currently offer internships.”（我们目前不提供实习岗位。）（[招聘 FAQ](https://www.anthropic.com/careers)）
  Anthropic 也没有列出专门的应届生项目；应届毕业生和其他候选人一样申请常规岗位，候选人反馈在软件和
  数据基础设施方向的流程里见过应届生。
- **Anthropic Fellows Program** 是 AI 安全研究方向一个结构化的入门渠道：为期四个月的全职实证研究，由
  Anthropic 的研究员提供指导，每周津贴 3,850 USD / 2,310 GBP / 4,300 CAD，此外每月还有大约 15,000 USD
  的算力经费。不需要博士学位、机器学习经验或论文发表；但需要熟练使用 Python、能全职投入，并且拥有在
  美国、英国或加拿大的工作许可。整个流程分为几个阶段：先是初始申请和背景推荐人核实，然后是技术测评与
  面试，最后是一次研究讨论；每年会开放好几期。Anthropic 表示，第一期 fellows 中有超过 40% 后来全职加入了
  Anthropic；而该项目 AI Safety & Security 方向的一份招聘启事给出的数字是，前几期里有 25–50% 的 fellows
  拿到了全职 offer。
  （[项目公告](https://alignment.anthropic.com/2025/anthropic-fellows-program-2026/)、
  [招聘启事](https://job-boards.greenhouse.io/anthropic/jobs/5183044008)）
- **候选人反馈**：Fellows 的技术测评是有摄像头监考的 CodeSignal 测试，不能用 AI 助手，也不能上网
  搜索：包括一个搭建小系统的测试（有一期是 DNS 解析器，另一期是在一个陌生、文档也不多的代码库里
  补全一个推理服务）和一个单独的调试测试。

### 公开的性能挑战

Anthropic 公开了它[最初的性能工程 take-home 作业](https://github.com/anthropics/original_performance_takehome)，
还配了一篇[工程博客文章](https://www.anthropic.com/engineering/AI-resistant-technical-evaluations)
（2026 年 1 月 21 日），解释了为什么会随着 Claude 模型能力提升而不断重新设计这道题。这个仓库邀请任何
能把内核优化到 1,487 个模拟周期以下——这是 Claude Opus 4.5 发布时的最好成绩——的人给
performance-recruiting@anthropic.com 发邮件，“with your code (and ideally a resume)”（附上你的代码，
最好也附一份简历）。

### 招聘诈骗

“Anthropic recruiters only contact you from @anthropic.com email addresses”（Anthropic 的招聘联系人
只会用 @anthropic.com 邮箱联系你）；在某些情况下，Anthropic 也可能与“vetted recruiting agencies who will
identify themselves as working on behalf of Anthropic”（经过审核、会表明自己代表 Anthropic 的招聘机构）
合作。“Legitimate Anthropic recruiters will never ask for money, fees, or banking information before your
first day.”（真正的 Anthropic 招聘联系人在你入职第一天之前，绝不会向你索要钱款、费用或银行账户信息。）
（[招聘启事](https://job-boards.greenhouse.io/anthropic/jobs/4980430008)）

## 流程各阶段

### 在线测评

**候选人反馈：**

- **形式**。CodeSignal 平台，90 分钟，通过邮件里的链接自行选择时间开始（有一次邀请给了两周的窗口期；
  也有一位候选人收到邀请一个月后才开始，也没出问题）。标准版本不需要开摄像头；有一份报告提到，
  正式计时开始前还有一个练习区。
- **内容**。一道分四关的题目。每一关要等上一关的测试全部通过才会解锁，后面几关都在同一份代码的基础上
  继续扩展。题目是一个个小系统，而不是算法题：内存数据库、支持两步转账和账户合并的银行系统、任务管理器、
  文件系统、菜谱管理系统、带配额的云文件存储、员工考勤与薪酬。代码是否正确、干净、易扩展，比渐进复杂度
  更重要。
- **分数**。部分分也算数：750/1000（最后一关只部分通过）和 800/1000 都进入了下一步，和招聘联系人
  通话。满分并不能保证进入下一轮。
- **时间与语言**。有好几份报告提到，提交之后一天之内就收到了招聘联系人的邮件，其中一次只隔了大约
  两小时。默认语言是 Python；有一个全栈岗位要求用 TypeScript。有一次邀请里提出，如果六个月内在别的
  公司的流程中做过 CodeSignal Industry Coding Assessment，可以直接复用那次的分数。
- **提示信息**。招聘联系人在准备邮件里提示的主题，并不能可靠地预测实际会遇到的题目。
- **会被跳过**：部分被内推的候选人、Staff 级别的岗位、报告中唯一的那个性能工程方向流程，以及
  部分研究岗位的流程里都出现过跳过这一步的情况。

### HR 初筛

**候选人反馈：**

- 通常是 30 分钟（有些报告里是 15–20 分钟），通过电话进行，对方是招聘联系人而不是工程师。
- 不只是聊流程事项。反复出现的问题包括：为什么是 Anthropic 而不是别的实验室；为什么是现在；AI 安全
  对你来说意味着什么；Anthropic 和其他 AI 实验室有什么不同；AI 可能会被怎样滥用；你觉得哪个 AI 相关的
  问题被低估了；你怎么看公司最近的新闻，或者 Responsible Scaling Policy。接下来会聊你的背景经历，以及
  和这个岗位最相关的项目，最后是工作许可、时间安排、是否还在走其他公司的流程，有时还会聊到职级。有一位
  从管理岗转向个人贡献者（IC）岗位的候选人，还被问到此前带过多大的团队、有多少直接下属。
- 这一轮就可能直接结束流程：有一份报告里，一位有十年经验的候选人，因为在 AI 安全相关问题上回答得比较
  笼统而被拒绝。有一位招聘联系人提前分享了阅读材料；也有招聘联系人在通话结束时简单介绍了接下来的几轮
  安排。
- 结果一般一到三天内出来；有一次说好的“48 小时内回复”，结果第二天就收到了一封自动生成的拒信。

### 用人经理初筛

**候选人反馈**：有些流程会在技术电面前后安排一次 40–60 分钟和用人经理（HM）的对话——聊一个过往项目，
同时穿插一些行为面问题。也有流程把用人经理这一轮放在现场面里面，还有少数流程根本没有这一轮。有一位
被内推的资深候选人，用人经理的这通电话直接替代了测评和技术电面，聊得不错之后才安排了现场面。

### 技术电面

**候选人反馈：**

- 通常是 55 分钟，在 Google Meet 上共享屏幕，在 CodeSignal 里写代码。有些流程会背靠背安排两场面试：
  编程加系统设计，或者编程加项目展示。
- **编程**题来自一个不大的、反复出现的实用题库，通常还会追问并发或规模相关的问题：图像处理流水线、
  并发网页爬虫、查找重复文件、一个需要修 bug 并加上持久化的记忆化缓存、从性能分析采样还原调用栈、
  分布式众数与中位数、最长匹配分词器、启动程序模拟器。2026 年年中之前，候选人门户会显示题号（编程
  Q1–Q6，系统设计 Q1–Q5）；大约从 2026 年 6 月到 7 月开始，门户只显示一句笼统的提示，比如“be
  prepared to handle concurrency using primitives/libraries of your choice”（准备好用你选择的原语或
  库来处理并发），这句提示并不能可靠地预测实际题目。
- **系统设计**（system design）对很多资深和 Staff 级别的候选人来说是第一轮技术面试。形式是在共享的
  Google Doc 里，或者你自己共享的任意白板工具上进行。越来越多的比例是**设计文档评审**
  （design-document review）：你会拿到一份写好的设计方案，里面埋了一些问题（比如一个做批处理的推理
  服务），给你几分钟读完，然后讨论哪里有问题、哪里没考虑到。其他反复出现的题目还有：向一个大型 GPU
  集群分发模型权重、一个提示词 playground、一对一聊天系统、一个遥测数据管道。
- **AI 辅助编程**。从 2026 年年中开始，部分电面和现场面会使用一个类似 VS Code 和 GitHub 的浏览器环境，
  里面可以用 Claude Code CLI：先在一个陌生的仓库里评审几个开着的 PR，再挑一个去改进它。还有一种相关
  的形式，是让你在 Claude API 之上搭建一个智能体循环（agent loop）。
- **研究方向的电面**会从一组形式里选择；参见[研究工程与研究科学](#研究工程与研究科学)。
- 编辑器里可能只有一个空的 `main()` 函数，没有任何测试：读文件之类的辅助函数和测试用的 fixture 都
  需要你自己写。
- 大多数报告里第二天就出结果，最长大约一周。

### Take-home 作业

Anthropic 官方怎么说：

- 性能工程团队从 2024 年初开始就在用一份 take-home 作业。它最初是 4 小时，后来缩短到了 2 小时；这个作业
  允许使用 AI 工具（“For this take-home, we explicitly indicate otherwise”，意思是这份 take-home 我们已经
  明确说明可以用），目前的版本是在一个极小、限制很严格的指令集上做题。（[工程博客](https://www.anthropic.com/engineering/AI-resistant-technical-evaluations)、
  [代码仓库](https://github.com/anthropics/original_performance_takehome)）
- 除非说明里另有交代，其他 take-home 作业都不能使用 AI。（[政策](https://www.anthropic.com/candidate-ai-guidance)）

**候选人反馈**：研究方向的流程会用一个 4 小时的探索性机器学习 take-home 作业（训练并评估一个基础模型，
之后做 10 分钟的讲解展示），部分团队用的是 72 小时的研究方向 take-home 作业；有一位候选人发现，具体内容
因团队而异。

### 虚拟现场面

Anthropic 官方怎么说：面试通过 Google Meet 进行；技术类岗位“we use live coding tools like Colab and
CodeSignal”（我们会用 Colab 和 CodeSignal 这类实时编程工具），并且“You can look things up—just be
comfortable with basic syntax and standard libraries so it doesn't eat up your time.”（你可以查资
料——只要对基本语法和标准库足够熟悉，不会因此浪费太多时间就行。）（[招聘页](https://www.anthropic.com/careers)）

**候选人反馈：**

- 软件岗位一般是四到五场 55 分钟的面试——编程、系统设计、项目深挖、文化面和用人经理——可能集中在
  一天，也可能分两到三天进行；有一个研究方向的流程有八场。有一位候选人要求在同一天的两场面试之间安排
  休息时间，也得到了同意。
- **编程**：还是从那个反复出现的题库里出题，追问的方向也差不多，还是并发、I/O 和 CPU 的取舍、崩溃
  恢复这些。
- **系统设计**：大多数流程只有一轮；Staff 级别的流程会有两到三轮，有时甚至完全没有编程环节。设计
  文档评审这种形式很常见。
- **项目深挖**：花大约 20 分钟展示一个过往项目（用幻灯片或者共享屏幕），然后回答关于设计决策、取舍、
  你自己承担的角色和最终结果的问题。
- **文化面**：围绕 AI 安全和价值观——如果一项突破有风险，你会不会推迟发布，如果别人不推迟、或者你因此
  落后了又该怎么办；如果公司股票跌到 0，你会怎么做；你对 Anthropic 的安全做法有哪些不同意见；还有讲一个
  你改变主意的经历，或者修复一段工作关系的经历。面试官不一定是工程师。有好几位候选人觉得自己技术轮表现
  不错，但认为是文化面这一轮让自己没通过；而在一个招聘联系人确实解释了拒绝原因的案例里，起决定作用的
  其实是一轮技术面。
- **用人经理**（如果没有提前安排过）：影响力、协作方式、成长空间、和团队的匹配度。
- **特定方向的轮次**：性能工程和基础设施类岗位会有性能建模（在一个类似电子表格的工具里估算矩阵乘法
  和切分的 roofline）、网络调试角色扮演、分布式编程；AI 辅助的 PR 评审；研究方向则有研究向的编程与
  调试，比如调试一个 GRPO 训练循环。
- 有一次招聘联系人在第一场文化面信号不够强时，额外加了一场文化面（单一个案），还有一位招聘联系人
  提醒说，如果第一天面试表现不好，第二天可能会被取消。
- 有一位候选人反馈说，每场面试都会打一个数字分，只要有一场是 2 分，流程基本就结束了，想通过需要大部分
  是 3 分和 4 分（未经证实）。

### 背景推荐人、定级、团队匹配与 offer

Anthropic 官方怎么说：

- Anthropic 面向澳大利亚、日本、新加坡、韩国、瑞士、英国和欧盟申请者的隐私声明（2025 年 10 月）写道，
  Anthropic “may contact your professional and academic referees to confirm the employment,
  professional and academic information that you have provided”（可能会联系你的职业和学业背景
  推荐人（reference），以核实你所提供的工作经历、职业和学业信息）；“if we decide to offer you
  employment”（如果我们决定向你发出 offer），还可能向第三方核实你的工作经历和职业资格；刑事背景调查的
  结果只会以“pass / fail only”（仅通过/不通过）的形式呈现。（[隐私声明](https://www-cdn.anthropic.com/files/4zrzovbb/website/a9d372d660e3de4e3a0abfc0a19fa15f507d078d.pdf)）
- 职位发布页面写道：“Years of experience required will correlate with the internal job level
  requirements for the position.”（要求的工作年限，会和这个职位的内部职级（level）要求相对应。）
  （[招聘启事](https://job-boards.greenhouse.io/anthropic/jobs/5283063008)）
- 关于 offer：“Take your time. We're happy to give you space to think and finish any other interview
  processes you're going through.”（慢慢考虑没关系，我们很乐意给你时间思考，也乐意等你走完正在进行的
  其他面试流程。）（[招聘 FAQ](https://www.anthropic.com/careers)）

**候选人反馈：**

- 背景推荐人（reference）核实是在现场面之后才发起的，从第二天到一周后不等——给出人数的报告里都是两位，
  其中一个案例是一位前领导和一位现在的同事。也有候选人是在背景推荐人核实之后才收到拒信的。
- 有一位招聘联系人说，职级（level）是流程结束后由招聘委员会决定的，而不是流程开始前就定好的。
- 有些流程从一开始就是面向某个具体团队；另一些（走通用招聘）则是在现场面之后才做 team match（团队
  匹配）。有候选人每一场面试都通过了，最后却没能匹配上团队。有两位候选人的面试反馈全部正面，但因为团队
  想要更有经验的人，还是没有拿到 offer；其中一位面的是单个团队的岗位，也没有触发冷冻期（cooldown）。
  另一个流程里，团队选了别的候选人，招聘联系人转而帮忙找其他团队。
- 股权是薪酬包里很重要的一部分，据说股权的定价会参考 offer 签署日期相对于最近一轮融资的时间点（未经
  证实）。

## 时间线

**候选人反馈：**

| 阶段 | 报告中的时间范围 |
| --- | --- |
| 申请 → 第一次收到回复 | 第二天到大约两个月 |
| 提交测评 → 收到招聘联系人邮件 | 大约两小时到几天 |
| 某一轮的结果 | 第二天到大约一周 |
| 两轮之间的间隔 | 几天到三周 |
| 现场面 → 出结果 | 一到五天；涉及 team match 时会更久 |
| 全流程走完 | 最快大约两周，慢的话两个月 |

- 报告中最快的一个流程（研究方向，招聘联系人主动联系）：第 1 天招聘联系人联系，第 4 天用人经理这
  一轮，第 10–11 天是两天的现场面，第 13 天被要求提供背景推荐人，第 15 天出结果。
- 一个通过内推走的 Staff 级别流程：第 1 天投递，第 6 天和招聘联系人通话，中间赶上公司放假一周，第 20 天
  系统设计电面，第 35–36 天是两天的现场面，第 38 天出结果。
- 公司整体放假会让招聘暂停：有一次为期一周的暑期假期拖慢了好几个流程；另一份报告则把回复变慢归因于
  内部的黑客周。
- 有一位候选人手上有其他公司的 offer，周一提出加快进度，当周四和周五就安排了现场面。
- 有一位候选人在某一轮结束大约一周后才收到拒信，并把它理解为没有团队愿意接手，而不是这一轮面试没通过
  （这是候选人自己的解读）。

## 赛道

### 软件工程（产品与基础设施）

**候选人反馈**的典型顺序是：在线测评 → HR 初筛 → 技术电面（编程、系统设计，或者两者都有）→ 现场面
（编程、系统设计、项目深挖、文化面、用人经理）→ 背景推荐人核实 → team match → offer。产品方向的
流程抽到过提示词 playground 这道设计题；基础设施方向的流程更偏向分布式编程和基础设施设计（权重分发、
推理服务、遥测）。有一次招聘联系人因为候选人的背景，把一个系统设计轮换成了数据基础设施方向的版本，还有一个全栈
岗位在测评里要求用 TypeScript。

### Staff 及以上

**候选人反馈**：Staff 级别的流程经常跳过测评，直接从系统设计面试开始；现场面会有两到三轮系统设计，
有时完全没有编程环节；有一次招聘联系人问了一位管理岗候选人此前带的团队规模，以及是否能接受转做个人
贡献者；有个团队只招“Staff+”，对应的招聘启事要求八年以上工作经验。

### 研究工程与研究科学

**候选人反馈**：申请时会对应具体的团队，然后第一轮技术面试会从一组形式里选一种（有时招聘联系人的
邮件里会直接给出选项）：在 Claude API 上搭建一个带工具调用的智能体；LLM 的提示词设计与工程；编程加设计
（比如一个加权数据批采样器）；一个机器学习实验配置系统；机器学习研究基础（用 PyTorch 写代码，并涉及基础
线性代数和画图）；强化学习基础（调试一个 GRPO 训练循环）；或者用一个 take-home 作业代替现场电面。现场面
在文化面和用人经理这两轮之外，还会加入研究方向的编程与调试；有一个研究方向的流程两天里安排了八场面试。

### 性能工程

招聘启事里，Performance Engineer 岗位分布在旧金山、纽约和西雅图，标注的薪资区间是 $280,000–$850,000
（[招聘启事](https://job-boards.greenhouse.io/anthropic/jobs/4020350008)）。这个团队从 2024 年初开始
使用自己的 take-home 作业（[工程博客](https://www.anthropic.com/engineering/AI-resistant-technical-evaluations)），
详见[Take-home 作业](#take-home-作业)。**候选人反馈**（一个性能工程方向的流程）：没有测评；HR 初筛、
用人经理初筛，然后是一轮分布式众数与中位数的编程电面；候选人门户里介绍的现场面包括一场围绕
“从第一性原理推导性能边界”展开的系统设计，以及一个网络调试角色扮演。另有一个基础设施方向的现场面考了性能建模
（矩阵乘法和切分的 roofline 估算）、分布式编程和文化面。

### Anthropic Fellows

参见[实习、应届生与 Fellows 项目](#实习应届生与-fellows-项目)：申请与背景推荐人核实、有人监考的
CodeSignal 测评、面试，然后是一次研究讨论。

## 职级与薪酬

### 职位名称与职级

招聘启事上用的是职能类头衔——Software Engineer、Research Engineer、Research Scientist、Performance
Engineer——有时会带一个资深程度的修饰词，比如 Staff 或 Staff+，文案里会提到“internal job level
requirements”（内部职级要求），但不会公开完整的职级体系。**候选人反馈**（来自同一个讨论）：内部职级是
有编号的，并且有一个大致的对照关系——Anthropic 的一个职级大致对应 Google 或 Meta 高一级的职级；不过
讨论中给出的对照关系并不一致，只能当作传闻参考。因为职级可能要到面试结束之后才决定，建议提前问招聘联系人
这个岗位对应的职级区间。按职级划分的薪酬数据（众包整理）：
[levels.fyi](https://www.levels.fyi/companies/anthropic/salaries/software-engineer)。

### 公开薪资区间

美国的招聘启事会写出年薪区间（销售岗位对应的则是目标年收入，OTE）。这些区间都很宽，招聘启事也没有说明
区间和内部职级之间怎么对应。举例如下，核实于 2026-09-22：

| 职位 | 地点 | 年薪（USD） |
| --- | --- | --- |
| [Software Engineer, Product (Full Stack)](https://job-boards.greenhouse.io/anthropic/jobs/4018206008) | 旧金山、纽约 | 405,000 – 485,000 |
| [Software Engineer, Research Infrastructure](https://job-boards.greenhouse.io/anthropic/jobs/5283063008) | 旧金山、纽约 | 405,000 – 625,000 |
| [Software Engineer, RL Data](https://job-boards.greenhouse.io/anthropic/jobs/5238606008) | 旧金山、纽约 | 320,000 – 485,000 |
| [Research Engineer, Interpretability](https://job-boards.greenhouse.io/anthropic/jobs/4980430008) | 旧金山 | 315,000 – 560,000 |
| [Research Scientist, Frontier Red Team (Emerging Risks)](https://job-boards.greenhouse.io/anthropic/jobs/5103788008) | 旧金山 | 320,000 – 850,000 |
| [Performance Engineer](https://job-boards.greenhouse.io/anthropic/jobs/4020350008) | 旧金山、纽约、西雅图 | 280,000 – 850,000 |

同一天核实的一个苏黎世研究岗位招聘启事就没有列出薪资区间。招聘启事本身会不断变化，想要最新数字，
还是要直接打开当前在招的那条启事。

### 福利与股权

[招聘页](https://www.anthropic.com/careers)上列出了薪资和股权方案；可选的股权捐赠 1:1 配捐，最高
为股权授予额的 25%；22 周带薪育儿假；医疗、牙科和视力保险，家属也覆盖；生育相关福利；
弹性带薪休假；带配比的退休金计划；每月 $500 的健康津贴；教育、居家办公、通勤和搬迁方面的支持；以及办公室提供餐食。Anthropic 是
一家未上市公司，所以要主动问清楚股权是怎么估值的、什么时候能卖。

## 面试规则

| 主题 | 具体规则 |
| --- | --- |
| 视频 | Google Meet（[招聘页](https://www.anthropic.com/careers)）；候选人反馈需要共享屏幕 |
| 编程环境 | CodeSignal 或 Colab（[招聘页](https://www.anthropic.com/careers)）；AI 辅助编程轮次用的是一个类似 VS Code 和 GitHub 的浏览器环境（候选人反馈） |
| 查资料 | 允许：“You can look things up”（你可以查资料）（[招聘页](https://www.anthropic.com/careers)）；候选人反馈，直接复制普通网页搜索找到的代码也是被接受的 |
| AI 助手 | 除非另有说明，否则不允许（[政策](https://www.anthropic.com/candidate-ai-guidance)）；性能方向的 take-home 作业允许使用（[工程博客](https://www.anthropic.com/engineering/AI-resistant-technical-evaluations)），据候选人反馈，AI 辅助编程轮次也允许。候选人还反馈遇到过被要求划过搜索引擎的 AI 摘要结果 |
| 编程语言 | Python 是最保险的选择；候选人反馈，事先和招聘联系人约好的话，有几道编程题也可以用 Java 或 TypeScript，不过也有一些轮次只能用 Python |
| 测试 | 通常不提供：需要自己写测试 fixture 并运行（候选人反馈） |
| 设计类轮次 | 一个共享的 Google Doc，或者你自己共享的任意白板工具（有候选人用过 Excalidraw 和 Figma）；设计文档评审不需要画图（候选人反馈） |

## 决定之后

- “We're not able to provide feedback on resumes or interviews.”（我们没办法针对简历或面试提供
  反馈。）（[招聘 FAQ](https://www.anthropic.com/careers)）**候选人反馈**：收到的是模板邮件，一般是
  在最后一轮结束后一到三天内。
- “Yes—you're welcome to re-apply after 12 months, or sooner if something materially changes about
  your experience or skills.”（可以的——欢迎你在 12 个月后重新申请，如果你的经历或技能有实质性变化，
  也可以更早重新申请。）（[招聘 FAQ](https://www.anthropic.com/careers)）
- **候选人反馈**：收到过一封拒信里写着面试反馈一年内有效，并且所有团队都能看到；还有一个案例，拒绝
  的唯一原因是团队想要更资深的人，这种情况下没有触发冷冻期（cooldown）。有一个职位停止招聘之后，
  某位候选人的流程被另一位招聘联系人重新启动了。

## 如何用这个仓库准备面试

| 阶段 | 页面 |
| --- | --- |
| 在线测评 | [内存数据库](coding/oa-in-memory-database/README.zh.md) · [银行系统](coding/oa-bank-system/README.zh.md) · [菜谱管理系统](coding/oa-recipe-manager/README.zh.md) · [任务管理](coding/oa-task-manager/README.zh.md) · [文件系统](coding/oa-file-system/README.zh.md) · [云存储](coding/oa-cloud-storage/README.zh.md) · [员工考勤与薪酬](coding/oa-worker-grant-management/README.zh.md) |
| HR 初筛 | [HR 初筛](behavioral/recruiter-screen/README.zh.md) |
| 编程电面与现场编程 | [图像处理](coding/image-processing-pipeline/README.zh.md) · [重复文件](coding/file-deduplication/README.zh.md) · [网页爬虫](coding/concurrent-web-crawler/README.zh.md) · [从采样还原调用栈](coding/stack-trace-reconstruction/README.zh.md) · [LRU 缓存](coding/lru-cache-bugfix-durability/README.zh.md) · [分布式众数与中位数](coding/distributed-mode-median/README.zh.md) · [分词器](coding/longest-match-tokenizer/README.zh.md) · [启动程序](coding/bootloader-loop-fix/README.zh.md) |
| AI 辅助编程环节 | [借助 AI 助手做 PR 评审](coding/agentic-pr-review/README.zh.md) · [带工具调用的智能体循环](coding/llm-agent-tool-loop/README.zh.md) |
| 系统设计 | [推理 API](system-design/inference-api-batching/README.zh.md) · [设计文档评审](system-design/design-doc-review/README.zh.md) · [模型权重分发](system-design/model-weight-distribution/README.zh.md) · [一对一聊天](system-design/one-on-one-chat/README.zh.md) · [提示词 Playground](system-design/prompt-playground/README.zh.md) · [遥测数据管道](system-design/telemetry-data-infrastructure/README.zh.md) |
| 研究方向电面 | [加权数据批采样器](coding/weighted-data-batcher/README.zh.md) · [机器学习配置系统](coding/ml-config-system/README.zh.md) · [LLM 提示工程](coding/prompting-with-llms/README.zh.md) · [用 einsum 实现注意力](coding/qkv-attention-einsum/README.zh.md) · [GRPO 调试](coding/grpo-debug/README.zh.md) |
| Take-home 作业 | [机器学习研究](take-home/ml-take-home/README.zh.md) · [双下降](take-home/interp-double-descent/README.zh.md) · [内核优化](take-home/performance-kernel-optimization/README.zh.md) |
| 性能工程方向 | [性能建模](coding/matmul-performance-modeling/README.zh.md) · [内核优化](take-home/performance-kernel-optimization/README.zh.md) · [分布式众数与中位数](coding/distributed-mode-median/README.zh.md) |
| Fellows 项目 | [DNS 解析器](coding/oa-dns-resolver/README.zh.md) · [银行系统](coding/oa-bank-system/README.zh.md) · [推理引擎](coding/oa-inference-engine/README.zh.md) · [机器学习研究](take-home/ml-take-home/README.zh.md) |
| 现场面沟通环节 | [文化面](behavioral/culture-ai-safety/README.zh.md) · [用人经理面](behavioral/hiring-manager/README.zh.md) · [项目深挖](behavioral/project-deep-dive/README.zh.md) |

[路线图](ROADMAP.zh.md) 按方向和优先级给每个页面排了序。

## 官方信息来源

- 招聘信息、面试形式、福利与常见问题：<https://www.anthropic.com/careers>；在招职位：
  <https://www.anthropic.com/careers/jobs>
- 招聘过程中使用 AI 的规定：<https://www.anthropic.com/candidate-ai-guidance>
- Anthropic Fellows Program：<https://alignment.anthropic.com/2025/anthropic-fellows-program-2026/>、
  <https://job-boards.greenhouse.io/anthropic/jobs/5183044008>
- 性能工程 take-home 作业：<https://github.com/anthropics/original_performance_takehome>、
  <https://www.anthropic.com/engineering/AI-resistant-technical-evaluations>
- 申请者隐私声明（AU、JP、SG、KR、CH、UK、EU；2025 年 10 月）：
  <https://www-cdn.anthropic.com/files/4zrzovbb/website/a9d372d660e3de4e3a0abfc0a19fa15f507d078d.pdf>
- 给 HR 初筛和文化面用的阅读材料：[Core Views on AI Safety](https://www.anthropic.com/news/core-views-on-ai-safety)
  （2023 年 3 月）· [Responsible Scaling Policy](https://www.anthropic.com/rsp)（3.4 版，2026 年 7 月
  8 日生效）· [Claude's constitution](https://www.anthropic.com/constitution) ·
  [The Long-Term Benefit Trust](https://www.anthropic.com/news/the-long-term-benefit-trust)（2023 年
  9 月）· [Company](https://www.anthropic.com/company)
