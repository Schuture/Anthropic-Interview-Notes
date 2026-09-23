# Building an Agent Loop with Tool Use

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Coding · LLM agents, tool use | ★★★★☆ | Medium | SWE · RE · RS · MLE | tool-use, agent-loop, llm-api, debugging, prompt-engineering | 3 parts / 55 min | Technical phone screen |
<!-- meta:end -->

## Problem

A notebook defines `client`, an object with one method, `client.create(model, system, messages, tools, max_tokens) -> Response`, which sends the running conversation to a language model and returns its next turn.

`messages` is a list of `{"role": "user" | "assistant", "content": [...]}` dictionaries, oldest first. `content` is a list of *content blocks*, each one of:

- `{"type": "text", "text": "..."}` — plain text.
- `{"type": "tool_use", "id": "...", "name": "...", "input": {...}}` — a request from the model to call one tool. `id` identifies this particular request; `name` names one of the tools offered in this call's `tools`; `input` is that tool's arguments, already validated against the tool's schema (below). A `tool_use` block appears only inside an assistant message, because only the model produces one.
- `{"type": "tool_result", "tool_use_id": "...", "content": "...", "is_error": False}` — the outcome of running a tool, handed back to the model. `tool_use_id` must equal the `id` of the `tool_use` block it answers; `content` is the tool's output, or a description of the failure, as a string; `is_error` is `True` exactly when the tool call failed. A `tool_result` block appears only inside a user message, because only the calling code produces one — never the model.

`tools` is a list of `{"name": "...", "description": "...", "input_schema": {...}}` dictionaries: each tool's name, a natural-language description the model uses to decide when and how to call it, and a JSON Schema object (`{"type": "object", "properties": {...}, "required": [...]}`) restricting `input` to those properties. `max_tokens` caps the length of the model's reply and does not otherwise affect correctness here; the notebook passes a fixed constant.

The return value, a `Response`, has two attributes: `.content`, a list of content blocks in the shapes above (a response only ever contains `text` and `tool_use` blocks, never `tool_result`), and `.stop_reason`, a string that in this notebook is always either `"end_turn"` (the model is done and is not asking for a tool) or `"tool_use"` (`.content` contains at least one `tool_use` block that must be answered before the conversation can continue). A single response's `.content` may hold more than one `tool_use` block at once, when the model asks for several tool calls together.

Answering one question is this loop:

1. Start `messages` with one message, `{"role": "user", "content": [{"type": "text", "text": question}]}`.
2. Call `client.create(...)` with the current `messages`.
3. If the response's `stop_reason` is `"end_turn"`, its `content` holds exactly one `text` block; that block's `text` is the answer, and the loop ends.
4. If `stop_reason` is `"tool_use"`, append `{"role": "assistant", "content": response.content}` to `messages` — the model's own turn, exactly as produced, including every block. Then, for every `tool_use` block in `response.content`, run the tool it names on its `input`, and build one `tool_result` block whose `tool_use_id` equals that block's own `id`; collect all of these `tool_result` blocks into one new message, `{"role": "user", "content": [...]}`, and append it to `messages`. Go back to step 2.

In this notebook, whenever `stop_reason` is `"tool_use"`, every block of `.content` is a `tool_use` block; whenever `stop_reason` is `"end_turn"`, `.content` holds exactly the one `text` block described above.

The notebook offers one tool at first, `get_quote`, backed by the closing-price table below:

```text
              2026-01-05   2026-01-06   2026-01-07   2026-01-08   2026-01-09   2026-01-12
AAPL             180.00       181.50       179.00       184.00       190.00       188.00
MSFT             410.00       406.00       412.00       415.00       420.00       418.00
GOOG             140.00       141.40       139.00       142.50       145.00       146.50
```

No other ticker, and no other date — including 2026-01-10 and 2026-01-11, a Saturday and a Sunday — has any data. `get_quote(ticker: str, date: str) -> float` returns the closing price at the row and column of this table; called with a ticker not listed, it fails with the message `f"unknown ticker: {ticker}"`; called with a date not listed for a ticker that is, it fails with `f"no trading data for {date}"`.

The notebook also provides `run_tool(name: str, input: dict) -> tuple[str, bool]`: it calls the Python function registered for the tool named `name` with the keyword arguments in `input`, and returns `(str(result), False)`. If that call raises, `run_tool` catches the exception instead of propagating it and returns `(str(exception), True)` instead — a failing tool never crashes the loop; carrying `(text, is_error)` into the `tool_result` block above is the loop's job.

### Part 1 — Fix the starter agent

`SINGLE_CALL_QUESTIONS` lists three questions that each need exactly one call to `get_quote`:

```text
"What was AAPL's closing price on 2026-01-07?"   -> AAPL, 2026-01-07 -> 179.00
"What was MSFT's closing price on 2026-01-09?"   -> MSFT, 2026-01-09 -> 420.00
"What was GOOG's closing price on 2026-01-05?"   -> GOOG, 2026-01-05 -> 140.00
```

The function below is a first attempt at the loop above; it has exactly four bugs against it. Fixed, it returns a string stating the right price for each of the three questions above, and — for these or any other single-tool-call question — never departs from the loop above: it always sends the model's own turn back before the tool results that answer it, always pairs a `tool_result` with the `id` of the `tool_use` block it answers (not the tool's `name`, which is the same string on every call to that tool), answers every `tool_use` block a response contains rather than only one of them, and keeps exchanging turns with the model until `stop_reason` is `"end_turn"` rather than assuming one round of tool calls is always enough. Fixed, it also takes exactly two calls to `client.create` per question: one that receives the `tool_use` request, one that receives the final text after seeing the `tool_result`.

```py
def answer_question(question: str) -> str:
    messages = [{"role": "user", "content": [{"type": "text", "text": question}]}]
    response = client.create(model=MODEL_NAME, system=SYSTEM_PROMPT, messages=messages,
                              tools=[GET_QUOTE_TOOL], max_tokens=MAX_TOKENS)

    if response.stop_reason == "tool_use":
        tool_use = next(b for b in response.content if b["type"] == "tool_use")
        text, is_error = run_tool(tool_use["name"], tool_use["input"])
        messages.append({
            "role": "user",
            "content": [{"type": "tool_result", "tool_use_id": tool_use["name"],
                         "content": text, "is_error": is_error}],
        })
        response = client.create(model=MODEL_NAME, system=SYSTEM_PROMPT, messages=messages,
                                  tools=[GET_QUOTE_TOOL], max_tokens=MAX_TOKENS)

    return response.content[0]["text"]
```

Trace of the first question above:

```text
messages = [{"role": "user", "content": [{"type": "text", "text":
             "What was AAPL's closing price on 2026-01-07?"}]}]

# call 1: no price is known yet, so the model asks for one
response.stop_reason = "tool_use"
response.content = [{"type": "tool_use", "id": "call_1", "name": "get_quote",
                      "input": {"ticker": "AAPL", "date": "2026-01-07"}}]

# the loop keeps the model's own turn, runs the tool, and hands back the result
messages += [{"role": "assistant", "content": response.content},
             {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "call_1",
                                            "content": "179.0", "is_error": False}]}]

# call 2: the model has the price and answers
response.stop_reason = "end_turn"
response.content = [{"type": "text", "text": "AAPL closed at $179.00 on 2026-01-07."}]
# the loop returns "AAPL closed at $179.00 on 2026-01-07."
```

### Part 2 — Questions that need several results

Generalize `answer_question` into

```py
def run_agent(question: str, tools: list[dict], system: str, max_calls: int = 8) -> str: ...
```

taking the tool catalog, the system prompt and a cap on rounds as arguments instead of the fixed globals above (Part 3 reuses it unchanged, with a different catalog and prompt). If the model has not reached `stop_reason == "end_turn"` after `max_calls` calls to `client.create`, stop and return the string `f"gave up after {max_calls} calls"` instead of calling `create` again.

With `tools=[GET_QUOTE_TOOL]` and `system=SYSTEM_PROMPT`, `run_agent` must answer every question in `MULTI_CALL_QUESTIONS`:

```text
"Between 2026-01-05 and 2026-01-09, did AAPL or MSFT gain more, in percentage terms?"
    -> names AAPL: (190.00 - 180.00) / 180.00 ~= 5.6%, against MSFT's (420.00 - 410.00) / 410.00 ~= 2.4%
"What was GOOG's closing price on 2026-01-10?"
    -> states that there is no trading data for that day
"Which of AAPL, MSFT and GOOG had the highest closing price on 2026-01-08?"
    -> names MSFT: 415.00 against 184.00 and 142.50
```

The first question needs four calls to `get_quote` (both tickers, at both dates); the third needs three (one call per ticker). With only `get_quote` offered, and nothing telling it to do otherwise, the model asks for one price at a time, so answering the three questions above costs `4 + 1`, `1 + 1` and `3 + 1` calls to `create` — 5, 2 and 4 — 11 in total. The second question's single call to `get_quote` fails, since there is no row for 2026-01-10; the loop must still hand the failure back as a `tool_result` with `is_error=True` and let the model produce an answer from it, rather than raising out of the loop.

Trace of the second question:

```text
messages[0] = {"role": "user", "content": [{"type": "text", "text":
               "What was GOOG's closing price on 2026-01-10?"}]}

# call 1
response.content = [{"type": "tool_use", "id": "call_1", "name": "get_quote",
                      "input": {"ticker": "GOOG", "date": "2026-01-10"}}]
response.stop_reason = "tool_use"

# run_tool("get_quote", {"ticker": "GOOG", "date": "2026-01-10"}) raises ValueError
# ("no trading data for 2026-01-10"); run_tool catches it and returns
# ("no trading data for 2026-01-10", True)
messages += [{"role": "assistant", "content": response.content},
             {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "call_1",
                                            "content": "no trading data for 2026-01-10", "is_error": True}]}]

# call 2: is_error does not stop the loop -- it is just another tool_result for the model to read
response.content = [{"type": "text", "text":
                      "I could not get GOOG's price on 2026-01-10: no trading data for 2026-01-10."}]
response.stop_reason = "end_turn"
```

### Part 3 — Fewer calls to `create`

Combined with Part 1's three questions (already at two calls each, the minimum for a question that needs a tool at all), the six questions of `SINGLE_CALL_QUESTIONS` and `MULTI_CALL_QUESTIONS` cost `6 + 11 = 17` calls to `client.create` under `tools=[GET_QUOTE_TOOL]` and `system=SYSTEM_PROMPT`. Without changing `run_agent`, bring this down to at most two calls per question — at most 12 in total — by changing only what is offered to the model:

- a second tool, `get_quotes`, that looks up several `(ticker, date)` pairs in one call:

  ```py
  def get_quotes(lookups: list[dict]) -> list[float]: ...
  ```

  `lookups` is a list of `{"ticker": str, "date": str}` dictionaries; the result lists one price per pair, in the same order. If any pair is invalid, `get_quotes` fails the same way `get_quote` would for that pair, scanning `lookups` in order, and returns nothing for the rest of the list — there is no partial success.
- a system prompt that tells the model to gather every price a question needs before answering: with `get_quotes` offered, in one call to it; without it, as several `get_quote` calls issued together, in parallel, in the same turn, rather than one call per turn.

Trace of the first `MULTI_CALL_QUESTIONS` question against this pair:

```text
messages[0] = {"role": "user", "content": [{"type": "text", "text":
               "Between 2026-01-05 and 2026-01-09, did AAPL or MSFT gain more, in percentage terms?"}]}

# call 1: every price the question needs is known up front, so the model asks for all four at once
response.content = [{"type": "tool_use", "id": "call_1", "name": "get_quotes",
                      "input": {"lookups": [{"ticker": "AAPL", "date": "2026-01-05"},
                                             {"ticker": "AAPL", "date": "2026-01-09"},
                                             {"ticker": "MSFT", "date": "2026-01-05"},
                                             {"ticker": "MSFT", "date": "2026-01-09"}]}}]
response.stop_reason = "tool_use"

# run_tool("get_quotes", ...) returns ("[180.0, 190.0, 410.0, 420.0]", False)
messages += [{"role": "assistant", "content": response.content},
             {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "call_1",
                                            "content": "[180.0, 190.0, 410.0, 420.0]", "is_error": False}]}]

# call 2: two calls total, against five for the same question in Part 2
response.content = [{"type": "text", "text":
                      "AAPL gained more in percentage terms between 2026-01-05 and 2026-01-09."}]
response.stop_reason = "end_turn"
```

## Reference solution

<details>
<summary>Show the reference solution</summary>

Confirm with the interviewer whether a failing tool call should ever stop the loop outright (assumed not here: `is_error` is handed back to the model like any other result) and whether the model may answer from memory without calling a tool at all (assumed not, stated directly in the system prompt below).

### Part 1

Reading the loop against the code line by line: the second call to `client.create` never receives the model's first turn, only the lone `tool_result`, because nothing appends `{"role": "assistant", "content": response.content}` — that is the first bug. The second pairs the `tool_result` with `tool_use["name"]`, the tool's name, a fixed string shared by every call to that tool, rather than `tool_use["id"]`, unique to this one request. The third reads only one block of a response that could hold several: `next(b for b in response.content if b["type"] == "tool_use")` throws away every `tool_use` block after the first. The fourth exits without checking the second response at all: the function calls `create` at most twice and returns `response.content[0]["text"]` unconditionally, rather than looping until `stop_reason` is `"end_turn"`.

```python
PRICES = {
    "AAPL": {"2026-01-05": 180.00, "2026-01-06": 181.50, "2026-01-07": 179.00,
             "2026-01-08": 184.00, "2026-01-09": 190.00, "2026-01-12": 188.00},
    "MSFT": {"2026-01-05": 410.00, "2026-01-06": 406.00, "2026-01-07": 412.00,
             "2026-01-08": 415.00, "2026-01-09": 420.00, "2026-01-12": 418.00},
    "GOOG": {"2026-01-05": 140.00, "2026-01-06": 141.40, "2026-01-07": 139.00,
             "2026-01-08": 142.50, "2026-01-09": 145.00, "2026-01-12": 146.50},
}


def get_quote(ticker: str, date: str) -> float:
    if ticker not in PRICES:
        raise ValueError(f"unknown ticker: {ticker}")
    if date not in PRICES[ticker]:
        raise ValueError(f"no trading data for {date}")
    return PRICES[ticker][date]


GET_QUOTE_TOOL = {
    "name": "get_quote",
    "description": "Look up a single stock's closing price on one trading day.",
    "input_schema": {
        "type": "object",
        "properties": {
            "ticker": {"type": "string", "description": "Stock ticker, e.g. 'AAPL'."},
            "date": {"type": "string", "description": "Trading day, formatted YYYY-MM-DD."},
        },
        "required": ["ticker", "date"],
    },
}

MODEL_NAME = "agent-loop-demo"
MAX_TOKENS = 1024
SYSTEM_PROMPT = (
    "You are a financial research assistant. Answer only using the get_quote tool; "
    "never state a price you did not just look up."
)

TOOL_FUNCTIONS = {"get_quote": get_quote}


def run_tool(name: str, tool_input: dict) -> tuple[str, bool]:
    try:
        return str(TOOL_FUNCTIONS[name](**tool_input)), False
    except Exception as e:
        return str(e), True


def _answered(block: dict) -> dict:
    text, is_error = run_tool(block["name"], block["input"])
    return {"type": "tool_result", "tool_use_id": block["id"], "content": text, "is_error": is_error}


def answer_question(question: str) -> str:
    messages = [{"role": "user", "content": [{"type": "text", "text": question}]}]
    while True:
        response = client.create(model=MODEL_NAME, system=SYSTEM_PROMPT, messages=messages,
                                  tools=[GET_QUOTE_TOOL], max_tokens=MAX_TOKENS)
        if response.stop_reason == "end_turn":                                # NOTE: loop until end_turn
            return response.content[0]["text"]
        messages.append({"role": "assistant", "content": response.content})   # NOTE: keep the model's own turn
        tool_uses = [b for b in response.content if b["type"] == "tool_use"]  # NOTE: every block, not just one
        messages.append({"role": "user", "content": [_answered(b) for b in tool_uses]})
```

`_answered` reads `block["id"]`, never `block["name"]`, so every `tool_result` is paired with the request it actually answers.

### Part 2

`run_agent` is Part 1's fixed loop with the tool catalog, the system prompt and a round cap as parameters instead of module-level constants; it reuses `_answered` unchanged, since pairing a `tool_result` with its `tool_use` block does not depend on which tools are offered.

```python
def run_agent(question: str, tools: list[dict], system: str, max_calls: int = 8) -> str:
    messages = [{"role": "user", "content": [{"type": "text", "text": question}]}]
    for _ in range(max_calls):                            # NOTE: a hard cap -- see Follow-ups
        response = client.create(model=MODEL_NAME, system=system, messages=messages,
                                  tools=tools, max_tokens=MAX_TOKENS)
        if response.stop_reason == "end_turn":
            return response.content[0]["text"]
        messages.append({"role": "assistant", "content": response.content})
        tool_uses = [b for b in response.content if b["type"] == "tool_use"]
        messages.append({"role": "user", "content": [_answered(b) for b in tool_uses]})
    return f"gave up after {max_calls} calls"
```

Nothing about error handling changes for the harder questions: `_answered` already carries whatever `run_tool` returns, success or failure, into `is_error`, so a failing lookup is just one more `tool_result` for the model to read on the next call, and the loop does not need to know that anything went wrong. What Part 1's question set never exercised is more than one `tool_use` block in a single response; `run_agent`'s comprehension over `response.content` answers every one of them, in one appended message, regardless of how many there are.

### Part 3

`run_agent` does not change; only what is passed to it does.

```python
def get_quotes(lookups: list[dict]) -> list[float]:
    return [get_quote(pair["ticker"], pair["date"]) for pair in lookups]   # NOTE: fails on the first bad pair


TOOL_FUNCTIONS["get_quotes"] = get_quotes

GET_QUOTES_TOOL = {
    "name": "get_quotes",
    "description": (
        "Look up closing prices for several (ticker, date) pairs in a single call. "
        "Prefer this over repeated get_quote calls whenever more than one price is needed."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "lookups": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {"ticker": {"type": "string"}, "date": {"type": "string"}},
                    "required": ["ticker", "date"],
                },
            },
        },
        "required": ["lookups"],
    },
}

SYSTEM_PROMPT_BATCHED = SYSTEM_PROMPT + (
    " When a question needs more than one price, request every price you already know you will "
    "need in a single turn: one call to get_quotes if it is offered, or one get_quote call per "
    "price, in parallel, in the same turn, if it is not."
)
TOOLS_BATCHED = [GET_QUOTE_TOOL, GET_QUOTES_TOOL]
```

`get_quotes` reuses `get_quote` pair by pair inside a list comprehension, so it raises on the first invalid pair exactly as the statement requires, with no separate validation pass. Measured on all six questions (checks below): 17 calls to `create` with `[GET_QUOTE_TOOL]` and `SYSTEM_PROMPT`, 12 with `TOOLS_BATCHED` and `SYSTEM_PROMPT_BATCHED` — every question that needed more than one price now takes exactly two calls, the same floor a question that only ever needed one was already at.

### Follow-ups

- Context growth: every call resends the full `messages` so far, so a long-running question pays for its own earlier rounds again on each later call; once a tool result's content has been folded into a later turn, a loop meant to run for many rounds typically drops or summarizes the raw result rather than keeping it verbatim forever.
- Cost and latency: fewer, larger calls save more than the token count alone suggests, because every tool result already sitting in `messages` is resent as prompt tokens on each later call, and because time-to-first-token is paid once per call to `create`, not once per tool used.
- A loop that never ends: `max_calls` is one instance of a general requirement — an agent needs a stopping condition the model cannot override, since nothing stops a model from requesting tools indefinitely; a call count alone does not bound wall-clock time, since a slow or hanging tool call can make even a low cap take a long time to reach, so production loops pair the call count with an explicit wall-clock deadline.
- Prompt injection through tool results: `tool_result.content` is text the loop did not write itself — here a fixed table, in general a file or a web page — and a model that treats every input the same way may follow instructions that appear inside it; stating in the system prompt that tool output is data, not instructions, and gating any consequential action behind a step the loop controls rather than the model alone, both narrow this.
- When an agent is the wrong tool: a question whose steps are fixed in advance, such as "look up X, then Y, then compare", is answered more cheaply and more reliably by code that calls `get_quote` directly; a loop earns its cost only when which lookups are needed cannot be known until the question itself is read.

<details>
<summary>Checks (runnable)</summary>

```python
# The checks below replace the real client with FakeModel, a deterministic stand-in used only here:
# it never calls a language model, it decides its next tool_use block (or its final answer) from
# fixed, per-question plans and from what the running `messages` already show was tried. It stands
# in for `client.create` only inside these checks, never for the interface described in the Problem.
import ast


class _Response:
    def __init__(self, content, stop_reason):
        self.content = content
        self.stop_reason = stop_reason


def _price(done, ticker, date):
    return float(done[(ticker, date)][0])


def _single(ticker, date):
    return [(ticker, date)], lambda done: f"price {_price(done, ticker, date):.2f}"


def _compare_change(t1, t2, d0, d1):
    pairs = [(t1, d0), (t1, d1), (t2, d0), (t2, d1)]

    def answer(done):
        pct = lambda t: (_price(done, t, d1) - _price(done, t, d0)) / _price(done, t, d0)
        return f"{t1 if pct(t1) > pct(t2) else t2} gained more"

    return pairs, answer


def _failing_lookup(ticker, date):
    return [(ticker, date)], lambda done: f"failed: {done[(ticker, date)][0]}"


def _highest(tickers, date):
    pairs = [(t, date) for t in tickers]
    return pairs, lambda done: f"{max(tickers, key=lambda t: _price(done, t, date))} is highest"


PLANS = {
    "What was AAPL's closing price on 2026-01-07?": _single("AAPL", "2026-01-07"),
    "What was MSFT's closing price on 2026-01-09?": _single("MSFT", "2026-01-09"),
    "What was GOOG's closing price on 2026-01-05?": _single("GOOG", "2026-01-05"),
    "Between 2026-01-05 and 2026-01-09, did AAPL or MSFT gain more, in percentage terms?":
        _compare_change("AAPL", "MSFT", "2026-01-05", "2026-01-09"),
    "What was GOOG's closing price on 2026-01-10?": _failing_lookup("GOOG", "2026-01-10"),
    "Which of AAPL, MSFT and GOOG had the highest closing price on 2026-01-08?":
        _highest(["AAPL", "MSFT", "GOOG"], "2026-01-08"),
}
SINGLE_CALL_QUESTIONS = list(PLANS)[:3]
MULTI_CALL_QUESTIONS = list(PLANS)[3:]
ALL_QUESTIONS = SINGLE_CALL_QUESTIONS + MULTI_CALL_QUESTIONS
CHECK_TEXT = {ALL_QUESTIONS[0]: "179.00", ALL_QUESTIONS[1]: "420.00", ALL_QUESTIONS[2]: "140.00",
             ALL_QUESTIONS[3]: "AAPL", ALL_QUESTIONS[4]: "no trading data", ALL_QUESTIONS[5]: "MSFT"}


class FakeModel:
    """Deterministic stand-in for a real model: correct, but not clever unless told to be."""

    def __init__(self, plans):
        self.plans = plans
        self.calls = 0
        self._next_id = 0

    def _new_id(self):
        self._next_id += 1
        return f"call_{self._next_id}"

    def create(self, model, system, messages, tools, max_tokens):
        self.calls += 1
        question = messages[0]["content"][0]["text"]
        pairs, answer_fn = self.plans[question]
        done = {}                                          # (ticker, date) -> (content, is_error), from messages
        for i, msg in enumerate(messages):
            if msg["role"] != "assistant":
                continue
            uses = {b["id"]: b for b in msg["content"] if b["type"] == "tool_use"}
            if not uses:
                continue
            for r in messages[i + 1]["content"]:
                assert r["type"] == "tool_result" and r["tool_use_id"] in uses, \
                    f"tool_result does not match any pending tool_use: {r}"
                block = uses[r["tool_use_id"]]
                if block["name"] == "get_quote":
                    done[(block["input"]["ticker"], block["input"]["date"])] = (r["content"], r["is_error"])
                elif r["is_error"]:
                    for pair in block["input"]["lookups"]:
                        done[(pair["ticker"], pair["date"])] = (r["content"], True)
                else:
                    values = ast.literal_eval(r["content"])
                    for pair, v in zip(block["input"]["lookups"], values):
                        done[(pair["ticker"], pair["date"])] = (str(v), False)
        remaining = [p for p in pairs if p not in done]
        if not remaining:
            return _Response([{"type": "text", "text": answer_fn(done)}], "end_turn")
        names = {t["name"] for t in tools}
        if "get_quotes" in names and len(remaining) > 1:               # a batch tool is offered: use it
            lookups = [{"ticker": t, "date": d} for t, d in remaining]
            return _Response([{"type": "tool_use", "id": self._new_id(), "name": "get_quotes",
                               "input": {"lookups": lookups}}], "tool_use")
        if "parallel" in system.lower() and len(remaining) > 1:        # told to parallelize: ask for all at once
            blocks = [{"type": "tool_use", "id": self._new_id(), "name": "get_quote",
                       "input": {"ticker": t, "date": d}} for t, d in remaining]
            return _Response(blocks, "tool_use")
        t, d = remaining[0]                                            # default: one lookup per turn
        return _Response([{"type": "tool_use", "id": self._new_id(), "name": "get_quote",
                           "input": {"ticker": t, "date": d}}], "tool_use")


def run_and_count(fn, *args, **kwargs):
    global client
    client = FakeModel(PLANS)
    return fn(*args, **kwargs), client.calls


def _expect_error(fn, *args, needle):
    try:
        fn(*args)
        assert False, "should have raised"
    except ValueError as e:
        assert needle in str(e), e


# --- get_quote, get_quotes and run_tool directly, including failures the example questions don't reach ---
assert get_quote("AAPL", "2026-01-07") == 179.00
_expect_error(get_quote, "TSLA", "2026-01-07", needle="unknown ticker")
_expect_error(get_quote, "AAPL", "2026-01-10", needle="no trading data")
assert get_quotes([{"ticker": "AAPL", "date": "2026-01-05"},
                   {"ticker": "MSFT", "date": "2026-01-05"}]) == [180.00, 410.00]
_expect_error(get_quotes, [{"ticker": "AAPL", "date": "2026-01-05"}, {"ticker": "AAPL", "date": "2026-01-10"}],
             needle="no trading data")                     # NOTE: all-or-nothing -- the first bad pair aborts
assert run_tool("get_quote", {"ticker": "AAPL", "date": "2026-01-07"}) == ("179.0", False)
text, is_error = run_tool("get_quote", {"ticker": "AAPL", "date": "2026-01-10"})
assert is_error and "no trading data" in text

# --- Part 1: answer_question on the three single-call questions, exactly two calls each ---
for q in SINGLE_CALL_QUESTIONS:
    answer, calls = run_and_count(answer_question, q)
    assert CHECK_TEXT[q] in answer, (q, answer)
    assert calls == 2, (q, calls)


# --- general contract: two tool_use blocks in one response, each answered with its own id ---
class _TwoToolStub:
    def __init__(self):
        self.calls = 0

    def create(self, model, system, messages, tools, max_tokens):
        self.calls += 1
        if self.calls == 1:
            return _Response([
                {"type": "tool_use", "id": "a1", "name": "get_quote",
                 "input": {"ticker": "AAPL", "date": "2026-01-07"}},
                {"type": "tool_use", "id": "a2", "name": "get_quote",
                 "input": {"ticker": "MSFT", "date": "2026-01-09"}},
            ], "tool_use")
        assistant_msg, result_msg = messages[-2], messages[-1]
        ids_asked = {b["id"] for b in assistant_msg["content"]}
        ids_answered = {r["tool_use_id"] for r in result_msg["content"]}
        assert ids_asked == ids_answered == {"a1", "a2"}, (ids_asked, ids_answered)
        contents = {r["tool_use_id"]: r["content"] for r in result_msg["content"]}
        assert contents == {"a1": "179.0", "a2": "420.0"}, contents
        return _Response([{"type": "text", "text": "done"}], "end_turn")


client = _TwoToolStub()
assert answer_question("(two independent lookups)") == "done"
assert client.calls == 2

# --- Part 2: run_agent with only get_quote offered, on all six questions ---
EXPECTED_BASELINE_CALLS = [2, 2, 2, 5, 2, 4]
baseline_total = 0
for q, expected_calls in zip(ALL_QUESTIONS, EXPECTED_BASELINE_CALLS):
    answer, calls = run_and_count(run_agent, q, [GET_QUOTE_TOOL], SYSTEM_PROMPT)
    assert CHECK_TEXT[q] in answer, (q, answer)
    assert calls == expected_calls, (q, calls, expected_calls)
    baseline_total += calls
assert baseline_total == 17, baseline_total


# --- Part 2: the cap on rounds, against a model that never stops asking for tools ---
class _NeverStopsStub:
    def __init__(self):
        self.calls = 0

    def create(self, model, system, messages, tools, max_tokens):
        self.calls += 1
        return _Response([{"type": "tool_use", "id": f"x{self.calls}", "name": "get_quote",
                           "input": {"ticker": "AAPL", "date": "2026-01-07"}}], "tool_use")


client = _NeverStopsStub()
assert run_agent("(never resolves)", [GET_QUOTE_TOOL], SYSTEM_PROMPT, max_calls=5) == "gave up after 5 calls"
assert client.calls == 5

# --- Part 3: run_agent with the batched tools and prompt, on the same six questions ---
batched_total = 0
for q in ALL_QUESTIONS:
    answer, calls = run_and_count(run_agent, q, TOOLS_BATCHED, SYSTEM_PROMPT_BATCHED)
    assert CHECK_TEXT[q] in answer, (q, answer)
    assert calls <= 2, (q, calls)
    batched_total += calls
assert batched_total == 12 < baseline_total
print(f"calls across all six questions: {baseline_total} with get_quote alone, {batched_total} batched")

print("all checks passed")
```

</details>

</details>
