# The Anthropic Hiring Process

English · [中文](PROCESS.zh.md)

> [!NOTE]
> **Unofficial, last updated September 2026.** This guide combines two kinds of information. Statements with a
> source link come from Anthropic's own pages (careers site, job postings, policies, engineering blog), checked on
> 2026-09-22. Paragraphs headed **Candidates report** summarise public write-ups by people who interviewed between
> roughly April and September 2026; they are not official, they are not linked, and one-off reports are marked as
> such. Teams run their loops differently and the process keeps changing: your recruiter's instructions and the
> job posting always take precedence over this page.

## At a glance

| Stage | What happens | Length | Who gets it |
| --- | --- | --- | --- |
| [Application](#getting-an-interview) | Apply online, get referred, or get contacted by a recruiter | — | everyone |
| [Online assessment](#online-assessment) | One multi-level "build a small system" problem on CodeSignal | 90 min | many software applicants; often skipped |
| [Recruiter screen](#recruiter-screen) | Motivation, AI safety, background, logistics | 15–30 min | everyone |
| [Hiring-manager screen](#hiring-manager-screen) | Past work and fit with the team | 40–60 min | some loops |
| [Technical screen](#technical-screen) | One coding or system-design interview, sometimes two back to back | 55 min each | most loops |
| [Take-home](#take-home-assignments) | Performance-engineering and research tracks | 2 h to 72 h | some tracks |
| [Virtual onsite](#virtual-onsite) | Four or five interviews (eight in one research loop) over one to three days | 55 min each | everyone who gets this far |
| [References and decision](#references-levelling-team-match-and-offer) | References, hiring committee, level, team match, offer | days to weeks | after the onsite |

From first contact to a decision: about two weeks in the fastest reported case, one to two months in most, longer
when a team pauses or a company-wide break falls in the middle (see [Timeline](#timeline)).

## Getting an interview

### What Anthropic says it looks for

- "We care about what you can do, not where you learned to do it." About half of the technical staff had no prior
  ML experience, and about half have PhDs, "but plenty of brilliant colleagues never went to college."
  ([careers](https://www.anthropic.com/careers))
- "If you've done interesting independent research, written a thoughtful blog post, or contributed to open source,
  put that at the top of your resume." ([careers](https://www.anthropic.com/careers))
- Job postings: "We encourage you to apply even if you do not believe you meet every single qualification." The
  stated minimum education is a bachelor's degree "or an equivalent combination of education, training, and/or
  experience". ([example posting](https://job-boards.greenhouse.io/anthropic/jobs/5283063008))

### Where and how to apply

Open roles are listed on the [careers site](https://www.anthropic.com/careers/jobs), filterable by team and office:
about twenty locations in September 2026, among them San Francisco, New York, Seattle, Boston, Washington D.C.,
London, Dublin, Zurich, Paris, Munich, Tokyo, Seoul, Singapore, Sydney and Bangalore, plus remote-friendly US roles
that require travel. Anthropic publishes neither a referral programme nor a limit on how many roles you may apply
to.

**Candidates report:**

- **Applying online works.** Several 2026 loops started from a cold application, including a staff-level loop
  and one that passed the onsite. The first reply came anywhere from the next day (an assessment invitation after
  an evening application) to two months later.
- **A referral speeds things up but decides nothing.** Most referred candidates heard back within a few days,
  and some skipped the online assessment or even the technical screen; yet one referred candidate failed the
  recruiter screen and two others their first technical round.
- **Recruiter outreach** started several loops, for senior engineers and for a research candidate; one research
  engineer was referred after meeting people at an AI-safety event.
- **Applying to several roles** at once (more than ten in one report) did not prevent a callback. When a role
  closed before the recruiter got to the candidate, the recruiter pointed them to open roles at another level.
- **The résumé can still be screened after the assessment.** Perfect assessment scores were followed by days of
  silence, and one candidate heard after the recruiter call that the team had passed on the résumé.

### Using AI in your application

Anthropic's policy, [How to collaborate with Claude during our hiring process](https://www.anthropic.com/candidate-ai-guidance)
(updated July 10, 2025):

- Application materials: "Please create your first draft yourself, then use Claude to refine it."
- Take-home assessments: "Complete these without Claude unless we indicate otherwise."
- Live interviews: "This is all you–no AI assistance unless we indicate otherwise."
- Preparation: using Claude to research the company, practise answers and prepare questions is allowed.

Some postings still carry a stricter sentence — "please do not use AI assistants during the application process"
([example](https://job-boards.greenhouse.io/anthropic/jobs/4018206008)) — so follow the posting you apply to. A few
rounds do indicate otherwise: the performance-engineering take-home allows AI tools explicitly, and candidates
quote the invitation to the AI-assisted coding interview as giving access to the Claude Code CLI (see
[Rules during interviews](#rules-during-interviews)).

### Location, visas and hybrid work

- "Currently, we expect all staff to be in one of our offices at least 25% of the time. However, some roles may
  require more time in our offices." ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5283063008))
- "Most staff are in the Bay Area and come to the office regularly. Some live further away and come in for one week
  a month." ([careers FAQ](https://www.anthropic.com/careers))
- "Yes. We sponsor visas and green cards for eligible roles." ([careers FAQ](https://www.anthropic.com/careers))
  Postings add that sponsorship is not possible for every role and candidate, "but if we make you an offer, we will
  make every reasonable effort to get you a visa, and we retain an immigration lawyer to help with this."
  ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5283063008))
- "All interviews are conducted over Google Meet, and we can accommodate a variety of timezones."
  ([careers](https://www.anthropic.com/careers)) No step of the process is in person.

### Internships, new graduates and the Fellows Program

- "We don't currently offer internships." ([careers FAQ](https://www.anthropic.com/careers)) Anthropic lists no
  separate new-graduate programme; recent graduates apply to regular roles, and candidates report new graduates in
  software and data-infrastructure loops.
- The **Anthropic Fellows Program** is the structured entry point for AI-safety research: four months of full-time
  empirical research with mentorship from Anthropic researchers, a weekly stipend of 3,850 USD / 2,310 GBP /
  4,300 CAD and about 15,000 USD a month for compute. No PhD, prior ML experience or publications are needed;
  fluency in Python, full-time availability and work authorisation in the US, UK or Canada are. The stages are an
  initial application and reference check, technical assessments and interviews, and a research discussion; cohorts
  start several times a year. Anthropic says that over 40% of the fellows in its first cohort subsequently joined it
  full-time, and a posting for the programme's AI Safety & Security track gives 25–50% of fellows in previous
  cohorts receiving a full-time offer.
  ([announcement](https://alignment.anthropic.com/2025/anthropic-fellows-program-2026/),
  [posting](https://job-boards.greenhouse.io/anthropic/jobs/5183044008))
- **Candidates report** the Fellows technical assessments as CodeSignal tests proctored by webcam, with no AI
  assistant and no web search: one build-a-system test (a DNS resolver in one cohort; completing an inference
  server inside an unfamiliar, sparsely documented codebase in another) and a separate debugging test.

### A public performance challenge

Anthropic has published its [original performance-engineering take-home](https://github.com/anthropics/original_performance_takehome),
together with an [engineering post](https://www.anthropic.com/engineering/AI-resistant-technical-evaluations)
(January 21, 2026) on why it keeps redesigning the test as Claude models improve. The repository invites anyone who
brings the kernel below 1,487 simulated cycles — Claude Opus 4.5's best result at launch — to email
performance-recruiting@anthropic.com "with your code (and ideally a resume)".

### Recruiting scams

"Anthropic recruiters only contact you from @anthropic.com email addresses"; in some cases Anthropic may partner
with "vetted recruiting agencies who will identify themselves as working on behalf of Anthropic". "Legitimate
Anthropic recruiters will never ask for money, fees, or banking information before your first day."
([posting](https://job-boards.greenhouse.io/anthropic/jobs/4980430008))

## The stages

### Online assessment

**Candidates report:**

- **Format.** CodeSignal, 90 minutes, started whenever you like from an emailed link (one invitation gave a two-week
  window; another candidate started a month after receiving theirs without trouble). No webcam in the standard
  version and, in one report, a practice area before the timer starts.
- **Content.** One problem in four levels. Each level unlocks when the previous one passes its tests, and later
  levels extend the same code. The problems are small systems rather than algorithm puzzles: an in-memory database,
  a bank with two-step transfers and account merges, a task manager, a file system, a recipe manager, cloud file
  storage with quotas, employee time tracking and pay. Correct, clean, extensible code matters more than asymptotic
  efficiency.
- **Scores.** Partial credit counts: 750/1000 (the last level only partly passing) and 800/1000 both advanced to
  the recruiter call. A perfect score did not guarantee the next step.
- **Timing and language.** The recruiter's email came within a day in several reports, once about two hours after
  submission. Python is the default; a full-stack role required TypeScript. One invitation offered to reuse a
  CodeSignal Industry Coding Assessment score from another company's process within the previous six months.
- **Hints.** The topic hinted in the recruiter's preparation email did not reliably predict the problem.
- **Skipped** in many loops: for some referred candidates, for staff-level roles, in the one performance loop
  reported and for some research roles.

### Recruiter screen

**Candidates report:**

- Usually 30 minutes (15–20 in some reports), by phone, with a recruiter rather than an engineer.
- It is not only logistics. Recurring questions: why Anthropic rather than another lab; why now; what AI safety
  means to you; how Anthropic differs from other labs; how AI could be misused; an AI problem you think is
  underrated; what you think of recent company news or the Responsible Scaling Policy. Then your background and the
  project most relevant to the role, and finally work authorisation, timeline, other processes and sometimes
  level. A people manager moving to an individual-contributor role was asked about team size and direct reports.
- It can end the process: in one report a candidate with ten years of experience was rejected after generic
  answers about AI safety. One recruiter shared reading material beforehand; others outlined the next rounds at
  the end of the call.
- Results in one to three days; a promised "within 48 hours" once became an automated rejection the next day.

### Hiring-manager screen

**Candidates report:** some loops put a 40–60 minute conversation with the hiring manager before or after the
technical screen — a walk through a past project combined with behavioural questions. Others hold the
hiring-manager interview inside the onsite, and a few have none. For one referred senior candidate the
hiring-manager call replaced both the assessment and the technical screen, and the onsite was scheduled once it
had gone well.

### Technical screen

**Candidates report:**

- Usually 55 minutes on Google Meet with screen sharing, coding in CodeSignal. Some loops schedule two interviews
  back to back: coding plus system design, or coding plus a project presentation.
- **Coding** problems come from a small recurring set of practical tasks with a concurrency or scale follow-up: an
  image-processing pipeline, a concurrent web crawler, finding duplicate files, a memoising cache with a bug to fix
  and durability to add, call stacks from profiler samples, distributed mode and median, a longest-match tokenizer,
  a boot-program simulator. Until mid-2026 the candidate portal showed a question number (Coding Q1–Q6, System
  design Q1–Q5); since about June–July 2026 it shows only a general note such as "be prepared to handle concurrency
  using primitives/libraries of your choice", which does not reliably predict the problem.
- **System design** is the first technical round for many senior and staff candidates. It happens in a shared
  Google Doc or on any whiteboard tool you share. A growing share are **design-document reviews**: you get a
  written design with planted problems (for example an inference service that batches requests), a few minutes to
  read it, and then discuss what is wrong and what is missing. Other recurring prompts: distributing model weights
  to a large GPU fleet, a prompt playground, a one-to-one chat system, a telemetry pipeline.
- **AI-assisted coding.** Since mid-2026 some screens and onsites use a browser environment resembling VS Code and
  GitHub with the Claude Code CLI available: review several open pull requests in an unfamiliar repository, then
  improve one. A related format asks you to build an agent loop on top of the Claude API.
- **Research-track screens** use a menu of formats; see [Research engineering and research
  science](#research-engineering-and-research-science).
- The editor may start with nothing but a `main()` function and no tests: you write the file-reading helpers and
  your own test fixtures.
- Results the next day in most reports, up to about a week.

### Take-home assignments

What Anthropic says:

- The performance engineering team has used a take-home since early 2024. It started at 4 hours and was cut to 2;
  it allows AI tools ("For this take-home, we explicitly indicate otherwise"), and its current version uses puzzles
  on a tiny, heavily constrained instruction set. ([engineering post](https://www.anthropic.com/engineering/AI-resistant-technical-evaluations),
  [repository](https://github.com/anthropics/original_performance_takehome))
- Other take-homes are done without AI unless the instructions say otherwise.
  ([policy](https://www.anthropic.com/candidate-ai-guidance))

**Candidates report:** research loops use a 4-hour exploratory ML take-home (train and evaluate a basic model, then
give a 10-minute presentation) and, in some teams, a 72-hour research take-home; one candidate found that the
content differs from team to team.

### Virtual onsite

What Anthropic says: interviews run over Google Meet; for technical roles "we use live coding tools like Colab and
CodeSignal", and "You can look things up—just be comfortable with basic syntax and standard libraries so it doesn't
eat up your time." ([careers](https://www.anthropic.com/careers))

**Candidates report:**

- Four or five 55-minute interviews for software roles — coding, system design, project deep dive, culture and
  hiring manager — on one day or split across two or three; one research loop had eight. One candidate asked for
  breaks between interviews on the same day and got them.
- **Coding:** another problem from the recurring set, with the same concurrency, I/O-versus-CPU and crash-recovery
  follow-ups.
- **System design:** one round in most loops; staff-level loops had two or three, sometimes with no coding at all.
  The design-document review format is frequent.
- **Project deep dive:** present a past project for about 20 minutes (slides or a shared screen), then answer
  questions on decisions, trade-offs, your own role and results.
- **Culture:** AI safety and values — whether you would delay a risky breakthrough and what if others do not or you
  fall behind, what you would do if the company's stock fell to zero, what you disagree with in Anthropic's approach
  to safety, stories about changing your mind or repairing a working relationship. The interviewer is not
  necessarily an engineer. Several candidates whose technical rounds felt fine believed the culture round decided
  against them; in one case where the recruiter did explain the rejection, the deciding round was a technical one.
- **Hiring manager** (if not held earlier): impact, collaboration, growth, team fit.
- **Track-specific rounds:** performance modelling (roofline estimates for matrix multiplications and sharding in a
  spreadsheet-like tool), a network-debugging role play and distributed coding for performance and infrastructure
  roles; AI-assisted pull-request review; research coding and debugging, such as a GRPO training loop.
- A recruiter added an extra culture interview when the first one's signal was not strong enough (single report),
  and one recruiter warned that a poor first day could cancel the second.
- One candidate reported that each interview gets a numeric rating, that a single 2 usually ends the loop and that
  passing takes mostly 3s and 4s (unverified).

### References, levelling, team match and offer

What Anthropic says:

- Anthropic's applicant privacy notice for Australia, Japan, Singapore, South Korea, Switzerland, the UK and the EU
  (October 2025) states that it "may contact your professional and academic referees to confirm the employment,
  professional and academic information that you have provided", that "if we decide to offer you employment" it may
  also collect verification of your employment history and professional qualifications from third parties, and
  that criminal background checks are reported "pass / fail only".
  ([notice](https://www-cdn.anthropic.com/files/4zrzovbb/website/a9d372d660e3de4e3a0abfc0a19fa15f507d078d.pdf))
- Postings: "Years of experience required will correlate with the internal job level requirements for the
  position." ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5283063008))
- On offers: "Take your time. We're happy to give you space to think and finish any other interview processes you're
  going through." ([careers FAQ](https://www.anthropic.com/careers))

**Candidates report:**

- References are requested after the onsite, from the next day to a week later — two in the reports that give a
  number, in one case a former manager and a current colleague. Rejections still followed some reference checks.
- One recruiter said the level is set at the hiring committee after the loop, not before it.
- Some loops interview for one team; in others (general hiring) team matching comes after the onsite. Candidates
  passed every interview and then failed team matching. Two candidates had all-positive feedback and no offer
  because the team wanted someone more experienced; for one of them, who had interviewed for a single team, no
  cooldown applied. In another loop the team chose a different candidate and the recruiter looked for other teams.
- Equity is a large part of the package, and its price was said to follow the offer-signing date relative to the
  latest funding round (unverified).

## Timeline

**Candidates report:**

| Step | Reported range |
| --- | --- |
| Application → first contact | next day to about two months |
| Assessment submitted → recruiter email | about two hours to a few days |
| Result of a round | next day to about a week |
| Gap between rounds | a few days to three weeks |
| Onsite → decision | one to five days; longer when team matching is involved |
| End to end | about two weeks (fastest) to two months |

- Fastest reported loop (research, recruiter outreach): recruiter contact on day 1, hiring manager on day 4, a
  two-day onsite on days 10–11, references requested on day 13, the decision on day 15.
- A staff-level loop from a referral: application on day 1, recruiter call on day 6, system-design screen on day 20
  after a week-long company break, a two-day onsite on days 35–36, decision on day 38.
- Company-wide breaks pause recruiting: a week-long summer break held up several loops, and one report put slow
  replies down to an internal hack week.
- With a competing offer, one candidate asked on a Monday to speed things up and had the onsite that Thursday and
  Friday.
- One candidate read a rejection that came about a week after a round as meaning that no team had picked them up,
  not that the round had failed (their own interpretation).

## Tracks

### Software engineering (product and infrastructure)

**Candidates report** this typical order: online assessment → recruiter screen → technical screen (coding, system
design or both) → onsite (coding, system design, project deep dive, culture, hiring manager) → references → team
match → offer. Product loops drew the prompt-playground design; infrastructure loops leaned on distributed coding and
infrastructure design (weight distribution, inference serving, telemetry). One recruiter switched a system-design
slot to a data-infrastructure version because of the candidate's background, and a full-stack role required
TypeScript in the assessment.

### Staff and above

**Candidates report:** staff-level loops often skip the assessment and open with a system-design interview; onsites
had two or three system-design rounds and sometimes no coding; a recruiter asked a people manager about team size
and comfort with an individual-contributor role; one team was hiring only at "Staff+", with a posting asking for
eight or more years of experience.

### Research engineering and research science

**Candidates report** applying to a specific team, then one first technical round chosen from a menu (sometimes the
recruiter email offers the choice): building an agent with tool use on the Claude API; prompting and engineering
with LLMs; coding and design (for example a weighted data batcher); an ML configuration system; ML research
fundamentals (coding in PyTorch, with basic linear algebra and plotting); RL fundamentals (debugging a GRPO training
loop); or a take-home in place of the live screen. Onsites add research coding and debugging to the culture and
hiring-manager rounds; one research loop had eight interviews over two days.

### Performance engineering

Postings list Performance Engineer roles in San Francisco, New York and Seattle, with a posted range of
$`280,000–`$850,000 ([posting](https://job-boards.greenhouse.io/anthropic/jobs/4020350008)). The team has used its
take-home since early 2024 ([engineering post](https://www.anthropic.com/engineering/AI-resistant-technical-evaluations));
see [Take-home assignments](#take-home-assignments). **Candidates report** (one performance loop): no assessment;
recruiter screen, hiring-manager screen, then a coding screen on distributed mode and median; the candidate portal
described an onsite with system design framed around performance bounds derived from first principles and a
network-debugging role play. An infrastructure onsite had performance modelling (roofline arithmetic for matrix
multiplications and sharding), distributed coding and culture.

### Anthropic Fellows

See [Internships, new graduates and the Fellows Program](#internships-new-graduates-and-the-fellows-program):
application and reference check, proctored CodeSignal assessments, interviews, then a research discussion.

## Levels and compensation

### Titles and levels

Postings use functional titles — Software Engineer, Research Engineer, Research Scientist, Performance Engineer —
sometimes with a seniority qualifier such as Staff or Staff+, and they refer to "internal job level requirements"
without publishing the ladder. **Candidates report** (one discussion) numbered internal levels and a rough mapping
in which an Anthropic level corresponds to one level higher at Google or Meta; the mappings given conflict, so treat
them as hearsay. Because the level may be decided only after the interviews, ask the recruiter early which levels
the role is scoped for. Crowd-sourced compensation by level: [levels.fyi](https://www.levels.fyi/companies/anthropic/salaries/software-engineer).

### Posted salary ranges

US postings state an annual salary range (for sales roles, on-target earnings). The ranges are wide, and postings do
not say how a range maps to internal levels. Examples, checked on 2026-09-22:

| Role | Location | Annual salary (USD) |
| --- | --- | --- |
| [Software Engineer, Product (Full Stack)](https://job-boards.greenhouse.io/anthropic/jobs/4018206008) | San Francisco, New York | 405,000 – 485,000 |
| [Software Engineer, Research Infrastructure](https://job-boards.greenhouse.io/anthropic/jobs/5283063008) | San Francisco, New York | 405,000 – 625,000 |
| [Software Engineer, RL Data](https://job-boards.greenhouse.io/anthropic/jobs/5238606008) | San Francisco, New York | 320,000 – 485,000 |
| [Research Engineer, Interpretability](https://job-boards.greenhouse.io/anthropic/jobs/4980430008) | San Francisco | 315,000 – 560,000 |
| [Research Scientist, Frontier Red Team (Emerging Risks)](https://job-boards.greenhouse.io/anthropic/jobs/5103788008) | San Francisco | 320,000 – 850,000 |
| [Performance Engineer](https://job-boards.greenhouse.io/anthropic/jobs/4020350008) | San Francisco, New York, Seattle | 280,000 – 850,000 |

A Zurich research posting checked on the same day listed no range. Postings come and go; open the current ones for
up-to-date numbers.

### Benefits and equity

The [careers page](https://www.anthropic.com/careers) lists salary and equity packages; optional equity donation
matching at 1:1, up to 25% of the equity grant; 22 weeks of paid parental leave; health, dental and vision insurance including
dependents; fertility benefits; flexible paid time off; retirement plans with matching; a $500 monthly wellness
stipend; education, home-office, commuter and relocation support; and meals in the office. Anthropic is a private
company, so ask how the equity is valued and when it can be sold.

## Rules during interviews

| Topic | What applies |
| --- | --- |
| Video | Google Meet ([careers](https://www.anthropic.com/careers)); candidates report sharing their screen |
| Coding environment | CodeSignal or Colab ([careers](https://www.anthropic.com/careers)); for AI-assisted rounds a browser environment resembling VS Code and GitHub (candidates report) |
| Looking things up | Allowed: "You can look things up" ([careers](https://www.anthropic.com/careers)); candidates report copying code found by ordinary web search was accepted |
| AI assistants | Not allowed unless indicated ([policy](https://www.anthropic.com/candidate-ai-guidance)); allowed in the performance take-home ([engineering post](https://www.anthropic.com/engineering/AI-resistant-technical-evaluations)) and, candidates report, in the AI-assisted coding round. Candidates also report being asked to scroll past a search engine's AI summary |
| Language | Python is the safe choice; candidates report Java and TypeScript for several coding problems when agreed with the recruiter beforehand, while some rounds are Python-only |
| Tests | Often none provided: write your own fixtures and run them (candidates report) |
| Design rounds | A shared Google Doc or any whiteboard tool you share (Excalidraw and Figma were used); design-document reviews need no diagram (candidates report) |

## After a decision

- "We're not able to provide feedback on resumes or interviews." ([careers FAQ](https://www.anthropic.com/careers))
  **Candidates report** template emails, usually one to three days after the last round.
- "Yes—you're welcome to re-apply after 12 months, or sooner if something materially changes about your experience
  or skills." ([careers FAQ](https://www.anthropic.com/careers))
- **Candidates report** a rejection email stating that interview feedback remains valid for a year and is visible to
  every team, and, in one case where the only reason was that the team wanted more seniority, that no cooldown
  applied. After a role stopped hiring, one candidate's process was restarted by a different recruiter.

## How to prepare with this repository

| Stage | Pages |
| --- | --- |
| Online assessment | [In-memory database](coding/oa-in-memory-database/README.md) · [Banking system](coding/oa-bank-system/README.md) · [Recipe manager](coding/oa-recipe-manager/README.md) · [Task manager](coding/oa-task-manager/README.md) · [File system](coding/oa-file-system/README.md) · [Cloud storage](coding/oa-cloud-storage/README.md) · [Office access and pay](coding/oa-worker-grant-management/README.md) |
| Recruiter screen | [Recruiter screen](behavioral/recruiter-screen/README.md) |
| Coding screens and onsite coding | [Image processing](coding/image-processing-pipeline/README.md) · [Duplicate files](coding/file-deduplication/README.md) · [Web crawler](coding/concurrent-web-crawler/README.md) · [Call stacks from samples](coding/stack-trace-reconstruction/README.md) · [LRU cache](coding/lru-cache-bugfix-durability/README.md) · [Distributed mode and median](coding/distributed-mode-median/README.md) · [Tokenizer](coding/longest-match-tokenizer/README.md) · [Boot program](coding/bootloader-loop-fix/README.md) |
| AI-assisted rounds | [Pull-request review with an AI assistant](coding/agentic-pr-review/README.md) · [Agent loop with tool use](coding/llm-agent-tool-loop/README.md) |
| System design | [Inference API](system-design/inference-api-batching/README.md) · [Design doc review](system-design/design-doc-review/README.md) · [Model weight distribution](system-design/model-weight-distribution/README.md) · [One-to-one chat](system-design/one-on-one-chat/README.md) · [Prompt playground](system-design/prompt-playground/README.md) · [Telemetry pipeline](system-design/telemetry-data-infrastructure/README.md) |
| Research screens | [Weighted data batcher](coding/weighted-data-batcher/README.md) · [ML configuration system](coding/ml-config-system/README.md) · [Prompting with LLMs](coding/prompting-with-llms/README.md) · [Attention with einsum](coding/qkv-attention-einsum/README.md) · [GRPO debugging](coding/grpo-debug/README.md) |
| Take-homes | [ML study](take-home/ml-take-home/README.md) · [Double descent](take-home/interp-double-descent/README.md) · [Kernel optimisation](take-home/performance-kernel-optimization/README.md) |
| Performance track | [Performance modelling](coding/matmul-performance-modeling/README.md) · [Kernel optimisation](take-home/performance-kernel-optimization/README.md) · [Distributed mode and median](coding/distributed-mode-median/README.md) |
| Fellows | [DNS resolver](coding/oa-dns-resolver/README.md) · [Banking system](coding/oa-bank-system/README.md) · [Inference engine](coding/oa-inference-engine/README.md) · [ML study](take-home/ml-take-home/README.md) |
| Onsite conversations | [Culture](behavioral/culture-ai-safety/README.md) · [Hiring manager](behavioral/hiring-manager/README.md) · [Project deep dive](behavioral/project-deep-dive/README.md) |

The [roadmap](ROADMAP.md) orders every page by track and priority.

## Official sources

- Careers, interview format, benefits and FAQ: <https://www.anthropic.com/careers>; open roles:
  <https://www.anthropic.com/careers/jobs>
- AI use during the hiring process: <https://www.anthropic.com/candidate-ai-guidance>
- Anthropic Fellows Program: <https://alignment.anthropic.com/2025/anthropic-fellows-program-2026/>,
  <https://job-boards.greenhouse.io/anthropic/jobs/5183044008>
- Performance take-home: <https://github.com/anthropics/original_performance_takehome>,
  <https://www.anthropic.com/engineering/AI-resistant-technical-evaluations>
- Applicant privacy notice (AU, JP, SG, KR, CH, UK, EU; October 2025):
  <https://www-cdn.anthropic.com/files/4zrzovbb/website/a9d372d660e3de4e3a0abfc0a19fa15f507d078d.pdf>
- Reading for the recruiter and culture rounds: [Core Views on AI Safety](https://www.anthropic.com/news/core-views-on-ai-safety)
  (March 2023) · [Responsible Scaling Policy](https://www.anthropic.com/rsp) (version 3.4, effective July 8, 2026) ·
  [Claude's constitution](https://www.anthropic.com/constitution) · [The Long-Term Benefit
  Trust](https://www.anthropic.com/news/the-long-term-benefit-trust) (September 2023) ·
  [Company](https://www.anthropic.com/company)
