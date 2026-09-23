# 机器学习实验配置系统

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 配置系统，校验 | ★★☆☆☆ | 困难 | RE · RS · MLE | deep-merge, cycle-detection, cli-overrides, schema-validation, reproducibility | 4 个部分 / 55 分钟 | 技术电面 |
<!-- meta:end -->

## 题目

*配置*（config）是由键为 `str` 的 `dict`、`list` 和标量——`int`、`float`、`bool`、`str` 或 `None`——嵌套任意
深度组成的值。*配置库*（config store）是一个 `dict[str, dict]`，把名字映射到一个本身在顶层就是 `dict` 的
配置；下面每个函数的第一个参数都是这样一个已经加载到内存中的配置库（这里不解析 YAML 或任何其他文本格式），
返回的都是新配置，不会修改配置库本身，也不会修改传入的任何配置。

### Part 1 —— 组合

一个配置可以在自己的顶层声明 `_base_: [n1, n2, ...]`——配置库中其他名字组成的列表——作为一个键；没有
`_base_` 键的配置就是没有 base。`DELETE` 是本模块导出的一个哨兵值（sentinel value），配置中凡是能放标量
的地方都能放它。

`deep_merge(base, overlay)` 返回一个新 dict，按 `base` 和 `overlay` 键的并集逐个键计算：

- 一个键如果在 `overlay` 中的值是 `DELETE`，结果里就没有这个键，不论 `base` 里有没有它；
- 一个键如果在 `overlay` 中的值是 `dict`，结果里就映射到 `deep_merge(b, overlay[key])`，其中 `b` 在
  `base[key]` 本身是 `dict` 时取 `base[key]`，否则取 `{}`——所以即使 `base` 那里原本没有 dict，
  `overlay[key]` 内部嵌套的 `DELETE` 仍然会删掉这个内层 dict 里对应的键；
- 一个键如果只出现在 `base` 中，结果里保留 `base` 的值；
- 其余的键——出现在 `overlay` 中，且既不是 `DELETE` 也不是 `dict`——结果里取 `overlay` 的值，完全替换掉
  `base` 在那里原来的值，不论原来是不是 dict。

结果中的任何值都不与 `base` 或 `overlay` 共享。

`resolve_bases(store, name)` 返回 `name` 完全组合后的配置：从一个空 dict 开始，把 `store[name]` 自己
`_base_` 列表中每个名字解析出的结果依次从左到右深度合并进来，再把 `store[name]` 自己的键（除 `_base_`
外的每个键）深度合并在最上层。`_base_` 里的名字自己也可以有 `_base_`，用同样的方式递归解析；结果中永远
不会出现 `_base_` 键。当 `name`，或者通过某个 `_base_` 列表可以到达的某个名字，不是 `store` 的键时，
`resolve_bases` 抛出 `ConfigNotFoundError`；当一个配置直接或通过其他配置间接把自己当作 base 时，抛出
`ConfigCycleError`——错误信息里会把这个环经过的名字序列原样列出，例如 `"a -> b -> a"`。

```py
DELETE = ...  # sentinel; usable as a value anywhere in a config

class ConfigError(Exception):
    """Base class for every error raised below."""

class ConfigNotFoundError(ConfigError):
    """name, or a name in some _base_ list, is not in the store."""

class ConfigCycleError(ConfigError):
    """The _base_ graph has a cycle reachable from name."""

def deep_merge(base: dict, overlay: dict) -> dict:
    """Returns a new dict combining base with overlay on top; see the rule above."""

def resolve_bases(store: dict[str, dict], name: str) -> dict:
    """Returns name's fully composed config. The result has no _base_ key."""
```

举例，考虑一个包含四个命名配置的配置库：

```text
store = {
    "base": {"trainer": {"epochs": 10, "batch_size": 32, "lr": 0.1},
             "optimizer": {"name": "sgd", "momentum": 0.9}},
    "wide": {"_base_": ["base"], "trainer": {"batch_size": 128}},
    "regularized": {"trainer": {"weight_decay": 0.01}},
    "experiment": {"_base_": ["wide", "regularized"], "trainer": {"epochs": 20},
                   "optimizer": {"name": "adam", "momentum": DELETE}},
}

resolve_bases(store, "experiment")
# trainer：base 给出 epochs=10, batch_size=32, lr=0.1
#        + wide 覆盖 batch_size=128                     -> epochs=10, batch_size=128, lr=0.1
#        + regularized 覆盖 weight_decay=0.01            -> ...,  weight_decay=0.01
#        + experiment 自己的 epochs=20                   -> epochs=20, batch_size=128, lr=0.1, weight_decay=0.01
# optimizer：base 给出 name="sgd", momentum=0.9
#          + experiment 自己的 name="adam", momentum=DELETE -> name="adam"（momentum 被删除）
-> {"trainer": {"epochs": 20, "batch_size": 128, "lr": 0.1, "weight_decay": 0.01},
    "optimizer": {"name": "adam"}}
```

### Part 2 —— 覆盖与插值

*覆盖*（override）是形如 `path=value` 或 `+path=value` 的字符串，其中 `path` 是一个或多个标识符
（`[A-Za-z_]\w*`）用 `.` 连接，指向嵌套 `dict` 内部的某个键——绝不是列表下标。没有 `+` 前缀时，`path`
的每一段，包括最后一段，都必须已经是被覆盖配置中的键，否则该覆盖抛出 `OverrideKeyError`；某一段虽然
存在但本身不是 `dict`，同样抛出 `OverrideKeyError`，因为无法继续往下走。带 `+` 前缀时，缺失的某一段
——包括最后一段——会被创建为一个空 `dict`（最后一段随后被设为给定的值，不会留空）；`+` 只会因为已存在
的非末尾段不是 `dict` 而失败，绝不会因为键缺失而失败。

`value` 只能是以下之一：`null`、`true`、`false`、一个整数（`-?[0-9]+`）、一个浮点数（`-?[0-9]+` 后面
跟一个 `.` 和至少一位数字、一个指数，或两者都有）、一个用匹配的单引号或双引号括起来且不含转义序列的
字符串，或者由前述几种之一组成的列表 `[v1, v2, ...]`（或 `[]`）——列表里绝不能再嵌套列表。空白只能
紧贴在列表的方括号内部、以及逗号前后出现；覆盖字符串中其他任何地方都不能有空白。其他任何形式都会抛出
`OverrideSyntaxError`。`apply_overrides(config, overrides)` 按顺序把 `overrides` 中的每个字符串应用
到 `config` 的一份拷贝上——对同一个 path 的更晚的覆盖会替换更早的——然后返回这份拷贝；`config` 本身
不受影响。

```py
class OverrideSyntaxError(ConfigError):
    """An override string does not match path=value or +path=value."""

class OverrideKeyError(ConfigError):
    """A path segment is missing without '+', or an existing segment is not a dict."""

def apply_overrides(config: dict, overrides: list[str]) -> dict:
    """Applies every override in order to a copy of config; returns the copy."""

class InterpolationError(ConfigError):
    """A ${...} reference addresses a missing path, or embeds a non-scalar in a string."""

class InterpolationCycleError(ConfigError):
    """Resolving some path's ${...} references needs that path's own value."""

def resolve_interpolations(config: dict) -> dict:
    """Returns a copy of config with every ${a.b} reference replaced by its value."""
```

base 和覆盖都解析完之后，字符串叶子节点可以包含一个或多个 `${a.b}` 引用，每个引用里的 `path` 与前面
形状相同，指向同一个配置里的另一个键（绝不是列表下标）。如果一个叶子节点的值*恰好*就是一个引用、字符串
里不含其他内容，它就直接取用被引用值本身，类型也照搬。如果引用只是更大字符串里的一部分，它所指向的值
就必须是 `int`、`float`、`bool` 或 `str`——绝不能是 `list`、`dict` 或 `None`——并被替换成对应的文本：
`int`、`float`、`str` 用 `str(x)`，`bool` 用 `"true"`/`"false"`。被引用的值自己也可以包含引用，会先
被解析。`resolve_interpolations(config)` 返回把每个引用都这样替换过的 `config` 拷贝；当某个引用指向
的 path 不存在于 `config` 中，或者在更大字符串里嵌入了 `list`、`dict` 或 `None` 时，抛出
`InterpolationError`；当解析某个 path 直接或间接需要它自己的值时，抛出 `InterpolationCycleError`——
错误信息里把这个环经过的 path 序列原样列出。

举例，接着 Part 1 的 `experiment`，再给它添加两个键：

```text
store["experiment"]["run_name"] = "exp-${optimizer.name}-bs${trainer.batch_size}"
store["experiment"]["eval_batch_size"] = "${trainer.batch_size}"
resolved = resolve_bases(store, "experiment")   # 同 Part 1，外加上面两个键的原样内容

overrides = ["trainer.epochs=30", "+trainer.warmup_steps=100", 'optimizer.name="adamw"']
overridden = apply_overrides(resolved, overrides)
# trainer.epochs：       20 -> 30（本来就存在）
# trainer.warmup_steps： 创建为 100（需要 '+'：之前不存在）
# optimizer.name：       "adam" -> "adamw"（一个带引号的字符串——不是裸词 adamw）

final = resolve_interpolations(overridden)
# run_name：        "exp-" + optimizer.name（"adamw"） + "-bs" + trainer.batch_size（128，转成字符串）
# eval_batch_size： 整个字符串就是一个引用 -> trainer.batch_size 自己的值，也就是 int 128
-> {"trainer": {"epochs": 30, "batch_size": 128, "lr": 0.1, "weight_decay": 0.01, "warmup_steps": 100},
    "optimizer": {"name": "adamw"},
    "run_name": "exp-adamw-bs128",
    "eval_batch_size": 128}
```

### Part 3 —— 校验

*模式*（schema）是一个用 `@dataclasses.dataclass` 装饰的类，字段的类型是 `int`、`float`、`bool`、
`str`，这些类型之一或另一个模式类组成的列表（`list[X]`），或者直接是另一个模式类，可以嵌套任意深度。
一个字段既没有 `default` 也没有 `default_factory` 时是*必需的*（required）；只要有其中之一就是可选
的，配置中缺失该字段不算错误。

`validate_config(config, schema)` 拿一个完全解析后的配置（Part 2 的输出——没有 `_base_`，没有剩下的
`${...}`）与 `schema` 比对，返回一个 `list[FieldError]`，每处不匹配一条，每条都带着从配置根开始的
*完整点号路径*，收集所有不匹配而不是找到第一个就停：

- `config` 中缺失一个必需字段——`"required field is missing"`；
- `config` 中任意层级存在一个匹配的模式类未声明的键——`"unexpected field, not declared in the schema"`；
- 一个存在的字段，其值与声明的类型不匹配：`int` 字段需要一个不是 `bool` 的 `int`（Python 里 `bool`
  是 `int` 的子类，但 `bool` 值永远不能满足 `int` 或 `float` 字段）；`float` 字段接受 `int` 或
  `float`，同样绝不接受 `bool`；`list[X]` 字段需要一个 `list`，其中每个元素都在自己的下标路径上
  （`....3`）针对 `X` 校验；嵌套模式字段需要一个 `dict`，会针对那个模式递归校验。

空列表意味着 `config` 满足 `schema`。`validate_config` 遇到不匹配不会抛出异常；只有当 `config` 本身
不是 `dict` 时才会抛出异常。

```py
@dataclasses.dataclass
class FieldError:
    path: str      # full dotted path from the config's root, e.g. "trainer.lr"
    message: str

def validate_config(config: dict, schema: type) -> list[FieldError]:
    """Returns every mismatch between config and schema, collected rather than raised."""
```

举例，针对 Part 2 得到的配置写一个模式：

```text
@dataclass
class OptimizerConfig:
    name: str
    momentum: float = 0.0

@dataclass
class TrainerConfig:
    epochs: int
    batch_size: int
    lr: float
    weight_decay: float = 0.0
    warmup_steps: int = 0

@dataclass
class ExperimentConfig:
    trainer: TrainerConfig
    optimizer: OptimizerConfig
    run_name: str
    eval_batch_size: int
    tags: list[str] = field(default_factory=list)

validate_config(final, ExperimentConfig)   # final，来自 Part 2 的例子 -> []（合法）

broken = {
    "trainer": {"epochs": 30, "batch_size": 128, "lr": "0.1", "weight_decay": 0.01,
                "warmup_steps": 100, "debug": True},
    "optimizer": {"name": "adamw"},
    "run_name": "exp-adamw-bs128",
}   # trainer.lr 是字符串；trainer 有一个未声明的 "debug" 键；缺少 eval_batch_size

validate_config(broken, ExperimentConfig) -> [
    FieldError("trainer.lr", "expected float, got str"),
    FieldError("trainer.debug", "unexpected field, not declared in the schema"),
    FieldError("eval_batch_size", "required field is missing"),
]   # 三条都在一次调用里返回——不只是第一条
```

### Part 4 —— 可复现性

`freeze_config(config)` 返回一个完全解析后的配置的不可变快照：每个 `dict` 变成一个只读映射
（`types.MappingProxyType`，递归地包住已经冻结的值），每个 `list` 变成由已经冻结的元素组成的
`tuple`；标量原样返回。对结果中任意层级赋值都会抛出 `TypeError`。

`config_hash(config)` 返回一个完全解析后的配置的稳定内容哈希：两个在嵌套 dict/list/标量意义上相等
的配置，不论它们的键插入顺序如何、不论组合与覆盖以什么顺序产生了它们，哈希值都相同；只要某个值不同，
哈希值就不同——包括 `2` 和 `2.0`，以及 `0.0` 和 `-0.0`，它们在 Python 里比较相等（`2 == 2.0`、
`-0.0 == 0.0`），但并不是同一个值。如果 `config` 中含有 `NaN` 或无穷大的 `float`，会抛出
`ValueError`，因为这个哈希无法表示它们。

在解答里说明：除了 `config` 和 `config_hash(config)` 之外，运行开始时还必须记录哪些东西，才能让这次
运行在几周后、可能在不同硬件上被精确复现。

```py
def freeze_config(config: dict):
    """Returns an immutable snapshot: nested dicts -> MappingProxyType, nested lists -> tuple."""

def config_hash(config: dict) -> str:
    """Returns a stable content hash of config, independent of key order."""
```

举例：

```text
frozen = freeze_config(final)
frozen["trainer"]["epochs"]          # 30 -- 正常读取
frozen["trainer"]["epochs"] = 999    # 抛出 TypeError

c1 = {"trainer": {"epochs": 30, "batch_size": 128}, "optimizer": {"name": "adamw"}}
c2 = {"optimizer": {"name": "adamw"}, "trainer": {"batch_size": 128, "epochs": 30}}  # 内容相同，键顺序不同
config_hash(c1) == config_hash(c2)                                    # True

c3 = {**c1, "trainer": {**c1["trainer"], "epochs": 31}}
config_hash(c3) != config_hash(c1)                                    # True
```

## 参考解答

<details>
<summary>展开参考解答</summary>

开始写代码前值得先确认两点：覆盖路径能不能索引到列表里（这里假定不能——覆盖和插值都只处理嵌套的映射），
以及一个无法识别的键应该让校验失败还是悄悄放过（这里假定严格失败，这也是 `pydantic` 的
`extra="forbid"` 的选择，因为一个被悄悄忽略的拼写错误，恰恰是模式存在的意义所在——就是要抓住这种错误）。

### Part 1

`resolve_bases` 是在 `_base_` 图上做环检测，形式与在任何依赖图上做环检测相同：`visiting` 是当前正在
解析路径上的名字序列，一个名字再次出现在其中就意味着闭合了一个环；`memo` 在整次调用中共享，让被两个
不同配置用作 base 的同一个名字——菱形结构——只被解析一次。`deep_merge` 唯一的微妙之处在于它处理
`overlay` 中 `dict` 值的分支：它总是递归，在 `base[key]` 本身不是 `dict`（包括根本不存在）的地方用
`{}` 代替，而不是直接采用整个 `overlay[key]`。这两种做法几乎总是等价的——唯独当 `overlay[key]` 内部
某处嵌着 `DELETE` 时不同：递归仍然能找到并删掉那个键，而直接拷贝会把这个哨兵值原样留在结果里。
`_clone` 让写入结果的每个值都独立于它来自哪个 dict 或 list，这样解析一个名字之后，对结果的任何修改
都不可能反过来影响 `store`。

```python
import dataclasses
import hashlib
import json
import re
import typing
from types import MappingProxyType


class ConfigError(Exception):
    """Base class for every error raised below."""


class ConfigNotFoundError(ConfigError):
    pass


class ConfigCycleError(ConfigError):
    pass


class _DeleteType:
    def __repr__(self) -> str:
        return "DELETE"


DELETE = _DeleteType()


def _clone(value):
    if isinstance(value, dict):
        return {k: _clone(v) for k, v in value.items()}
    if isinstance(value, list):
        return [_clone(v) for v in value]
    return value


def deep_merge(base: dict, overlay: dict) -> dict:
    result = {k: _clone(v) for k, v in base.items()}
    for key, value in overlay.items():
        if value is DELETE:
            result.pop(key, None)
        elif isinstance(value, dict):
            prior = result.get(key)
            # NOTE: recurse against {}, not a wholesale _clone(value), even when base has no dict
            # here yet -- otherwise a DELETE nested inside value would never take effect.
            result[key] = deep_merge(prior if isinstance(prior, dict) else {}, value)
        else:
            result[key] = _clone(value)
    return result


def resolve_bases(store: dict, name: str) -> dict:
    return _resolve(store, name, [], {})


def _resolve(store, name, visiting, memo):
    if name in memo:                  # already resolved earlier in this call -- a diamond base
        return memo[name]
    if name not in store:
        raise ConfigNotFoundError(f"no config named {name!r} in the store")
    if name in visiting:               # NOTE: visiting is the current _base_ path, not the whole call
        chain = " -> ".join(visiting[visiting.index(name):] + [name])
        raise ConfigCycleError(f"cycle in _base_: {chain}")
    visiting.append(name)
    merged = {}
    for base_name in store[name].get("_base_", []):
        merged = deep_merge(merged, _resolve(store, base_name, visiting, memo))
    own_keys = {k: v for k, v in store[name].items() if k != "_base_"}
    merged = deep_merge(merged, own_keys)
    visiting.pop()
    memo[name] = merged
    return merged
```

`DELETE` 只在它作为 overlay 的那一次合并中起作用：如果配置 `a` 在某个 `_base_` 之上设置
`x.y: DELETE`，单独解析 `a` 时，`x` 下面根本没有 `y` 这个键——而不是有一个值为 `DELETE` 的 `y`。如果
在 `a` *之后*列出的另一个 base，通过它自己独立的路径又达到了原来的 `x.y`，这个值就会被恢复，因为
`DELETE` 从来不会留存到解析结果里。`_base_` 组合扮演的角色和 Hydra 的 defaults list 或 OmegaConf
自己的 merge 一样：一个配置是它的各个 base 能够产生的格（lattice）中的一个点，而不是产生它的那些
编辑操作的记录。

`memo` 让每个名字最多被解析一次，不论它被多少个配置用作 base；但解析一个名字本身的开销，仍然与它自己
合并结果的大小成正比，所以一条只由单一父级组成的 `_base_` 链——每个配置都只把上一个当作自己的
base——开销会是链长的平方，因为每一层都要把下面已经合并好的一切重新克隆一遍。

### Part 2

解析一个覆盖字符串是两套独立的语法在 `=` 处拼接起来：`_PATH_RE` 完整匹配左边，所以一旦匹配成功，
`path_part.split(".")` 就是安全的——不可能有空的段；`_parse_value`/`_parse_scalar` 依次用
`null`/`true`/`false`、浮点数模式、整数模式、带引号的字符串、然后是列表，去完整匹配右边。浮点数必须
排在整数前面，因为带小数点的值绝不能落到 `int(...)` 上，那样会直接报错。把列表内部文本按 `,` 拆开时，
必须跳过带引号元素内部的逗号：`_split_list_items` 逐字符跟踪当前是否处于引号内，因为 `str.split(",")`
无法把 `"a,b"` 和两个独立元素 `"a"`、`"b"` 区分开。应用一个覆盖绝不会合并：`_walk_and_set` 要么创建
一个段（仅在 `+` 下），要么要求它已经存在，并且总是直接整体覆盖最后一段，这与 `deep_merge` 处理非
dict 类型 overlay 值时的规则相同。

```python
class OverrideSyntaxError(ConfigError):
    pass


class OverrideKeyError(ConfigError):
    pass


_PATH_RE = re.compile(r'[A-Za-z_]\w*(?:\.[A-Za-z_]\w*)*')
_INT_RE = re.compile(r'-?\d+', re.ASCII)
_FLOAT_RE = re.compile(r'-?\d+\.\d+(?:[eE][+-]?\d+)?|-?\d+[eE][+-]?\d+', re.ASCII)
_STRING_RE = re.compile(r"'([^']*)'|\"([^\"]*)\"")


def _parse_scalar(token: str):
    if token == "null":
        return None
    if token == "true":
        return True
    if token == "false":
        return False
    if _FLOAT_RE.fullmatch(token):
        return float(token)
    if _INT_RE.fullmatch(token):
        return int(token)
    m = _STRING_RE.fullmatch(token)
    if m:
        return m.group(1) if m.group(1) is not None else m.group(2)
    raise OverrideSyntaxError(f"not a valid value: {token!r}")


def _split_list_items(inner: str) -> list:
    if not inner.strip():
        return []
    items, quote, start = [], None, 0
    for i, ch in enumerate(inner):
        if quote:
            if ch == quote:
                quote = None
        elif ch in "'\"":
            quote = ch
        elif ch == ",":                    # NOTE: a comma inside a quoted element is not a separator
            items.append(inner[start:i])
            start = i + 1
    items.append(inner[start:])
    return items


def _parse_value(token: str):
    if token.startswith("[") and token.endswith("]"):
        return [_parse_scalar(item.strip()) for item in _split_list_items(token[1:-1])]
    return _parse_scalar(token)


def parse_override(text: str):
    create = text.startswith("+")
    body = text[1:] if create else text
    path_part, sep, value_part = body.partition("=")
    if not sep or not _PATH_RE.fullmatch(path_part):
        raise OverrideSyntaxError(f"not a valid override: {text!r}")
    return create, path_part.split("."), _parse_value(value_part)


def _walk_and_set(config: dict, path: list, value, create: bool, text: str) -> None:
    node = config
    for segment in path[:-1]:
        if segment not in node:
            if not create:
                raise OverrideKeyError(f"{text}: {'.'.join(path)} does not exist (use '+' to create it)")
            node[segment] = {}
        elif not isinstance(node[segment], dict):
            raise OverrideKeyError(f"{text}: {segment!r} is not a mapping, cannot descend into it")
        node = node[segment]
    last = path[-1]
    if last not in node and not create:
        raise OverrideKeyError(f"{text}: {'.'.join(path)} does not exist (use '+' to create it)")
    node[last] = value


def apply_overrides(config: dict, overrides: list[str]) -> dict:
    result = _clone(config)
    for text in overrides:
        create, path, value = parse_override(text)
        _walk_and_set(result, path, value, create, text)
    return result
```

插值解析每次处理一个点号路径，递归形式与 `resolve_bases` 相同——用一个 `visiting` 列表记录当前引用链，
用一个 `memo` 记录已经解析过的 path——只是这里的图跑在字符串值上而不是 `_base_` 列表上，是随着
`${...}` 模式被发现而惰性发掘出来的，而不是预先声明好的。`_resolve_ref` 从 `root`（原始的、插值之前
的配置）里取出一个 path 的原始值，所以不论有多少其他引用指向它，一个引用总是针对该 path 实际被设置的
值来解析。`_resolve_value` 是唯一决定要不要把一个引用当作整串原样返回、还是当作嵌在更大字符串里的一
部分、只取其文本的地方：用 `fullmatch` 对 `sub` 做的这次判断，与 `deep_merge` 区分“替换整个值”和
“合并进一部分”是同一种区分。在 `isinstance(target, (int, float, str))` 之前先检查
`isinstance(target, bool)`，原因和处理 `bool` 时通常一样：不这样做的话它会走进 `int`/`float`/`str`
分支，打印成 `"True"`，而不是这套系统自己的覆盖语法能读回去的小写 `"true"`。

```python
class InterpolationError(ConfigError):
    pass


class InterpolationCycleError(ConfigError):
    pass


_REF_RE = re.compile(r'\$\{([A-Za-z_]\w*(?:\.[A-Za-z_]\w*)*)\}')


def _get_path(root: dict, path: list):
    node = root
    for segment in path:
        if not isinstance(node, dict) or segment not in node:
            raise InterpolationError(f"unknown path: {'.'.join(path)}")
        node = node[segment]
    return node


def _resolve_ref(root, path_str, memo, visiting):
    if path_str in memo:
        return memo[path_str]
    if path_str in visiting:
        chain = " -> ".join(visiting[visiting.index(path_str):] + [path_str])
        raise InterpolationCycleError(f"circular interpolation: {chain}")
    visiting.append(path_str)
    value = _resolve_value(root, _get_path(root, path_str.split(".")), memo, visiting)
    visiting.pop()
    memo[path_str] = value
    return value


def _resolve_value(root, value, memo, visiting):
    if isinstance(value, str):
        whole = _REF_RE.fullmatch(value)
        if whole:                                         # the entire string is one reference
            return _resolve_ref(root, whole.group(1), memo, visiting)

        def substitute(m):
            target = _resolve_ref(root, m.group(1), memo, visiting)
            if isinstance(target, bool):                   # NOTE: check bool before int -- bool is an int
                return "true" if target else "false"
            if isinstance(target, (int, float, str)):
                return str(target)
            raise InterpolationError(f"cannot embed {m.group(1)!r} (a {type(target).__name__}) in a string")

        return _REF_RE.sub(substitute, value)
    if isinstance(value, dict):
        return {k: _resolve_value(root, v, memo, visiting) for k, v in value.items()}
    if isinstance(value, list):
        return [_resolve_value(root, v, memo, visiting) for v in value]
    return value


def resolve_interpolations(config: dict) -> dict:
    return _resolve_value(config, config, {}, [])
```

解析并应用一个覆盖所花的时间与它自身的长度成线性关系。解析插值所花的时间与配置的总大小加上其中
`${...}` 出现的次数成线性关系，因为不论有多少字符串引用同一个 path，`memo` 都保证它最多被解析一次。

### Part 3

`validate_config` 同时遍历 `schema` 和 `config`，而不是只看 `config`——只有模式知道什么是必需的、
一个存在的字段的类型应该是什么，所以只扫一遍 `config` 能报出类型不匹配，却永远报不出缺失字段。对每个
已声明字段，它先检查是否存在，存在的话再交给 `_check_type`，本质上是共享同一种结构的三种情况：
`list[X]` 递归检查每个元素，各自带着自己的下标路径；嵌套模式递归调用 `validate_config`，带着一个
嵌套路径；基本类型就是一次 `isinstance` 检查，并且明确把 `bool` 排除在 `int` 和 `float` 之外，因为
Python 里 `isinstance(True, int)` 是 `True`，但布尔值永远不能替代一个数值字段。未声明的键在每一层
检查一次，在所有已声明字段检查完之后，把 `config` 自己的键与这一层已收集到的字段名比对——所以哪怕是
嵌套模式的 `list[X]` 里第三层深处多出来的一个键，也会带着它自己的完整路径被报告出来。

```python
@dataclasses.dataclass
class FieldError:
    path: str
    message: str


def _is_schema(tp) -> bool:
    return isinstance(tp, type) and dataclasses.is_dataclass(tp)


def _required(f: dataclasses.Field) -> bool:
    return f.default is dataclasses.MISSING and f.default_factory is dataclasses.MISSING


def _check_type(value, tp, path: str) -> list[FieldError]:
    if typing.get_origin(tp) is list:
        elem_type = typing.get_args(tp)[0]
        if not isinstance(value, list):
            return [FieldError(path, f"expected list, got {type(value).__name__}")]
        return [err for i, item in enumerate(value) for err in _check_type(item, elem_type, f"{path}.{i}")]
    if _is_schema(tp):
        return validate_config(value, tp, path)
    if tp is float:                                 # NOTE: an int widens to float; a bool never does
        ok = isinstance(value, (int, float)) and not isinstance(value, bool)
    elif tp in (int, bool, str):
        ok = isinstance(value, tp) and not (tp is int and isinstance(value, bool))
    else:
        return [FieldError(path, f"unsupported schema type {tp!r}")]
    return [] if ok else [FieldError(path, f"expected {tp.__name__}, got {type(value).__name__}")]


def validate_config(config: dict, schema: type, _path: str = "") -> list[FieldError]:
    if not isinstance(config, dict):
        return [FieldError(_path or "<root>", f"expected a mapping, got {type(config).__name__}")]
    hints = typing.get_type_hints(schema)
    fields = {f.name: f for f in dataclasses.fields(schema)}
    errors = []
    for name, f in fields.items():
        full = f"{_path}.{name}" if _path else name
        if name not in config:
            if _required(f):
                errors.append(FieldError(full, "required field is missing"))
            continue
        errors.extend(_check_type(config[name], hints[name], full))
    for name in config:
        if name not in fields:
            full = f"{_path}.{name}" if _path else name
            errors.append(FieldError(full, "unexpected field, not declared in the schema"))
    return errors
```

每个分支都是往一个列表里追加，然后继续往下走，而不是发现第一个问题就返回——一个缺失字段不会让它周围
的字段跳过类型检查，`list[X]` 里下标 2 处的一个坏元素也不会让下标 5 被跳过。`_path` 贯穿每一次递归
调用，最顶层默认为 `""`，正是它让每处检查都能报告*在哪里*失败，而不需要要求对外的两参数签名的调用者
自己提供这个参数。开销与 schema 和 config 一起能到达的字段数与列表元素数之和成线性关系，因为每一个
都恰好被访问一次。

### Part 4

`freeze_config` 的递归形式和 `_clone` 一样，只是生成的是只读容器而不是全新的可变容器：
`MappingProxyType` 包住一个 `dict` 时不会拷贝它，所以它必须包住一个*新建的*、内部元素已经冻结过的
dict，绝不能包住原始的那个——直接包住原始 dict 的话，之后对它的任何修改都会直接透过这个所谓冻结的
视图显现出来。`config_hash` 完全依赖 `json.dumps`：`sort_keys=True` 让键的顺序变得无关紧要，
`separators=(",", ":")` 去掉了默认分隔符带来的空白差异，而 Python 的浮点数格式化本身就是能还原出
那个确切比特模式的最短十进制字符串——对每个值都是确定的，所以 `0.1` 和 `1e-1`（同一个 `float`）
格式化结果相同，而 `0.0` 和 `-0.0`（比特模式不同、但比较结果相等）分别格式化成 `"0.0"` 和 `"-0.0"`，
从而正确地哈希出两个不同的摘要。`allow_nan=False` 把 Python 里非标准的 `NaN`/`Infinity` 输出变成了
一个 `ValueError`，而不是悄悄地在一段其他 JSON 解析器都读不回来的文本上算出一个哈希。

```python
def freeze_config(config):
    if isinstance(config, dict):
        return MappingProxyType({k: freeze_config(v) for k, v in config.items()})
    if isinstance(config, list):
        return tuple(freeze_config(v) for v in config)
    return config


def config_hash(config: dict) -> str:
    canonical = json.dumps(config, sort_keys=True, separators=(",", ":"), allow_nan=False)
    return hashlib.sha256(canonical.encode("utf-8")).hexdigest()
```

解析后配置的哈希只能证明两次运行从完全相同的配置开始；对配置没有捕捉到的一切，它什么都说明不了。在
运行开始时和 `config_hash` 一起保存的清单，还需要：确切的代码，以一次干净工作树的提交哈希的形式记录，
因为配置的含义取决于训练脚本拿它做了什么；数据的快照，用内容哈希或不可变的数据集版本号标识，而不是
一个可变的路径，因为“这个路径下的文件”到下次运行时可能已经悄悄变了；每一个相关的库版本，理想情况下
是一份完整的依赖锁定文件或容器镜像摘要，因为即使代码和配置完全相同，数值结果也可能在框架或 CUDA 版本
之间发生变化；用到的每一个种子——Python 自己的 `random`、NumPy、训练框架，以及数据加载器的乱序分别
都要算，因为它们各自独立取值；以及硬件本身，加速器的型号和数量，因为并行方式会改变一个 batch 的组成
方式和浮点数归约的顺序，而且对同一个操作、同一份输入，有些内核在不同加速器架构之间也不是逐比特一致的。

### 追问

- 一次 sweep 是若干覆盖列表的笛卡尔积，每个轴一个列表——`lr` 取 `{0.01, 0.1}`、`batch_size` 取
  `{32, 128}`，就有四次运行——每次都独立组合、独立哈希；sweep 自己的清单是这些 `config_hash` 值组成
  的列表，而不是对整个 sweep 算一个哈希。
- 密钥绝不应该出现在会被哈希、冻结、记录下来的配置里，因为哈希和冻结对象恰恰就是会把它们泄露出去的
  东西；配置里应该只用引用来指代一个密钥——一个环境变量名或者密钥管理服务里的一个 key——在加载时才
  解析，解析结果留在 `config_hash` 的输入之外。
- 对两次运行做 diff，是逐个键遍历两份解析后的配置，在匹配的嵌套 dict 处递归、其余地方按值是否相同
  比较，报告出每一个被新增、删除或修改了的点号路径；单看 `config_hash` 只能回答两次运行是否不同，
  回答不了差异在哪里。
- 要做静态检查，模式就得是配置构建出来所依据的东西，而不只是事后拿来校验的对象：直接构造
  `TrainerConfig(epochs=30, lr=0.1)`，能让类型检查器在代码运行前就抓住类型错误或缺失参数，代价是
  失去了组合与 CLI 覆盖赖以操作的那种自由形式的 dict 字面量；`pydantic` 介于两者之间，一步把一个
  dict 解析并强制转换成一个带类型、经过校验的对象。
- `.py` 配置在导入时会运行任意代码，这恰恰是它难以被通用地合并、diff、覆盖或哈希的原因：这四件事都
  需要在一个静止不变的结构上检查和改写值，普通的嵌套 dict 满足这一点，一个可执行的模块不满足。

<details>
<summary>验证代码（可运行）</summary>

```python
import copy
import random


def expect(exc, fn, *args):
    try:
        fn(*args)
    except exc:
        return
    raise AssertionError(f"expected {exc.__name__} from {fn.__name__}{args!r}")


# ---- worked examples, exactly as stated (Part 1-4) ----
store = {
    "base": {"trainer": {"epochs": 10, "batch_size": 32, "lr": 0.1},
             "optimizer": {"name": "sgd", "momentum": 0.9}},
    "wide": {"_base_": ["base"], "trainer": {"batch_size": 128}},
    "regularized": {"trainer": {"weight_decay": 0.01}},
    "experiment": {"_base_": ["wide", "regularized"], "trainer": {"epochs": 20},
                   "optimizer": {"name": "adam", "momentum": DELETE}},
}
resolved = resolve_bases(store, "experiment")
assert resolved == {"trainer": {"epochs": 20, "batch_size": 128, "lr": 0.1, "weight_decay": 0.01},
                     "optimizer": {"name": "adam"}}

fresh = deep_merge({}, {"data": {"path": DELETE, "shuffle": True}})
assert fresh == {"data": {"shuffle": True}}  # DELETE nested under a fresh key still takes effect

diamond_store = {"base": {"optimizer": {"name": "sgd", "momentum": 0.9}},
                  "a": {"_base_": ["base"], "optimizer": {"momentum": DELETE}},
                  "b": {"_base_": ["base"]}, "combo": {"_base_": ["a", "b"]}}
assert resolve_bases(diamond_store, "combo") == {"optimizer": {"name": "sgd", "momentum": 0.9}}

expect(ConfigCycleError, resolve_bases, {"a": {"_base_": ["b"]}, "b": {"_base_": ["a"]}}, "a")
expect(ConfigCycleError, resolve_bases, {"a": {"_base_": ["a"]}}, "a")
expect(ConfigNotFoundError, resolve_bases, {"a": {"_base_": ["ghost"]}}, "a")

store["experiment"]["run_name"] = "exp-${optimizer.name}-bs${trainer.batch_size}"
store["experiment"]["eval_batch_size"] = "${trainer.batch_size}"
resolved = resolve_bases(store, "experiment")
overrides = ["trainer.epochs=30", "+trainer.warmup_steps=100", 'optimizer.name="adamw"']
overridden = apply_overrides(resolved, overrides)
assert resolved["trainer"]["epochs"] == 20  # apply_overrides did not touch its input
final = resolve_interpolations(overridden)
assert final == {
    "trainer": {"epochs": 30, "batch_size": 128, "lr": 0.1, "weight_decay": 0.01, "warmup_steps": 100},
    "optimizer": {"name": "adamw"}, "run_name": "exp-adamw-bs128", "eval_batch_size": 128,
}
assert type(final["eval_batch_size"]) is int  # whole-value reference keeps the target's own type

expect(OverrideKeyError, apply_overrides, resolved, ["nope.x=1"])
expect(OverrideKeyError, apply_overrides, resolved, ["trainer.epochs.nested=1"])
expect(OverrideSyntaxError, parse_override, "a.b=nope")
assert parse_override("a.b=[1, 2.5, true, null, 'x,y']") == (False, ["a", "b"], [1, 2.5, True, None, "x,y"])

expect(InterpolationCycleError, resolve_interpolations, {"a": "${b}", "b": "${a}"})
expect(InterpolationCycleError, resolve_interpolations, {"a": "${a}"})
expect(InterpolationError, resolve_interpolations, {"a": "${nope.x}"})
expect(InterpolationError, resolve_interpolations, {"a": [1, 2], "b": "value is ${a}"})
assert resolve_interpolations({"a": [1, 2], "b": "${a}"})["b"] == [1, 2]


@dataclasses.dataclass
class OptimizerConfig:
    name: str
    momentum: float = 0.0


@dataclasses.dataclass
class TrainerConfig:
    epochs: int
    batch_size: int
    lr: float
    weight_decay: float = 0.0
    warmup_steps: int = 0


@dataclasses.dataclass
class ExperimentConfig:
    trainer: TrainerConfig
    optimizer: OptimizerConfig
    run_name: str
    eval_batch_size: int
    tags: list[str] = dataclasses.field(default_factory=list)


assert validate_config(final, ExperimentConfig) == []
broken = {"trainer": {"epochs": 30, "batch_size": 128, "lr": "0.1", "weight_decay": 0.01,
                       "warmup_steps": 100, "debug": True},
          "optimizer": {"name": "adamw"}, "run_name": "exp-adamw-bs128"}
assert validate_config(broken, ExperimentConfig) == [
    FieldError("trainer.lr", "expected float, got str"),
    FieldError("trainer.debug", "unexpected field, not declared in the schema"),
    FieldError("eval_batch_size", "required field is missing"),
]

frozen = freeze_config(final)
assert frozen["trainer"]["epochs"] == 30
try:
    frozen["trainer"]["epochs"] = 999
    raise AssertionError("expected TypeError")
except TypeError:
    pass

c1 = {"trainer": {"epochs": 30, "batch_size": 128}, "optimizer": {"name": "adamw"}}
c2 = {"optimizer": {"name": "adamw"}, "trainer": {"batch_size": 128, "epochs": 30}}
assert config_hash(c1) == config_hash(c2)
c3 = {**c1, "trainer": {**c1["trainer"], "epochs": 31}}
assert config_hash(c3) != config_hash(c1)
assert -0.0 == 0.0 and config_hash({"x": -0.0}) != config_hash({"x": 0.0})
assert 2 == 2.0 and config_hash({"x": 2}) != config_hash({"x": 2.0})
expect(ValueError, config_hash, {"x": float("nan")})
print("worked examples OK")


# ==================== randomised cross-validation ====================

# ---- Part 1: random base graphs against an independent, non-memoized resolver ----
LEAF_POOL = ["trainer.epochs", "trainer.batch_size", "optimizer.name", "optimizer.momentum", "data.path"]


def _set_dotted(d, dotted, value):
    node, parts = d, dotted.split(".")
    for part in parts[:-1]:
        node = node.setdefault(part, {})
    node[parts[-1]] = value


def random_own_keys(rng, deletable=False):
    own = {}
    for _ in range(rng.randint(0, 3)):
        v = DELETE if deletable and rng.random() < 0.25 else rng.choice(
            [rng.randint(0, 100), round(rng.uniform(0, 1), 3), rng.choice(["sgd", "adam"]), rng.choice([True, False])])
        _set_dotted(own, rng.choice(LEAF_POOL), v)
    return own


def random_acyclic_store(rng, n):
    names = [f"c{i}" for i in range(n)]
    store = {}
    for i, name in enumerate(names):
        k = min(i, rng.randint(0, 2))
        cfg = random_own_keys(rng, deletable=True)
        if k:
            cfg["_base_"] = rng.sample(names[:i], k=k)
        store[name] = cfg
    return store, names


def naive_deep_merge(base, overlay):
    """Independent restatement of the merge rule, sharing no helper with deep_merge."""
    result = dict(base)
    for key, value in overlay.items():
        if value is DELETE:
            result.pop(key, None)
        elif isinstance(value, dict):
            prior = result.get(key)
            result[key] = naive_deep_merge(prior if isinstance(prior, dict) else {}, value)
        else:
            result[key] = value
    return result


def naive_resolve(store, name, stack=()):
    if name not in store:
        raise ConfigNotFoundError(name)
    if name in stack:
        raise ConfigCycleError(name)
    merged = {}
    for base_name in store[name].get("_base_", []):
        merged = naive_deep_merge(merged, naive_resolve(store, base_name, stack + (name,)))
    return naive_deep_merge(merged, {k: v for k, v in store[name].items() if k != "_base_"})


rng = random.Random(0)
for _ in range(500):
    trial_store, names = random_acyclic_store(rng, rng.randint(1, 8))
    name = rng.choice(names)
    assert resolve_bases(trial_store, name) == naive_resolve(trial_store, name)
for _ in range(150):  # every config on a ring points at the next: always a cycle
    n = rng.randint(2, 6)
    ring = [f"r{i}" for i in range(n)]
    ring_store = {nm: {"_base_": [ring[(i + 1) % n]]} for i, nm in enumerate(ring)}
    expect(ConfigCycleError, resolve_bases, ring_store, rng.choice(ring))
print("Part 1 random cross-check OK (500 base graphs, 150 forced cycles)")

# ---- Part 2: random overrides, predicted with _set_dotted instead of re-parsing ----
def _literal(value):
    if isinstance(value, bool):
        return "true" if value else "false"
    return f"'{value}'" if isinstance(value, str) else str(value)


FRESH_PATHS = ["trainer.warmup_steps", "data.shuffle", "optimizer.beta"]
EXISTING_PATHS = ["trainer.epochs", "trainer.batch_size", "optimizer.name"]
rng = random.Random(1)
for _ in range(400):
    cfg = {"trainer": {"epochs": 10, "batch_size": 32}, "optimizer": {"name": "sgd"}}
    expected = copy.deepcopy(cfg)
    texts = []
    for _ in range(rng.randint(1, 4)):
        create = rng.random() < 0.5
        path = rng.choice(FRESH_PATHS if create else EXISTING_PATHS)
        value = rng.choice([rng.randint(-50, 50), rng.choice(["adamw", "resnet"]), rng.choice([True, False])])
        texts.append(("+" if create else "") + path + "=" + _literal(value))
        _set_dotted(expected, path, value)
    assert apply_overrides(cfg, texts) == expected
print("Part 2 overrides random cross-check OK (400 trials)")

# ---- Part 2: random interpolation DAGs, predicted while they are built ----
rng = random.Random(2)
for _ in range(300):
    names = [f"n{i}" for i in range(rng.randint(1, 5))]
    cfg, expected = {}, {}
    for i, name in enumerate(names):
        r = rng.random()
        if i == 0 or r < 0.35:
            v = rng.choice([rng.randint(0, 50), rng.choice(["x", "y", "z"])])
            cfg[name] = expected[name] = v
        elif r < 0.65:
            target = rng.choice(names[:i])
            cfg[name] = "${" + target + "}"
            expected[name] = expected[target]                        # whole-value: keeps target's type
        else:
            target = rng.choice(names[:i])
            cfg[name] = f"v-${{{target}}}-end"
            expected[name] = f"v-{expected[target]}-end"              # embedded: always text
    assert resolve_interpolations(cfg) == expected
for _ in range(100):  # every leaf points at the next, wrapping around: always a cycle
    ring = [f"m{i}" for i in range(rng.randint(2, 5))]
    ring_cfg = {nm: "${" + ring[(i + 1) % len(ring)] + "}" for i, nm in enumerate(ring)}
    expect(InterpolationCycleError, resolve_interpolations, ring_cfg)
print("Part 2 interpolation random cross-check OK (300 DAGs, 100 forced cycles)")


# ---- Part 3: random schemas and corrupted configs against an independent validator ----
def naive_validate(config, schema, path=""):
    if not isinstance(config, dict):
        return [(path or "<root>", "not a mapping")]
    fields = {f.name: f for f in dataclasses.fields(schema)}
    hints = typing.get_type_hints(schema)
    out = []
    for fname, f in fields.items():
        full = f"{path}.{fname}" if path else fname
        no_default = f.default is dataclasses.MISSING and f.default_factory is dataclasses.MISSING
        if fname not in config:
            if no_default:
                out.append((full, "missing"))
            continue
        out.extend(naive_check(config[fname], hints[fname], full))
    for k in config:
        if k not in fields:
            out.append((f"{path}.{k}" if path else k, "extra"))
    return out


def naive_check(val, ftype, path):
    if dataclasses.is_dataclass(ftype):
        return naive_validate(val, ftype, path)
    if getattr(ftype, "__origin__", None) is list:
        elem = ftype.__args__[0]
        if not isinstance(val, list):
            return [(path, "not a list")]
        return [e for i, item in enumerate(val) for e in naive_check(item, elem, f"{path}.{i}")]
    ok = {float: isinstance(val, (int, float)) and not isinstance(val, bool),
          int: isinstance(val, int) and not isinstance(val, bool),
          bool: isinstance(val, bool), str: isinstance(val, str)}.get(ftype, False)
    return [] if ok else [(path, "type")]


@dataclasses.dataclass
class LayerConfig:
    hidden_size: int
    activation: str = "relu"


@dataclasses.dataclass
class ModelConfig:
    layers: list[LayerConfig]
    dropout: float = 0.0


def sample_value(rng, tp):
    if dataclasses.is_dataclass(tp):
        return {f.name: sample_value(rng, typing.get_type_hints(tp)[f.name]) for f in dataclasses.fields(tp)}
    if getattr(tp, "__origin__", None) is list:
        return [sample_value(rng, tp.__args__[0]) for _ in range(rng.randint(0, 3))]
    return {int: rng.randint(0, 50), float: round(rng.uniform(0, 10), 2),
            bool: rng.choice([True, False])}.get(tp, rng.choice(["relu", "run-a"]))


WRONG_TYPE_POOL = [3, 2.5, "text", True, None, [1, 2], {"x": 1}]


def corrupt(rng, config, schema):
    config = copy.deepcopy(config)
    hints = typing.get_type_hints(schema)
    if config and rng.random() < 0.3:
        del config[rng.choice(list(config))]
    if rng.random() < 0.3:
        config[f"extra_{rng.randint(0, 999)}"] = 1
    if config and rng.random() < 0.4:
        key = rng.choice(list(config))
        tp = hints.get(key)
        if dataclasses.is_dataclass(tp) and isinstance(config[key], dict):
            config[key] = corrupt(rng, config[key], tp)
        elif getattr(tp, "__origin__", None) is list and config[key]:
            i, elem = rng.randrange(len(config[key])), tp.__args__[0]
            config[key][i] = (corrupt(rng, config[key][i], elem) if dataclasses.is_dataclass(elem)
                              else rng.choice(WRONG_TYPE_POOL))
        elif tp is not None:
            config[key] = rng.choice(WRONG_TYPE_POOL)
    return config


rng = random.Random(3)
for schema in (ExperimentConfig, ModelConfig):
    for _ in range(400):
        valid = sample_value(rng, schema)
        assert validate_config(valid, schema) == []
        mutant = corrupt(rng, valid, schema)
        sol_paths = {e.path for e in validate_config(mutant, schema)}
        naive_paths = {p for p, _ in naive_validate(mutant, schema)}
        assert sol_paths == naive_paths, (schema.__name__, mutant, sol_paths, naive_paths)
print("Part 3 fuzz cross-check OK (800 trials across two schemas)")

# ---- Part 4: hash stability under key order and across equal merge orders ----
def shuffle_dict(rng, value):
    if isinstance(value, dict):
        items = [(k, shuffle_dict(rng, v)) for k, v in value.items()]
        rng.shuffle(items)
        return dict(items)
    return [shuffle_dict(rng, v) for v in value] if isinstance(value, list) else value


def flat_paths(node, prefix=()):
    for k, v in node.items():
        p = prefix + (k,)
        yield from flat_paths(v, p) if isinstance(v, dict) else [p]


rng = random.Random(4)
for _ in range(300):
    trial_store, names = random_acyclic_store(rng, rng.randint(1, 6))
    resolved_config = resolve_bases(trial_store, rng.choice(names))
    assert config_hash(resolved_config) == config_hash(shuffle_dict(rng, resolved_config))

    sibling = random_own_keys(rng)
    if sibling and not (set(sibling) & set(resolved_config)):     # disjoint keys: merge order should not matter
        assert config_hash(deep_merge(resolved_config, sibling)) == config_hash(deep_merge(sibling, resolved_config))

    paths = list(flat_paths(resolved_config))
    if paths:
        target = rng.choice(paths)
        mutated = copy.deepcopy(resolved_config)
        node = mutated
        for seg in target[:-1]:
            node = node[seg]
        node[target[-1]] = 999999 if node[target[-1]] != 999999 else 888888
        assert config_hash(mutated) != config_hash(resolved_config)
print("Part 4 hash-stability sweep OK (300 trials)")

print("all checks passed")
```

</details>

</details>
