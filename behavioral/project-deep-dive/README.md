# Project Deep Dive: Presentation and Technical Q&A

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Presentation and Q&A | ★★★★★ | — | All | presentation, project-deep-dive, trade-offs, metrics, retrospective | 55 min | Onsite |
<!-- meta:end -->

## Problem

This is a 55-minute onsite round with one or more interviewers. You choose one past technical project —
one you drove yourself, in enough depth to sustain detailed follow-up — and present it for about 20–25
minutes, either from a short slide deck or by sharing your screen to walk through diagrams and code
directly; confirm which format your loop expects. A live demonstration of the running system is
uncommon. About 25–30 minutes of question-and-answer follow the presentation; a question can also
arrive during the presentation itself, before you reach the slide that would have answered it. The
questions fall into the following groups.

### Problem and context

- What problem existed before this project, and who had it — a specific team, a specific set of users, a
  specific metric that was stuck?
- What was in place before your project — nothing, a manual process, an older system — and what was
  wrong with it?
- Why was this the right project to take on at that time, rather than a year earlier, a year later, or a
  different problem entirely?
- What was explicitly out of scope, and why?

### Architecture and key decisions

- Walk through the architecture: the main components, and how a request or a piece of data moves through
  them.
- Which single decision in this design mattered most to how the project turned out?
- Where did you compromise on what you considered the ideal design, and because of what constraint —
  time, headcount, an existing system you had to integrate with?
- Which part of the design would you defend most strongly if it were pushed on?

### Trade-offs and alternatives rejected

- Why didn't you use a specific, plausible alternative — an existing internal system, a managed product,
  a simpler design?
- What did you know at the time that ruled that alternative out, as opposed to what you know now?
- Was there a simpler design that would have captured most of the benefit for a fraction of the effort?
- Under what condition would you make the opposite choice today?

### Your contribution versus the team's

- What did you personally design, build, or decide, distinct from what the rest of the team did?
- Who else worked on this project, and what did they own?
- For a part of the system you did not build yourself, what do you actually know about it — its
  interface, or its internals too?
- Who made the final call on the project's biggest decision — you, or someone else?

### Measurement and results

- What metric did you use to show the project worked, and why that metric rather than another one?
- What was the number before and after, and what was it measured against — a prior baseline, a control
  group, a rollback?
- Who defined that metric, and how was the target set?
- How do you know the change was caused by your project, and not by something else that changed at the
  same time?
- Translate the project's cost into a number and its measured return into a number: was it worth it?

### Failures and what you would do differently

- What was the hardest challenge you did not see coming, and how did you find out about it?
- What was the single biggest mistake in the project, and when did you realize it?
- If you started the project again today, what would you do differently?
- What have you carried into a later project because of what this one taught you?

### Scaling and operational concerns

- What happens to the system if load or data volume grows 10x? Which part breaks first?
- What was the actual bottleneck once the system was in production, and how did you find it?
- What is the failure mode when a specific resource is exhausted, and how does the system behave then?
- Who operated the system day to day after launch, and what did that cost?

### For research roles: experimental design

When the project is empirical research — a new method, a measured model behavior, a claim evaluated
against baselines — the deep dive adds this group.

- How was the comparison designed — what was held constant, what was varied, and what was the baseline?
- How do you know the result is a real effect and not noise, a leak between training and evaluation
  data, or an artifact of the evaluation setup?
- What would have shown your hypothesis to be wrong, and did you check for it?
- How many seeds, runs, or repetitions support the headline number, and what was the spread across them?

## Reference solution

<details>
<summary>Show the reference solution</summary>

Confirm two things with your recruiter before you prepare: how the 55 minutes actually splits if your
loop differs from the default, and whether the round expects a shared deck or a shared screen.
Everything below assumes the default split.

### Choosing the project

Pick a project you can go two or three follow-ups deep on every major decision in it, not only the one
you would lead with. Two further things matter more than how impressive the project sounds on a slide:
decisions that were actually yours rather than execution of someone else's design, and a measurable
outcome — a number that moved, not merely a system that shipped. A project that maps to the team's domain
is convenient but not required; a sharp deep dive on a tangential project holds up better under the Q&A
half than a shallow one on a project that only sounds relevant.

Confidential detail is not a reason to set aside an otherwise strong project — sanitize it instead.
Replace an exact absolute number with the relative change or the order of magnitude it represents ("cut
p99 latency by roughly 40%" in place of the millisecond figures on an internal dashboard), replace a
proprietary system's internal name with a description of what it does, and keep every piece of reasoning
and every trade-off intact — sanitizing changes which numbers and names appear, not the substance of what
you argue. If a follow-up asks for a number you have deliberately rounded or normalized this way, say so
directly rather than inventing a precise-sounding figure to match the rounded one.

### Slide outline

A slide-by-slide structure with time budgets that sum to about eighteen minutes, leaving slack inside the
20–25 minute window for whatever arrives mid-presentation before you reach the later slides.

- **Slide 1 (1 min): title and one-line summary** — the system, who used it, over what period. Not: your
  background or the team's org chart.
- **Slide 2 (2 min): problem and context** — what existed before the project, why this problem and why
  now, what was explicitly out of scope. Not: a full requirements document.
- **Slide 3 (3 min): architecture** — a diagram of the main components and one request or one piece of
  data traced through them. Not: every internal service, only what later slides refer back to.
- **Slides 4–5 (2 min each): two key decisions, one slide each** — the alternative you did not take and
  the constraint that ruled it out. Not: a list of technology names with no reasoning attached.
- **Slide 6 (2 min): your role** — precisely what you personally designed, built, or decided, distinct
  from what the rest of the team did. Not: a claim on work that was actually a teammate's.
- **Slide 7 (3 min): results** — the metric, the before/after numbers, what they are measured against,
  and the cost-versus-return figure if the project's scale makes ROI a likely question. Not: a number you
  cannot explain how you computed.
- **Slide 8 (3 min): what you'd change** — the single biggest mistake, and what breaks first at 10x. Not:
  a vague "add more monitoring."
- **Slide 9 (backup, on standby): further detail** — a fuller architecture diagram, a third decision you
  did not have room for on slides 4–5, secondary metrics, scaling numbers. Shown only if asked.

### Problem and context

What it probes: whether the motivation is a real problem that existed before you touched it, sized and
dated, rather than a justification assembled afterward for a project you wanted to build regardless.

Skeleton: name who had the problem, what they were doing about it before your project — nothing, a
manual step, a worse tool — one number that shows its size, why this particular moment made it worth
doing rather than a different time, and one thing you explicitly left out of scope and why.

Most common way to lose points: opening with the solution — "we built a system that..." — before the
problem has been named, sized, or dated, so the motivation reads as invented to fit what was already
built. Fix: rehearse the first ninety seconds so a specific person's problem, with a number attached,
appears before you say what you built.

### Architecture and key decisions

What it probes: whether you can trace a request or a unit of data through the real system from memory,
and whether every component in your own diagram is one you can open up on demand.

Skeleton: name the components in the order data or a request actually passes through them, trace one
concrete example end to end, and for each decision that mattered, have one sentence ready for why it was
made and what would have to change for you to decide differently today.

Most common way to lose points: a diagram with a box you cannot describe the interface of, because you
inherited it or only ever called it. Fix: before the round, mark on your own diagram which components you
built and which you only used, and prepare the one-sentence contract — input, output, guarantee — for
every box you did not build, instead of guessing at what is inside it.

### Trade-offs and alternatives rejected

What it probes: whether a decision was actually weighed against real alternatives, or was the only option
that was ever considered.

Skeleton: for each decision you expect to be pushed on, have the alternative named, the evidence or
constraint that ruled it out at the time — not in hindsight — and the condition under which you would
choose it today. When a follow-up proposes an alternative you had not prepared for, restate the
assumption that alternative depends on and check it against your actual problem before defending against
it; if the assumption does not hold in your case, say so and name what would have to be true for the
alternative to apply, rather than forcing a defense of a comparison that does not fit.

Fictional example, for a deploy pipeline that gates each rollout stage on live error-rate metrics before
advancing it:

| Alternative | Why not, at the time | Under what condition you'd switch |
| --- | --- | --- |
| Add the gate as a plugin to the existing CI tool | The CI tool's plugin API only saw build-time signals; it had no way to read the post-deploy metrics the gate needed to decide on | The CI tool added a post-deploy hook API |
| Have an on-call engineer manually approve each rollout stage | Deploys were frequent enough that manual approval became the bottleneck the gate was meant to remove, and different engineers approved by different, inconsistent standards | Deploy frequency dropped low enough that manual review stopped being the bottleneck |

Most common way to lose points: naming the choice with no alternative attached — "we used X" with no Y or
Z that was actually considered. Fix: before the round, write down every alternative you seriously weighed
for your two or three biggest decisions, one row each, with the deciding evidence rather than just the
alternative's name.

### Your contribution versus the team's

What it probes: whether "we" stands in for a division of labor you can state precisely — what was yours,
and just as clearly, what was not.

Skeleton: one sentence naming your own share, tied to something verifiable ("I designed and built the
ranking change; a teammate built the logging that measured it; the rollout call was the team's").

Most common way to lose points: an answer that is all "I," claiming work that was a teammate's, or all
"we," using the team as cover when asked for your specific share — both read as avoiding the question.
Fix: write the one sentence above before the round, naming your part, a named teammate's or role's part,
and what was a joint or someone else's call.

### Measurement and results

What it probes: whether the metric is well defined and who actually owns it, whether other explanations
for the number moving have been ruled out, and whether the project's return is defensible against its
cost.

Skeleton: define the metric and how it is computed; name who defined it and owns it — you, product, a
cross-functional group — and how the target was set; name what the result is measured against, a prior
baseline, a control group, or a rollback; name one plausible confound and why you believe it is not the
explanation; and if the project's scale invites an ROI question, translate the effort into a cost —
engineer-time at a fully loaded rate is a defensible unit, so fifty engineer-years at roughly $400K per
engineer-year is about $20M — and state what the measured return bought against that figure.

Most common way to lose points: a number you cannot explain the computation of, credit for a metric's
definition claimed as your own when it was actually negotiated with another function, or "it was worth
it" with no cost side to compare against. Fix: before the round, write the metric's formula, who signed
off on the target, what it is compared against, and a rough cost figure for the project, so the ROI
question has a rehearsed answer instead of an improvised one.

### Failures and what you would do differently

What it probes: a real cost you can name specifically, not a stylistic regret or a claim that nothing
went wrong.

Skeleton: the single biggest mistake and the moment you actually realized it, not the moment it became
convenient to admit; the hardest challenge you did not see coming and how you found out about it; two
concrete things you would do differently with hindsight; and one thing you have since carried into a
later project because of this one.

Most common way to lose points: "nothing, it went well," a trivial regret standing in for a real one, or
a hardest-challenge answer vague enough to read as secondhand. Fix: pick a mistake with a real, nameable
cost — rework, an incident, months of delay — have the specific moment you realized it ready, and name
the one pattern from it you have since reused.

### Scaling and operational concerns

What it probes: whether you can reason about the actual bottleneck with real numbers, and who was
responsible for the system once it was running.

Skeleton: name the resource that was the real bottleneck once the system was live and how you found it,
say what breaks first at 10x, name the concrete fix and where that fix hits its own new limit, and say
who operated the system day to day after launch and what that cost.

Most common way to lose points: a generic "we'd scale horizontally" with no resource named and no real
number from the actual system. Fix: memorize two or three real numbers from the project — current
throughput or volume, the actual bottleneck you hit, the cost of running it — before the round, and know
which one moves first if the others hold steady.

### For research roles: experimental design

What it probes: whether the result the project claims is actually established — a fair comparison, an
effect distinguishable from noise, checked for the ways it could be an artifact rather than real.

Skeleton: state what was held constant and what was varied, name the control or baseline the result is
measured against and how it was tuned, state how you distinguished a real effect from noise — a
significance test, or multiple seeds — and name one way the result could have been an artifact of the
evaluation itself, such as a leak between training and evaluation data, or a baseline that was not tuned
as carefully as the proposed method, and what you did to check for it.

Most common way to lose points: reporting a single run as the result with no measure of variance, or a
baseline that was never given a fair chance. Fix: before the round, have the exact comparison ready — the
same data, the same compute budget, the same tuning effort on both sides — and the specific check you ran
for the most likely artifact in your own setup.

### When you do not know the answer

Separate a genuine unknown from something you can estimate live. For a number you have not memorized, say
you do not have it exactly, then give the order of magnitude you are confident of or say how you would go
find it; for a component you did not build, the same honesty applies to its internals, not only its
interface. Never state a number you do not have as if you do: a stated gap costs you one question, while
an invented number that turns out wrong undermines every other number you gave in the round.

### Recovering time when a question arrives early

A follow-up during the presentation itself draws on the same budget as the slide you have not reached
yet. Mark two things on your own outline before the round: which slides carry content a later question
depends on — context, architecture, results, since skipping these leaves the rest of the Q&A without a
shared reference point — and which are compressible on the spot, such as one of the two decision slides
reduced to its one-line verdict, or the backup slide dropped entirely. When you compress, say so out loud
— "I'll fold the second decision into one line and move to results" — rather than letting the cut look
unplanned. A presentation that has not reached the results and retrospective slides by the middle of its
budget leaves no room to reach them, which shifts the follow-ups that would otherwise be about trade-offs
and results onto whatever you did cover instead.

### Prep outline

Copy this into `my/` and fill it in with your own project.

```text
Project in one sentence:
Format (slides or shared screen), confirmed with the recruiter:
Constraints that shaped the design (time / team size / existing systems / budget):

Slide budget (minutes):
  1. Title and summary:
  2. Problem and context:
  3. Architecture:
  4-5. Two key decisions:
  6. Your role:
  7. Results:
  8. What you'd change:
  9. Backup, shown only if asked:

Sanitizing:
  Absolute numbers replaced with relative change or order of magnitude:
  Proprietary names replaced with a description of function:

Key decision 1:
  Alternative:
  Why not, at the time:
  Under what condition you'd switch:

Key decision 2:
  Alternative:
  Why not, at the time:
  Under what condition you'd switch:

Key decision 3:
  Alternative:
  Why not, at the time:
  Under what condition you'd switch:

Your role in one sentence (yours / a named teammate's / the team's):

The metric:
  Definition and how it's computed:
  Who defined and owns it, and how the target was set:
  Compared against (baseline / control / rollback):
  A plausible confound, and why it isn't the explanation:
  Cost of the project and the return against it, if ROI is a likely question:

Failures:
  Biggest mistake, and when you actually realized it:
  Hardest unanticipated challenge, and how you found out:
  Two things you'd do differently with hindsight:
  What you've since carried into a later project:

Scale:
  Current throughput or volume:
  The actual bottleneck you hit, and how you found it:
  What breaks first at 10x, and the fix:
  Who operated it after launch, and what that cost:

If the project is research:
  Held constant / varied / baseline:
  How you distinguished signal from noise:
  The likeliest artifact in your setup, and how you checked for it:

Three parts you didn't build (the interface you'd state, not a guess):
  1.
  2.
  3.

Ten likely follow-ups, one line each, with a 60-second answer ready:
  1.
  2.
  3.
  4.
  5.
  6.
  7.
  8.
  9.
  10.
```

</details>
