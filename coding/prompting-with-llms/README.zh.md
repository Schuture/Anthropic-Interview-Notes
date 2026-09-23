# LLM 提示与工程：结构化输出与评测

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 提示词与评测 | ★★☆☆☆ | 中等 | RE · RS · MLE | prompt-design, structured-output, parsing, evaluation, error-analysis | 3 个部分 / 55 分钟 | 技术电面 |
<!-- meta:end -->

## 题目

笔记本（notebook）中定义了 `complete`，这是一个把一轮对话发给语言模型、并返回其纯文本回复的函数。

```py
def complete(system: str, prompt: str, temperature: float = 0.0) -> str:
    """Sends `prompt` to a language model under the given system instructions and returns its text
    reply. Calling complete with the same system and prompt at temperature 0.0 returns the same
    string every time it is called; above 0.0 it may return a different string on each call."""
```

任务是对收到的客服工单做分诊（triage）。一条*工单*（ticket）是一个带有 `id`、`text` 和 `label` 的字典；
`TICKETS` 是 28 条这样的工单组成的列表，`label` 是 `LABELS` 中四个字符串之一，定义如下。

- `billing`：涉及扣费、账单、支付方式、退款或订阅价格。
- `bug`：产品的行为不符合文档说明——报错、崩溃，或某个本该可用的控件失灵。
- `how_to`：发件人想知道如何使用产品已经支持的某项功能。
- `access`：发件人无法进入自己的账户——登录、密码、验证码，或账户所有权问题。

*平局裁定规则*（tie-break rule）：当一条工单的措辞同时涉及多个类别时，标签应该是客服此刻必须采取的行动，
而不是顺带提到的次要原因。如果工单最紧迫的问题是发件人无法登录，即使账单问题是其表述的起因，也应标为
`access`；如果工单以提问的形式描述某个本应可用的控件却无法使用，应标为 `bug`，而不是 `how_to`。

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

下面是同事第一次尝试评测分诊提示词（prompt）的代码。它运行时不会报错，也确实打印出一个数字，但那个数字是
错的：这段代码里恰好有四个 bug。

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

### Part 1 —— 提示词、输出约定与健壮的解析器

为这个任务设计一个系统提示词（system prompt）和一个解析模型回复的解析器，再把它们组合成 `classify`。

在系统提示词中写明：上面的四个标签定义和平局裁定规则；回复必须恰好包含一个 `<label>...</label>` 标签，其
内容是四个标签名之一，随后是一个 `<reason>...</reason>` 标签给出一句话的理由，此外不得包含任何别的内容；
以及一到两个采用同样标签格式的*示例*（worked example）——每个示例是一个带有 `text`、`label` 和 `reason`
的字典，向模型展示一条已经被正确分类的工单。每个示例都必须是一条不属于 `TICKETS` 中 28 条工单的工单——
如果示例取自被评测的工单，就等于把模型即将被再次问到的问题的答案提前告诉了它。

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

### Part 2 —— 评测框架

把打分和调用模型分开：`score` 从预测结果字典计算指标，不调用 `complete`；`evaluate` 把它和 `classify`
组合起来。

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

然后找出 bug。上面的流水线恰好有四个 bug：说明每一个 bug 是哪一行代码导致的、为什么是错的，并基于上面的
`classify` 和 `score` 给出修正后的流水线。修正后，它必须让 28 条工单每条都恰好被评测一次，绝不能把某条
正在被评测的工单本身用作示例展示给模型，并且无论标签周围有什么文字，都必须能正确识别出格式正确的回复。

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

### Part 3 —— 错误分析，以及判断什么才是真正的提升

读一读你的 Part 1 提示词在 `evaluate` 中判错的工单，针对每一条，依据上面的标签定义和平局裁定规则判断它
本该属于哪个类别，以及是什么样的措辞会让第一版提示词判断错误。针对你发现的模式，为提示词补充一到两个新
的示例——同样不能是 28 条工单中的任何一条——然后用扩充后的示例集重新运行 `evaluate`。

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

精确说明你会如何用 `paired_improvement` 的输出，判断 `EXEMPLARS_V2` 相对 `EXEMPLARS_V1` 是不是一次
真正的提升，而不是一个抛硬币也同样容易得到的差异。

## 参考解答

<details>
<summary>展开参考解答</summary>

有两点值得向面试官确认：解析失败在计算准确率时，应该直接算作一次错误预测（本文假设如此，这样提示词就
不能靠拒绝作答来提高准确率），还是应该从分母中剔除；以及示例是否可以取自被评测的集合本身（本文假设永
远不可以，原因见 Part 1）。

### Part 1

系统提示词逐字写明了上面的四个标签定义和平局裁定规则，确保模型看到的规则与金标签（gold label）标注时
所依据的规则完全一致；写明了准确的标签格式；并给出两个采用该格式的示例，两者都不在被评测的 28 条工单
之中。

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

`parse_reply` 用的是 `search` 而不是 `match`，所以即使回复把标签包在一句话中间，也依然能解析成功。题目
中给出的那段有 bug 的流水线享受不到这个好处：它的正则表达式要求整条回复除了标签内容之外什么都不能有，
而且即便是一个干净的 `how_to`，也会被它拒绝，因为字符类 `[a-z]+` 无法匹配下划线。

### Part 2

`score` 是一个只依赖真实标签和预测结果字典的纯函数；`evaluate` 是唯一调用 `classify` 的地方，`compare`
则在同一批工单上运行两次 `evaluate`。

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

题目中给出的那段代码恰好有四个 bug。`SYSTEM` 完全没有为回复规定任何格式，模型也就没有任何格式约定可
以遵循——这一点和解析过程无关，任何正则表达式都补救不了。`LABEL_RE = re.compile(r"^([a-z]+)$")` 要求
整条回复除了小写字母以外什么都不能有：只要回复多出一点点文字就会被拒绝，就连一个干净的 `how_to` 也会
被直接拒绝，因为 `[a-z]+` 无法匹配下划线。`examples = tickets[:2]` 把用作示例的工单取自正在被评测的
同一份列表，于是 `tickets[0]` 和 `tickets[1]` 会先被当作已经给出答案的示例展示给模型，紧接着又被拿来
重新提问——模型在这两条工单上的表现，说明不了它在一条从未见过答案的工单上会怎么做。最后，`for t in
tickets[:20]` 只评测了 28 条工单中的前 20 条，悄悄丢掉了最后 8 条（`t21` 到 `t28`——最后一条 `how_to`
工单和全部 `access` 工单），而 `return correct / len(tickets)` 却仍然除以全部 28 条；这样算出来的数字，
既漏掉了一整个标签的评测，也永远达不到真实的准确率，哪怕被评测的工单全部判对也是如此。

两个示例，覆盖四个标签中的两个，已经足够做出第一次测量：

```python
EXEMPLAR_A = {"text": "I was charged for the Pro plan but I'm still on the Free plan.",
              "label": "billing", "reason": "A charge does not match the plan the sender is actually on."}
EXEMPLAR_B = {"text": "How do I change my workspace name?",
              "label": "how_to", "reason": "Asks how to do something the product already supports."}
EXEMPLARS_V1 = [EXEMPLAR_A, EXEMPLAR_B]
```

`evaluate(TICKETS, EXEMPLARS_V1)` 在 28 条工单中判对 26 条（准确率 `26/28 ~= 0.929`），解析失败率为
`0.0`：只有一条工单（`t03`）用到了那一次重试，其余工单第一次就能解析成功。判错的两条是 `t08`（真实标
签 `bug`，预测成 `how_to`）和 `t22`（真实标签 `access`，预测成 `billing`）；其余工单全部正确，所以
`bug` 和 `access` 各自保住了满分精确率（precision），但召回率（recall）各损失一点（`6/7`），而
`billing` 和 `how_to` 各自保住了满分召回率，但精确率各损失一点（`7/8`）——这两次误判，分别成了它们被
误标到的那个标签的一次误报（false positive）。

### Part 3

`t08` 的文字问的是“怎么才能启用它”，表面上像是一个 `how_to` 问题，但它描述的控件本该可用——变灰正是
标签定义中 `bug` 所指的情形。`t22` 提到了一次被拒的扣款，这会把人拉向 `billing`，但这条工单此刻要求
的行动是重新获得登录权限，这正是 `access` 的情形。两次判错背后是同一个原因：基线提示词写明了平局裁定
规则，却从未示范过它——提示词里没有任何一处，展示了在这么含糊的工单上遵循这条规则应该是什么样子。

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

`evaluate(TICKETS, EXEMPLARS_V2)` 的准确率达到 `1.0`：`t08` 和 `t22` 都判对了，其余部分没有变化——
`t03` 的那一次重试照样发生，因为它只取决于格式约定和是否处于重试状态，与提供了哪些示例无关。比较这两
次结果，

```text
result_v1, result_v2 = evaluate(TICKETS, EXEMPLARS_V1), evaluate(TICKETS, EXEMPLARS_V2)
paired_improvement(TICKETS, result_v1, result_v2)
-> {"a_only": 0, "b_only": 2, "p_value": 0.5}
```

两次改动的方向完全一致（`b_only = 2`，`a_only = 0`），而且上面两个示例已经把原因解释得很清楚——但只
有两条工单意见不一致时，精确双侧符号检验（sign test）算出的 `p` 值永远不会低于 `0.5`，无论这两条工单
怎么分配都是如此，所以单凭这么少的不一致工单，配对检验永远无法独立判定这个差异是显著的。反复运行
`evaluate(TICKETS, EXEMPLARS_V1)`，每次得到的 `predictions` 字典完全相同，这证实了在温度 `0.0` 下，
这里没有任何东西是由采样噪声造成的；因此诚实的结论不是“不显著，所以可以忽略”，而是在只有两条工单意见
不一致的情况下，配对检验根本没有能力去确认那条已经直接展示出来的判断轨迹（trace）。只有当意见不一致
的工单足够多、且全部倒向同一边时，双侧符号检验才会降到 `0.05` 以下，而这至少需要 6 条意见不一致的工
单，而不是 2 条。温度高于 `0.0` 时所做的改动完全不能只靠一次运行来判断：对每个提示词的每条工单多跑几
次（例如 5 次），再比较两个提示词每条工单的平均正确率或多数投票结果，而不是可能只是运气使然的单次采
样。

### 追问

- 提示缓存（prompt caching）与成本：在同一次 `evaluate` 运行中，系统提示词和示例在每次调用里都完全
  相同；把它们放在固定的前缀里，只把每条工单各自的文字放在末尾，就能让缓存以远低于首次的成本命中这个
  前缀——当同一个评测框架要把同一种提示词形状调用成百上千次时，这一点最为关键。
- 温度与自洽性（self-consistency）：温度 `0.0` 能让一次评测运行可复现；实际部署的提示词则可以在更高
  的温度下多次采样，取多数票作为最终标签（自洽性），用 k 倍的调用次数换取那些单次采样容易随机判错的
  模糊工单上的几个百分点准确率。
- 用模型来打分：对于没有固定标签集的任务，常见做法是用第二次模型调用依据一份评分标准（rubric）给第一
  次的输出打分，而不是精确匹配打分；这比标注更多人工数据搭建得快，但它不是独立的证据——评分模型往往
  偏爱更长、语气更笃定的回答，而不管其是否正确，给和自己相似的模型打分时也会更宽松，所以只要任务允许，
  像本文这样针对固定标签集做精确匹配打分是更好的选择。
- 保持评测集干净：一旦某条工单的答案被提示词“见过”——无论是作为泄漏的示例，还是提示词的措辞就是针对
  这条工单反复试错调出来的——它就不再能衡量任何新东西；留一部分提示词作者从不查看的留出（held-out）
  工单，并定期换入新的工单，才能让前后两次的数字具有可比性。
- 什么时候该微调（fine-tune）而不是写提示词：当一个稳定不变的任务需要大量低延迟、低成本的调用，并且
  已标注样本多到足以把指标推得比最好的提示词还远——通常是成百上千条，而不是本文这 28 条——微调的固定
  成本才划算；如果任务的标签或指令还在变化，或者只有几十条样本，写提示词仍然更便宜，改一次只需要几分
  钟，不需要重新训练。

<details>
<summary>验证代码（可运行）</summary>

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