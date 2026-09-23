# 在线测评：菜谱管理系统

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 在线测评 · 4 关 | ★★★☆☆ | 中等 | SWE · MLE | oop-design, case-folding, sorting, permissions, versioning | 4 关 / 90 分钟 | 在线测评（OA） |
<!-- meta:end -->

## 题目

按顺序实现下面四关：只有当前一关自己的测试全部通过，下一关的测试才会开始运行，且每一关都在同一个
类上继续扩展，而不是另起一个新类。`recipe_id` 是调用方提供的非空字符串，绝不由类自己生成；从
Level 3 起出现的 `user_id` 也是如此。`ingredients` 是原料名称字符串组成的列表，按传入的顺序存储和
返回，类不会对它重新排序或去重。这个类接受或返回的每一份 `ingredients` 列表，都会在边界处被复制
一份：某个方法把它存下来之后再修改传入的列表，或者修改某个方法返回的列表，都绝不能改变类内部已经
存储的内容。

菜谱的 `name` 是传给创建或修改它的那个方法的原始字符串，这是它的*显示名称*（display name），任何
返回菜谱的方法都会原样返回这个字符串。名称的*查找键*（lookup key）是 `name.casefold()`，只用于判断
两个名称是否冲突：两个名称冲突当且仅当它们的查找键相同，所以 `"Congee"` 和 `"CONGEE"` 冲突，但
`"Congee"` 和 `"Congee "`（末尾多一个空格）不冲突。下文中凡是要求名称唯一的地方，唯一性都按查找键
判断，而不是按显示名称本身判断。

### Level 1 —— 菜谱

```py
class RecipeManager:
    def add_recipe(self, recipe_id: str, name: str, ingredients: list[str]) -> bool:
        """Creates a recipe with this id, display name and ingredient list. Returns True. Returns
        False, without effect, if recipe_id already names a recipe, or if name's lookup key is
        already used by any recipe (regardless of recipe_id)."""

    def get_recipe(self, recipe_id: str) -> dict | None:
        """Returns {"id": recipe_id, "name": ..., "ingredients": [...]} for this recipe's current
        state, or None if recipe_id names no recipe."""

    def update_recipe(self, recipe_id: str, name: str, ingredients: list[str]) -> bool:
        """Replaces recipe_id's name and ingredients. Returns True. Returns False, without effect,
        if recipe_id names no recipe, or if name's lookup key is used by a different recipe --
        keeping (or only re-casing) a recipe's own current name is not a collision."""

    def delete_recipe(self, recipe_id: str) -> bool:
        """Removes recipe_id, freeing both its id and its name's lookup key for reuse -- a later
        add_recipe with either creates an unrelated new recipe. Returns True if recipe_id named a
        recipe, False if it did not."""
```

示例：

```text
mgr = RecipeManager()
mgr.add_recipe("r1", "Miso Soup", ["miso paste", "tofu", "scallion"])   # -> True
mgr.add_recipe("r2", "MISO SOUP", ["dashi", "kombu"])                   # -> False -- "miso soup" 已被占用（属于 r1）
mgr.add_recipe("r1", "Ramen", ["noodles", "broth"])                     # -> False -- "r1" 已经存在
mgr.get_recipe("r1")
# -> {"id": "r1", "name": "Miso Soup", "ingredients": ["miso paste", "tofu", "scallion"]}
mgr.get_recipe("r9")                                                    # -> None
mgr.update_recipe("r1", "miso soup", ["miso paste", "tofu"])            # -> True -- 换个大小写用自己的名字不算冲突
mgr.get_recipe("r1")["name"]                                            # -> "miso soup"  -- 原样存储
mgr.add_recipe("r2", "Chicken Ramen", ["noodles", "broth", "chicken"])  # -> True -- "r2" 和这个名字都还没被占用
mgr.update_recipe("r2", "Miso Soup", ["dashi"])                         # -> False -- 与 r1 冲突（大小写不敏感）
mgr.delete_recipe("r2")                                                 # -> True
mgr.delete_recipe("r2")                                                 # -> False -- 已经不存在了
mgr.get_recipe("r2")                                                    # -> None
```

### Level 2 —— 列表与搜索

```py
class RecipeManager:
    def list_recipes(self) -> list[dict]:
        """Every recipe, as get_recipe's dict, sorted by number of ingredients ascending. Recipes
        tied on ingredient count keep the order in which they were first created by add_recipe,
        earliest first; deleting a recipe and later reusing its id counts as a new creation, at
        the end, while update_recipe never changes a recipe's place in this order. [] if empty."""

    def find_by_ingredient(self, ingredient: str) -> list[dict]:
        """Every recipe with an ingredient equal to ingredient, case-insensitive (an exact match
        against one entry of its ingredient list, not a substring match), in the same order as
        list_recipes."""

    def search_by_name_prefix(self, prefix: str) -> list[dict]:
        """Every recipe whose name starts with prefix, case-insensitive; an empty prefix matches
        every recipe. Same order as list_recipes."""
```

示例：

```text
mgr = RecipeManager()
mgr.add_recipe("r1", "Ramen", ["noodles", "broth", "egg"])          # 3 种原料，第 1 个创建
mgr.add_recipe("r2", "Fried Rice", ["rice", "egg", "soy sauce"])    # 3 种原料，第 2 个创建
mgr.add_recipe("r3", "Toast", ["bread"])                            # 1 种原料，第 3 个创建
mgr.add_recipe("r4", "Congee", ["rice", "water"])                   # 2 种原料，第 4 个创建

[r["id"] for r in mgr.list_recipes()]
# -> ["r3", "r4", "r1", "r2"]     -- 按数量 1、2、3、3 排列；r1 比 r2 先创建，两者都比对方先出现

[r["id"] for r in mgr.find_by_ingredient("EGG")]
# -> ["r1", "r2"]                 -- 大小写不敏感匹配；排序规则相同

[r["id"] for r in mgr.search_by_name_prefix("CO")]
# -> ["r4"]                       -- 大小写不敏感的前缀匹配，命中 "Congee"

[r["id"] for r in mgr.search_by_name_prefix("")]
# -> ["r3", "r4", "r1", "r2"]     -- 空前缀匹配所有菜谱
```

### Level 3 —— 用户与权限

从这一关开始，`add_recipe` 和 `delete_recipe` 改为下面这两种形式，替换掉它们在 Level 1 的形式；评测
不会再调用三个参数的 `add_recipe`，也不会再调用一个参数的 `delete_recipe`。`update_recipe` 同样不会
再被调用：从现在起，菜谱的名称和原料改由下面的 `edit_recipe` 来修改，用户出现之后 `update_recipe`
该做什么，这个问题不再规定。读取菜谱——`get_recipe` 以及 Level 2 的所有方法——对所有人保持开放，不
受这一关任何规则的影响。

```py
class RecipeManager:
    def add_user(self, user_id: str, username: str) -> bool:
        """Registers user_id. Returns True, or False, without effect, if user_id is already
        registered. username is stored nowhere else and checked for nothing."""

    def add_recipe(self, recipe_id: str, name: str, ingredients: list[str], owner_id: str) -> bool:
        """As Level 1, plus: the new recipe's owner is owner_id. Returns False, without effect, if
        owner_id is not a registered user (checked in addition to Level 1's two conditions)."""

    def grant_editor(self, owner_id: str, recipe_id: str, user_id: str) -> bool:
        """Grants user_id permission to edit recipe_id. Returns True. Returns False, without
        effect, if recipe_id names no recipe, owner_id is not its current owner, or user_id is not
        a registered user. Granting a user who already has the permission returns True and changes
        nothing further."""

    def edit_recipe(self, user_id: str, recipe_id: str, name: str, ingredients: list[str]) -> bool:
        """Changes recipe_id's name and ingredients, exactly as update_recipe did. Returns True if
        user_id is recipe_id's owner or a user granted edit permission on it. Returns False,
        without effect, if recipe_id names no recipe, user_id is neither its owner nor a granted
        editor, or the new name's lookup key is used by a different recipe."""

    def delete_recipe(self, user_id: str, recipe_id: str) -> bool:
        """Removes recipe_id, as Level 1 did. Returns True only if user_id is recipe_id's owner --
        a granted editor may edit a recipe but not delete it. Returns False, without effect, for
        every other case, including an unknown recipe_id or user_id."""
```

和这个问题里的每一种失败情形一样，对 `edit_recipe`、`delete_recipe` 或 `grant_editor` 的未授权调用
不会有任何效果，返回 `False`；三者都不会抛出异常。

示例：

```text
mgr = RecipeManager()
mgr.add_user("u1", "Alice")                                              # -> True
mgr.add_user("u2", "Bob")                                                # -> True
mgr.add_user("u1", "Alice2")                                             # -> False -- "u1" 已被占用
mgr.add_recipe("r1", "Congee", ["rice", "water"], "u1")                  # -> True -- u1 拥有 r1
mgr.add_recipe("r2", "Toast", ["bread"], "nope")                         # -> False -- "nope" 不是用户；r2 未被创建
mgr.edit_recipe("u2", "r1", "Rice Congee", ["rice", "water", "ginger"])  # -> False -- u2 既不是拥有者也不是编辑者
mgr.grant_editor("u2", "r1", "u2")                                       # -> False -- u2 不是 r1 的拥有者，无权授予
mgr.grant_editor("u1", "r1", "u2")                                       # -> True  -- 拥有者 u1 把编辑权限授予 u2
mgr.edit_recipe("u2", "r1", "Rice Congee", ["rice", "water", "ginger"])  # -> True  -- u2 现在是被授权的编辑者
mgr.delete_recipe("u2", "r1")                                            # -> False -- 编辑者可以编辑，但不能删除
mgr.delete_recipe("u1", "r1")                                            # -> True  -- 拥有者可以删除
```

### Level 4 —— 版本历史与回滚

从这一关开始，每次返回 `True` 的 `edit_recipe` 调用都不再原地覆盖 recipe_id 的名称和原料：它会把新
状态*追加*（append）为一个新版本，已经记录下来的内容永远不会被改动或删除。`add_recipe` 本身就已经
创建了菜谱的第一个版本。同一个菜谱的版本号从 1 开始、按创建顺序编号，版本号不会被重用或重新分配，
下面的 `rollback_recipe` 也不例外。`get_recipe` 没有变化：它仍然返回 recipe_id 当前（最新）版本的
状态，不附带版本号。

```py
class RecipeManager:
    def get_recipe_history(self, recipe_id: str) -> list[dict] | None:
        """Returns [{"version": 1, "name": ..., "ingredients": [...]}, ...], oldest first, one
        entry per version recipe_id has ever had. None if recipe_id names no recipe."""

    def get_version(self, recipe_id: str, version: int) -> dict | None:
        """Returns the single {"version": ..., "name": ..., "ingredients": [...]} entry for this
        version of recipe_id. None if recipe_id names no recipe, or version is less than 1 or
        greater than the number of versions recipe_id currently has."""

    def rollback_recipe(self, user_id: str, recipe_id: str, version: int) -> bool:
        """Appends a copy of get_version(recipe_id, version) as recipe_id's newest version --
        version may be its current latest, which appends an identical copy. Authorization is
        exactly edit_recipe's: user_id must be recipe_id's owner or a granted editor. Returns
        False, without effect, if that authorization fails, version is out of range, or the name
        recorded at that version is currently used by a different recipe -- a recipe's own
        earlier versions never conflict with its own current name. rollback_recipe never changes
        who owns recipe_id or who is a granted editor on it: only edit_recipe's authorization rule
        and its name-collision rule apply to it, and nothing about permissions themselves changes,
        no matter which version is restored."""
```

示例：

```text
mgr = RecipeManager()
mgr.add_user("u1", "Alice")
mgr.add_recipe("r1", "Congee", ["rice", "water"], "u1")                     # v1 = "Congee"
mgr.edit_recipe("u1", "r1", "Rice Porridge", ["rice", "water", "salt"])     # v2 = "Rice Porridge" -- "Congee" 现在空出来了
mgr.edit_recipe("u1", "r1", "Jook", ["rice", "water", "ginger"])            # v3 = "Jook"
mgr.get_recipe_history("r1")
# -> [{"version": 1, "name": "Congee", "ingredients": ["rice", "water"]},
#     {"version": 2, "name": "Rice Porridge", "ingredients": ["rice", "water", "salt"]},
#     {"version": 3, "name": "Jook", "ingredients": ["rice", "water", "ginger"]}]
mgr.get_version("r1", 1)   # -> {"version": 1, "name": "Congee", "ingredients": ["rice", "water"]}
mgr.get_version("r1", 5)   # -> None -- 目前只有 3 个版本

mgr.add_recipe("r2", "Congee", ["rice"], "u1")                              # -> True -- "Congee" 又空出来了
mgr.rollback_recipe("u1", "r1", 1)                                          # -> False -- v1 的名字现在属于 r2
mgr.grant_editor("u1", "r1", "u2")                                          # -> False -- "u2" 还不是注册用户

mgr.add_user("u2", "Bob")
mgr.grant_editor("u1", "r1", "u2")                                          # -> True
mgr.rollback_recipe("u2", "r1", 2)                                          # -> True -- v4 = v2 的副本 = "Rice Porridge"
mgr.get_recipe("r1")
# -> {"id": "r1", "name": "Rice Porridge", "ingredients": ["rice", "water", "salt"]}
len(mgr.get_recipe_history("r1"))                                           # -> 4
mgr.edit_recipe("u2", "r1", "Jook", ["rice", "water", "ginger"])            # -> True -- 回滚并没有撤销 u2 的授权
mgr.delete_recipe("u2", "r1")                                               # -> False -- u2 依然只是编辑者，从未成为拥有者
mgr.delete_recipe("u1", "r1")                                               # -> True
```

## 参考解答

<details>
<summary>展开参考解答</summary>

有两点值得事先确认，因为题目本身没有把它们钉死：被授权的编辑者是否也可以删除菜谱（这里选择不行——
只有拥有者可以删除，或者再授予别人编辑权限，这也是为什么 `grant_editor` 和 `delete_recipe` 专门检查
的是拥有权，而不只是编辑权限）；以及 `rollback_recipe` 需要的授权是不是和 `edit_recipe` 一样，还是要
更严格（这里选择一样，因为回滚说到底也只是把菜谱当前的名称和原料改掉，和一次普通的编辑没有区别）。

### Level 1

`_Recipe` 从一开始就带上了以后每一关都会用到的字段——`owner_id`、`editors` 和 `versions` 从创建时就
保留着，不需要额外成本，所以 Level 3 和 Level 4 只需要开始读取和追加内容，不需要再补充字段。
`_name_index` 把查找键映射到当前持有它的那个 `recipe_id`，这样每次冲突检查都只是一次字典查找，而
不用扫描全部菜谱；`_name_taken` 被每一个会改动名称的方法共用，从这里的 `add_recipe` 一直到 Level 4
的 `rollback_recipe`，`exclude_id` 在一个地方统一实现了“不是被另一个菜谱占用”这一半规则。`_rename`
让 `_name_index` 和菜谱自己的 `.name` 保持同步，也是这样被反复复用。

```python
def _key(name: str) -> str:
    return name.casefold()


class _Recipe:
    def __init__(self, recipe_id: str, name: str, ingredients: list[str]):
        self.id = recipe_id
        self.name = name
        self.ingredients = list(ingredients)             # NOTE: a fresh copy -- never alias the caller's list
        self.owner_id: str | None = None                  # set by Level 3's add_recipe
        self.editors: set[str] = set()                     # grown by Level 3's grant_editor (membership only)
        self.versions: list[tuple[str, list[str]]] = [(name, list(ingredients))]   # read from Level 4 on

    def public(self) -> dict:
        return {"id": self.id, "name": self.name, "ingredients": list(self.ingredients)}   # NOTE: copy out too


class RecipeManager:
    def __init__(self) -> None:
        self._recipes: dict[str, _Recipe] = {}      # recipe_id -> _Recipe, in creation order
        self._name_index: dict[str, str] = {}         # lookup key -> the recipe_id currently holding it

    def _name_taken(self, name: str, exclude_id: str | None = None) -> bool:
        holder = self._name_index.get(_key(name))
        return holder is not None and holder != exclude_id

    def _rename(self, recipe: _Recipe, name: str) -> None:
        del self._name_index[_key(recipe.name)]
        recipe.name = name
        self._name_index[_key(name)] = recipe.id

    def add_recipe(self, recipe_id: str, name: str, ingredients: list[str]) -> bool:
        if recipe_id in self._recipes or self._name_taken(name):
            return False
        self._recipes[recipe_id] = _Recipe(recipe_id, name, ingredients)
        self._name_index[_key(name)] = recipe_id
        return True

    def get_recipe(self, recipe_id: str) -> dict | None:
        recipe = self._recipes.get(recipe_id)
        return recipe.public() if recipe is not None else None

    def update_recipe(self, recipe_id: str, name: str, ingredients: list[str]) -> bool:
        recipe = self._recipes.get(recipe_id)
        if recipe is None or self._name_taken(name, exclude_id=recipe_id):
            return False
        self._rename(recipe, name)
        recipe.ingredients = list(ingredients)           # NOTE: copy again -- do not alias the caller's list
        return True

    def delete_recipe(self, recipe_id: str) -> bool:
        recipe = self._recipes.pop(recipe_id, None)
        if recipe is None:
            return False
        del self._name_index[_key(recipe.name)]          # NOTE: frees the lookup key for reuse
        return True
```

每个方法都是 $O(1)$。重放这一关的示例：

```python
mgr = RecipeManager()
assert mgr.add_recipe("r1", "Miso Soup", ["miso paste", "tofu", "scallion"]) is True
assert mgr.add_recipe("r2", "MISO SOUP", ["dashi", "kombu"]) is False
assert mgr.add_recipe("r1", "Ramen", ["noodles", "broth"]) is False
assert mgr.get_recipe("r1") == {"id": "r1", "name": "Miso Soup", "ingredients": ["miso paste", "tofu", "scallion"]}
assert mgr.get_recipe("r9") is None
assert mgr.update_recipe("r1", "miso soup", ["miso paste", "tofu"]) is True
assert mgr.get_recipe("r1")["name"] == "miso soup"
assert mgr.add_recipe("r2", "Chicken Ramen", ["noodles", "broth", "chicken"]) is True
assert mgr.update_recipe("r2", "Miso Soup", ["dashi"]) is False
assert mgr.delete_recipe("r2") is True
assert mgr.delete_recipe("r2") is False
assert mgr.get_recipe("r2") is None
print("Level 1 example replayed")
```

### Level 2

`list_recipes`、`find_by_ingredient` 和 `search_by_name_prefix` 共用一个 `_sorted` 辅助方法。它只按
原料数量排序，没有第二个键来打破平局，而这样就够用，原因是 `self._recipes.values()` 本来就已经按
创建顺序产出菜谱——`dict` 会保持键第一次插入时的顺序，先 `del` 再重新赋值 `[...] =` 会把重用的 id
移到末尾，而原地修改 `recipe.name`/`.ingredients` 从不触碰这个顺序——并且因为 `sorted` 是*稳定*
排序，会把比较结果相等的菜谱（原料数量相同）留在它们本来到达的相对顺序上。如果先把候选列表建立在某
个不保序的结构上，比如一个匹配结果的 `set`，那么平局的顺序就会随字符串哈希种子而变。

```python
class RecipeManager(RecipeManager):
    def list_recipes(self) -> list[dict]:
        return self._sorted(self._recipes.values())

    def find_by_ingredient(self, ingredient: str) -> list[dict]:
        key = _key(ingredient)
        return self._sorted(r for r in self._recipes.values() if any(_key(i) == key for i in r.ingredients))

    def search_by_name_prefix(self, prefix: str) -> list[dict]:
        key = _key(prefix)
        return self._sorted(r for r in self._recipes.values() if _key(r.name).startswith(key))

    @staticmethod
    def _sorted(recipes) -> list[dict]:
        return [r.public() for r in sorted(recipes, key=lambda r: len(r.ingredients))]   # NOTE: stable -- see above
```

`list_recipes` 的开销是 $O(n \log n)$，$n$ 是菜谱总数。`find_by_ingredient` 和 `search_by_name_prefix`
的开销是 $O(n \cdot m + k \log k)$，$m$ 是原料列表的平均长度，$k \le n$ 是匹配到的数量：每份菜谱的
原料或名称都要先看一遍，才能对匹配结果排序，这和没有索引时的全表扫描是一样的代价。重放这一关的示例：

```python
mgr = RecipeManager()
mgr.add_recipe("r1", "Ramen", ["noodles", "broth", "egg"])
mgr.add_recipe("r2", "Fried Rice", ["rice", "egg", "soy sauce"])
mgr.add_recipe("r3", "Toast", ["bread"])
mgr.add_recipe("r4", "Congee", ["rice", "water"])
assert [r["id"] for r in mgr.list_recipes()] == ["r3", "r4", "r1", "r2"]
assert [r["id"] for r in mgr.find_by_ingredient("EGG")] == ["r1", "r2"]
assert [r["id"] for r in mgr.search_by_name_prefix("CO")] == ["r4"]
assert [r["id"] for r in mgr.search_by_name_prefix("")] == ["r3", "r4", "r1", "r2"]
print("Level 2 example replayed")
```

### Level 3

`add_recipe` 通过 `super()` 调用 Level 1 自己的 `add_recipe` 来完成 id/名称检查以及对象本身的创建，
之后才把 `owner_id` 补上去，这样冲突规则就只用写在一个地方。`_authorized` 是一个被 `edit_recipe`、
`delete_recipe` 以及从 Level 4 起的 `rollback_recipe` 共用的函数；`allow_editor` 是唯一把
`delete_recipe` 和另外两个方法区分开的地方——拥有者总能通过，被授权的编辑者只有在 `allow_editor`
为真时才能通过。它自己并不检查 `self._users`：`add_recipe` 在存下 `owner_id` 之前已经验证过它，
下面的 `grant_editor` 在把 `user_id` 加进 `editors` 之前也已经验证过它，所以一个未注册的 id 不可能
等于已存下的 `owner_id`，也不可能已经出现在某个 `editors` 集合里——在这一点上 `_authorized` 已经
没有什么可查的了。`grant_editor` 直接检查拥有权，而不是通过 `_authorized`，因为 `allow_editor=True`
会让编辑者也能授予别人编辑权限，而这不是这里的规则。

```python
class RecipeManager(RecipeManager):
    def __init__(self) -> None:
        super().__init__()
        self._users: set[str] = set()   # membership only, like _Recipe.editors -- never iterated for output

    def add_user(self, user_id: str, username: str) -> bool:
        if user_id in self._users:
            return False
        self._users.add(user_id)
        return True

    def add_recipe(self, recipe_id: str, name: str, ingredients: list[str], owner_id: str) -> bool:
        if owner_id not in self._users:
            return False
        if not super().add_recipe(recipe_id, name, ingredients):
            return False
        self._recipes[recipe_id].owner_id = owner_id
        return True

    def _authorized(self, user_id: str, recipe: _Recipe | None, allow_editor: bool) -> bool:
        if recipe is None:
            return False
        return user_id == recipe.owner_id or (allow_editor and user_id in recipe.editors)

    def grant_editor(self, owner_id: str, recipe_id: str, user_id: str) -> bool:
        recipe = self._recipes.get(recipe_id)
        if recipe is None or owner_id != recipe.owner_id or user_id not in self._users:
            return False
        recipe.editors.add(user_id)
        return True

    def edit_recipe(self, user_id: str, recipe_id: str, name: str, ingredients: list[str]) -> bool:
        recipe = self._recipes.get(recipe_id)
        if not self._authorized(user_id, recipe, allow_editor=True) or self._name_taken(name, exclude_id=recipe_id):
            return False
        self._rename(recipe, name)
        recipe.ingredients = list(ingredients)          # NOTE: copy again -- do not alias the caller's list
        return True

    def delete_recipe(self, user_id: str, recipe_id: str) -> bool:
        recipe = self._recipes.get(recipe_id)
        if not self._authorized(user_id, recipe, allow_editor=False):
            return False
        return super().delete_recipe(recipe_id)         # NOTE: reuses Level 1's removal, not reimplemented
```

Level 3 的每个方法都是 $O(1)$。重放这一关的示例：

```python
mgr = RecipeManager()
assert mgr.add_user("u1", "Alice") is True
assert mgr.add_user("u2", "Bob") is True
assert mgr.add_user("u1", "Alice2") is False
assert mgr.add_recipe("r1", "Congee", ["rice", "water"], "u1") is True
assert mgr.add_recipe("r2", "Toast", ["bread"], "nope") is False
assert mgr.edit_recipe("u2", "r1", "Rice Congee", ["rice", "water", "ginger"]) is False
assert mgr.grant_editor("u2", "r1", "u2") is False
assert mgr.grant_editor("u1", "r1", "u2") is True
assert mgr.edit_recipe("u2", "r1", "Rice Congee", ["rice", "water", "ginger"]) is True
assert mgr.delete_recipe("u2", "r1") is False
assert mgr.delete_recipe("u1", "r1") is True
print("Level 3 example replayed")
```

### Level 4

`edit_recipe` 又被重写了一次，但只是为了加上追加这一步：`super().edit_recipe(...)` 已经完成了
Level 3 的授权检查、冲突检查、改名和原料赋值，所以如果它返回 `False`，就说明什么都没发生，这一层的
重写也照样返回 `False`，不去碰 `versions`；如果它返回 `True`，`recipe.name`/`recipe.ingredients`
已经是新状态了，剩下唯一要做的就是把它们的副本追加进去。`rollback_recipe` 读出目标版本，直接交给
`edit_recipe`：授权、冲突检查（“旧名字现在被另一个菜谱占用了”）和追加操作都直接是 `edit_recipe` 自己
的，不需要重新实现一遍——回滚说到底就是一次编辑，只不过新的名称和原料来自历史记录，而不是来自调用方。

```python
class RecipeManager(RecipeManager):
    def edit_recipe(self, user_id: str, recipe_id: str, name: str, ingredients: list[str]) -> bool:
        if not super().edit_recipe(user_id, recipe_id, name, ingredients):
            return False
        recipe = self._recipes[recipe_id]
        recipe.versions.append((recipe.name, list(recipe.ingredients)))   # NOTE: its own copy, aliasing nothing
        return True

    def get_recipe_history(self, recipe_id: str) -> list[dict] | None:
        recipe = self._recipes.get(recipe_id)
        if recipe is None:
            return None
        return [{"version": i, "name": n, "ingredients": list(ing)}
                for i, (n, ing) in enumerate(recipe.versions, start=1)]

    def get_version(self, recipe_id: str, version: int) -> dict | None:
        recipe = self._recipes.get(recipe_id)
        if recipe is None or not 1 <= version <= len(recipe.versions):
            return None
        name, ingredients = recipe.versions[version - 1]     # NOTE: O(1) -- history is 1-based, never shrinks
        return {"version": version, "name": name, "ingredients": list(ingredients)}

    def rollback_recipe(self, user_id: str, recipe_id: str, version: int) -> bool:
        recipe = self._recipes.get(recipe_id)
        if recipe is None or not 1 <= version <= len(recipe.versions):
            return False
        name, ingredients = recipe.versions[version - 1]
        return self.edit_recipe(user_id, recipe_id, name, ingredients)   # NOTE: same auth, same collision rule
```

`get_version` 除了复制那一个版本的原料列表之外都是 $O(1)$；`get_recipe_history` 的开销是 $O(v)$，
$v$ 是版本数量，因为它要把每一个版本都复制一遍。`edit_recipe` 和 `rollback_recipe` 仍然是 $O(1)$。
重放这一关的示例：

```python
mgr = RecipeManager()
mgr.add_user("u1", "Alice")
mgr.add_recipe("r1", "Congee", ["rice", "water"], "u1")
mgr.edit_recipe("u1", "r1", "Rice Porridge", ["rice", "water", "salt"])
mgr.edit_recipe("u1", "r1", "Jook", ["rice", "water", "ginger"])
assert mgr.get_recipe_history("r1") == [
    {"version": 1, "name": "Congee", "ingredients": ["rice", "water"]},
    {"version": 2, "name": "Rice Porridge", "ingredients": ["rice", "water", "salt"]},
    {"version": 3, "name": "Jook", "ingredients": ["rice", "water", "ginger"]},
]
assert mgr.get_version("r1", 1) == {"version": 1, "name": "Congee", "ingredients": ["rice", "water"]}
assert mgr.get_version("r1", 5) is None

assert mgr.add_recipe("r2", "Congee", ["rice"], "u1") is True
assert mgr.rollback_recipe("u1", "r1", 1) is False
assert mgr.grant_editor("u1", "r1", "u2") is False

mgr.add_user("u2", "Bob")
assert mgr.grant_editor("u1", "r1", "u2") is True
assert mgr.rollback_recipe("u2", "r1", 2) is True
assert mgr.get_recipe("r1") == {"id": "r1", "name": "Rice Porridge", "ingredients": ["rice", "water", "salt"]}
assert len(mgr.get_recipe_history("r1")) == 4
assert mgr.edit_recipe("u2", "r1", "Jook", ["rice", "water", "ginger"]) is True
assert mgr.delete_recipe("u2", "r1") is False
assert mgr.delete_recipe("u1", "r1") is True
print("Level 4 example replayed")
```

### 追问

- 撤销编辑权限：这里没有要求实现；一个对称的 `revoke_editor(owner_id, recipe_id, user_id)` 只需要
  $O(1)$ 就能从 `editors` 里去掉一个 id，完全不需要碰 `versions`，因为权限是记在菜谱上的，不属于
  任何一个版本。
- 并发调用：这里没有任何加锁，所以两个线程同时对同一个 `recipe_id` 调用 `edit_recipe`，有可能在
  任何一方真正改名之前，都通过了冲突检查。给每个 `recipe_id` 加一把锁，把“授权-检查-修改”这一整套
  动作都框在锁里（`edit_recipe` 和 `rollback_recipe` 本来就已经把这套动作归并成了一个方法），就能
  恰好把真正会触碰共享状态的调用串行化，同时不会阻塞对其他菜谱的编辑。
- 被编辑过几千次的菜谱：`get_recipe_history` 每次调用都会把每一个版本复制一遍，所以它的开销随菜谱
  整个编辑历史增长，而不是随调用方实际想要的内容增长。改成支持分页，或者接受一个版本区间的形式，
  就能避免复制调用方根本没要的那些版本。
- 转让拥有权：这个问题里，菜谱的拥有者一旦在 `add_recipe` 时确定，就再也不会改变。要加上转让功能，
  就得决定它算不算自己的一个版本（这里的答案是不算，因为它既不改名称也不改原料），以及应该由当前
  拥有者、某个编辑者，还是只能由外部管理员来发起。
- 非 ASCII 名称：查找键用的是 `casefold()` 而不是 `lower()`，因为 `casefold` 是专门为跨文字系统的
  大小写不敏感匹配设计的——`"STRASSE".casefold() == "straße".casefold()` 结果是 `True`，换成
  `lower()` 做同样的比较则是 `False`。

<details>
<summary>验证代码（可运行）</summary>

```python
# ---- edge cases the level examples do not reach ----

# mutating a list after passing it in, or a dict/list a method returned, must never reach the store
mgr = RecipeManager()
mgr.add_user("u1", "Alice")
ingredients_in = ["rice", "water"]
mgr.add_recipe("r1", "Congee", ingredients_in, "u1")
ingredients_in.append("salt")
assert mgr.get_recipe("r1")["ingredients"] == ["rice", "water"]

fetched = mgr.get_recipe("r1")
fetched["ingredients"].append("intruder")
assert mgr.get_recipe("r1")["ingredients"] == ["rice", "water"]

mgr.edit_recipe("u1", "r1", "Rice Porridge", ["rice", "water", "salt"])
old_version = mgr.get_version("r1", 1)
old_version["ingredients"].append("intruder")
assert mgr.get_version("r1", 1)["ingredients"] == ["rice", "water"]          # history unaffected

# an empty store
empty = RecipeManager()
assert empty.list_recipes() == [] and empty.find_by_ingredient("rice") == [] and empty.search_by_name_prefix("") == []
assert empty.get_recipe("r1") is None and empty.get_recipe_history("r1") is None and empty.get_version("r1", 1) is None

# unknown recipe_id, unknown user_id, and version 0 are all in-range failures, not crashes
assert mgr.edit_recipe("u1", "ghost", "X", []) is False
assert mgr.delete_recipe("u1", "ghost") is False
assert mgr.grant_editor("u1", "ghost", "u1") is False
assert mgr.rollback_recipe("u1", "ghost", 1) is False
assert mgr.get_version("r1", 0) is None

# editing a recipe never moves it in the tie-break order, even though it is the most recently touched
mgr2 = RecipeManager()
mgr2.add_user("u1", "Alice")
mgr2.add_recipe("r1", "A-recipe", ["x", "y"], "u1")     # 2 ingredients, created 1st
mgr2.add_recipe("r2", "B-recipe", ["x"], "u1")          # 1 ingredient,  created 2nd
assert [r["id"] for r in mgr2.list_recipes()] == ["r2", "r1"]
mgr2.edit_recipe("u1", "r2", "B-recipe", ["x", "y"])    # now tied with r1 on count, but r1 was created first
assert [r["id"] for r in mgr2.list_recipes()] == ["r1", "r2"]

# rollback to the current latest version appends an identical copy rather than failing
mgr3 = RecipeManager()
mgr3.add_user("u1", "Alice")
mgr3.add_recipe("r1", "Soup", ["water"], "u1")
assert mgr3.rollback_recipe("u1", "r1", 1) is True
assert mgr3.get_recipe_history("r1") == [
    {"version": 1, "name": "Soup", "ingredients": ["water"]},
    {"version": 2, "name": "Soup", "ingredients": ["water"]},
]

# deleting a recipe and reusing its id starts an unrelated recipe: history and permissions do not carry over
mgr4 = RecipeManager()
mgr4.add_user("u1", "Alice")
mgr4.add_user("u2", "Bob")
mgr4.add_recipe("r1", "Old", ["x"], "u1")
mgr4.edit_recipe("u1", "r1", "Old Two", ["x", "y"])
mgr4.grant_editor("u1", "r1", "u2")
assert mgr4.delete_recipe("u1", "r1") is True
assert mgr4.add_recipe("r1", "New", ["z"], "u2") is True
assert mgr4.get_recipe_history("r1") == [{"version": 1, "name": "New", "ingredients": ["z"]}]
assert mgr4.edit_recipe("u1", "r1", "New Two", ["z"]) is False   # u1 owned the old r1, not this one
assert mgr4.edit_recipe("u2", "r1", "New Two", ["z"]) is True    # u2 is this recipe's actual owner
print("edge cases OK")
```

```python
# ---- independent reference model, straight from the statement ----
import random
from collections import Counter


class _NaiveRecipeManager:
    """Recipes and users are flat containers; every uniqueness or permission check re-scans them from
    scratch, and rollback is written out on its own rather than calling edit_recipe."""

    def __init__(self) -> None:
        self.recipes: dict[str, dict] = {}   # recipe_id -> {name, ingredients, owner, editors, versions}
        self.users: set[str] = set()

    def _name_used(self, name: str, exclude_id: str | None = None) -> bool:
        key = name.casefold()
        return any(rid != exclude_id and r["name"].casefold() == key for rid, r in self.recipes.items())

    def add_user(self, user_id, username):
        if user_id in self.users:
            return False
        self.users.add(user_id)
        return True

    def add_recipe(self, recipe_id, name, ingredients, owner_id):
        if recipe_id in self.recipes or self._name_used(name) or owner_id not in self.users:
            return False
        self.recipes[recipe_id] = {"name": name, "ingredients": list(ingredients), "owner": owner_id,
                                    "editors": set(), "versions": [(name, list(ingredients))]}
        return True

    def get_recipe(self, recipe_id):
        r = self.recipes.get(recipe_id)
        return None if r is None else {"id": recipe_id, "name": r["name"], "ingredients": list(r["ingredients"])}

    def _can_edit(self, user_id, r):
        return r is not None and (user_id == r["owner"] or user_id in r["editors"])

    def grant_editor(self, owner_id, recipe_id, user_id):
        r = self.recipes.get(recipe_id)
        if r is None or owner_id != r["owner"] or user_id not in self.users:
            return False
        r["editors"].add(user_id)
        return True

    def edit_recipe(self, user_id, recipe_id, name, ingredients):
        r = self.recipes.get(recipe_id)
        if not self._can_edit(user_id, r) or self._name_used(name, exclude_id=recipe_id):
            return False
        r["name"], r["ingredients"] = name, list(ingredients)
        r["versions"].append((name, list(ingredients)))
        return True

    def delete_recipe(self, user_id, recipe_id):
        r = self.recipes.get(recipe_id)
        if r is None or user_id != r["owner"]:
            return False
        del self.recipes[recipe_id]
        return True

    def _ordered(self, pairs):
        return sorted(pairs, key=lambda kv: len(kv[1]["ingredients"]))

    def list_recipes(self):
        return [{"id": rid, "name": r["name"], "ingredients": list(r["ingredients"])}
                for rid, r in self._ordered(self.recipes.items())]

    def find_by_ingredient(self, ingredient):
        key = ingredient.casefold()
        matches = [(rid, r) for rid, r in self.recipes.items() if any(i.casefold() == key for i in r["ingredients"])]
        return [{"id": rid, "name": r["name"], "ingredients": list(r["ingredients"])}
                for rid, r in self._ordered(matches)]

    def search_by_name_prefix(self, prefix):
        key = prefix.casefold()
        matches = [(rid, r) for rid, r in self.recipes.items() if r["name"].casefold().startswith(key)]
        return [{"id": rid, "name": r["name"], "ingredients": list(r["ingredients"])}
                for rid, r in self._ordered(matches)]

    def get_recipe_history(self, recipe_id):
        r = self.recipes.get(recipe_id)
        if r is None:
            return None
        return [{"version": i, "name": n, "ingredients": list(ing)} for i, (n, ing) in enumerate(r["versions"], 1)]

    def get_version(self, recipe_id, version):
        r = self.recipes.get(recipe_id)
        if r is None or not 1 <= version <= len(r["versions"]):
            return None
        n, ing = r["versions"][version - 1]
        return {"version": version, "name": n, "ingredients": list(ing)}

    def rollback_recipe(self, user_id, recipe_id, version):
        r = self.recipes.get(recipe_id)
        if not self._can_edit(user_id, r) or not 1 <= version <= len(r["versions"]):
            return False
        n, ing = r["versions"][version - 1]
        if self._name_used(n, exclude_id=recipe_id):
            return False
        r["name"], r["ingredients"] = n, list(ing)
        r["versions"].append((n, list(ing)))
        return True


def _apply(mgr, op):
    return getattr(mgr, op[0])(*op[1:])


def _random_ops(rng, n, recipe_ids, user_ids, names, ingredient_pool):
    ops = []
    for _ in range(n):
        kind = rng.choices(
            ["add_user", "add_recipe", "get_recipe", "edit_recipe", "delete_recipe", "grant_editor",
             "list_recipes", "find_by_ingredient", "search_by_name_prefix",
             "get_recipe_history", "get_version", "rollback_recipe"],
            weights=[2, 4, 3, 4, 2, 3, 2, 3, 3, 2, 3, 3])[0]
        rid, uid, name = rng.choice(recipe_ids), rng.choice(user_ids), rng.choice(names)
        ingredients = rng.sample(ingredient_pool, rng.randint(1, 4))
        if kind == "add_user":
            ops.append(("add_user", uid, f"name-{uid}"))
        elif kind == "add_recipe":
            ops.append(("add_recipe", rid, name, ingredients, uid))
        elif kind == "get_recipe":
            ops.append(("get_recipe", rid))
        elif kind == "edit_recipe":
            ops.append(("edit_recipe", uid, rid, name, ingredients))
        elif kind == "delete_recipe":
            ops.append(("delete_recipe", uid, rid))
        elif kind == "grant_editor":
            ops.append(("grant_editor", uid, rid, rng.choice(user_ids)))
        elif kind == "list_recipes":
            ops.append(("list_recipes",))
        elif kind == "find_by_ingredient":
            ops.append(("find_by_ingredient", rng.choice(ingredient_pool)))
        elif kind == "search_by_name_prefix":
            ops.append(("search_by_name_prefix", name[:rng.randint(0, len(name))]))
        elif kind == "get_recipe_history":
            ops.append(("get_recipe_history", rid))
        elif kind == "get_version":
            ops.append(("get_version", rid, rng.randint(0, 5)))
        else:
            ops.append(("rollback_recipe", uid, rid, rng.randint(0, 5)))
    return ops


def _agree(seed, n_ops):
    rng = random.Random(seed)
    recipe_ids = ["r1", "r2", "r3"]
    user_ids = ["u1", "u2", "u3", "ghost"]           # small pool -- ids collide and get reused across ops
    names = ["Congee", "congee", "Toast", "Jook"]     # "Congee"/"congee" deliberately share a lookup key
    ingredient_pool = ["rice", "water", "egg", "bread", "salt"]
    real, naive = RecipeManager(), _NaiveRecipeManager()
    counts = Counter()
    for op in _random_ops(rng, n_ops, recipe_ids, user_ids, names, ingredient_pool):
        got, want = _apply(real, op), _apply(naive, op)
        assert got == want, (seed, op, got, want)
        if op[0] in ("add_recipe", "edit_recipe", "delete_recipe", "grant_editor", "rollback_recipe"):
            counts[(op[0], bool(want))] += 1
    return counts


totals = Counter()
for seed in range(400):
    totals += _agree(seed, n_ops=40 + seed % 40)
assert min(totals.values()) > 10, totals   # every mutator, both succeeding and blocked, fired repeatedly
print(f"cross-validated 400 random operation sequences against an independent reference model: {dict(totals)}")
print("all checks passed")
```

</details>

</details>
