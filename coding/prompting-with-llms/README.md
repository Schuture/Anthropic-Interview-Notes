# Prompting and Engineering with LLMs: Structured Output and Evaluation

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Coding · prompting and evaluation | ★★☆☆☆ | Medium | RE · RS · MLE | prompt-design, structured-output, parsing, evaluation, error-analysis | 3 parts / 55 min | Technical phone screen |
<!-- meta:end -->

## Problem

A notebook defines `complete`, a function that sends one turn to a language model and returns its
reply as plain text.

```py
def complete(system: str, prompt: str, temperature: float = 0.0) -> str:
    """Sends `prompt` to a language model under the given system instructions and returns its text
    reply. Calling complete with the same system and prompt at temperature 0.0 returns the same
    string every time it is called; above 0.0 it may return a different string on each call."""
```

The task is to triage incoming support tickets. A *ticket* is a dict with `id`, `text` and `label`;
`TICKETS` is a list of 28 such tickets, and `label` is one of the four strings in `LABELS`, defined
as follows.

- `billing`: about a charge, invoice, payment method, refund, or subscription price.
- `bug`: the product is not behaving as documented — an error, a crash, or a control that stops
  working.
- `how_to`: the sender wants to know how to do something the product already supports.
- `access`: the sender cannot get into their account — login, password, a verification code, or
  account ownership.

The *tie-break rule*: when a ticket's wording touches more than one category, its label is the
action support must take right now, not any secondary cause mentioned in passing. A ticket whose
immediate problem is that the sender cannot sign in is labelled `access` even when a billing issue
is the stated cause, and a ticket phrased as a question about a control that does not work is
labelled `bug`, not `how_to`, whenever that control is supposed to work already.

```python
LABELS = ["billing", "bug", "how_to", "access"]

TICKETS = [
    {"id": "t01", "text": "You charged me twice for my March subscription, can you refund the extra payment?", "label": "billing"},
    {"id": "t02", "text": "My invoice shows a $49 charge but my plan should only cost $29 a month.", "label": "billing"},
    {"id": "t03", "text": "I cancelled my subscription last week but was still billed today.", "label": "billing"},
    {"id": "t04", "text": "Can I get a receipt for tax purposes for my last three payments?", "label": "billing"},
    {"id": "t05", "text": "The annual plan discount wasn't applied at checkout, I paid full price.", "label": "billing"},
    {"id": "t06", "text": "I want to downgrade my plan and get a prorated refund for the unused days.", "label": "billing"},
    {"id": "t07", "text": "My credit card was charged in a currency I did not select at signup.", "label": "billing"},
    {"id": "t08", "text": "The export as CSV option is grayed out and I can't click it, how do I enable it?", "label": "bug"},
    {"id": "t09", "text": "Every time I upload a PNG larger than 2MB the page just goes blank.", "label": "bug"},
    {"id": "t10", "text": "Search results are missing items that were there yesterday, nothing was deleted.", "label": "bug"},
    {"id": "t11", "text": "The dashboard chart shows negative numbers for a metric that can never be negative.", "label": "bug"},
    {"id": "t12", "text": "Dark mode resets to light mode every time I refresh the page.", "label": "bug"},
    {"id": "t13", "text": "Saving a draft sometimes duplicates it into two identical drafts.", "label": "bug"},
    {"id": "t14", "text": "The mobile app crashes immediately after I tap the notifications bell.", "label": "bug"},
    {"id": "t15", "text": "How do I invite a teammate to my workspace?", "label": "how_to"},
    {"id": "t16", "text": "Is there a way to schedule a report to send every Monday morning?", "label": "how_to"},
    {"id": "t17", "text": "Where do I change the default currency shown on invoices?", "label": "how_to"},
    {"id": "t18", "text": "How can I export my project data as a CSV file?", "label": "how_to"},
    {"id": "t19", "text": "What's the difference between the viewer and editor roles?", "label": "how_to"},
    {"id": "t20", "text": "How do I set up a keyboard shortcut for creating a new task?", "label": "how_to"},
    {"id": "t21", "text": "Can you point me to where I turn off email notifications for comments?", "label": "how_to"},
    {"id": "t22", "text": "I can't log in because it says my payment failed and my account is suspended until I update my card.", "label": "access"},
    {"id": "t23", "text": "My account got locked after too many failed login attempts, please unlock it.", "label": "access"},
    {"id": "t24", "text": "Two-factor authentication won't accept the code from my authenticator app.", "label": "access"},
    {"id": "t25", "text": "I'm logged out every few minutes even though I checked 'stay signed in'.", "label": "access"},
    {"id": "t26", "text": "I forgot my password and the reset email never arrives.", "label": "access"},
    {"id": "t27", "text": "Single sign-on redirects me back to the login page without any error.", "label": "access"},
    {"id": "t28", "text": "My teammate left the company and I need to be added as the new account owner.", "label": "access"},
]
```

A colleague's first attempt at evaluating a triage prompt is below. It runs without raising an
exception, and it prints a number, but that number is wrong: the cell has exactly four bugs.

```py
LABELS = ["billing", "bug", "how_to", "access"]
SYSTEM = "You are a support triage assistant. Read the ticket and decide which category it belongs to."


def build_prompt(ticket_text, examples):
    shots = "\n\n".join(f"Ticket: {ex['text']}\nLabel: {ex['label']}" for ex in examples)
    return f"{shots}\n\nTicket: {ticket_text}\nLabel:"


LABEL_RE = re.compile(r"^([a-z]+)$")


def parse_label(reply):
    match = LABEL_RE.match(reply.strip())
    return match.group(1) if match else None


def run_eval(tickets):
    examples = tickets[:2]
    correct = 0
    for t in tickets[:20]:
        reply = complete(system=SYSTEM, prompt=build_prompt(t["text"], examples))
        pred = parse_label(reply)
        if pred == t["label"]:
            correct += 1
    return correct / len(tickets)


print(f"accuracy: {run_eval(TICKETS):.2%}")
```

### Part 1 — Prompt, output contract and a robust parser

Design a system prompt for this task and a parser for the model's reply, then combine them into
`classify`.

State in the system prompt: the four label definitions and the tie-break rule above; that the reply
must contain exactly one `<label>...</label>` tag whose content is one of the four label names,
followed by one `<reason>...</reason>` tag with a short justification, and nothing else; and one or
two *worked examples* in that same tag format — each a dict with `text`, `label` and `reason`, shown
to the model as a demonstration of a ticket already correctly classified. Every worked example must
be a ticket that is never one of the 28 in `TICKETS` — an example drawn from the tickets being scored
would show the model the answer to a question it is about to be asked again.

```py
def parse_reply(reply: str) -> dict | None:
    """Extracts a prediction from one raw reply. Finds the first `<label>...</label>` tag anywhere
    in `reply` (any text before, between or after the tags is ignored); if none is present, or its
    content -- stripped of surrounding whitespace and compared case-insensitively -- is not one of
    LABELS, returns None. Otherwise returns {"label": <the matched label, lower-cased>, "reason":
    the content of the first `<reason>...</reason>` tag if present, else ""}."""


def classify(ticket_text: str, examples: list[dict], temperature: float = 0.0) -> dict:
    """Builds a prompt from `ticket_text` and the worked `examples` (each a dict with "text",
    "label" and "reason"), calls complete(), and parses the reply with parse_reply. If parsing
    fails, calls complete() exactly once more, with a prompt that includes the failed reply and
    states why it did not parse, and parses that second reply instead. Returns {"label": ... (None
    if both attempts fail to parse), "reason": ... ("" if label is None), "retried": True iff a
    second call was made}."""
```

```text
ticket_text = "Can I get a receipt for tax purposes for my last three payments?"

# first call: the model answers in prose instead of the required tags
complete(system, prompt) -> "This is a billing question -- they want a receipt for past payments."
parse_reply(...) -> None                                    # no <label> tag anywhere

# classify() retries once, with the parse failure fed back into a new prompt
complete(system, retry_prompt) -> ('<label>billing</label>\n'
                                    '<reason>Requests a receipt for past payments.</reason>')
parse_reply(...) -> {"label": "billing", "reason": "Requests a receipt for past payments."}

classify(ticket_text, examples) -> {"label": "billing",
                                     "reason": "Requests a receipt for past payments.",
                                     "retried": True}
```

### Part 2 — An evaluation harness

Separate scoring from calling the model: `score` computes metrics from a predictions dict with no
calls to `complete`, and `evaluate` combines it with `classify`.

```py
def score(tickets: list[dict], predictions: dict[str, str | None]) -> dict:
    """Pure function of `tickets` (using each entry's "id" and "label") and `predictions` (ticket id
    -> predicted label, or None for a ticket classify() could not parse). Returns:
    "accuracy": (# tickets whose prediction equals their true label) / len(tickets) -- a missing or
      None prediction is never equal to any label, so it counts as incorrect, not excluded;
    "parse_failure_rate": (# tickets with a None prediction) / len(tickets);
    "per_label": {label: {"precision": ..., "recall": ...} for label in LABELS}, using the standard
      multi-class definitions (precision is 0.0 for a label that was never predicted; a None
      prediction can lower a label's recall but is never a false positive for any label, since it
      was not predicted as anything)."""


def evaluate(tickets: list[dict], examples: list[dict], temperature: float = 0.0) -> dict:
    """Classifies every ticket in `tickets` with classify(ticket["text"], examples, temperature),
    then returns score(tickets, predictions) with two keys added: "predictions" (the dict used) and
    "retry_count" (the number of tickets whose classify() call had "retried": True)."""


def compare(tickets: list[dict], examples_a: list[dict], examples_b: list[dict],
            temperature: float = 0.0) -> dict:
    """Runs evaluate(tickets, examples_a, temperature) and evaluate(tickets, examples_b,
    temperature) on the same tickets. Returns {"a": <first result>, "b": <second result>,
    "changed": sorted list of ticket ids where the two predicted labels differ}."""
```

Then find the bugs. The pipeline shown above has exactly four of them: state each one (which line
causes it and why it is wrong) and give a corrected pipeline built from `classify` and `score`
above. Fixed, it must score every one of the 28 tickets exactly once, must never show the model a
worked example that is also one of the tickets being scored, and must accept a correctly tagged
reply regardless of what text surrounds the tags.

```text
tickets = [{"id": "x1", "label": "billing"}, {"id": "x2", "label": "billing"},
           {"id": "x3", "label": "bug"},      {"id": "x4", "label": "bug"},
           {"id": "x5", "label": "how_to"},   {"id": "x6", "label": "access"}]
predictions = {"x1": "billing", "x2": "access", "x3": "bug", "x4": None,
               "x5": "how_to", "x6": "access"}
# x2 is wrong (predicted access, true billing); x4 failed to parse (true bug); the other four are correct

score(tickets, predictions) -> {
    "accuracy": 4 / 6,                                    # x1, x3, x5, x6 correct
    "parse_failure_rate": 1 / 6,                          # x4
    "per_label": {
        "billing": {"precision": 1.0, "recall": 0.5},     # the 1 ticket predicted "billing" (x1) is right,
                                                            # but only 1 of the 2 true billing tickets was recalled
        "bug":     {"precision": 1.0, "recall": 0.5},      # x4's parse failure costs bug a recall point
        "how_to":  {"precision": 1.0, "recall": 1.0},
        "access":  {"precision": 0.5, "recall": 1.0},      # x2's wrong guess costs access its precision
    },
}
```

### Part 3 — Error analysis and deciding what is a real improvement

Read the tickets `evaluate` gets wrong with your Part 1 prompt, and for each one, decide from the
label definitions and the tie-break rule which category it should have been, and why the wording
could mislead a first pass. Add one or two further worked examples to the prompt that address the
pattern you find — never an example that is one of the 28 tickets — and re-run `evaluate` with the
enlarged example set.

```py
def paired_improvement(tickets: list[dict], result_a: dict, result_b: dict) -> dict:
    """Compares two evaluate() results computed on the same tickets. Among tickets where exactly one
    side predicted the true label, counts "a_only" (a correct, b incorrect) and "b_only" (b correct,
    a incorrect). Returns {"a_only": ..., "b_only": ..., "p_value": the two-sided exact binomial
    (sign) test p-value for a_only against n = a_only + b_only trials at p = 0.5 -- the probability,
    if the two prompts were equally good and every discordant ticket were a coin flip, of a split at
    least this lopsided; 1.0 when a_only + b_only == 0}."""
```

```text
tickets = [{"id": "y1", "label": "bug"}, {"id": "y2", "label": "access"}, {"id": "y3", "label": "billing"}]
result_a = {"predictions": {"y1": "how_to", "y2": "billing", "y3": "billing"}}   # wrong on y1, y2
result_b = {"predictions": {"y1": "bug",    "y2": "access",  "y3": "billing"}}   # right on all three

paired_improvement(tickets, result_a, result_b)
-> {"a_only": 0, "b_only": 2, "p_value": 0.5}
# both disagreements favor b and none favor a -- but with only two disagreements, the exact
# two-sided sign test can never report p below 0.5, whichever way a two-item split goes
```

State precisely how you would use `paired_improvement`'s output here to decide whether
`EXEMPLARS_V2` is a genuine improvement over `EXEMPLARS_V1`, and not a difference a coin flip
could as easily have produced.

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two points worth confirming with the interviewer: whether a parse failure should simply count as an
incorrect prediction in accuracy (assumed here, so a prompt cannot raise its accuracy by refusing to
answer) or be excluded from the denominator; and whether a worked example may ever be drawn from the
set being scored (assumed never, for the reason Part 1 states).

### Part 1

The system prompt states the four label definitions and the tie-break rule verbatim, so the model
sees the same rule the gold labels were assigned by; the exact tag format; and two worked examples
in that format, neither of them one of the 28 scored tickets.

```python
import re

LABELS = ["billing", "bug", "how_to", "access"]

SYSTEM_PROMPT = (
    "You triage incoming support tickets into exactly one of four categories.\n\n"
    "billing: about a charge, invoice, payment method, refund, or subscription price.\n"
    "bug: the product is not behaving as documented -- an error, a crash, or a control that stops working.\n"
    "how_to: the sender wants to know how to do something the product already supports.\n"
    "access: the sender cannot get into their account -- login, password, a verification code, or account ownership.\n\n"
    "When a ticket touches more than one category, label it by the action support must take right now, "
    "not by any secondary cause mentioned in passing.\n\n"
    "Reply with exactly one <label>...</label> tag containing one of billing, bug, how_to, access, "
    "followed by one <reason>...</reason> tag with a one-sentence justification. Do not use any other "
    "tag and do not repeat the ticket text."
)


def render_example(ex: dict) -> str:
    return f"Ticket: {ex['text']}\n<label>{ex['label']}</label>\n<reason>{ex['reason']}</reason>"


def build_user_prompt(ticket_text: str, examples: list[dict]) -> str:
    blocks = [render_example(ex) for ex in examples] + [f"Ticket: {ticket_text}"]
    return "\n\n".join(blocks)


LABEL_TAG = re.compile(r"<label>\s*(.*?)\s*</label>", re.S)
REASON_TAG = re.compile(r"<reason>\s*(.*?)\s*</reason>", re.S)


def parse_reply(reply: str) -> dict | None:
    m = LABEL_TAG.search(reply)                 # NOTE: search, not match -- tolerates text around the tags
    if not m:
        return None
    label = m.group(1).strip().lower()
    if label not in LABELS:                     # NOTE: validated against the label set, not just "looks like a tag"
        return None
    r = REASON_TAG.search(reply)
    return {"label": label, "reason": r.group(1).strip() if r else ""}


def _parse_complaint(reply: str) -> str:
    return ("Your reply could not be parsed: no valid <label> tag was found, or its content was not "
            "one of " + ", ".join(LABELS) + ".")


def classify(ticket_text: str, examples: list[dict], temperature: float = 0.0) -> dict:
    prompt = build_user_prompt(ticket_text, examples)
    reply = complete(SYSTEM_PROMPT, prompt, temperature)
    parsed = parse_reply(reply)
    retried = False
    if parsed is None:                           # NOTE: exactly one retry, with the failure fed back verbatim
        retried = True
        retry_prompt = (f"{prompt}\n\nYour previous reply was:\n{reply}\n\n{_parse_complaint(reply)} "
                         "Reply again, using exactly the tag format above and nothing else.")
        parsed = parse_reply(complete(SYSTEM_PROMPT, retry_prompt, temperature))
    if parsed is None:
        return {"label": None, "reason": "", "retried": retried}
    return {"label": parsed["label"], "reason": parsed["reason"], "retried": retried}
```

`parse_reply` uses `search`, not `match`, so a reply that wraps the tags in a sentence still parses.
The flawed pipeline in the Problem statement never gets this benefit: its regex requires the entire
reply to be nothing else, and it rejects a clean `how_to` outright too, since a character class of
`[a-z]+` cannot match an underscore.

### Part 2

`score` is a pure function of the true labels and a predictions dict; `evaluate` is the only place
that calls `classify`, and `compare` runs `evaluate` twice on the same tickets.

```python
def score(tickets: list[dict], predictions: dict) -> dict:
    n = len(tickets)
    correct = sum(1 for t in tickets if predictions.get(t["id"]) == t["label"])
    per_label = {}
    for lab in LABELS:
        tp = sum(1 for t in tickets if t["label"] == lab and predictions.get(t["id"]) == lab)
        pred_n = sum(1 for t in tickets if predictions.get(t["id"]) == lab)
        true_n = sum(1 for t in tickets if t["label"] == lab)
        per_label[lab] = {"precision": tp / pred_n if pred_n else 0.0,   # NOTE: 0.0, not a division error
                           "recall": tp / true_n if true_n else 0.0}
    return {
        "accuracy": correct / n,
        "parse_failure_rate": sum(1 for t in tickets if predictions.get(t["id"]) is None) / n,
        "per_label": per_label,
    }


def evaluate(tickets: list[dict], examples: list[dict], temperature: float = 0.0) -> dict:
    predictions, retry_count = {}, 0
    for t in tickets:
        out = classify(t["text"], examples, temperature)
        predictions[t["id"]] = out["label"]
        retry_count += out["retried"]
    result = score(tickets, predictions)
    result["predictions"], result["retry_count"] = predictions, retry_count
    return result


def compare(tickets: list[dict], examples_a: list[dict], examples_b: list[dict],
            temperature: float = 0.0) -> dict:
    a = evaluate(tickets, examples_a, temperature)
    b = evaluate(tickets, examples_b, temperature)
    changed = sorted(t["id"] for t in tickets if a["predictions"][t["id"]] != b["predictions"][t["id"]])
    return {"a": a, "b": b, "changed": changed}
```

The cell in the Problem statement has exactly four bugs. `SYSTEM` never states a format for the
reply, so the model has no contract to follow at all — independent of parsing, and no regex fixes
it. `LABEL_RE = re.compile(r"^([a-z]+)$")` requires the entire reply to be nothing but lower-case
letters: it rejects any reply with the smallest amount of surrounding text, and it rejects a clean
`how_to` outright, since `[a-z]+` cannot match an underscore. `examples = tickets[:2]` draws the
worked examples from the very list being scored, so `tickets[0]` and `tickets[1]` are shown to the
model, answered, immediately before being asked about again — whatever the model does with those two
tickets proves nothing about a ticket it has not already seen the answer to. Finally, `for t in
tickets[:20]` scores only the first 20 of the 28 tickets, silently dropping the last 8 (`t21` through
`t28` — the last `how_to` ticket and every `access` ticket), while `return correct / len(tickets)`
still divides by all 28; the reported number both omits an entire label from scoring and cannot reach
the true accuracy even if every scored ticket were right.

Two worked examples, covering two of the four labels, are enough to reach a first measurement:

```python
EXEMPLAR_A = {"text": "I was charged for the Pro plan but I'm still on the Free plan.",
              "label": "billing", "reason": "A charge does not match the plan the sender is actually on."}
EXEMPLAR_B = {"text": "How do I change my workspace name?",
              "label": "how_to", "reason": "Asks how to do something the product already supports."}
EXEMPLARS_V1 = [EXEMPLAR_A, EXEMPLAR_B]
```

`evaluate(TICKETS, EXEMPLARS_V1)` scores 26 of the 28 tickets correctly (accuracy `26/28 ~= 0.929`),
with a parse-failure rate of `0.0`: one ticket (`t03`) needs its one retry, and every other ticket
parses on the first attempt. The two misses are `t08` (true `bug`, predicted `how_to`) and `t22`
(true `access`, predicted `billing`); every other ticket is correct, so `bug` and `access` each keep
perfect precision but lose one point of recall (`6/7`), while `billing` and `how_to` each keep
perfect recall but lose one point of precision (`7/8`) — the two misclassifications land as false
positives for the labels they were wrongly assigned to.

### Part 3

`t08`'s text asks "how do I enable it", which reads like a `how_to` question on the surface, but the
control it describes is supposed to work already — greyed out is exactly the `bug` case the label
definition names. `t22` mentions a declined payment, which pulls toward `billing`, but the action the
ticket asks for right now is to be let back in, which is the `access` case. Both misses share one
cause: the baseline prompt states the tie-break rule but never demonstrates it, so nothing in the
prompt shows what following that rule looks like on a ticket this ambiguous.

```python
EXEMPLAR_C = {"text": "The 'Share' button is disabled and I can't invite anyone to the document.",
              "label": "bug",
              "reason": "A control that should work is disabled; that is broken behavior, not a request for instructions."}
EXEMPLAR_D = {"text": "I can't sign in because the system says my trial expired and my workspace is frozen.",
              "label": "access",
              "reason": "The action needed right now is regaining entry to the account, even though the cause is billing-related."}
EXEMPLARS_V2 = EXEMPLARS_V1 + [EXEMPLAR_C, EXEMPLAR_D]
```

```python
from scipy.stats import binomtest


def paired_improvement(tickets: list[dict], result_a: dict, result_b: dict) -> dict:
    a_only = b_only = 0
    for t in tickets:
        a_correct = result_a["predictions"][t["id"]] == t["label"]
        b_correct = result_b["predictions"][t["id"]] == t["label"]
        if a_correct and not b_correct:
            a_only += 1
        elif b_correct and not a_correct:
            b_only += 1
    n = a_only + b_only
    p_value = 1.0 if n == 0 else binomtest(min(a_only, b_only), n, 0.5, alternative="two-sided").pvalue
    return {"a_only": a_only, "b_only": b_only, "p_value": p_value}
```

`evaluate(TICKETS, EXEMPLARS_V2)` reaches accuracy `1.0`: both `t08` and `t22` are now correct, and
nothing else changes — the same one retry on `t03` still happens, since it depends only on the
format contract and the retry state, never on which examples are offered. Comparing the two runs,

```text
result_v1, result_v2 = evaluate(TICKETS, EXEMPLARS_V1), evaluate(TICKETS, EXEMPLARS_V2)
paired_improvement(TICKETS, result_v1, result_v2)
-> {"a_only": 0, "b_only": 2, "p_value": 0.5}
```

Both changes go the same way (`b_only = 2`, `a_only = 0`), and the mechanism is fully explained by
the two examples above — yet the exact two-sided sign test on only two discordant tickets can never
fall below `p = 0.5`, whichever way a two-item split lands, so no paired test run on this few
discordant items could ever call the difference significant by itself. Repeating
`evaluate(TICKETS, EXEMPLARS_V1)` returns an identical `predictions` dict every time, confirming that
at temperature `0.0` nothing here is left to sampling noise; the honest conclusion is not "not
significant, so ignore it" but that a two-item paired test has no power to confirm what the worked
trace already shows directly. A p-value only becomes informative here once enough tickets disagree
for the two-sided sign test to drop below `0.05`, which — with every disagreement favoring the same
side — takes at least 6 discordant tickets, not 2. A prompt change made at temperature above `0.0`
cannot lean on a single run at all: repeat every ticket several times per prompt (for instance 5) and
pair the two prompts' mean per-ticket correctness, or their majority label, rather than one sample
that could be a matter of luck rather than the prompt.

### Follow-ups

- Prompt caching and cost: the system prompt and worked examples are identical on every call within
  one `evaluate` run; keeping them as a fixed prefix and putting only the per-ticket text at the end
  lets a cache serve that prefix at a fraction of its first cost on every call after the first, which
  matters most exactly when a harness calls the same prompt shape hundreds of times.
- Temperature and self-consistency: temperature `0.0` keeps an evaluation run reproducible; a
  deployed prompt can instead sample several times at a higher temperature and take the majority
  label (self-consistency), trading k times the calls for a few points of accuracy on the ambiguous
  items a single sample would get wrong at random.
- Grading with a model: a task without a fixed label set often uses a second model call to grade the
  first's output against a rubric instead of exact-match scoring; this is faster to set up than more
  human labels, but it is not independent evidence — a grader tends to favor longer or more
  confident-sounding answers regardless of correctness, and grades its own kind of model more
  generously, so exact-match scoring against a fixed label set, as here, is preferred whenever the
  task allows it.
- Keeping an eval set clean: once a specific ticket's answer has been seen by the prompt — as a
  leaked few-shot example, or through wording tuned by trial and error against that same ticket — it
  stops measuring anything new; a held-out slice that the prompt's author never reads, refreshed with
  new tickets periodically, is what keeps later numbers comparable to earlier ones.
- When to fine-tune instead: fine-tuning pays for itself once one stable task needs a large volume of
  low-latency, cheap calls and enough labeled examples exist to move the metric further than the best
  prompt reaches — typically hundreds to thousands, not the 28 tickets here; a task whose labels or
  instructions still change, or that has only a few dozen examples, stays cheaper to iterate on as a
  prompt, which can be updated in minutes rather than a retraining run.

<details>
<summary>Checks (runnable)</summary>

```python
# The checks below replace complete() with a deterministic scripted stand-in: it never calls a
# language model. It decides its reply from simple, fixed rules -- whether the system prompt states
# the tag format, whether this call is a retry, and, for two specific ambiguous tickets, whether a
# relevant worked example has been shown -- so that the parser, the retry, the metrics and the Part 3
# improvement are all exactly measurable. It stands in for complete() only here, never for the
# interface described in the Problem statement.
FLAKY_TICKET_ID = "t03"
CONFUSABLE = {"t08": ("how_to", "disabled"), "t22": ("billing", "billing-related")}
_CALL_LOG = []


def _find_ticket(prompt: str) -> dict:
    hits = [t for t in TICKETS if t["text"] in prompt]
    assert len(hits) == 1
    return hits[0]


def complete(system: str, prompt: str, temperature: float = 0.0) -> str:
    _CALL_LOG.append(1)
    has_format = "<label>" in system
    is_retry = "could not be parsed" in prompt
    ticket = _find_ticket(prompt)
    predicted = ticket["label"]
    if ticket["id"] in CONFUSABLE:
        wrong, trigger = CONFUSABLE[ticket["id"]]
        if trigger not in (system + "\n" + prompt).lower():
            predicted = wrong
    reason = f"scripted reason for {predicted}"
    if not has_format:
        return f"I'd call this a {predicted} issue. {reason}"
    if ticket["id"] == FLAKY_TICKET_ID and not is_retry:
        return f"label: {predicted}\nreason: {reason}"
    return f"<label>{predicted}</label>\n<reason>{reason}</reason>"


# --- Part 1: parse_reply, exactly on the trace given in the Problem statement, plus edge cases ---
assert parse_reply("This is a billing question -- they want a receipt for past payments.") is None
assert parse_reply("<label>billing</label>\n<reason>Requests a receipt for past payments.</reason>") == \
    {"label": "billing", "reason": "Requests a receipt for past payments."}
assert parse_reply("Sure! <label>Bug</label> <reason>ok</reason> Thanks!") == {"label": "bug", "reason": "ok"}
assert parse_reply("<label>bug</label><label>billing</label>") == {"label": "bug", "reason": ""}
assert parse_reply("<label>spam</label>") is None                    # not one of LABELS
assert parse_reply("no tags in this reply at all") is None

# classify(): the flaky ticket needs its one retry, an ordinary ticket does not
_CALL_LOG.clear()
out_flaky = classify(TICKETS[2]["text"], EXEMPLARS_V1)                # t03
assert out_flaky == {"label": "billing", "reason": "scripted reason for billing", "retried": True}
assert len(_CALL_LOG) == 2

_CALL_LOG.clear()
out_plain = classify(TICKETS[3]["text"], EXEMPLARS_V1)                # t04
assert out_plain == {"label": "billing", "reason": "scripted reason for billing", "retried": False}
assert len(_CALL_LOG) == 1

# classify(): with no format contract at all, the retry cannot save it either
_GOOD_SYSTEM_PROMPT = SYSTEM_PROMPT
SYSTEM_PROMPT = "You triage incoming support tickets. Decide which of billing, bug, how_to, access fits best."
assert "<label>" not in SYSTEM_PROMPT
_CALL_LOG.clear()
out_bad = classify(TICKETS[0]["text"], EXEMPLARS_V1)
assert out_bad == {"label": None, "reason": "", "retried": True}
assert len(_CALL_LOG) == 2
bad_format_result = evaluate(TICKETS, EXEMPLARS_V1)
assert bad_format_result["parse_failure_rate"] == 1.0
assert bad_format_result["retry_count"] == 28
SYSTEM_PROMPT = _GOOD_SYSTEM_PROMPT

# --- the four bugs of the Problem statement's flawed cell, demonstrated directly ---
FLAWED_LABEL_RE = re.compile(r"^([a-z]+)$")
assert FLAWED_LABEL_RE.match("how_to") is None                       # a clean, correct label is still rejected
assert parse_reply("<label>how_to</label><reason>ok</reason>")["label"] == "how_to"


def _flawed_build_prompt(ticket_text, examples):
    shots = "\n\n".join(f"Ticket: {ex['text']}\nLabel: {ex['label']}" for ex in examples)
    return f"{shots}\n\nTicket: {ticket_text}\nLabel:"


_leaked_prompt = _flawed_build_prompt(TICKETS[0]["text"], TICKETS[:2])
assert _leaked_prompt.count(TICKETS[0]["text"]) == 2                  # once as the "example", once as the question

_scored_by_flawed_cell = TICKETS[:20]
assert not any(t["label"] == "access" for t in _scored_by_flawed_cell)
assert sum(1 for t in TICKETS if t["label"] == "access") == 7         # all 7 dropped, none of them scored

# --- Part 2: score() on the toy example given in the Problem statement ---
_toy_tickets = [{"id": "x1", "label": "billing"}, {"id": "x2", "label": "billing"},
                {"id": "x3", "label": "bug"}, {"id": "x4", "label": "bug"},
                {"id": "x5", "label": "how_to"}, {"id": "x6", "label": "access"}]
_toy_predictions = {"x1": "billing", "x2": "access", "x3": "bug", "x4": None,
                    "x5": "how_to", "x6": "access"}
_toy_result = score(_toy_tickets, _toy_predictions)
assert _toy_result["accuracy"] == 4 / 6
assert _toy_result["parse_failure_rate"] == 1 / 6
assert _toy_result["per_label"] == {
    "billing": {"precision": 1.0, "recall": 0.5},
    "bug": {"precision": 1.0, "recall": 0.5},
    "how_to": {"precision": 1.0, "recall": 1.0},
    "access": {"precision": 0.5, "recall": 1.0},
}

# --- Part 2: evaluate() with the baseline prompt (EXEMPLARS_V1) over the full 28 tickets ---
result_v1 = evaluate(TICKETS, EXEMPLARS_V1)
assert result_v1["accuracy"] == 26 / 28
assert result_v1["parse_failure_rate"] == 0.0
assert result_v1["retry_count"] == 1
assert result_v1["predictions"]["t08"] == "how_to"                    # wrong: true label is "bug"
assert result_v1["predictions"]["t22"] == "billing"                   # wrong: true label is "access"
assert result_v1["per_label"] == {
    "billing": {"precision": 7 / 8, "recall": 1.0},
    "bug": {"precision": 1.0, "recall": 6 / 7},
    "how_to": {"precision": 7 / 8, "recall": 1.0},
    "access": {"precision": 1.0, "recall": 6 / 7},
}

# repeating the same call at temperature 0.0 changes nothing
assert evaluate(TICKETS, EXEMPLARS_V1)["predictions"] == result_v1["predictions"]

# --- Part 3: evaluate() with the enlarged prompt (EXEMPLARS_V2) ---
result_v2 = evaluate(TICKETS, EXEMPLARS_V2)
assert result_v2["accuracy"] == 1.0
assert result_v2["parse_failure_rate"] == 0.0
assert result_v2["retry_count"] == 1                                  # t03's retry is unaffected by the examples
assert all(v == {"precision": 1.0, "recall": 1.0} for v in result_v2["per_label"].values())

cmp = compare(TICKETS, EXEMPLARS_V1, EXEMPLARS_V2)
assert cmp["changed"] == ["t08", "t22"]

# --- Part 3: paired_improvement(), on the toy example given in the Problem statement ---
_y_tickets = [{"id": "y1", "label": "bug"}, {"id": "y2", "label": "access"}, {"id": "y3", "label": "billing"}]
_y_result_a = {"predictions": {"y1": "how_to", "y2": "billing", "y3": "billing"}}
_y_result_b = {"predictions": {"y1": "bug", "y2": "access", "y3": "billing"}}
assert paired_improvement(_y_tickets, _y_result_a, _y_result_b) == {"a_only": 0, "b_only": 2, "p_value": 0.5}

# --- Part 3: paired_improvement() on the real V1-vs-V2 comparison ---
pi = paired_improvement(TICKETS, result_v1, result_v2)
assert pi == {"a_only": 0, "b_only": 2, "p_value": 0.5}

# the general fact behind the Part 3 discussion: at p = 0.5, an all-one-sided split needs at least
# 6 discordant tickets before the two-sided exact sign test drops below 0.05
assert binomtest(0, 5, 0.5, alternative="two-sided").pvalue > 0.05
assert binomtest(0, 6, 0.5, alternative="two-sided").pvalue < 0.05

print("all checks passed")
```

</details>

</details>
