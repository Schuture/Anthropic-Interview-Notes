# ML Experiment Configuration System

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Coding · configuration systems, validation | ★★☆☆☆ | Hard | RE · RS · MLE | deep-merge, cycle-detection, cli-overrides, schema-validation, reproducibility | 4 parts / 55 min | Technical phone screen |
<!-- meta:end -->

## Problem

A *config* is a value built from `dict`s with `str` keys, `list`s, and scalars — `int`, `float`,
`bool`, `str`, or `None` — nested to any depth. A *config store* is a `dict[str, dict]` mapping a
name to a config that is itself a `dict` at its top level; every function below takes a store as
its first argument, already loaded in memory (nothing here parses YAML or any other text format),
and returns new configs without mutating the store or any config it was given.

### Part 1 — Composition

A config may declare `_base_: [n1, n2, ...]` — a list of other names in the store — as a key at its
own top level; a config without a `_base_` key has none. `DELETE` is a sentinel value exported by
this module, usable anywhere a config may hold a scalar.

`deep_merge(base, overlay)` returns a new dict, computed key by key over the union of `base`'s and
`overlay`'s keys:

- a key whose `overlay` value is `DELETE` is absent from the result, whether or not `base` had it;
- a key whose `overlay` value is a `dict` maps, in the result, to `deep_merge(b, overlay[key])`,
  where `b` is `base[key]` when that is itself a `dict`, and `{}` otherwise — so a `DELETE` nested
  inside `overlay[key]` still removes a key of that inner dict even where `base` had no dict there
  to begin with;
- a key present only in `base` keeps `base`'s value;
- every other key — present in `overlay`, and neither `DELETE` nor a `dict` — takes `overlay`'s
  value, replacing whatever `base` held there entirely, dict or not.

No value in the result is shared with `base` or `overlay`.

`resolve_bases(store, name)` returns `name`'s fully composed config: an empty dict, with the
resolution of every name in `store[name]`'s own `_base_` list deep-merged in, left to right, then
`store[name]`'s own keys (every key except `_base_`) deep-merged in on top. A name in `_base_` may
itself have a `_base_`, resolved the same way, recursively; the result never contains a `_base_`
key. `resolve_bases` raises `ConfigNotFoundError` when `name`, or some name reachable through a
`_base_` list, is not a key of `store`, and raises `ConfigCycleError` when a config is its own base,
directly or through others — with the cycle spelled out in the message as the sequence of names it
passes through, e.g. `"a -> b -> a"`.

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

Example, over a store of four named configs:

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
# trainer:   base gives epochs=10, batch_size=32, lr=0.1
#          + wide overlays batch_size=128                 -> epochs=10, batch_size=128, lr=0.1
#          + regularized overlays weight_decay=0.01        -> ...,  weight_decay=0.01
#          + experiment's own epochs=20                    -> epochs=20, batch_size=128, lr=0.1, weight_decay=0.01
# optimizer: base gives name="sgd", momentum=0.9
#          + experiment's own name="adam", momentum=DELETE -> name="adam" (momentum removed)
-> {"trainer": {"epochs": 20, "batch_size": 128, "lr": 0.1, "weight_decay": 0.01},
    "optimizer": {"name": "adam"}}
```

### Part 2 — Overrides and interpolation

An *override* is a string `path=value` or `+path=value`, where `path` is one or more identifiers
(`[A-Za-z_]\w*`) joined by `.`, addressing a key inside nested `dict`s — never a list index.
Without the `+` prefix, every segment of `path`, including the last, must already exist as a key of
the config being overridden, or the override raises `OverrideKeyError`; a segment that exists but is
not itself a `dict` also raises `OverrideKeyError`, since there is nothing to descend into further.
With the `+` prefix, a missing segment — including the last — is created as an empty `dict` (the
last one is then set, not left empty); `+` never fails because a key is missing, only because an
existing, non-final segment is not a `dict`.

`value` is exactly one of: `null`, `true`, `false`, an integer (`-?[0-9]+`), a float (a `-?[0-9]+`
with a `.` and at least one following digit, an exponent, or both), a string in matching single or
double quotes with no escape sequences, or a list `[v1, v2, ...]` (or `[]`) of any of the former —
never of another list. Whitespace may appear only immediately inside a list's brackets and around
its commas; nowhere else in an override. Anything else raises `OverrideSyntaxError`.
`apply_overrides(config, overrides)` applies every string in `overrides` to a copy of `config`, in
order — a later override to the same path replaces an earlier one — and returns the copy; `config`
itself is left untouched.

Once bases and overrides are both resolved, a string leaf may contain one or more `${a.b}`
references, each a `path` of the same shape as above, addressing another key of the same config
(never a list index). A leaf whose value is *exactly* one reference, with nothing else in the
string, takes on the referenced value verbatim, whatever type it is. A reference that is only part
of a larger string requires the value it addresses to be an `int`, `float`, `bool`, or `str` — never
a `list`, `dict`, or `None` — and is replaced by its text: `str(x)` for an `int`, `float`, or `str`,
and `"true"`/`"false"` for a `bool`. A referenced value may itself contain references, resolved
first. `resolve_interpolations(config)` returns a copy of `config` with every reference replaced
this way; it raises `InterpolationError` when a reference addresses a path absent from `config`, or
embeds a `list`, `dict`, or `None` inside a larger string, and raises `InterpolationCycleError` when
resolving some path needs, directly or indirectly, its own value — with the cycle spelled out as the
sequence of paths it passes through.

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

Example, continuing Part 1's `experiment`, with two more keys added to it:

```text
store["experiment"]["run_name"] = "exp-${optimizer.name}-bs${trainer.batch_size}"
store["experiment"]["eval_batch_size"] = "${trainer.batch_size}"
resolved = resolve_bases(store, "experiment")   # as in Part 1, plus the two keys above verbatim

overrides = ["trainer.epochs=30", "+trainer.warmup_steps=100", 'optimizer.name="adamw"']
overridden = apply_overrides(resolved, overrides)
# trainer.epochs:      20 -> 30 (already existed)
# trainer.warmup_steps: created at 100 (needed '+': did not exist yet)
# optimizer.name:      "adam" -> "adamw" (a quoted string -- not the bare word adamw)

final = resolve_interpolations(overridden)
# run_name:        "exp-" + optimizer.name ("adamw") + "-bs" + trainer.batch_size (128, stringified)
# eval_batch_size: the whole string is one reference -> trainer.batch_size's own value, the int 128
-> {"trainer": {"epochs": 30, "batch_size": 128, "lr": 0.1, "weight_decay": 0.01, "warmup_steps": 100},
    "optimizer": {"name": "adamw"},
    "run_name": "exp-adamw-bs128",
    "eval_batch_size": 128}
```

### Part 3 — Validation

A *schema* is a class decorated with `@dataclasses.dataclass` whose fields are typed `int`, `float`,
`bool`, `str`, a list of one of these or of another schema class (`list[X]`), or another schema
class directly, nested to any depth. A field is *required* when it has neither a `default` nor a
`default_factory`; a field with either is optional, and its absence from a config is not an error.

`validate_config(config, schema)` checks a fully resolved config (Part 2's output — no `_base_`, no
`${...}` left) against `schema` and returns a `list[FieldError]`, one entry per mismatch, each
carrying the *full dotted path* from the config's root, collecting every mismatch rather than
stopping at the first:

- a required field absent from `config` — `"required field is missing"`;
- a key of `config`, at any level, not declared by the matching schema class —
  `"unexpected field, not declared in the schema"`;
- a present field whose value does not match its declared type: an `int` field needs an `int` that
  is not a `bool` (Python's `bool` is a subclass of `int`, but a `bool` value never satisfies an
  `int` or `float` field); a `float` field accepts an `int` or a `float`, again never a `bool`; a
  `list[X]` field needs a `list`, each of whose elements is checked against `X` at its own indexed
  path (`....3`); a nested-schema field needs a `dict`, checked recursively against that schema.

An empty list means `config` satisfies `schema`. `validate_config` does not raise on a mismatch; it
only raises if `config` itself is not a `dict`.

```py
@dataclasses.dataclass
class FieldError:
    path: str      # full dotted path from the config's root, e.g. "trainer.lr"
    message: str

def validate_config(config: dict, schema: type) -> list[FieldError]:
    """Returns every mismatch between config and schema, collected rather than raised."""
```

Example, against a schema for the config Part 2 produced:

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

validate_config(final, ExperimentConfig)   # final, from Part 2's example -> [] (valid)

broken = {
    "trainer": {"epochs": 30, "batch_size": 128, "lr": "0.1", "weight_decay": 0.01,
                "warmup_steps": 100, "debug": True},
    "optimizer": {"name": "adamw"},
    "run_name": "exp-adamw-bs128",
}   # trainer.lr is a string; trainer has an undeclared "debug" key; eval_batch_size is missing

validate_config(broken, ExperimentConfig) -> [
    FieldError("trainer.lr", "expected float, got str"),
    FieldError("trainer.debug", "unexpected field, not declared in the schema"),
    FieldError("eval_batch_size", "required field is missing"),
]   # all three, from one call -- not just the first one found
```

### Part 4 — Reproducibility

`freeze_config(config)` returns an immutable snapshot of a fully resolved config: every `dict`
becomes a read-only mapping (a `types.MappingProxyType`, recursively, over already-frozen values)
and every `list` becomes a `tuple` of already-frozen elements; scalars are returned as is. Assigning
into any level of the result raises `TypeError`.

`config_hash(config)` returns a stable content hash of a fully resolved config: the same hash for
two configs equal as nested dicts/lists/scalars regardless of the order their keys were inserted in
or which order composition and overrides produced them in, and a different hash whenever some value
differs — including `2` versus `2.0`, and `0.0` versus `-0.0`, which compare equal in Python
(`2 == 2.0`, `-0.0 == 0.0`) but are not the same value. It raises `ValueError` if `config` contains a
`NaN` or an infinite `float`, neither of which this hash can represent.

State, in your solution, what else — beyond `config` and `config_hash(config)` — must be recorded
when a run starts, for that exact run to be reproducible weeks later, possibly on different
hardware.

```py
def freeze_config(config: dict):
    """Returns an immutable snapshot: nested dicts -> MappingProxyType, nested lists -> tuple."""

def config_hash(config: dict) -> str:
    """Returns a stable content hash of config, independent of key order."""
```

Example:

```text
frozen = freeze_config(final)
frozen["trainer"]["epochs"]          # 30 -- reads normally
frozen["trainer"]["epochs"] = 999    # raises TypeError

c1 = {"trainer": {"epochs": 30, "batch_size": 128}, "optimizer": {"name": "adamw"}}
c2 = {"optimizer": {"name": "adamw"}, "trainer": {"batch_size": 128, "epochs": 30}}  # same content, different key order
config_hash(c1) == config_hash(c2)                                    # True

c3 = {**c1, "trainer": {**c1["trainer"], "epochs": 31}}
config_hash(c3) != config_hash(c1)                                    # True
```

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two points are worth confirming before coding: whether an override path may index into a list
(assumed no — overrides and interpolation only ever address nested mappings), and whether an
unrecognized key should fail validation or pass silently (assumed a strict failure, the same choice
`pydantic`'s `extra="forbid"` makes, since a silently-ignored misspelled key is exactly the mistake a
schema exists to catch).

### Part 1

`resolve_bases` is cycle detection over the `_base_` graph, the same shape as over any dependency
graph: `visiting` is the path of names currently being resolved, so a name reappearing on it closes a
cycle, and `memo`, shared across the whole call, resolves a name used as a base by two different
configs — a diamond — exactly once. `deep_merge`'s one subtlety is its branch for a `dict` value in
`overlay`: it always recurses, using `{}` in place of a `base[key]` that is not itself a `dict`
(including one simply absent), rather than adopting `overlay[key]` outright. The two look identical
almost always — except when `overlay[key]` contains a `DELETE` somewhere inside it, where recursing
still finds and drops that key, while a direct copy would leave the sentinel sitting in the result.
`_clone` keeps every value written into a result independent of whatever dict or list it came from,
so resolving one name can never let a later mutation of the result reach back into `store`.

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

A `DELETE` only takes effect at the merge step where it is the overlay: if config `a` sets
`x.y: DELETE` over some `_base_`, `resolve_bases` of `a` alone has no `y` key under `x` at all —
not one holding `DELETE`. A second config listed as a base *after* `a`, reaching the same original
`x.y` through its own separate path, restores it, since `DELETE` never survives into a resolved
config. `_base_` composition plays the same role as Hydra's defaults list or OmegaConf's own merge: a
config is a point in the lattice its bases could produce, not a record of the edits that got it there.

`memo` resolves each name at most once regardless of how many configs use it as a base; resolving
one name still costs time proportional to the size of its own merged result, so a long chain of
single-parent `_base_` links — each config naming only the previous one as its base — costs
quadratic time in the chain's length, since every level re-clones everything merged below it.

### Part 2

Parsing one override is two independent grammars glued at `=`: `_PATH_RE` fullmatches the left side,
so `path_part.split(".")` is safe once it has matched — no segment can be empty — and
`_parse_value`/`_parse_scalar` fullmatch the right side against `null`/`true`/`false`, the float
pattern, the int pattern, a quoted string, then a list, in that order; float has to come before int
since a decimal point must never let a value fall through to `int(...)`, which would raise on it.
Splitting a list's inner text on `,` has to skip commas inside a quoted element: `_split_list_items`
tracks whether it is currently inside a quote character by character, since `str.split(",")` cannot
tell `"a,b"` apart from two separate elements `"a"` and `"b"`. Applying an override never merges:
`_walk_and_set` either creates a segment (only under `+`) or requires it to already exist, and always
overwrites the final segment outright, the same rule `deep_merge` applies to a non-dict overlay value.

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

Interpolation resolves one dotted path at a time with the same recursive shape as `resolve_bases` —
a `visiting` list for the current chain of references and a `memo` for paths already resolved —
except the graph here runs over string values instead of `_base_` lists, discovered lazily as
`${...}` patterns are found rather than declared up front. `_resolve_ref` fetches a path's raw value
from `root`, the original pre-interpolation config, so a reference always resolves against what that
path was actually set to, however many other references point at it. `_resolve_value` is the one
place that decides between a whole-string reference, returned verbatim, and one embedded in a larger
string, kept only as text: checked with `fullmatch` against `sub`, the same distinction `deep_merge`
draws between replacing an entire value and merging into part of one. Checking `isinstance(target,
bool)` before `isinstance(target, (int, float, str))` matters for the usual reason with `bool`: it
would otherwise take the `int`/`float`/`str` branch and print as `"True"`, not the lowercase `"true"`
this same system's own override grammar reads back.

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

Parsing and applying one override costs time linear in its own length. Resolving interpolations costs
time linear in the config's total size plus its `${...}` occurrences, since `memo` resolves any one
path at most once however many other strings reference it.

### Part 3

`validate_config` walks `schema` and `config` together, not `config` alone — only the schema knows
what is required and what a present field's type must be, so a pass over `config` alone could report
a type mismatch but never a missing field. For each declared field it checks presence and, if present,
delegates to `_check_type`, really three cases sharing one shape: a `list[X]` recurses into every
element at its own indexed path, a nested schema recurses into `validate_config` at a nested path, and
a primitive is one `isinstance` check with `bool` excluded from `int` and `float` explicitly, since
`isinstance(True, int)` is `True` in Python even though a boolean is never an acceptable substitute
for a numeric field. Unexpected keys are checked once per level, after every declared field, by
comparing `config`'s own keys against that level's collected field names — so an extra key three
levels deep, inside a `list[X]` of nested schemas, is still reported at its own full path.

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

Every branch appends to a list and keeps going instead of returning on the first problem found — a
missing field does not stop the type check of the fields around it, and a bad element at index 2 of a
`list[X]` does not stop index 5 from being checked too. `_path`, threaded through every recursive call
and defaulted to `""` at the top, is what lets each check report *where* it failed, without asking a
caller of the public two-argument signature to supply it. Cost is linear in the number of fields and
list elements schema and config together reach, since each is visited exactly once.

### Part 4

`freeze_config` mirrors `_clone`'s recursion but produces read-only containers instead of fresh
mutable ones: `MappingProxyType` wraps a `dict` without copying it, so it has to wrap a *freshly
built* dict of already-frozen values, never the original — wrapping the original directly would
leave any later mutation of it visible straight through the supposedly frozen view. `config_hash`
leans entirely on `json.dumps`: `sort_keys=True` makes key order irrelevant, `separators=(",", ":")`
removes whitespace variance, and Python's float formatting is already the shortest decimal string
that round-trips to that exact bit pattern — deterministic per value, so `0.1` and `1e-1` (the same
`float`) format identically, while `0.0` and `-0.0` (different bit patterns that happen to compare
equal) format as `"0.0"` and `"-0.0"` and correctly hash to two different digests. `allow_nan=False`
turns Python's non-standard `NaN`/`Infinity` output into a `ValueError` instead of a hash silently
built over text no other JSON reader could parse back.

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

A hash of the resolved config only proves two runs started from the identical configuration; it says
nothing about everything the configuration does not capture. A manifest saved alongside `config_hash`
at run start also needs: the exact code, as a commit hash of a clean working tree, since a config's
meaning depends on what the training script does with it; a snapshot of the data, addressed by a
content hash or an immutable dataset version rather than a mutable path, since "the file at this
path" can silently change by the next run; every relevant library version, ideally a full dependency
lockfile or container image digest, since numeric results can shift between framework or CUDA
versions under identical code and config; every seed used — Python's own `random`, NumPy, the
training framework, and data-loader shuffling separately, since each draws independently; and the
hardware itself, accelerator type and count, since parallelism changes batch composition and
floating-point reduction order, and some kernels are not bit-identical across accelerator
architectures for the same operation on the same input.

### Follow-ups

- A sweep is the Cartesian product of several override lists, one per axis — `lr` in `{0.01, 0.1}`
  times `batch_size` in `{32, 128}` gives four runs — each composed and hashed independently; the
  sweep's manifest is the list of resulting `config_hash` values, not a hash of the sweep itself.
- Secrets never belong inside a config that gets hashed, frozen, and logged, since the hash and the
  frozen object are exactly what would leak them; a config instead names a secret by reference — an
  environment variable or a secret-manager key — resolved at load time, outside `config_hash`'s input.
- Diffing two runs walks both resolved configs key by key, recursively through matching nested dicts
  and by identity elsewhere, reporting every dotted path whose value was added, removed, or changed;
  `config_hash` alone only answers whether two runs differ, never where.
- Static checking needs the schema to be what configs are built from, not merely checked against
  afterward: constructing `TrainerConfig(epochs=30, lr=0.1)` directly lets a type checker catch a
  wrong type or missing argument before the code runs, at the cost of losing the free-form dict
  literals composition and CLI overrides operate on; `pydantic` sits in between, parsing and coercing
  a dict into a typed, validated object in one step.
- A `.py` config runs arbitrary code at import time, which is exactly what makes it hard to merge,
  diff, override, or hash generically: all four need to inspect and rewrite values on a structure
  that holds still, which a plain nested dict does and an executable module does not.

<details>
<summary>Checks (runnable)</summary>

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
