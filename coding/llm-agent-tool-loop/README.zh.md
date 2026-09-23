# 用工具调用搭建智能体循环

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · LLM 智能体，工具调用 | ★★★★☆ | 中等 | SWE · RE · RS · MLE | tool-use, agent-loop, llm-api, debugging, prompt-engineering | 3 个部分 / 55 分钟 | 技术电面 |
<!-- meta:end -->

## 题目

一个 notebook 定义了 `client`，一个只有一个方法的对象：`client.create(model, system, messages, tools, max_tokens) -> Response`，它把当前的对话发给一个语言模型，并返回模型的下一轮回复。

`messages` 是一个由 `{"role": "user" | "assistant", "content": [...]}` 字典组成的列表，按时间顺序从旧到新排列。`content` 是一个由*内容块*（content block）组成的列表，每一块是以下几种之一：

- `{"type": "text", "text": "..."}` —— 纯文本。
- `{"type": "tool_use", "id": "...", "name": "...", "input": {...}}` —— 模型发出的一次工具调用请求。`id` 标识这一次具体的请求；`name` 是本次调用的 `tools` 中某个工具的名字；`input` 是该工具的参数，已经按照工具的 schema（见下文）校验过。`tool_use` 块只会出现在 assistant 消息里，因为只有模型会产生它。
- `{"type": "tool_result", "tool_use_id": "...", "content": "...", "is_error": False}` —— 一次工具调用的结果，回传给模型。`tool_use_id` 必须等于它所回答的那个 `tool_use` 块的 `id`；`content` 是工具的输出，或者失败原因的说明，类型是字符串；`is_error` 在工具调用失败时为 `True`。`tool_result` 块只会出现在 user 消息里，因为只有调用方代码会产生它——模型不会。

`tools` 是一个由 `{"name": "...", "description": "...", "input_schema": {...}}` 字典组成的列表：每个工具的名字、模型用来判断何时以及如何调用它的自然语言描述，以及一个 JSON Schema 对象（`{"type": "object", "properties": {...}, "required": [...]}`），把 `input` 限制在这些属性范围内。`max_tokens` 限制模型回复的长度，这里不影响正确性；notebook 里固定传一个常量。

返回值 `Response` 有两个属性：`.content`，一个形状如上的内容块列表（一次响应只会包含 `text` 和 `tool_use` 块，绝不会有 `tool_result`），以及 `.stop_reason`，这是一个字符串，在这个 notebook 里始终是 `"end_turn"`（模型已经完成，不再请求工具）或 `"tool_use"`（`.content` 中至少有一个 `tool_use` 块，必须先被回答对话才能继续）二者之一。同一个响应的 `.content` 里可能一次出现不止一个 `tool_use` 块，这发生在模型一次请求多个工具调用的时候。

回答一个问题就是下面这个循环：

1. 用一条消息初始化 `messages`：`{"role": "user", "content": [{"type": "text", "text": question}]}`。
2. 用当前的 `messages` 调用 `client.create(...)`。
3. 如果响应的 `stop_reason` 是 `"end_turn"`，它的 `content` 里正好有一个 `text` 块；这个块的 `text` 就是答案，循环结束。
4. 如果 `stop_reason` 是 `"tool_use"`，把 `{"role": "assistant", "content": response.content}` 追加到 `messages`——原样保留模型自己这一轮产生的每一个块。然后，对 `response.content` 里的每一个 `tool_use` 块，用它的 `input` 运行它指定的工具，构造一个 `tool_result` 块，其 `tool_use_id` 等于该块自己的 `id`；把这些 `tool_result` 块合并进一条新消息 `{"role": "user", "content": [...]}`，追加到 `messages`。回到第 2 步。

在这个 notebook 里，只要 `stop_reason` 是 `"tool_use"`，`.content` 里的每一块都是 `tool_use` 块；只要 `stop_reason` 是 `"end_turn"`，`.content` 就正好是上面说的那一个 `text` 块。

notebook 一开始只提供一个工具，`get_quote`，背后是下面这张收盘价表：

```text
              2026-01-05   2026-01-06   2026-01-07   2026-01-08   2026-01-09   2026-01-12
AAPL             180.00       181.50       179.00       184.00       190.00       188.00
MSFT             410.00       406.00       412.00       415.00       420.00       418.00
GOOG             140.00       141.40       139.00       142.50       145.00       146.50
```

除了表里列出的股票代码和日期组合，其余一概没有数据——包括 2026-01-10 和 2026-01-11，一个周六和一个周日。`get_quote(ticker: str, date: str) -> float` 返回这张表里对应行列的收盘价；传入表中没有的股票代码会失败，报错信息是 `f"unknown ticker: {ticker}"`；对一个存在的股票代码传入表中没有的日期，同样会失败，报错信息是 `f"no trading data for {date}"`。

notebook 还提供了 `run_tool(name: str, input: dict) -> tuple[str, bool]`：它用 `input` 里的关键字参数，调用 `name` 这个工具名下注册的 Python 函数，返回 `(str(result), False)`。如果这次调用抛出异常，`run_tool` 会捕获它而不是让它往外传播，转而返回 `(str(exception), True)`——失败的工具调用不会让循环崩溃；把 `(text, is_error)` 装进上面的 `tool_result` 块，是循环自己的职责。

### Part 1 —— 修复初版 agent

`SINGLE_CALL_QUESTIONS` 列出了三个问题，每个都只需要恰好一次 `get_quote` 调用：

```text
"What was AAPL's closing price on 2026-01-07?"   -> AAPL, 2026-01-07 -> 179.00
"What was MSFT's closing price on 2026-01-09?"   -> MSFT, 2026-01-09 -> 420.00
"What was GOOG's closing price on 2026-01-05?"   -> GOOG, 2026-01-05 -> 140.00
```

下面的函数是对上述循环的第一版实现，相对这个循环有恰好四个 bug。修复之后，对上面三个问题中的每一个，它都要返回一个陈述了正确价格的字符串；并且——对这三个问题或任何其他只需单次工具调用的问题——都不能偏离上面的循环：先把模型自己这一轮的内容发回去，再发回答它的那些工具结果；`tool_result` 要用它所回答的那个 `tool_use` 块的 `id` 配对（而不是工具的 `name`——`name` 在每次调用同一个工具时都是同一个字符串）；要回答一次响应里的每一个 `tool_use` 块，而不只是其中一个；要一直和模型交换轮次直到 `stop_reason` 变成 `"end_turn"`，而不是假定一轮工具调用总是够用。修复之后，它对每个问题恰好调用两次 `client.create`：一次收到 `tool_use` 请求，一次在看到 `tool_result` 之后收到最终文本。

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

第一个问题的完整轨迹：

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

### Part 2 —— 需要多个结果的问题

把 `answer_question` 推广成

```py
def run_agent(question: str, tools: list[dict], system: str, max_calls: int = 8) -> str: ...
```

把工具目录、system prompt（系统提示词）和轮数上限都变成参数，而不是像上面那样用固定的全局变量（Part 3 会原样复用它，只换一套目录和提示词）。如果在 `max_calls` 次 `client.create` 调用之后，模型仍未让 `stop_reason == "end_turn"`，就停下来，返回字符串 `f"gave up after {max_calls} calls"`，而不是继续调用 `create`。

用 `tools=[GET_QUOTE_TOOL]` 和 `system=SYSTEM_PROMPT`，`run_agent` 必须回答 `MULTI_CALL_QUESTIONS` 里的每一个问题：

```text
"Between 2026-01-05 and 2026-01-09, did AAPL or MSFT gain more, in percentage terms?"
    -> names AAPL: (190.00 - 180.00) / 180.00 ~= 5.6%, against MSFT's (420.00 - 410.00) / 410.00 ~= 2.4%
"What was GOOG's closing price on 2026-01-10?"
    -> states that there is no trading data for that day
"Which of AAPL, MSFT and GOOG had the highest closing price on 2026-01-08?"
    -> names MSFT: 415.00 against 184.00 and 142.50
```

第一个问题需要四次 `get_quote` 调用（两只股票、各两个日期）；第三个需要三次（每只股票各一次）。在只提供 `get_quote`、又没有任何提示告诉它换一种做法的前提下，模型每次只会要一个价格，回答上面三个问题因此分别要花 `4 + 1`、`1 + 1` 和 `3 + 1` 次 `create` 调用——也就是 5、2、4 次，合计 11 次。第二个问题里唯一一次 `get_quote` 调用会失败，因为表里没有 2026-01-10 这一行；循环仍必须把这次失败包装成一个 `is_error=True` 的 `tool_result` 传回去，让模型据此给出回答，而不是从循环里直接抛出异常。

第二个问题的轨迹：

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

### Part 3 —— 减少 `create` 调用次数

加上 Part 1 的三个问题（每个已经是两次调用，这是任何需要用到工具的问题所能达到的下限），`SINGLE_CALL_QUESTIONS` 和 `MULTI_CALL_QUESTIONS` 合计六个问题，在 `tools=[GET_QUOTE_TOOL]`、`system=SYSTEM_PROMPT` 下一共要花 `6 + 11 = 17` 次 `client.create` 调用。在不改动 `run_agent` 的前提下，只靠改变提供给模型的东西，把这个数字降到每个问题最多两次——总共最多 12 次：

- 一个新工具 `get_quotes`，一次调用就能查多个 `(ticker, date)` 组合：

  ```py
  def get_quotes(lookups: list[dict]) -> list[float]: ...
  ```

  `lookups` 是一个由 `{"ticker": str, "date": str}` 字典组成的列表；返回结果按同样的顺序，每个组合给出一个价格。只要其中有一个组合无效，`get_quotes` 就会按 `lookups` 的顺序扫描，在第一个无效的组合处失败（失败方式与 `get_quote` 对该组合的失败方式相同），并且不为列表里剩下的部分返回任何结果——没有部分成功这回事。
- 一段 system prompt，告诉模型在回答之前先一次性凑齐一个问题需要的所有价格：如果 `get_quotes` 可用，就用一次调用；如果不可用，就把多个 `get_quote` 调用放在同一轮里一起发出，而不是一轮只发一个。

针对这一对改动，`MULTI_CALL_QUESTIONS` 第一个问题的轨迹：

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

## 参考解答

<details>
<summary>展开参考解答</summary>

不妨先和面试官确认两点：工具调用失败是否应该让循环直接终止（这里假定不会——`is_error` 就像任何其他结果一样原样传回给模型）；模型是否可以凭记忆直接作答而不调用工具（这里假定不行，下面的系统提示里已经直接写明）。

### Part 1

把上面的循环规则对着代码逐行核对：第二次 `client.create` 调用根本没收到模型第一轮产生的内容，只收到了孤零零的 `tool_result`，因为代码里没有任何一处追加 `{"role": "assistant", "content": response.content}`——这是第一个 bug。第二处把 `tool_result` 和 `tool_use["name"]` 配对，而 `name` 是工具的名字，每次调用同一个工具都是同一个字符串；本该用的是这一次请求独有的 `tool_use["id"]`。第三处只读了一次响应里可能有好几个的块中的一个：`next(b for b in response.content if b["type"] == "tool_use")` 会把第一个之后的所有 `tool_use` 块都扔掉。第四处压根没检查第二次响应：函数最多调用两次 `create`，之后无条件返回 `response.content[0]["text"]`，而不是一直循环到 `stop_reason` 变成 `"end_turn"`。

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

`_answered` 读的是 `block["id"]`，而不是 `block["name"]`，所以每个 `tool_result` 都和它真正回答的那次请求配对。

### Part 2

`run_agent` 就是 Part 1 修好之后的循环，只是把工具目录、system prompt 和轮数上限从模块级常量换成了参数；它原样复用 `_answered`，因为把 `tool_result` 和它对应的 `tool_use` 块配对，跟当前提供了哪些工具无关。

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

面对更难的问题，错误处理这一块什么都不用改：`_answered` 已经把 `run_tool` 的返回值——不管成功还是失败——原样装进了 `is_error`，所以一次失败的查询，对模型来说只是下一次调用时又多读到的一个 `tool_result`，循环本身不需要知道出了什么问题。Part 1 的问题集从没用到的是一次响应里出现不止一个 `tool_use` 块；`run_agent` 对 `response.content` 的推导式会把它们全部答上，合并进同一条追加的消息，不管一共有几个。

### Part 3

`run_agent` 不用变；变的只是传给它的东西。

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

`get_quotes` 在一个列表推导式里逐对复用 `get_quote`，所以它会在第一个无效的组合处报错，正如题目要求的那样，不需要另外再写一遍校验。在全部六个问题上实测（见下面的验证代码）：用 `[GET_QUOTE_TOOL]` 和 `SYSTEM_PROMPT` 要 17 次 `create` 调用，用 `TOOLS_BATCHED` 和 `SYSTEM_PROMPT_BATCHED` 只要 12 次——每一个原本需要不止一个价格的问题，现在都恰好两次调用，和只需要一个价格的问题原本就有的下限一样。

### 追问

- 上下文增长：每次调用都要把迄今为止完整的 `messages` 重新发一遍，所以一个问题跑的轮次越多，后面每次调用都要为前面已经问过的内容重新付一遍代价；一旦某个工具结果的内容已经被后续某一轮消化吸收，一个打算跑很多轮的循环通常会丢弃或压缩这段原始结果，而不是把它逐字保留到最后。
- 成本与延迟：更少、更大的调用省下的不只是 token 数字本身能看出的那么多，因为 `messages` 里已有的每一条工具结果都会在之后每次调用时被当作提示词 token 重新计费一遍，而首字延迟（time-to-first-token）是按每次 `create` 调用计的，不是按用了多少个工具计的。
- 永不停止的循环：`max_calls` 只是一类更一般要求的一个具体实例——agent 需要一个模型自己无法绕开的停止条件，因为没有什么能阻止模型无休止地请求工具；只靠调用次数无法限定墙钟时间，因为一次又慢又卡的工具调用，能让哪怕很低的调用次数上限也要花很久才撞到，所以生产环境的循环会在调用次数之外再加一个明确的墙钟时限。
- 通过工具结果的提示词注入：`tool_result.content` 是循环自己没有写过的文本——这里是一张固定的表，一般情况下可能是一个文件或一张网页——如果模型对所有输入一视同仁，就可能听从藏在其中的指令；在 system prompt 里说明工具的输出是数据、不是指令，并且把任何有实际后果的动作放在循环自己把关的一步之后，而不是完全交给模型，都能缩小这个风险。
- 什么时候 agent 是错的工具：一个步骤在提问之前就已经固定下来的问题——比如“先查 X，再查 Y，再比较”——用直接调用 `get_quote` 的代码来回答，更省成本、也更可靠；只有当需要查哪些东西要等看到问题本身才能确定时，循环才值得它的开销。

<details>
<summary>验证代码（可运行）</summary>

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
