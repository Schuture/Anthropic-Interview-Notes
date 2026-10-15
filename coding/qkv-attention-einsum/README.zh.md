# 用 einsum 实现多头注意力

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程（含数学推导）· NumPy | ★★☆☆☆ | 困难 | MLE · RE · RS | attention, einsum, multi-head, masking, numerical-stability, debugging | 3 个部分 / 55 分钟 | 技术电面 |
<!-- meta:end -->

## 题目

*缩放点积注意力*（scaled dot-product attention）把一组*查询*（query）向量映射为一组*值*（value）向量的加权平均，每个值的权重由该查询与这个值对应的*键*（key）向量之间的匹配得分算出。用 NumPy 实现下面三个部分。

### Part 1 —— 缩放点积注意力

`Q` 是形状为 `(B, T_q, d)` 的浮点数组：`B` 个互相独立的批次，每个批次有 `T_q` 个 $\mathbb{R}^d$ 中的查询向量。`K` 的形状是 `(B, T_k, d)`：`T_k` 个与查询同处 $\mathbb{R}^d$ 的键向量。`V` 的形状是 `(B, T_k, d_v)`：`T_k` 个值向量，每个键对应一个值，它的维度 $d_v$ 不必等于 $d$。对批次 $b$、查询 $i$、键 $j$，

```math
\mathrm{score}_{b,i,j} = \frac{Q_{b,i} \cdot K_{b,j}}{\sqrt{d}}, \qquad
\mathrm{weights}_{b,i,:} = \mathrm{softmax}(\mathrm{score}_{b,i,:}), \qquad
\mathrm{output}_{b,i} = \sum_{j=0}^{T_k-1} \mathrm{weights}_{b,i,j}\, V_{b,j},
```

softmax 在 $j$ 上进行，也就是 `score` 的最后一个轴。可选的布尔数组 `mask`，形状为 `(B, T_q, T_k)`，限制每个查询能用哪些键：`mask[b, i, j]` 为 `True` 当且仅当批次 $b$ 的查询 $i$ 可以关注键 $j$，为 `False` 则不能；`mask=None` 表示每个查询都可以关注每个键。被掩码的键要在 softmax 之前把它的得分设为 $-\infty$ 来排除，绝不能在归一化之后把它的权重清零。如果某个查询的所有键都被掩码（$`\mathrm{mask}_{b,i,:}`$ 全为 `False`），约定它的权重全部是 $0$，输出是形状为 $(d_v,)$ 的零向量，而不是 $0/0$。

上面两次张量收缩——$`QK^\top`$ 和 $\mathrm{weights} \cdot V$——都要用 `np.einsum` 计算；softmax 是沿一个轴的归约，不是张量收缩，仍用普通的 NumPy 运算。让 softmax 数值稳定：在调用 `np.exp` 之前先减去每行的最大值。

```py
def scaled_dot_product_attention(Q: np.ndarray, K: np.ndarray, V: np.ndarray,
                                  mask: np.ndarray | None = None) -> np.ndarray:
    """Q: (B, T_q, d), K: (B, T_k, d), V: (B, T_k, d_v). mask: (B, T_q, T_k) bool or None,
    True where the query may attend to the key. Returns (B, T_q, d_v)."""
```

举例，$`B=1`$、$`T_q=2`$、$`T_k=3`$、$`d=2`$、$`d_v=1`$：

```text
Q[0] = [[1, 0],      K[0] = [[1, 0],      V[0] = [[10],
        [0, 1]]              [0, 1],             [20],
                              [1, 1]]             [30]]

mask[0] = [[False, False, False],   # 查询 0：所有键都被掩码
           [True,  False, True ]]   # 查询 1：可以关注键 0 和键 2，不能关注键 1

查询 0 -> weights[0, 0] = [0, 0, 0]，output[0, 0] = [0]                       # 整行都被掩码

查询 1 -> 原始得分 Q[0,1].K[0,j]，j = 0, 1, 2 时分别是 0, 1, 1；乘以 1/sqrt(2) 后是 0, 0.7071, 0.7071
          键 1 被掩码，所以只有键 0 和键 2 进入 softmax
          weights[0, 1] ~= [0.3302, 0, 0.6698]
          output[0, 1] = 0.3302 * 10 + 0.6698 * 30 ~= [23.3952]
```

### Part 2 —— 多头自注意力

*自注意力*（self-attention）是查询、键、值都来自同一个序列的缩放点积注意力。`X` 是形状为 `(B, T, d_model)` 的浮点数组。权重矩阵 `W_q`、`W_k`、`W_v`，形状都是 `(d_model, d_model)`，算出 $Q = XW_q$、$`K = XW_k`$、$`V = XW_v`$，形状都是 `(B, T, d_model)`。`d_model` 必须能被*头*（head）的个数 $H$ 整除，每个头的大小是 $d_{\text{head}} = d_{\text{model}} / H$。$`Q`$ 的头 $h$（$`0 \le h < H`$）的第 $e$ 个特征（$`0 \le e < d_{\text{head}}`$）就是 $Q$ 的第 $h \cdot d_{\text{head}} + e$ 列，$`K`$、$`V`$ 同理。头 $h$ 在 $T$ 个位置之间跑一次缩放点积注意力（Part 1 的公式，把 $d$ 换成 $d_{\text{head}}$），只用它自己那部分 $Q$、$`K`$、$`V`$，并且加上*因果掩码*（causal mask）：位置 $i$ 可以关注位置 $j$ 当且仅当 $j \le i$（查询不能关注更靠后的位置）。把 $H$ 个头的输出沿特征轴拼接起来，再乘以形状为 `(d_model, d_model)` 的输出投影 $W_o$，得到最终输出，形状 `(B, T, d_model)`。

用 `np.reshape`（拆分或合并头和特征这两个轴）和 `np.einsum`（把头轴移到时间轴之前，以及移回去）实现头的拆分与合并；三次输入投影、每个头的得分、每个头对 $V$ 的加权求和、输出投影，这些张量收缩全部用 `np.einsum` 实现。说明每个中间数组的形状。在给得分定标之前，先推导为什么要除以 $\sqrt{d_{\text{head}}}$：如果查询向量 $q$ 和键向量 $k$ 的 $d_{\text{head}}$ 个分量都是均值为 $0$、方差为 $1$ 的独立随机变量，求 $\mathrm{Var}(q \cdot k)$，并说明除以 $\sqrt{d_{\text{head}}}$ 起到了什么作用。

```py
def multi_head_self_attention(X: np.ndarray, W_q: np.ndarray, W_k: np.ndarray, W_v: np.ndarray,
                               W_o: np.ndarray, n_heads: int) -> np.ndarray:
    """X: (B, T, d_model). W_q, W_k, W_v, W_o: (d_model, d_model). d_model must be divisible by
    n_heads; d_head = d_model // n_heads. Applies a causal mask. Returns (B, T, d_model)."""
```

举例，$`B=1`$、$`T=2`$、$`d_{\text{model}}=4`$、$`H=2`$（于是 $d_{\text{head}}=2$）、$`W_q=W_k=W_v=W_o=I_4`$：

```text
X[0] = [[1, 0, 2, 0],       # 头 0 = 特征 [0, 1]，头 1 = 特征 [2, 3]
        [0, 1, 0, 2]]       # 这里权重矩阵都是单位矩阵，所以 Q = K = V = X

查询 0 只能关注键 0（因果）-> output[0, 0] = V[0, 0] = X[0, 0] = [1, 0, 2, 0]

查询 1 可以关注键 0 和键 1：
  头 0：原始得分 0, 1，乘以 1/sqrt(2) -> 0, 0.7071；weights ~= [0.3302, 0.6698]；头输出 ~= [0.3302, 0.6698]
  头 1：原始得分 0, 4，乘以 1/sqrt(2) -> 0, 2.8284；weights ~= [0.0558, 0.9442]；头输出 ~= [0.1116, 1.8884]
  拼接两个头的输出，再乘 W_o = I -> output[0, 1] ~= [0.3302, 0.6698, 0.1116, 1.8884]
```

### Part 3 —— 调试：层归一化与因果自注意力

*层归一化*（layer normalization）把一个 $d$ 维行向量 $v$ 缩放为零均值、单位方差，再做一次可学习的逐元素仿射变换，缩放为 $\gamma \in \mathbb{R}^d$，平移为 $\beta \in \mathbb{R}^d$：

```math
\mathrm{LN}(v)_k = \frac{v_k - \mu}{\sqrt{\sigma^2 + \epsilon}}\,\gamma_k + \beta_k, \qquad
\mu = \frac{1}{d}\sum_{k=0}^{d-1} v_k, \qquad \sigma^2 = \frac{1}{d}\sum_{k=0}^{d-1} (v_k - \mu)^2,
```

其中 $\epsilon > 0$ 是为数值安全加的一个小常数。作用在形状为 `(B, T, d)` 的数组 `x` 上时，$`B \times T`$ 行各自独立地归一化，每一行只用它自己的 $d$ 个特征。下面这个函数本应该这样归一化输入 `x`，再对归一化后的结果跑单头因果自注意力（Part 2 的公式，取 $H=1$），最后把结果作为残差连接加回 `x`——但它有四个 bug：一个在 `layer_norm` 求均值时归约的轴上，一个在它算出的方差的形状上，一个在 `causal_self_attention` 做指数运算前取最大值的轴上，一个在它给原始得分定标的方式上。把这四个都找出来并修复；代码里其余部分都是对的。修好之后的实现，在 `atol=1e-8` 的误差范围内，要与用显式的批次、时间、特征下标循环写出的同一公式（不用 NumPy 广播）完全一致，对任何输入都成立；它还要有 Part 2 那样的因果性质：改动 `x` 的第 $T-1$ 个 token，不会改变它之前任何位置的输出。

```py
def layer_norm(x: np.ndarray, gamma: np.ndarray, beta: np.ndarray, eps: float = 1e-5) -> np.ndarray:
    """x: (B, T, d). gamma, beta: (d,). Normalizes each (batch, time) row over its d features."""
    mean = x.mean(axis=0, keepdims=True)
    var = ((x - mean) ** 2).mean(axis=-1)
    return (x - mean) / np.sqrt(var + eps) * gamma + beta


def causal_self_attention(x: np.ndarray, W_q: np.ndarray, W_k: np.ndarray, W_v: np.ndarray) -> np.ndarray:
    """x: (B, T, d). W_q, W_k, W_v: (d, d). Single-head causal self-attention. Returns (B, T, d)."""
    T, d = x.shape[1], x.shape[2]
    Q, K, V = x @ W_q, x @ W_k, x @ W_v
    scores = (Q @ np.swapaxes(K, -1, -2)) // np.sqrt(d)
    visible = np.tril(np.ones((T, T), dtype=bool))
    scores = np.where(visible, scores, -np.inf)
    m = scores.max(axis=-2, keepdims=True)
    exp_scores = np.exp(scores - m)
    probs = exp_scores / exp_scores.sum(axis=-1, keepdims=True)
    return probs @ V


def transformer_block_forward(x: np.ndarray, gamma: np.ndarray, beta: np.ndarray,
                               W_q: np.ndarray, W_k: np.ndarray, W_v: np.ndarray) -> np.ndarray:
    """x: (B, T, d). Pre-norm residual block: x + causal_self_attention(layer_norm(x))."""
    return x + causal_self_attention(layer_norm(x, gamma, beta), W_q, W_k, W_v)
```

举例，$`B=1`$、$`T=2`$、$`d=2`$、$`\gamma=(1,1)`$、$`\beta=(0,0)`$、$`W_q=W_k=W_v=I_2`$：

```text
x[0] = [[1, 0],
        [0, 1]]

查询 0 只能关注键 0（因果）-> transformer_block_forward(x, ...)[0, 0] ~= [2.0000, -1.0000]
查询 1 可以关注两个键     -> transformer_block_forward(x, ...)[0, 1] ~= [-0.8884, 1.8884]
```

## 参考解答

<details>
<summary>展开参考解答</summary>

值得先和面试官确认：`mask` 的约定（这里假定 `True` 表示可以关注，与 `torch.nn.functional.scaled_dot_product_attention` 的 `attn_mask` 一致）、整行都被掩码时应该返回什么（这里假定是零向量，而不是报错）、`d_model` 是否总能被 `n_heads` 整除；以及在 Part 3 里，是否只需要修复这四个已知的 bug，其余代码都正确。

### Part 1

两次张量收缩各是一次分批的矩阵乘法。`np.einsum('bqd,bkd->bqk', Q, K)` 沿共有的特征轴 `d` 求和，剩下每个
`(批次, 查询, 键)` 三元组一个得分；`np.einsum('bqk,bkd->bqd', weights, V)` 沿键轴求和，剩下每个
`(批次, 查询)` 一个输出向量。在 `np.exp` 之前减去行最大值，在精确算术下不改变 softmax 的结果：对任何不依赖被求和下标
$j$ 的常数 $m$，

$$\frac{e^{s_j - m}}{\sum_k e^{s_k - m}} = \frac{e^{-m}e^{s_j}}{e^{-m}\sum_k e^{s_k}} = \frac{e^{s_j}}{\sum_k e^{s_k}},$$

同时让每个指数都 $\le 0$，`np.exp` 就不会溢出。整行都被掩码时，每个得分都是 $-\infty$，行最大值也是
$-\infty$，$`-\infty - (-\infty)`$ 是 `nan`；实现里把这一行的 $m$ 换成 $0$ 再相减，这一行的每个元素仍然是
$\exp(-\infty) = 0$（这一行的得分本来就是 $-\infty$，减去的有限值是多少并不影响这一点），并单独防住
softmax 分母——同样在这一行恰好为 $0$——引发的 $0/0$。

```python
import numpy as np


def scaled_dot_product_attention(Q, K, V, mask=None):
    d = Q.shape[-1]
    scores = np.einsum('bqd,bkd->bqk', Q, K) / np.sqrt(d)            # (B, T_q, T_k)
    if mask is not None:
        scores = np.where(mask, scores, -np.inf)
    m = np.max(scores, axis=-1, keepdims=True)                       # (B, T_q, 1)
    # NOTE: a fully masked row has every score -inf, so m is -inf too, and -inf - (-inf) is nan.
    #       Replace m with 0 there; exp(scores - 0) is still exp(-inf) = 0 on that row.
    m = np.where(np.isneginf(m), 0.0, m)
    exp_scores = np.exp(scores - m)                                  # (B, T_q, T_k)
    denom = np.sum(exp_scores, axis=-1, keepdims=True)                # (B, T_q, 1)
    # NOTE: denom is exactly 0 on a fully masked row; dividing 0 / 0 is nan, so guard the denominator
    #       (the numerator is 0 there regardless, so the guard value only avoids the nan).
    weights = exp_scores / np.where(denom == 0.0, 1.0, denom)
    return np.einsum('bqk,bkd->bqd', weights, V)
```

复杂度：得分那次收缩的时间是 $O(BT_qT_kd)$，加权求和那次收缩的时间是 $O(BT_qT_kd_v)$，`scores`/`weights`
占用 $O(BT_qT_k)$ 的额外内存，此外还有输入和输出本身。

### Part 2

$Q$、$`K`$、$`V`$ 各是一次与对应权重矩阵的收缩：`np.einsum('btd,de->bte', X, W_q)`，`W_k`、`W_v` 同理——形状
`(B, T, d_model)`。把最后一个轴拆成 $(H, d_{\text{head}})$ 是一次 `reshape`，不是 `einsum`（`einsum` 只对已有的轴做置换和求和，不能把一个轴拆成两个），得到形状
`(B, T, H, d_head)`；把头轴移到时间轴之前是一次纯置换，`np.einsum('bthe->bhte', ...)`，得到
`(B, H, T, d_head)`。每个头的得分是 `np.einsum('bhqe,bhke->bhqk', Qh, Kh)`，形状 `(B, H, T, T)`——和
Part 1 的收缩一样，只是多了一个类似批次的轴 `h`；对 $V$ 的加权求和是
`np.einsum('bhqk,bhke->bhqe', weights, Vh)`，形状 `(B, H, T, d_head)`。合并头正好把拆分反过来做：
`np.einsum('bhte->bthe', ...)` 把头轴移回时间轴之后，得到 `(B, T, H, d_head)`，`reshape` 再把最后两个轴合并成
`(B, T, d_model)`；最后 `np.einsum('btd,de->bte', merged, W_o)` 给出 `(B, T, d_model)`。因为 $i \le i$
恒成立，查询总能看到自己，所以这里 `scores.max(axis=-1)` 永远不是 $-\infty$，Part 1 里整行掩码的情形在因果的模块里不会出现。

**$`1/\sqrt{d_{\text{head}}}`$ 这个定标。** 设 $q, k \in \mathbb{R}^{d_{\text{head}}}$ 的各分量都是均值 $0$、方差
$1$ 的独立随机变量，并且每个 $q_i$ 都与每个 $k_j$ 独立。于是 $q \cdot k = \sum_{i=0}^{d_{\text{head}}-1} q_ik_i$，

$$\mathrm{E}[q \cdot k] = \sum_i \mathrm{E}[q_i]\\,\mathrm{E}[k_i] = 0, \qquad \mathrm{Var}(q\cdot k) = \mathrm{E}\Bigl[\bigl(\textstyle\sum_i q_ik_i\bigr)^2\Bigr] = \sum_i\sum_j \mathrm{E}[q_ik_iq_jk_j].$$

当 $i \ne j$ 时，$`q_i, k_i, q_j, k_j`$ 是四个相互独立、均值为零的变量，所以
$\mathrm{E}[q_ik_iq_jk_j] = \mathrm{E}[q_i]\mathrm{E}[k_i]\mathrm{E}[q_j]\mathrm{E}[k_j] = 0$；当
$i = j$ 时，$`\mathrm{E}[q_i^2k_i^2] = \mathrm{E}[q_i^2]\,\mathrm{E}[k_i^2] = 1 \cdot 1 = 1`$，用到了
$q_i$ 与 $k_i$ 的独立性，以及 $\mathrm{E}[q_i^2] = \mathrm{Var}(q_i) = 1$。于是
$\mathrm{Var}(q\cdot k) = \sum_{i=0}^{d_{\text{head}}-1} 1 = d_{\text{head}}$：在这个模型下，点积的标准差、也就是它典型的大小，按
$\sqrt{d_{\text{head}}}$ 增长。$`d_{\text{head}}`$ 越大，未定标的得分就分散得越开，softmax 越会趋于饱和、逼近
one-hot 向量，它的雅可比 $\partial p_i/\partial s_j = p_i(\delta_{ij}-p_j)$ 在每一处都趋于
$0$——对得分的梯度趋于消失。把 $q \cdot k$ 除以 $\sqrt{d_{\text{head}}}$，能把它的方差重新缩放回恰好
$1$，且对任意 $d_{\text{head}}$ 都成立，与头的大小无关，这正是不论每个头有多少特征都能让 softmax 处在良态区域的原因。

```python
def multi_head_self_attention(X, W_q, W_k, W_v, W_o, n_heads):
    B, T, d_model = X.shape
    d_head = d_model // n_heads

    Q = np.einsum('btd,de->bte', X, W_q)                              # (B, T, d_model)
    K = np.einsum('btd,de->bte', X, W_k)                              # (B, T, d_model)
    V = np.einsum('btd,de->bte', X, W_v)                              # (B, T, d_model)

    Qh = np.einsum('bthe->bhte', Q.reshape(B, T, n_heads, d_head))    # (B, H, T, d_head)
    Kh = np.einsum('bthe->bhte', K.reshape(B, T, n_heads, d_head))    # (B, H, T, d_head)
    Vh = np.einsum('bthe->bhte', V.reshape(B, T, n_heads, d_head))    # (B, H, T, d_head)

    scores = np.einsum('bhqe,bhke->bhqk', Qh, Kh) / np.sqrt(d_head)   # (B, H, T, T)
    visible = np.tril(np.ones((T, T), dtype=bool))                    # (T, T); visible[i, j] = query i sees key j
    scores = np.where(visible, scores, -np.inf)

    m = np.max(scores, axis=-1, keepdims=True)                        # (B, H, T, 1); never -inf: i sees itself
    exp_scores = np.exp(scores - m)
    weights = exp_scores / np.sum(exp_scores, axis=-1, keepdims=True)  # (B, H, T, T)

    head_out = np.einsum('bhqk,bhke->bhqe', weights, Vh)               # (B, H, T, d_head)
    merged = np.einsum('bhte->bthe', head_out).reshape(B, T, d_model)  # (B, T, d_model)
    return np.einsum('btd,de->bte', merged, W_o)                       # (B, T, d_model)
```

复杂度：Part 2 里四个 `(d_model, d_model)` 投影合计花费 $O(BTd_{\text{model}}^2)$；每个头的收缩花费
$O(BHT^2d_{\text{head}}) = O(BT^2d_{\text{model}})$，因为 $H \cdot d_{\text{head}} = d_{\text{model}}$；
`scores`/`weights` 占用 $O(BHT^2)$ 的内存。序列很长时（$`T \gg d_{\text{model}}`$）$`T^2`$ 那一项占主导；序列短时，
$d_{\text{model}}^2$ 的投影占主导。

### Part 3

按下面的顺序一个个修复，检查代码用的那组随机输入（$`B=3`$、$`T=5`$、$`d=4`$）会依次暴露出不同的症状：

| 已修复的 bug | 结果 | 指向 |
| --- | --- | --- |
| 无 | `ValueError: operands could not be broadcast together with shapes (3,5,4) (3,5)` | `layer_norm` 的 `var`，缺了 `keepdims` |
| `var` 的 `keepdims` | 能跑，但与公式最多相差 $1.32$ | `layer_norm` 的 `mean`，轴错了 |
| + `mean` 的轴 | 仍然有差，最多相差 $0.56$ | softmax 的 `max`，轴错了 |
| + softmax 的 `max` | 仍然有差，最多相差 $0.17$ | 得分定标，`//` |
| 全部四个 | 与浮点精度相符 | |

**`layer_norm` 的 `var`。** `x.mean(axis=-1, keepdims=True)` 已经是形状 `(B, T, 1)`，但
`((x - mean) ** 2).mean(axis=-1)` 把被归约的轴直接丢掉，而不是保留成大小 $1$，形状变成 `(B, T)`。用它去除形状为
`(B, T, d)` 的 `x - mean` 时，NumPy 按末尾对齐两个形状，也就是拿 `d` 去对 `T`；只要 $d \ne T$，NumPy 就无法广播，直接报错。

**`layer_norm` 的 `mean`。** `axis=0` 归约的是批次轴，而不是特征轴，于是每一行都被*同一批次里所有序列在这一特征上的均值*居中，而不是*它自己那
$`d`$ 个特征的均值*——当 $\gamma=1,\beta=0$ 且批次大小为 $1$ 时格外明显：一个元素的均值就是它自己，所以 `x - mean`
恒为 $0$，`layer_norm` 不管 `x` 是什么都只返回常数 $\beta$。

**softmax 的 `max`。** `axis=-2` 取的是查询轴上的最大值，而不是键轴。Part 1 里用来说明减去一个常数不改变结果的等式，要求这个常数不依赖被求和的那个下标（键轴，`axis=-1`）；`axis=-2`
算出的最大值却随键的下标变化，减去它会改变同一行里各元素的相对大小，得到的是一个不同的 softmax，而不仅仅是算法不同、结果相同。

**得分定标。** `//` 会把 $QK^\top$ 的每个元素在送入 softmax 之前先向下取整成不大于它的整数——因为 NumPy 里浮点数的
`//` 就是向下取整除法，softmax 根本看不到小数部分。这个错误不会报错也不会有警告，只是把数字在指数运算之前悄悄地向下砍掉了最多
$1$。

```python
def layer_norm(x, gamma, beta, eps=1e-5):
    mean = x.mean(axis=-1, keepdims=True)                             # NOTE: was axis=0
    var = ((x - mean) ** 2).mean(axis=-1, keepdims=True)               # NOTE: was missing keepdims=True
    return (x - mean) / np.sqrt(var + eps) * gamma + beta


def causal_self_attention(x, W_q, W_k, W_v):
    T, d = x.shape[1], x.shape[2]
    Q, K, V = x @ W_q, x @ W_k, x @ W_v
    scores = (Q @ np.swapaxes(K, -1, -2)) / np.sqrt(d)                 # NOTE: was // np.sqrt(d)
    visible = np.tril(np.ones((T, T), dtype=bool))
    scores = np.where(visible, scores, -np.inf)
    m = scores.max(axis=-1, keepdims=True)                             # NOTE: was axis=-2
    exp_scores = np.exp(scores - m)
    probs = exp_scores / exp_scores.sum(axis=-1, keepdims=True)
    return probs @ V


def transformer_block_forward(x, gamma, beta, W_q, W_k, W_v):
    return x + causal_self_attention(layer_norm(x, gamma, beta), W_q, W_k, W_v)
```

### 追问

- **复杂度。** Part 2 里四个 `(d_model, d_model)` 投影花费 $O(BTd_{\text{model}}^2)$；每个头的得分收缩和加权求和收缩合计花费
  $O(BT^2d_{\text{model}})$；`scores`/`weights` 占用 $O(BHT^2)$ 内存。序列长时 $T^2$ 那一项占主导；序列短时
  $d_{\text{model}}^2$ 的投影占主导。
- **KV 缓存。** 在因果掩码下，位置 $j$ 的键和值不依赖它之后的任何位置。逐个生成新 token 的解码过程因此可以把之前每个位置的
  $K$、$`V`$ 缓存起来：新的一步只需要新位置的 $Q$，把新的一行拼进缓存的 $K$、$`V`$，花费 $O(T)$，而不必从头重新计算全部
  $T$ 个位置、做一次 $O(T^2)$ 的注意力。
- **`-inf` 还是一个很大的负数。** `-inf` 让被掩码位置的权重恰好是 $0$，这正是整行掩码时 `denom` 恰好为 $0$、因而可以被检测到的原因。用
  `-1e9` 这样的有限值代替，能避免整行掩码时算出 `nan`（softmax 会悄悄地变成均匀分布），但它本身只在足够宽的类型里才能被表示：转成
  `float16`（能表示的最大有限值约为 $65504$）之后，`-1e9` 照样会变成 `-inf`。
- **多查询与分组查询注意力。** 让 $K$、$`V`$ 的头数比 $Q$ 少——共用一个头（多查询，multi-query）或者分成几组、每组由若干查询头共用（分组查询，grouped-query）——做法是投影到
  `g < d_model` 个特征，`np.einsum('btd,dg->btg', X, W_k)`，再 reshape 成 `(B, T, H_kv, d_head)`。同一组里的每个查询头都和同一份
  `Kh`/`Vh` 做收缩；换来的好处是 KV 缓存只需要存 $H_{kv}$ 个头的键和值，而不是 $H$ 个。
- **`einsum` 与 `@`。** `np.einsum(subscripts, *arrays)` 不加 `optimize=True` 时会按从左到右的顺序求值，不会调用
  BLAS，在同样的数组上可能比 `@` 慢好几倍；加上 `optimize=True`（或者把这次收缩写成永远调用 BLAS 的
  `@`/`np.matmul`）能补上几乎全部差距。`einsum` 显式写出下标，对上面这种多轴收缩来说更清楚，值得使用——只是在把它放进热循环之前，记得先跑一下基准测试。

<details>
<summary>验证代码（可运行）</summary>

```python
import torch
import torch.nn.functional as F


def loop_attention(Q, K, V, mask=None):                           # Part 1, straight from the formula
    B, Tq, d = Q.shape
    Tk, dv = K.shape[1], V.shape[2]
    out = np.zeros((B, Tq, dv))
    for b in range(B):
        for i in range(Tq):
            visible = [j for j in range(Tk) if mask is None or mask[b, i, j]]
            if not visible:
                continue                                          # out[b, i] stays the zero vector
            scores = [np.dot(Q[b, i], K[b, j]) / np.sqrt(d) for j in range(Tk)]
            m = max(scores[j] for j in visible)
            exps = {j: np.exp(scores[j] - m) for j in visible}
            s = sum(exps.values())
            for j in visible:
                out[b, i] += (exps[j] / s) * V[b, j]
    return out


# Part 1: the worked example of the statement
Q = np.array([[[1., 0.], [0., 1.]]])
K = np.array([[[1., 0.], [0., 1.], [1., 1.]]])
V = np.array([[[10.], [20.], [30.]]])
mask = np.array([[[False, False, False], [True, False, True]]])
out = scaled_dot_product_attention(Q, K, V, mask)
assert np.array_equal(out[0, 0], [0.0])                            # fully masked row
assert round(float(out[0, 1, 0]), 4) == 23.3952
assert np.allclose(out, loop_attention(Q, K, V, mask))

# Part 1: random trials against the loop version, exercising the fully masked case
rng = np.random.default_rng(0)
fully_masked = 0
for _ in range(200):
    B, Tq, Tk, d, dv = (int(rng.integers(1, 4)) for _ in range(5))
    Qr, Kr, Vr = rng.normal(size=(B, Tq, d)), rng.normal(size=(B, Tk, d)), rng.normal(size=(B, Tk, dv))
    m = rng.random((B, Tq, Tk)) < 0.5 if rng.random() < 0.7 else None
    if m is not None:
        fully_masked += int(np.sum(~m.any(axis=-1)))
    assert np.allclose(scaled_dot_product_attention(Qr, Kr, Vr, m), loop_attention(Qr, Kr, Vr, m), atol=1e-8)
assert fully_masked > 20                                           # the fully masked case was really exercised

# Part 1: random trials against torch (no mask, so torch's own fully-masked-row nan never comes up)
for _ in range(20):
    B, Tq, Tk, d, dv = (int(rng.integers(1, 4)) for _ in range(5))
    Qr, Kr, Vr = rng.normal(size=(B, Tq, d)), rng.normal(size=(B, Tk, d)), rng.normal(size=(B, Tk, dv))
    theirs = F.scaled_dot_product_attention(torch.tensor(Qr), torch.tensor(Kr), torch.tensor(Vr)).numpy()
    assert np.allclose(scaled_dot_product_attention(Qr, Kr, Vr), theirs, atol=1e-8)


def loop_multi_head(X, W_q, W_k, W_v, W_o, n_heads):                # Part 2, straight from the formula
    B, T, d_model = X.shape
    d_head = d_model // n_heads
    Q, K, V = X @ W_q, X @ W_k, X @ W_v
    out = np.zeros((B, T, d_model))
    for b in range(B):
        for h in range(n_heads):
            sl = slice(h * d_head, (h + 1) * d_head)
            for i in range(T):
                scores = [np.dot(Q[b, i, sl], K[b, j, sl]) / np.sqrt(d_head) for j in range(i + 1)]
                m = max(scores)
                exps = [np.exp(s - m) for s in scores]
                s = sum(exps)
                for j, e in enumerate(exps):
                    out[b, i, sl] += (e / s) * V[b, j, sl]
    return out @ W_o


def split_heads_torch(t, n_heads):
    b, t_, dm = t.shape
    return t.reshape(b, t_, n_heads, dm // n_heads).transpose(1, 2)


# Part 2: the worked example of the statement
X = np.array([[[1., 0., 2., 0.], [0., 1., 0., 2.]]])
I4 = np.eye(4)
out = multi_head_self_attention(X, I4, I4, I4, I4, n_heads=2)
assert np.array_equal(out[0, 0], [1., 0., 2., 0.])                 # a token that can only see itself reproduces itself
assert [round(float(v), 4) for v in out[0, 1]] == [0.3302, 0.6698, 0.1116, 1.8884]

# Part 2: random trials against the loop version and against torch
for _ in range(20):
    B, T, H = int(rng.integers(1, 4)), int(rng.integers(1, 6)), int(rng.choice([1, 2, 4]))
    d_model = H * int(rng.integers(1, 4))
    X = rng.normal(size=(B, T, d_model)) * 0.5
    W_q, W_k, W_v, W_o = (rng.normal(size=(d_model, d_model)) * 0.5 for _ in range(4))
    ours = multi_head_self_attention(X, W_q, W_k, W_v, W_o, n_heads=H)
    assert np.allclose(ours, loop_multi_head(X, W_q, W_k, W_v, W_o, H), atol=1e-8)

    Xt, Wqt, Wkt, Wvt, Wot = (torch.tensor(a) for a in (X, W_q, W_k, W_v, W_o))
    heads = F.scaled_dot_product_attention(split_heads_torch(Xt @ Wqt, H), split_heads_torch(Xt @ Wkt, H),
                                            split_heads_torch(Xt @ Wvt, H), is_causal=True)
    theirs = (heads.transpose(1, 2).reshape(B, T, d_model) @ Wot).numpy()
    assert np.allclose(ours, theirs, atol=1e-6)

# Part 2: the causal-mask property -- changing a later token never changes an earlier output
X = rng.normal(size=(2, 5, 6))
weights = [rng.normal(size=(6, 6)) * 0.5 for _ in range(4)]
before = multi_head_self_attention(X, *weights, n_heads=3)
X_changed = X.copy()
X_changed[:, -1, :] += rng.normal(size=6)
after = multi_head_self_attention(X_changed, *weights, n_heads=3)
assert np.allclose(before[:, :-1], after[:, :-1], atol=1e-8)
assert not np.allclose(before[:, -1], after[:, -1], atol=1e-4)     # the changed token's own output does move

# Part 2: the 1/sqrt(d_head) variance derivation, checked by simulation
for d_head in (8, 16, 64):
    q = rng.normal(size=(200_000, d_head))
    k = rng.normal(size=(200_000, d_head))
    dot = np.sum(q * k, axis=1)
    assert abs(dot.var() - d_head) < 0.05 * d_head                 # Var(q . k) ~= d_head, unscaled
    assert abs((dot / np.sqrt(d_head)).var() - 1.0) < 0.05          # Var(scaled) ~= 1, for every d_head


def loop_block(x, gamma, beta, W_q, W_k, W_v, eps=1e-5):            # Part 3, straight from the formula, no broadcasting
    B, T, d = x.shape
    normed = np.zeros_like(x)
    for b in range(B):
        for t in range(T):
            row = x[b, t]
            mean = sum(row) / d
            var = sum((v - mean) ** 2 for v in row) / d
            for k_ in range(d):
                normed[b, t, k_] = (row[k_] - mean) / np.sqrt(var + eps) * gamma[k_] + beta[k_]
    Q, K, V = normed @ W_q, normed @ W_k, normed @ W_v
    out = np.zeros_like(x)
    for b in range(B):
        for i in range(T):
            scores = [np.dot(Q[b, i], K[b, j]) / np.sqrt(d) for j in range(i + 1)]
            m = max(scores)
            exps = [np.exp(s - m) for s in scores]
            s = sum(exps)
            for j, e in enumerate(exps):
                out[b, i] += (e / s) * V[b, j]
            out[b, i] += x[b, i]
    return out


# Part 3: the worked example of the statement
x = np.array([[[1., 0.], [0., 1.]]])
gamma, beta = np.array([1., 1.]), np.array([0., 0.])
I2 = np.eye(2)
out = transformer_block_forward(x, gamma, beta, I2, I2, I2)
assert [round(float(v), 4) for v in out[0, 0]] == [2.0, -1.0]
assert [round(float(v), 4) for v in out[0, 1]] == [-0.8884, 1.8884]

# Part 3: random trials against the loop version, and the causal property
B, T, d = 3, 5, 4
x = rng.normal(size=(B, T, d))
gamma, beta = rng.normal(size=d), rng.normal(size=d)
W_q, W_k, W_v = rng.normal(size=(d, d)), rng.normal(size=(d, d)), rng.normal(size=(d, d))
fixed = transformer_block_forward(x, gamma, beta, W_q, W_k, W_v)
reference = loop_block(x, gamma, beta, W_q, W_k, W_v)
assert np.allclose(fixed, reference, atol=1e-8)
x_changed = x.copy()
x_changed[:, -1, :] += 3.0
assert np.allclose(fixed[:, :-1], transformer_block_forward(x_changed, gamma, beta, W_q, W_k, W_v)[:, :-1], atol=1e-8)


# Part 3: reintroducing any one of the four bugs breaks the output, on the same input
def layer_norm_bug_axis(x, gamma, beta, eps=1e-5):
    mean = x.mean(axis=0, keepdims=True)                            # bug: was axis=-1
    var = ((x - mean) ** 2).mean(axis=-1, keepdims=True)
    return (x - mean) / np.sqrt(var + eps) * gamma + beta


def layer_norm_bug_keepdims(x, gamma, beta, eps=1e-5):
    mean = x.mean(axis=-1, keepdims=True)
    var = ((x - mean) ** 2).mean(axis=-1)                            # bug: missing keepdims=True
    return (x - mean) / np.sqrt(var + eps) * gamma + beta


def attention_bug_max_axis(x, W_q, W_k, W_v):
    T, d = x.shape[1], x.shape[2]
    Q, K, V = x @ W_q, x @ W_k, x @ W_v
    scores = (Q @ np.swapaxes(K, -1, -2)) / np.sqrt(d)
    visible = np.tril(np.ones((T, T), dtype=bool))
    scores = np.where(visible, scores, -np.inf)
    m = scores.max(axis=-2, keepdims=True)                          # bug: was axis=-1
    exp_scores = np.exp(scores - m)
    probs = exp_scores / exp_scores.sum(axis=-1, keepdims=True)
    return probs @ V


def attention_bug_floor_div(x, W_q, W_k, W_v):
    T, d = x.shape[1], x.shape[2]
    Q, K, V = x @ W_q, x @ W_k, x @ W_v
    scores = (Q @ np.swapaxes(K, -1, -2)) // np.sqrt(d)              # bug: was / np.sqrt(d)
    visible = np.tril(np.ones((T, T), dtype=bool))
    scores = np.where(visible, scores, -np.inf)
    m = scores.max(axis=-1, keepdims=True)
    exp_scores = np.exp(scores - m)
    probs = exp_scores / exp_scores.sum(axis=-1, keepdims=True)
    return probs @ V


out_bug_axis = x + causal_self_attention(layer_norm_bug_axis(x, gamma, beta), W_q, W_k, W_v)
assert not np.allclose(out_bug_axis, reference, atol=1e-8)

try:
    layer_norm_bug_keepdims(x, gamma, beta)                          # d != T, so this cannot silently broadcast
    assert False, "expected a broadcasting error"
except ValueError:
    pass

out_bug_max = x + attention_bug_max_axis(layer_norm(x, gamma, beta), W_q, W_k, W_v)
assert not np.allclose(out_bug_max, reference, atol=1e-8)

out_bug_div = x + attention_bug_floor_div(layer_norm(x, gamma, beta), W_q, W_k, W_v)
assert not np.allclose(out_bug_div, reference, atol=1e-8)


def attention_both_bugs(x, W_q, W_k, W_v):                          # softmax-max axis and floor-div together
    T, d = x.shape[1], x.shape[2]
    Q, K, V = x @ W_q, x @ W_k, x @ W_v
    scores = (Q @ np.swapaxes(K, -1, -2)) // np.sqrt(d)              # bug: was / np.sqrt(d)
    visible = np.tril(np.ones((T, T), dtype=bool))
    scores = np.where(visible, scores, -np.inf)
    m = scores.max(axis=-2, keepdims=True)                          # bug: was axis=-1
    exp_scores = np.exp(scores - m)
    probs = exp_scores / exp_scores.sum(axis=-1, keepdims=True)
    return probs @ V


# Part 3: the cumulative fixing order of the solution's table, on this same input
out_var_only = x + attention_both_bugs(layer_norm_bug_axis(x, gamma, beta), W_q, W_k, W_v)
assert round(float(np.max(np.abs(out_var_only - reference))), 2) == 1.32

out_var_and_mean = x + attention_both_bugs(layer_norm(x, gamma, beta), W_q, W_k, W_v)
assert round(float(np.max(np.abs(out_var_and_mean - reference))), 2) == 0.56

assert round(float(np.max(np.abs(out_bug_div - reference))), 2) == 0.17

print("all checks passed")
```

</details>

</details>
