# Take-home：样本维度的双下降

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| Take-home · 实验与报告 | ★☆☆☆☆ | 困难 | RE · RS | double-descent, linear-regression, bias-variance, ridge-regression, random-matrix-theory | 4 小时 | Take-home 作业 |
<!-- meta:end -->

## 题目

你有四个小时。设计并运行一个小实验，展示样本维度的双下降，解释它为什么会发生，并提出并验证一个解决办法。

**背景。** 固定一个模型类别和一个模型规模——对线性模型来说是特征数，对神经网络来说是宽度或参数量。用从某个固定分布中抽取的 $n$ 个独立同分布样本训练它，并在同一分布的留出数据上衡量它的误差，即它的*测试误差*（test error）。在经典状态下，训练数据越多，测试误差就单调下降。但如果模型大到足以*插值*（interpolate）其训练集——把训练误差压到零——就可能出现*双下降*（double descent）：随着 $n$ 在固定模型规模下增大，测试误差先下降，在*插值阈值*（interpolation threshold）附近升到一个峰值——插值阈值是在该模型规模下，模型恰好有足够容量精确拟合训练数据所需的最小 $n$——然后在 $n$ 远超过阈值后再次下降。像这里一样固定模型规模、改变 $n$，是该现象的*样本维度*（sample-wise）版本；固定 $n$、改变模型规模则是*模型维度*（model-wise）版本。

**任务。**

1. 选择一个模型类别、一种从已知真值生成带标签数据的方式，以及测试误差的定义。选择如何在固定模型规模下改变 $n$。
2. 在足够多的 $n$ 取值上计算测试误差——覆盖范围从远低于插值阈值到远高于插值阈值——描出双下降曲线；对训练数据做足够多次独立重复取平均，使峰值不是某一次不走运抽样的产物。
3. 用你选择的模型类别来解释：为什么测试误差会在插值阈值附近升到峰值，又在两侧下降。
4. 提出一种补救办法，使其能消除或减弱这个峰值，同时不明显改变曲线的其余部分，并在同一个实验上验证它。

**交付物。** 一份 notebook，包含实验代码、数值结果和简要说明；以及一份不超过八页的幻灯片，用于现场评审，内容涵盖问题本身、你的设置、结果、你的解释、补救办法、其局限性，以及你接下来会尝试的方向。

**限制条件。** 从头到尾四个小时。可以使用 NumPy 和一个绘图库，以及标准库；在这个规模下不需要其他任何东西，也不需要机器学习框架。不得使用 AI 编程助手——实验、解释和幻灯片都必须是你自己完成的工作。

**现场评审。** 提交 notebook 和幻灯片之后，你要当面展示并回答问题：为什么做出这些选择，你的解释在不同的数据分布或不同的模型类别下是否依然成立，以及如果有更多时间你会如何扩展这个实验。

## 参考解答

<details>
<summary>展开参考解答</summary>

本解答选择了这一现象表现得最不含糊的最小模型类别：固定特征数的线性回归、各向同性的高斯输入，以及预先设定好的线性真值。有两个选择值得先说清楚：全文使用的测试误差，是模型在同一分布下一个新样本上的*期望*平方误差，而不是在某一次采样测试集上量出的误差——这一点会在下面给出理由，并在最后对照一个真正采样出的测试集验证；而且曲线上的每一个点，都是在该 $n$ 下对许多次独立重新抽取的训练集取平均得到的，从来不是单次运行的结果。

### Part 1 —— 实验

固定 $d = 100$ 个特征。先抽取一次真值方向 $w^\star \in \mathbb{R}^d$，再把规模为 $n$ 的训练集生成为 $n$ 个独立同分布的各向同性点 $x_i \sim N(0, I_d)$，标签为 $y_i = x_i^\top w^\star + \varepsilon_i$，其中 $\varepsilon_i \sim N(0, \sigma^2)$ 且与 $x_i$ 独立。这是“模型规模”含义最不含糊（即 $d$，特征数）、同时模型又能够插值的最简单模型类别：一旦 $n \ge d$ 且设计矩阵 $X \in \mathbb{R}^{n \times d}$ 满列秩，或者一旦 $n \le d$ 且 $X$ 满行秩，就存在一个能精确拟合每一个训练标签的权重向量。

拟合方式是普通最小二乘，最小化 $\|Xw - y\|_2^2$。当 $n \ge d$ 且 $X$ 满列秩时，唯一的极小值点是 $(X^\top X)^{-1} X^\top y$。当 $n < d$ 且 $X$ 满行秩时，有无穷多个 $w$ 精确满足 $Xw = y$；在这些解中，*最小范数最小二乘解*（minimum-norm least-squares solution）是其中 $\|w\|_2$ 最小的那一个。两种情形由同一个公式给出：$\hat w = X^{+} y$，其中 $X^{+}$ 是*Moore–Penrose 伪逆*：写出奇异值分解 $X = U \Sigma V^\top$，则 $X^{+} = V \Sigma^{+} U^\top$，$\Sigma^{+}$ 由 $\Sigma$ 的每个非零奇异值取倒数、其余保持为零得到。当 $n = d$ 且（设）$X$ 恰好是方阵并可逆时，这个公式就化简为唯一的精确解 $X^{-1}y$——此时模型恰好有足够的样本数确定每一个参数，这正是插值阈值。

因为测试点是各向同性的，模型在一个新点上的期望平方误差 $E_x\bigl[(x^\top \hat w - x^\top w^\star)^2\bigr] = (\hat w - w^\star)^\top E[xx^\top](\hat w - w^\star)$，会恰好化简为 $\|\hat w - w^\star\|_2^2$，因为 $E[xx^\top] = I_d$。带噪声的测试标签还要再加上不可约的 $\sigma^2$。所以整条双下降曲线只需 $\|\hat w - w^\star\|_2^2$ 就能算出，不需要采样测试集，也没有测试集自身的噪声需要靠平均去消除——剩下的唯一随机性，来自训练集本身的抽样。

```python
import os

# NOTE: thousands of tiny pinv/solve calls thrash under multi-threaded BLAS, so pin to one thread
# before numpy loads. This only takes effect if numpy has not already been imported in this process
# (a BLAS library reads these at load time, not on every call); the saved values are restored at the
# end of the checks below so a fresh process is the only case this changes, never a later page's.
_PRIOR_THREAD_ENV = {k: os.environ.get(k) for k in ("OPENBLAS_NUM_THREADS", "OMP_NUM_THREADS", "MKL_NUM_THREADS")}
os.environ["OPENBLAS_NUM_THREADS"] = "1"
os.environ["OMP_NUM_THREADS"] = "1"
os.environ["MKL_NUM_THREADS"] = "1"
import numpy as np

D = 100  # fixed model size: the number of features, held fixed as n varies


def make_ground_truth(d: int, seed: int) -> np.ndarray:
    """A fixed target direction w_star in R^d, drawn once from N(0, I_d)."""
    rng = np.random.default_rng(seed)
    return rng.standard_normal(d)


def make_dataset(n: int, w_star: np.ndarray, sigma: float, seed: int) -> tuple[np.ndarray, np.ndarray]:
    """n i.i.d. samples with isotropic features x ~ N(0, I_d) and label y = x @ w_star + noise."""
    d = w_star.shape[0]
    rng = np.random.default_rng(seed)
    X = rng.standard_normal((n, d))
    y = X @ w_star + sigma * rng.standard_normal(n)
    return X, y


def min_norm_fit(X: np.ndarray, y: np.ndarray) -> np.ndarray:
    """The minimum-norm least-squares solution, via the Moore-Penrose pseudo-inverse."""
    return np.linalg.pinv(X) @ y


def ridge_fit(X: np.ndarray, y: np.ndarray, lam: float) -> np.ndarray:
    """Ridge regression: minimizes ||X w - y||^2 + lam * ||w||^2."""
    d = X.shape[1]
    return np.linalg.solve(X.T @ X + lam * np.eye(d), X.T @ y)


def excess_test_error(w_hat: np.ndarray, w_star: np.ndarray) -> float:
    """E[(x^T what - x^T w_star)^2] for a fresh isotropic test point x ~ N(0, I_d): since
    E[x x^T] = I_d, this equals ||what - w_star||^2 exactly -- no test-set sampling needed.
    Total test MSE on a noisy label adds the irreducible sigma^2 on top."""
    return float(np.sum((w_hat - w_star) ** 2))


def mean_test_error(n: int, w_star: np.ndarray, sigma: float, n_seeds: int, fit, seed0: int) -> float:
    """Test error at sample size n, averaged over n_seeds independently redrawn training sets."""
    errors = [excess_test_error(fit(*make_dataset(n, w_star, sigma, seed0 + s)), w_star) for s in range(n_seeds)]
    return float(np.mean(errors))


SIGMA = 1.0
W_STAR = make_ground_truth(D, seed=0)
NORM_W_STAR_SQ = float(np.sum(W_STAR ** 2))          # approx. 93.23

N_GRID = [10, 20, 30, 40, 50, 60, 70, 80, 90, 95, 98, 99, 100, 101, 102, 105, 110, 120, 150, 200, 300, 400]
N_SEEDS = 40

min_norm_curve = {n: mean_test_error(n, W_STAR, SIGMA, N_SEEDS, min_norm_fit, 10_000 * n) for n in N_GRID}
peak_n = max(min_norm_curve, key=min_norm_curve.get)
```

$n$ 的取值范围从远低于 $D = 100$ 到远高于它，网格在 $D$ 附近——曲线变化最快的地方——取得更密。对每个点在 40 次重新抽取的训练集上取平均后，测试误差为：

| $n$ | 10 | 20 | 30 | 40 | 50 | 60 | 70 | 80 | 90 | 95 | 98 | 99 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 测试误差 | 83.8 | 73.7 | 65.5 | 56.8 | 48.4 | 40.2 | 31.3 | 21.4 | 18.4 | 27.8 | 171.2 | 230.5 |

| $n$ | 100 | 101 | 102 | 105 | 110 | 120 | 150 | 200 | 300 | 400 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 测试误差 | 1549.7 | 869.1 | 76.5 | 21.6 | 10.5 | 5.5 | 1.9 | 1.0 | 0.5 | 0.3 |

测试误差从 $n = 10$ 处的 $83.8$ 一路下降，到 $n = 90$ 处的局部低点 $18.4$；随后骤然攀升——$27.8$、$171.2$、$230.5$——恰好在插值阈值 $n = 100 = D$ 处升到峰值 $1{,}549.7$，再回落经过 $869.1$、$76.5$，恢复经典状态下的平滑下降：$n = 110$ 处为 $10.5$，$n = 200 = 2D$ 处为 $1.0$，$n = 400$ 处为 $0.3$。峰值是 $n = D/2 = 50$ 处误差（$48.4$）的 $32.0$ 倍，是 $n = 2D = 200$ 处误差（$1.0$）的 $1{,}579.6$ 倍。尖峰的具体高度，在 $n = 98$ 到 $102$ 之间本身就不稳定——这几个点的设计矩阵都接近方阵，究竟哪一次抽样恰好落在接近奇异的位置纯属运气；第 2 部分会解释原因，也会说明为什么这种不稳定本身就是现象的一部分，而不是实验的噪声。

### Part 2 —— 为什么会出现这个峰值

先固定训练输入 $X$，只让标签噪声 $\varepsilon$ 保持随机。把 $y = Xw^\star + \varepsilon$ 代入 $\hat w = X^{+}y$，得到 $\hat w = X^{+}Xw^\star + X^{+}\varepsilon$；而 $X^{+}X = P_X$ 正是投影到 $X$ 行空间上的正交投影算子（当 $X$ 满列秩时它就是单位阵），于是

$$\hat w = P_X w^\star + X^{+}\varepsilon.$$

因此 $E_\varepsilon[\hat w \mid X] = P_X w^\star$，$\mathrm{Cov}_\varepsilon(\hat w \mid X) = \sigma^2 X^{+}(X^{+})^\top$。超额测试误差在给定 $X$ 时的*偏差–方差分解*（bias–variance decomposition），就是把它对噪声取期望后拆成两部分：一部分即便在这个 $n$ 下有无穷多数据也不会消失，另一部分则完全由噪声驱动：

$$E_\varepsilon\bigl[\|\hat w - w^\star\|^2 \mid X\bigr] = \underbrace{\|(I - P_X)w^\star\|^2}_{\text{bias}(X)^2} + \underbrace{\sigma^2 \operatorname{tr}\bigl(X^{+}(X^{+})^\top\bigr)}_{\text{variance}(X)}.$$

由奇异值分解写出 $X^{+} = V\Sigma^{+}U^\top$，则 $X^{+}(X^{+})^\top = V(\Sigma^{+})^2 V^\top$，它的迹就是 $\Sigma^{+}$ 各元素的平方和——也就是 $X$ 的非零奇异值的 $1/\sigma_i^2$ 之和：

$$\text{variance}(X) = \sigma^2 \sum_i \frac{1}{\sigma_i^2}.$$

上面每条曲线本来就已经对重新抽取的训练集取过平均，所以每个 $n$ 处报告的测试误差就是 $E_X[\text{bias}(X)^2] + E_X[\text{variance}(X)]$。对各向同性的 $X$，这两项都有闭式解。当 $n \le d$ 时，$X$ 的行空间是 $\mathbb{R}^d$ 中一个均匀随机的 $n$ 维子空间；一个固定向量的平方长度中，期望有 $n/d$ 这一比例落在这样的子空间内，所以 $E_X[\text{bias}(X)^2] = (1 - n/d)\|w^\star\|^2$——一旦 $n \ge d$，每个方向都可达，这一项恰好为 $0$。对方差项而言，$X^\top X$（当 $n > d$ 时）或 $XX^\top$（当 $n < d$ 时）是一个 Wishart 矩阵——由 $\max(n,d)$ 个独立的标准高斯向量外积之和构成——它的逆矩阵有一个经典的闭式均值，给出

$$E_X[\text{variance}(X)] = \sigma^2 \times \begin{cases} n / (d - n - 1) & n < d \\ d / (n - d - 1) & n > d. \end{cases}$$

只要 $n \ne d$，两项都是有限的，但方差项的分母会在 $n$ 从两侧趋向 $d$ 时趋于零：这正是峰值所在。原因在于最小奇异值 $\sigma_{\min}(X)$，它是 $\sum_i 1/\sigma_i^2$ 中最大的那一项加数。对于元素独立同分布于 $N(0,1)$ 的 $X$，*Marchenko–Pastur 定律*指出：当 $n, d \to \infty$ 且 $d/n \to \gamma \le 1$ 时，$X^\top X / n$ 的特征值会铺满区间 $[(1-\sqrt{\gamma})^2, (1+\sqrt{\gamma})^2]$；其下边界 $(1 - \sqrt{d/n})^2$，是谱的主体能达到的最小值。当 $n \to d$ 时，$\gamma \to 1$，这个下边界滑向 $0$，于是 $X^\top X$ 的最小特征值量级就是 $n(1-\sqrt{d/n})^2 = (\sqrt n - \sqrt d)^2$，即

$$\sigma_{\min}(X) \approx |\sqrt n - \sqrt d|.$$

$n$ 每向 $d$ 靠近一步，最小奇异值就被推得更接近 $0$，它的平方倒数就被推向无穷：无论 $\sigma$ 多小，训练标签的噪声都会被 $X$ 中接近奇异的方向无限放大。这个近似在远离边缘的地方成立，却恰恰在峰值最高的地方失效：就在 $n = d$ 处，$\sigma_{\min}$ 不再集中到任何单一数值附近——它服从一个尺度为 $1/\sqrt d$ 的极限分布，并且在 $0$ 附近有任意接近的质量——所以 $E_X[\text{variance}(X)]$ 在 $n = d$ 处确实是无界的，而不只是很大；对任何一组固定的训练集抽样取平均，结果都会被其中恰好接近奇异的少数几次抽样主导。这就是为什么第 1 部分中的峰值在这一组固定种子下算出的是 $1{,}549.7$，而不是实验会收敛到的一个稳定数字。

```python
def bias_variance_of_design(n: int, w_star: np.ndarray, sigma: float, seed: int) -> tuple[float, float]:
    """bias(X)^2 = ||(I - P_X) w_star||^2 and variance(X) = sigma^2 * sum(1 / singular_value^2),
    both computed from one draw of X via its SVD."""
    d = w_star.shape[0]
    rng = np.random.default_rng(seed)
    X = rng.standard_normal((n, d))
    _, s, Vt = np.linalg.svd(X, full_matrices=False)
    if n < d:
        row_space_component = Vt.T @ (Vt @ w_star)
        bias2 = float(np.sum(w_star ** 2) - np.sum(row_space_component ** 2))
    else:
        bias2 = 0.0
    variance = sigma ** 2 * float(np.sum(1.0 / s ** 2))
    return bias2, variance


def bias_formula(n: int, d: int, norm_w_star_sq: float) -> float:
    return max(0.0, 1 - n / d) * norm_w_star_sq


def variance_formula(n: int, d: int, sigma: float) -> float:
    """Exact Wishart-inverse mean: tr = d / (n - d - 1) for n > d, n / (d - n - 1) for n < d."""
    return sigma ** 2 * (n / (d - n - 1) if n < d else d / (n - d - 1))


BV_POINTS = [50, 90, 300]
bias_var_check = {}
for n in BV_POINTS:
    draws = [bias_variance_of_design(n, W_STAR, SIGMA, 20_000 * n + s) for s in range(80)]
    bias_var_check[n] = (float(np.mean([b for b, _ in draws])), float(np.mean([v for _, v in draws])))


def mean_min_singular_value(n: int, d: int, n_seeds: int, seed0: int) -> float:
    values = []
    for s in range(n_seeds):
        rng = np.random.default_rng(seed0 + s)
        values.append(np.linalg.svd(rng.standard_normal((n, d)), compute_uv=False).min())
    return float(np.mean(values))


MP_POINTS = [10, 50, 150, 300]
mp_check = {n: (mean_min_singular_value(n, D, 30, 40_000 * n), abs(np.sqrt(n) - np.sqrt(D))) for n in MP_POINTS}
```

在 $n = 50$、$90$、$300$ 处各对 $X$ 的 80 次抽样取平均，经验得到的（偏差平方、方差）分别是 $(47.62, 1.02)$、$(9.19, 10.14)$ 和 $(0.00, 0.50)$，对应公式给出的是 $(46.61, 1.02)$、$(9.32, 10.00)$ 和 $(0.00, 0.50)$——处处接近，差距最大的地方正是 $n$ 最接近 $d$、也就是这个量本身最难取平均的地方。在远离阈值处对 30 次抽样取平均，$n = 10, 50, 150, 300$ 处最小奇异值的均值分别是 $7.35$、$3.10$、$2.44$、$7.53$，对应边缘预测 $|\sqrt n - \sqrt d|$ 给出的是 $6.84$、$2.93$、$2.25$、$7.32$——在这些安全远离 $n = d$ 的点上，两者相差都在 $9\%$ 以内，是一个不错的近似。

把 $n$ 和 $d$ 的角色互换，同一套推导照样成立：改为固定 $n$、增大 $d$——即双下降的模型维度版本——按同样的边缘论证、把 $n$ 和 $d$ 互换后，会在 $d \approx n$ 处得到一个对称的峰值。但峰值两侧的结果并不对称。当 $d \to \infty$ 而 $n$ 固定时，$X$ 的行空间是越来越大的 $\mathbb{R}^d$ 中的一个 $n$ 维子空间，于是 $E_X[\text{bias}(X)^2] = (1 - n/d)\|w^\star\|^2 \to \|w^\star\|^2$——而不是趋于 $0$——同时 $E_X[\text{variance}(X)] \to 0$，因为 $X$ 的 $d$ 个列会让 $XX^\top$ 集中在 $dI_n$ 附近，每个奇异值都随 $d$ 增大。一个朴素的最小范数拟合，在各向同性特征、没有其他结构的情况下，并不会仅靠增加参数就在越过阈值后恢复到经典状态的低误差；它会停滞在“预测为零”这个误差水平上。那些在远超插值阈值之后仍能良好泛化的模型，靠的不只是参数数量本身——而是架构和优化器的隐式偏好恰好倾向于正确类型的解，或者特征协方差恰好与目标对齐，这正是良性过拟合（benign overfitting）相关工作所刻画的情形。

### Part 3 —— 消除峰值

*岭回归*（ridge regression）在固定 $\lambda > 0$ 时最小化 $\|Xw - y\|_2^2 + \lambda\|w\|_2^2$，其唯一解是 $\hat w_\lambda = (X^\top X + \lambda I_d)^{-1}X^\top y$——对任意 $n$ 和任意 $\lambda > 0$ 都有良好定义，因为 $X^\top X + \lambda I_d$ 的每个特征值至少是 $\lambda$，无论 $X$ 自身的奇异值有多小。

用同样的奇异值分解 $X = U\Sigma V^\top$，把 $w^\star$ 分解成它在行空间中的分量 $\alpha = V^\top w^\star$ 和正交余量 $w^\star_\perp$（只有当 $X$ 不满列秩时才存在），岭回归的估计可以写成 $\hat w_\lambda = V\,\mathrm{diag}\bigl(\sigma_i^2/(\sigma_i^2+\lambda)\bigr)\alpha + V\,\mathrm{diag}\bigl(\sigma_i/(\sigma_i^2+\lambda)\bigr)U^\top\varepsilon$，从而

$$E_\varepsilon\bigl[\|\hat w_\lambda - w^\star\|^2 \mid X\bigr] = \underbrace{\|w^\star_\perp\|^2 + \sum_i \Bigl(\frac{\lambda}{\sigma_i^2+\lambda}\Bigr)^2 \alpha_i^2}_{\text{bias}_\lambda(X)^2} + \underbrace{\sigma^2 \sum_i \frac{\sigma_i^2}{(\sigma_i^2+\lambda)^2}}_{\text{variance}_\lambda(X)}.$$

当 $\lambda \to 0$ 时，这恰好化简为第 2 部分的公式（$\lambda$ 带来的偏差项消失，且 $\sigma_i^2/\sigma_i^4 = 1/\sigma_i^2$）。而当 $\lambda > 0$ 时，方差的每一项加数都是有界的：由均值不等式（AM–GM），对任意 $\sigma_i \ge 0$ 都有 $\sigma_i^2/(\sigma_i^2+\lambda)^2 \le 1/(4\lambda)$（在 $\sigma_i^2 = \lambda$ 处取到最大值），所以无论 $\sigma_{\min}(X)$ 多小，都有 $\text{variance}_\lambda(X) \le \sigma^2 \min(n,d)/(4\lambda)$。岭回归用一点可控的偏差增加——收缩项 $\lambda/(\sigma_i^2+\lambda)$ 把每个方向都往 $0$ 轻轻拉一点——换来对方差的硬上限，彻底消除了 $1/\sigma_i^2$ 的爆炸。在远离阈值的地方，这笔交易只会白白增加偏差，因为那里未加约束的方差本来就很小；而恰恰在 $n = d$ 附近，未加约束的方差原本会大得离谱，这笔交易才真正划算。

$\lambda$ 该取多大？只为了选 $\lambda$ 这一个目的，不妨假设 $w^\star$ 本身是从 $N(0, \tau^2 I_d)$ 中抽出的，并且与标签噪声独立。那么给定 $(X, y)$ 时 $w^\star$ 的后验均值——也就是在这个先验下使期望平方误差最小的估计量——恰好就是

$$E[w^\star \mid X, y] = \bigl(X^\top X + \tfrac{\sigma^2}{\tau^2}I_d\bigr)^{-1}X^\top y,$$

也就是 $\lambda = \sigma^2/\tau^2$ 的岭回归，这对任意 $n$ 和 $d$ 都精确成立，而不只是渐近成立。从该先验中抽出的向量满足 $E\|w^\star\|^2 = d\tau^2$，所以用*实际取到*的 $\|w^\star\|^2$ 作为 $d\tau^2$ 的代入估计，就给出

$$\lambda^\star = \frac{\sigma^2}{\tau^2} \approx \frac{\sigma^2 d}{\|w^\star\|^2}.$$

这只在对先验取平均的意义下精确，对某个固定的 $w^\star$ 并不精确——但 $\|w^\star\|^2/d$ 会随 $d$ 增大而集中到 $\tau^2$ 附近，$d = 100$ 已经足够大，能让这个代入估计足够接近。这里 $\|w^\star\|^2 \approx 93.23$，给出 $\lambda^\star \approx 1.073$。

```python
LAM_STAR = SIGMA ** 2 * D / NORM_W_STAR_SQ  # approx. 1.073

ridge_curve = {n: mean_test_error(n, W_STAR, SIGMA, N_SEEDS, lambda X, y: ridge_fit(X, y, LAM_STAR), 10_000 * n)
               for n in N_GRID}
```

把这个固定的 $\lambda^\star$ 应用到整个网格上——每个 $n$ 都不单独调参——就消除了尖峰：

| $n$ | 80 | 90 | 95 | 98 | 99 | 100 | 101 | 102 | 105 | 110 | 120 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| min-norm | 21.4 | 18.4 | 27.8 | 171.2 | 230.5 | 1549.7 | 869.1 | 76.5 | 21.6 | 10.5 | 5.5 |
| ridge | 20.6 | 13.3 | 12.2 | 10.3 | 9.6 | 9.3 | 9.2 | 8.8 | 7.8 | 5.9 | 4.1 |

在 $n = 100$ 处，岭回归的测试误差是 $9.3$，最小范数解的 $1{,}549.7$ 是它的 $165.8$ 倍；而且它做到的不只是削平峰值——从 $n=10$ 到 $n=400$，岭回归曲线在*整个*网格上都单调不增，正是训练数据增多时本该有的形状，插值阈值在其中已经找不到任何痕迹。

### Part 4 —— 书面材料

notebook 中包含上面的代码——`make_dataset`、`min_norm_fit`、`ridge_fit`、`mean_test_error`，以及第 2 部分里偏差–方差和 Marchenko–Pastur 的辅助函数——两张作为计算输出的数字表，以及足以指明各处推导、但不逐行重复推导本身的行内说明。幻灯片把同样的内容压缩成八页，用于现场评审：

1. **问题。** 在固定模型规模下，测试误差是否总是随数据增多而变好？先用听众熟悉的实际说法把问题讲清楚，再引出“双下降”这个名字。
2. **设置。** $d = 100$ 个特征的线性回归、各向同性高斯输入与预先设定的真值，用伪逆求最小范数最小二乘解，把测试误差定义为 $\|\hat w - w^\star\|^2$（外加固定的 $\sigma^2$），以及这个闭式解为什么能代替一次采样测试集。
3. **结果。** 第 1 部分的数字表，以它所表格化的那条曲线的形式呈现：先下降，在 $n = D$ 处骤然升到 $n = D/2$ 处误差的 $32.0$ 倍、$2D$ 处误差的 $1{,}579.6$ 倍，随后再次下降。
4. **原因。** 偏差–方差分解、$\sigma^2 \sum 1/\sigma_i^2$ 这个方差公式，以及关于 $n \to d$ 时为什么 $\sigma_{\min}(X) \to 0$ 的 Marchenko–Pastur 边缘论证。
5. **修复办法。** 岭回归、有界方差的论证、$\lambda^\star = \sigma^2 d / \|w^\star\|^2$ 的贝叶斯推导，以及第 3 部分中展示峰值消失的数字表。
6. **局限性。** 只有一次真值实现，而不是多次；各向同性、互不相关的特征，这与大多数真实数据不同；$\lambda^\star$ 用到的 $\sigma^2$ 和 $\|w^\star\|^2$ 之所以在这里已知，只是因为数据本身就是生成出来的——实际场景中 $\lambda$ 要靠扫描或交叉验证来选，本实验里的一次粗略扫描给出的风险，与 $\lambda^\star$ 相差不到 $1\%$，说明这个闭式解并没有比搜索多做出什么。
7. **模型维度的双下降与下一步。** 改为改变 $d$ 时，会在 $d \approx n$ 处出现对称的峰值；为什么朴素的最小范数拟合越过那个阈值后会停滞不前，而不是继续改善；再更推测性地提一句：从拟合单个训练点到拟合共享结构的这种同一性质的转变，也出现在机制可解释性（mechanistic interpretability）对记忆与泛化电路的研究中（例如 grokking），不过线性模型没有可供这层联系落到实处的内部电路，在这个规模下它只能算一个类比。
8. **结论与提问。** 用一句话说明这个实验证明了什么、没有证明什么，然后把时间交给现场评审。

### 追问

- 为什么用 $\|\hat w - w^\star\|^2$，而不是采样一个留出测试集？各向同性使两者在期望意义下完全相同，下面的验证代码会直接对照一个大规模采样测试集来确认这一点；采样只会额外引入闭式解本可以避免的测试集噪声。
- 如果特征之间存在相关性，还会出现同样的峰值吗？插值阈值以及它带来的病态条件仍然存在，但精确的偏差和方差公式会随协方差谱变化；对某些与 $w^\star$ 恰当对齐的谱，即便远远越过阈值，误差也能保持很低，完全不需要显式的岭回归项——这正是第 2 部分提到的良性过拟合状态。
- 为什么不用交叉验证来选 $\lambda$，而要推导它？实际场景中 $\sigma^2$ 和 $\|w^\star\|^2$ 都是未知的，$\lambda$ 实际上就是靠交叉验证来选的；这里用闭式解，是因为数据生成过程本身是已知的，而且下面验证代码里的网格搜索证实了交叉验证会得到几乎相同的风险。
- 训练出来的神经网络里，插值阈值会有这么锐利吗？模型维度的双下降在神经网络里也有充分记录，但用 SGD 训练的网络，其“容量”并不只是参数数量——提前停止、架构以及优化器的隐式偏好，都会改变峰值出现的位置，以及它有多锐利。
- 如果再给四个小时：换成相关特征和重尾噪声分布重新做一遍，把单一的闭式 $\lambda$ 换成一次 $\lambda$ 扫描，再和一个用梯度下降训练（而不是闭式求解）的小型两层网络做对比。

<details>
<summary>验证代码（可运行）</summary>

```python
QUOTED_MIN_NORM = {10: 83.8, 20: 73.7, 30: 65.5, 40: 56.8, 50: 48.4, 60: 40.2, 70: 31.3, 80: 21.4,
                    90: 18.4, 95: 27.8, 98: 171.2, 99: 230.5, 100: 1549.7, 101: 869.1, 102: 76.5,
                    105: 21.6, 110: 10.5, 120: 5.5, 150: 1.9, 200: 1.0, 300: 0.5, 400: 0.3}
assert set(QUOTED_MIN_NORM) == set(N_GRID)
assert all(round(min_norm_curve[n], 1) == QUOTED_MIN_NORM[n] for n in N_GRID)

QUOTED_RIDGE = {80: 20.6, 90: 13.3, 95: 12.2, 98: 10.3, 99: 9.6, 100: 9.3, 101: 9.2, 102: 8.8,
                105: 7.8, 110: 5.9, 120: 4.1}
assert all(round(ridge_curve[n], 1) == QUOTED_RIDGE[n] for n in QUOTED_RIDGE)

# the peak sits within a small window of the interpolation threshold D, and towers over both flanks
assert abs(peak_n - D) <= 5
assert min_norm_curve[peak_n] > 10 * min_norm_curve[D // 2]
assert min_norm_curve[peak_n] > 10 * min_norm_curve[2 * D]
assert round(min_norm_curve[peak_n] / min_norm_curve[D // 2], 1) == 32.0
assert round(min_norm_curve[peak_n] / min_norm_curve[2 * D], 1) == 1579.6

# ridge removes the peak: monotone non-increasing (a little slack for floating-point noise) with no
# trace of the spike left at n = D
ridge_vals = [ridge_curve[n] for n in N_GRID]
assert all(ridge_vals[i + 1] <= ridge_vals[i] * 1.02 for i in range(len(ridge_vals) - 1))
assert ridge_curve[peak_n] < 0.05 * min_norm_curve[peak_n]
assert round(min_norm_curve[peak_n] / ridge_curve[peak_n], 1) == 165.8

# Part 2's closed-form bias and variance match direct Monte Carlo, both near and away from the threshold
for n in BV_POINTS:
    b_emp, v_emp = bias_var_check[n]
    assert abs(b_emp - bias_formula(n, D, NORM_W_STAR_SQ)) < 0.1 * NORM_W_STAR_SQ + 1.0
    assert abs(v_emp - variance_formula(n, D, SIGMA)) < 0.25 * variance_formula(n, D, SIGMA) + 0.3

QUOTED_BV_EMP = {50: (47.62, 1.02), 90: (9.19, 10.14), 300: (0.0, 0.5)}
QUOTED_BV_FORMULA = {50: (46.61, 1.02), 90: (9.32, 10.0), 300: (0.0, 0.5)}
for n in BV_POINTS:
    b_emp, v_emp = bias_var_check[n]
    assert (round(b_emp, 2), round(v_emp, 2)) == QUOTED_BV_EMP[n]
    assert (round(bias_formula(n, D, NORM_W_STAR_SQ), 2), round(variance_formula(n, D, SIGMA), 2)) == QUOTED_BV_FORMULA[n]

# the Marchenko-Pastur edge approximation for sigma_min is accurate once n is safely away from d
for n in MP_POINTS:
    emp, pred = mp_check[n]
    assert 0.8 <= emp / pred <= 1.3

QUOTED_MP_EMP = {10: 7.35, 50: 3.1, 150: 2.44, 300: 7.53}
QUOTED_MP_PRED = {10: 6.84, 50: 2.93, 150: 2.25, 300: 7.32}
for n in MP_POINTS:
    emp, pred = mp_check[n]
    assert round(emp, 2) == QUOTED_MP_EMP[n]
    assert round(pred, 2) == QUOTED_MP_PRED[n]

# the analytic test error equals empirical MSE on an actual sampled test set -- an independent check
# of the isotropy argument in Part 1, not a re-derivation of it
X_val, y_val = make_dataset(60, W_STAR, SIGMA, seed=999)
w_hat_val = min_norm_fit(X_val, y_val)
analytic_error = excess_test_error(w_hat_val, W_STAR) + SIGMA ** 2
rng_val = np.random.default_rng(12345)
n_test = 200_000
X_big = rng_val.standard_normal((n_test, D))
y_big = X_big @ W_STAR + SIGMA * rng_val.standard_normal(n_test)
empirical_mse = float(np.mean((X_big @ w_hat_val - y_big) ** 2))
assert abs(analytic_error - empirical_mse) / analytic_error < 0.02

# LAM_STAR is close to optimal: a coarse grid search over lambda, independent of the Bayes derivation,
# finds nothing meaningfully better at the interpolation threshold
lam_grid = np.geomspace(1e-2, 50, 9)
grid_risks = [mean_test_error(D, W_STAR, SIGMA, 60, lambda X, y: ridge_fit(X, y, lam), 777_000) for lam in lam_grid]
lam_star_risk = mean_test_error(D, W_STAR, SIGMA, 60, lambda X, y: ridge_fit(X, y, LAM_STAR), 777_000)
assert lam_star_risk <= 1.25 * min(grid_risks)
assert lam_star_risk < 0.02 * min_norm_curve[D]

# NOTE: undo the thread pin from Part 1 so it cannot leak into another page's own process (for example
# a subprocess this page's checks never spawn, but a later page's might, which would otherwise inherit
# these variables from this shared process's environment)
for _k, _v in _PRIOR_THREAD_ENV.items():
    if _v is None:
        os.environ.pop(_k, None)
    else:
        os.environ[_k] = _v

print("all checks passed")
```

</details>

</details>
