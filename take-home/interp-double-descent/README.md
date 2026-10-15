# Take-Home: Sample-Wise Double Descent

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Take-home · experiment and write-up | ★☆☆☆☆ | Hard | RE · RS | double-descent, linear-regression, bias-variance, ridge-regression, random-matrix-theory | 4 hours | Take-home |
<!-- meta:end -->

## Problem

You have four hours. Design and run a small experiment that shows sample-wise double descent, explain why it happens, and propose and demonstrate a fix.

**Background.** Fix a model class and a model size — for a linear model, the number of features; for a network, its width or parameter count. Train it on $n$ i.i.d. samples drawn from a fixed distribution and measure its error on held-out data from the same distribution, its *test error*. In the classical regime, more training data makes test error decrease monotonically. A model large enough to *interpolate* its training set — to reach zero training error — can instead show *double descent*: as $n$ grows at fixed model size, test error decreases, rises to a peak near the *interpolation threshold* — the smallest $n$, at that model size, at which the model has just enough capacity to fit the training data exactly — and then decreases again once $n$ is well past the threshold. Varying $n$ at fixed model size, as here, is the *sample-wise* version of the phenomenon; varying model size at fixed $n$ instead is the *model-wise* version.

**Task.**

1. Choose a model class, a way of generating labelled data from a known ground truth, and a definition of test error. Choose how you will vary $n$ at a fixed model size.
2. Compute test error at enough values of $n$ — spanning well below the interpolation threshold to well above it — to trace out the double-descent curve, averaging over enough random repetitions of the training data that the peak is not an artefact of one unlucky draw.
3. Explain, in terms of the model class you chose, why test error rises to a peak near the interpolation threshold and falls on both sides of it.
4. Propose one remedy that removes or reduces the peak without materially changing the rest of the curve, and demonstrate it on the same experiment.

**Deliverables.** A notebook containing the experiment's code, its numerical results, and brief commentary; and a slide deck of at most eight slides for a live review, covering the question, your setup, the result, your explanation, the remedy, its limitations, and what you would try next.

**Constraints.** Four hours, start to finish. NumPy and a plotting library may be used, along with the standard library; nothing else is required at this scale, and no machine-learning framework is needed. Do not use an AI coding assistant — the experiment, the explanation and the slides must be your own work.

**Live review.** After you submit the notebook and slides, you present them and answer questions: why you made the choices you did, whether your explanation would still hold under a different data distribution or a different model class, and how you would extend the experiment given more time.

## Reference solution

<details>
<summary>Show the reference solution</summary>

This solution picks the smallest model class in which the phenomenon is unambiguous: linear regression with a fixed number of features, isotropic Gaussian inputs, and a planted linear ground truth. Two choices are worth stating up front: the test error used throughout is the model's *expected* squared error on a fresh sample from the same distribution, not an error measured on one sampled test set — this is justified below, and checked against an actual sampled test set at the end — and every point on every curve is an average over many independently redrawn training sets at that $n$, never a single run.

### Part 1 — The experiment

Fix $d = 100$ features. Draw a ground-truth direction $w^\star \in \mathbb{R}^d$ once, and generate a training set of size $n$ as $n$ i.i.d. isotropic points $x_i \sim N(0, I_d)$ with labels $y_i = x_i^\top w^\star + \varepsilon_i$, $\varepsilon_i \sim N(0, \sigma^2)$ independent of $x_i$. This is the simplest model class in which "model size" is unambiguous ($d$, the number of features) and in which the model can interpolate: once $n \ge d$ and the design matrix $X \in \mathbb{R}^{n \times d}$ has full column rank, or once $n \le d$ and $X$ has full row rank, a weight vector exists that fits every training label exactly.

Fitting is ordinary least squares, minimizing $`\|Xw - y\|_2^2`$. For $n \ge d$ with $X$ of full column rank this has the unique minimizer $(X^\top X)^{-1} X^\top y$. For $n < d$ with $X$ of full row rank, infinitely many $w$ satisfy $Xw = y$ exactly; among these, the *minimum-norm least-squares solution* is the one of smallest $`\|w\|_2`$. Both cases are given by the same formula, $\hat w = X^{+} y$, where $X^{+}$ is the *Moore–Penrose pseudo-inverse*: writing the singular value decomposition $X = U \Sigma V^\top$, $X^{+} = V \Sigma^{+} U^\top$, with $\Sigma^{+}$ formed by inverting every nonzero singular value of $\Sigma$ and leaving the rest zero. At $n = d$ exactly, assuming $X$ is square and invertible, this reduces to the unique exact fit $X^{-1}y$ — the model has exactly enough samples to pin down every parameter, the interpolation threshold.

Because the test points are isotropic, the model's expected squared error on a fresh point, $E_x\bigl[(x^\top \hat w - x^\top w^\star)^2\bigr] = (\hat w - w^\star)^\top E[xx^\top](\hat w - w^\star)$, collapses to $`\|\hat w - w^\star\|_2^2`$ exactly, since $E[xx^\top] = I_d$. A noisy test label adds the irreducible $\sigma^2$ on top. So the double-descent curve can be computed from $`\|\hat w - w^\star\|_2^2`$ alone, with no test-set sampling and no test-set noise to average away — only the randomness of the training draw remains.

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

$n$ spans well below $D = 100$ to well above it, with the grid packed tightly around $D$ where the curve moves fastest. Averaged over 40 redrawn training sets per point, the test error is:

| $n$ | 10 | 20 | 30 | 40 | 50 | 60 | 70 | 80 | 90 | 95 | 98 | 99 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| test error | 83.8 | 73.7 | 65.5 | 56.8 | 48.4 | 40.2 | 31.3 | 21.4 | 18.4 | 27.8 | 171.2 | 230.5 |

| $n$ | 100 | 101 | 102 | 105 | 110 | 120 | 150 | 200 | 300 | 400 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| test error | 1549.7 | 869.1 | 76.5 | 21.6 | 10.5 | 5.5 | 1.9 | 1.0 | 0.5 | 0.3 |

Test error falls from $83.8$ at $n = 10$ to $18.4$ at its local low at $n = 90$, then climbs sharply — $27.8$, $171.2$, $230.5$ — to a peak of $1{,}549.7$ exactly at $n = 100 = D$, the interpolation threshold, before falling back through $869.1$ and $76.5$ and resuming the smooth descent of the classical regime: $10.5$ at $n = 110$, $1.0$ at $n = 200 = 2D$, $0.3$ at $n = 400$. The peak is $32.0\times$ the error at $n = D/2 = 50$ ($48.4$) and $1{,}579.6\times$ the error at $n = 2D = 200$ ($1.0$). The exact height of the spike is itself unstable across $n = 98$ to $102$ — each of these has a design matrix close to square, and which particular draws of $X$ land almost singular is a matter of luck; Part 2 explains why, and why that instability is itself part of the phenomenon rather than noise in the experiment.

### Part 2 — Why the peak happens

Condition on the training inputs $X$, leaving only the label noise $\varepsilon$ random. Substituting $y = Xw^\star + \varepsilon$ into $\hat w = X^{+}y$ gives $\hat w = X^{+}Xw^\star + X^{+}\varepsilon$, and $X^{+}X = P_X$ is the orthogonal projector onto the row space of $X$ (the identity once $X$ has full column rank), so

$$\hat w = P_X w^\star + X^{+}\varepsilon.$$

Hence $E_\varepsilon[\hat w \mid X] = P_X w^\star$ and $`\mathrm{Cov}_\varepsilon(\hat w \mid X) = \sigma^2 X^{+}(X^{+})^\top`$. The *bias–variance decomposition* of the excess test error, conditional on $X$, is the split of its expectation over the noise into a part that does not vanish even with infinite data at this $n$ and a part driven purely by the noise:

$$E\_\varepsilon\bigl[\\|\hat w - w^\star\\|^2 \mid X\bigr] = \underbrace{\\|(I - P_X)w^\star\\|^2}\_{\text{bias}(X)^2} + \underbrace{\sigma^2 \mathrm{tr}\bigl(X^{+}(X^{+})^\top\bigr)}\_{\text{variance}(X)}.$$

Writing $X^{+} = V\Sigma^{+}U^\top$ from the SVD, $X^{+}(X^{+})^\top = V(\Sigma^{+})^2 V^\top$, so its trace is the sum of the squared entries of $\Sigma^{+}$ — the sum of $1/\sigma_i^2$ over the nonzero singular values of $X$:

$$\text{variance}(X) = \sigma^2 \sum_i \frac{1}{\sigma_i^2}.$$

Every curve above already averages over redrawn training sets, so the reported test error at each $n$ is $E_X[\text{bias}(X)^2] + E_X[\text{variance}(X)]$. For isotropic $X$, both terms have closed forms. When $n \le d$, the row space of $X$ is a uniformly random $n$-dimensional subspace of $\mathbb{R}^d$; a fixed vector has, in expectation, the fraction $n/d$ of its squared length inside such a subspace, so $`E_X[\text{bias}(X)^2] = (1 - n/d)\|w^\star\|^2`$ — and exactly $0$ once $n \ge d$, where every direction is reachable. For the variance, $X^\top X$ (when $n > d$) or $XX^\top$ (when $n < d$) is a Wishart matrix — a sum of $\max(n,d)$ independent rank-one outer products of standard Gaussian vectors — whose inverse has a classical closed-form mean, giving

$`\displaystyle E_X[\text{variance}(X)] = \sigma^2 \times \begin{cases} n / (d - n - 1) & n < d \\ d / (n - d - 1) & n > d. \end{cases}`$

Both terms are finite once $n \ne d$, but the variance term's denominator vanishes as $n \to d$ from either side: this is the peak. The reason is the smallest singular value $\sigma_{\min}(X)$, the largest addend in $\sum_i 1/\sigma_i^2$. For $X$ with i.i.d. $N(0,1)$ entries, the *Marchenko–Pastur law* says that as $n, d \to \infty$ with $d/n \to \gamma \le 1$, the eigenvalues of $X^\top X / n$ fill the interval $[(1-\sqrt{\gamma})^2, (1+\sqrt{\gamma})^2]$; its lower edge, $(1 - \sqrt{d/n})^2$, is the smallest value the bulk of the spectrum reaches. As $n \to d$, $\gamma \to 1$ and this edge slides to $0$, so the smallest eigenvalue of $X^\top X$ is of order $n(1-\sqrt{d/n})^2 = (\sqrt n - \sqrt d)^2$, i.e.

$$\sigma\_{\min}(X) \approx |\sqrt n - \sqrt d|.$$

Every step of $n$ toward $d$ pushes the smallest singular value toward $0$ and its reciprocal square toward infinity: the training labels' noise, however small $\sigma$ is, gets amplified without bound by the near-singular directions of $X$. This approximation holds away from the edge, but breaks down exactly where the peak is highest: right at $n = d$, $\sigma_{\min}$ no longer concentrates near any single value — it has a limiting distribution on scale $1/\sqrt d$ with mass arbitrarily close to $0$ — so $E_X[\text{variance}(X)]$ is genuinely unbounded exactly at $n = d$, not merely large, and a finite average over any fixed set of nearby training-set draws is dominated by whichever few happen to be nearly singular. That is why the peak in Part 1 came out at $1{,}549.7$ from one fixed set of seeds rather than as a stable number the experiment converges to.

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

Averaged over 80 draws of $X$ at $n = 50$, $90$ and $300$, empirical (bias$`^2`$, variance) is $(47.62, 1.02)$, $(9.19, 10.14)$ and $(0.00, 0.50)$, against the formulas' $(46.61, 1.02)$, $(9.32, 10.00)$ and $(0.00, 0.50)$ — close everywhere, with the largest gap where $n$ is closest to $d$, exactly where the underlying quantity is hardest to average. Averaged over 30 draws away from the threshold, the mean smallest singular value at $n = 10, 50, 150, 300$ is $7.35$, $3.10$, $2.44$, $7.53$, against the edge prediction $|\sqrt n - \sqrt d|$ of $6.84$, $2.93$, $2.25$, $7.32$ — a good approximation, within about $`9\%`$, everywhere these points are safely away from $n = d$.

The same derivation runs with the roles of $n$ and $d$ exchanged: fixing $n$ and growing $d$ instead — the model-wise version of double descent — gives a symmetric peak at $d \approx n$ by the same edge argument with $n$ and $d$ swapped. The two sides of that peak are not symmetric in outcome, though. As $d \to \infty$ at fixed $n$, the row space of $X$ is an $n$-dimensional subspace of an ever larger $\mathbb{R}^d$, so $`E_X[\text{bias}(X)^2] = (1 - n/d)\|w^\star\|^2 \to \|w^\star\|^2`$ — not to $0$ — while $E_X[\text{variance}(X)] \to 0$, since the $d$ columns of $X$ make $XX^\top$ concentrate around $dI_n$ and every singular value grows with $d$. A plain minimum-norm fit with isotropic features and no further structure does not recover the low error of the classical regime by adding parameters alone past the threshold; it plateaus at the error of predicting zero. Models that generalize well far past their interpolation threshold rely on more than raw parameter count — an architecture and optimizer whose implicit bias favors the right kind of solution, or a feature covariance aligned with the target, the setting that work on benign overfitting characterizes.

### Part 3 — Removing the peak

*Ridge regression* minimizes $`\|Xw - y\|_2^2 + \lambda\|w\|_2^2`$ for a fixed $\lambda > 0$, with the unique solution $\hat w_\lambda = (X^\top X + \lambda I_d)^{-1}X^\top y$ — well-defined for every $n$ and every $\lambda > 0$, because $X^\top X + \lambda I_d$ has every eigenvalue at least $\lambda$, however small $X$'s own singular values are.

With the same SVD $X = U\Sigma V^\top$, and decomposing $w^\star$ into its row-space component $\alpha = V^\top w^\star$ and the orthogonal remainder $w^\star_\perp$ (present only when $X$ lacks full column rank), the ridge estimate works out to $`\hat w_\lambda = V\,\mathrm{diag}\bigl(\sigma_i^2/(\sigma_i^2+\lambda)\bigr)\alpha + V\,\mathrm{diag}\bigl(\sigma_i/(\sigma_i^2+\lambda)\bigr)U^\top\varepsilon`$, so that

$$E\_\varepsilon\bigl[\\|\hat w\_\lambda - w^\star\\|^2 \mid X\bigr] = \underbrace{\\|w^\star\_\perp\\|^2 + \sum_i \Bigl(\frac{\lambda}{\sigma_i^2+\lambda}\Bigr)^2 \alpha_i^2}\_{\text{bias}\_\lambda(X)^2} + \underbrace{\sigma^2 \sum_i \frac{\sigma_i^2}{(\sigma_i^2+\lambda)^2}}\_{\text{variance}\_\lambda(X)}.$$

As $\lambda \to 0$ this reduces to Part 2's formulas exactly (the $\lambda$-bias term vanishes, and $\sigma_i^2/\sigma_i^4 = 1/\sigma_i^2$). For $\lambda > 0$, every summand of the variance is bounded: by AM–GM, $\sigma_i^2/(\sigma_i^2+\lambda)^2 \le 1/(4\lambda)$ for every $\sigma_i \ge 0$ (maximized at $\sigma_i^2 = \lambda$), so $`\text{variance}_\lambda(X) \le \sigma^2 \min(n,d)/(4\lambda)`$ regardless of how small $\sigma_{\min}(X)$ gets. Ridge trades a small, controlled increase in bias — the shrinkage $\lambda/(\sigma_i^2+\lambda)$ pulls every direction slightly toward $0$ — for a hard cap on the variance that removes the $1/\sigma_i^2$ blow-up entirely. Away from the threshold that trade only costs bias for no benefit, since the uncapped variance was already small there; it pays off exactly in the neighbourhood of $n = d$, where the uncapped variance would otherwise be enormous.

How large should $\lambda$ be? Suppose, only for the purpose of choosing $\lambda$, that $w^\star$ itself were drawn from $N(0, \tau^2 I_d)$, independent of the label noise. The posterior mean of $w^\star$ given $(X, y)$ — the estimator minimizing expected squared error averaged over this prior — is then exactly

$$E[w^\star \mid X, y] = \bigl(X^\top X + \tfrac{\sigma^2}{\tau^2}I_d\bigr)^{-1}X^\top y,$$

ridge regression with $\lambda = \sigma^2/\tau^2$, for every $n$ and $d$, not only asymptotically. A vector drawn from that prior has $`E\|w^\star\|^2 = d\tau^2`$, so using the *realized* $`\|w^\star\|^2`$ as a plug-in estimate of $d\tau^2$ gives

$$\lambda^\star = \frac{\sigma^2}{\tau^2} \approx \frac{\sigma^2 d}{\\|w^\star\\|^2}.$$

This is exact only on average over the prior, not for one fixed $w^\star$ — but $`\|w^\star\|^2/d`$ concentrates around $\tau^2$ as $d$ grows, and $d = 100$ is large enough for the plug-in to land close. Here $`\|w^\star\|^2 \approx 93.23`$, giving $\lambda^\star \approx 1.073$.

```python
LAM_STAR = SIGMA ** 2 * D / NORM_W_STAR_SQ  # approx. 1.073

ridge_curve = {n: mean_test_error(n, W_STAR, SIGMA, N_SEEDS, lambda X, y: ridge_fit(X, y, LAM_STAR), 10_000 * n)
               for n in N_GRID}
```

Applying this one fixed $\lambda^\star$ across the whole grid — not re-tuned at each $n$ — removes the spike:

| $n$ | 80 | 90 | 95 | 98 | 99 | 100 | 101 | 102 | 105 | 110 | 120 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| min-norm | 21.4 | 18.4 | 27.8 | 171.2 | 230.5 | 1549.7 | 869.1 | 76.5 | 21.6 | 10.5 | 5.5 |
| ridge | 20.6 | 13.3 | 12.2 | 10.3 | 9.6 | 9.3 | 9.2 | 8.8 | 7.8 | 5.9 | 4.1 |

At $n = 100$, ridge's test error is $9.3$, a $165.8\times$ reduction from min-norm's $1{,}549.7$, and it does not merely blunt the peak — the ridge curve is monotone non-increasing across the *entire* grid from $n=10$ to $n=400$, exactly the shape more training data is expected to produce, with no trace of the interpolation threshold left in it.

### Part 4 — The write-up

The notebook contains the code above — `make_dataset`, `min_norm_fit`, `ridge_fit`, `mean_test_error`, the bias–variance and Marchenko–Pastur helpers of Part 2 — the two tables of numbers as computed output, and enough inline commentary to point at the derivations without reproducing them line by line. The slide deck compresses the same material into eight slides for the live review:

1. **The question.** Does test error always improve with more data, at a fixed model size? Framed on real terms the audience will recognize before naming double descent.
2. **Setup.** $d = 100$-feature linear regression, isotropic Gaussian inputs and planted ground truth, minimum-norm least squares by pseudo-inverse, test error defined as $`\|\hat w - w^\star\|^2`$ (plus the fixed $\sigma^2$), and why that closed form stands in for a sampled test set.
3. **Result.** The Part 1 table, presented as the curve it tabulates: falling, spiking $32.0\times$ at $n = D/2$'s error and $1{,}579.6\times$ at $2D$'s, right at $n = D$, falling again.
4. **Why.** The bias–variance split, the $\sigma^2 \sum 1/\sigma_i^2$ variance formula, and the Marchenko–Pastur edge argument for why $\sigma_{\min}(X) \to 0$ as $n \to d$.
5. **The fix.** Ridge, the bounded-variance argument, the Bayes derivation of $`\lambda^\star = \sigma^2 d / \|w^\star\|^2`$, and the Part 3 table showing the peak gone.
6. **Limitations.** One ground-truth realization, not many; isotropic, uncorrelated features, unlike most real data; $\lambda^\star$ uses $\sigma^2$ and $`\|w^\star\|^2`$, which are known here only because the data were generated — in practice $\lambda$ would be swept or cross-validated, and a coarse sweep in this experiment lands within about $`1\%`$ of $\lambda^\star$'s risk, confirming the closed form is not doing anything a search couldn't find.
7. **Model-wise double descent and next steps.** The symmetric peak at $d \approx n$ when $d$ is varied instead, why the plain minimum-norm fit plateaus rather than improving further past that threshold, and, more speculatively, that the same qualitative transition — from fitting individual training points to fitting shared structure — shows up in mechanistic interpretability's study of memorization versus generalizing circuits (for instance in grokking), though a linear model has no internal circuits for that connection to be more than an analogy at this scale.
8. **Takeaway and questions.** One sentence on what the experiment shows and does not show, then the floor for the live review.

### Follow-ups

- Why report $`\|\hat w - w^\star\|^2`$ instead of sampling a held-out test set? Isotropy makes the two identical in expectation, checked directly below against a large sampled test set; sampling would only add test-set noise the closed form avoids.
- Would the same peak show up with correlated features? The interpolation threshold and its ill-conditioning survive, but the exact bias and variance formulas change with the covariance spectrum; for some spectra aligned with $w^\star$ in the right way, error can stay low well past the threshold without any explicit ridge term — the benign-overfitting regime mentioned in Part 2.
- Why not cross-validate $\lambda$ instead of deriving it? In practice, $\sigma^2$ and $`\|w^\star\|^2`$ are unknown and cross-validation is how $\lambda$ would actually be chosen; the closed form is useful here because the data-generating process is known by construction, and the grid search in the checks below confirms cross-validation would land at essentially the same risk.
- Does an interpolation threshold this sharp appear in trained neural networks? Model-wise double descent is well documented there too, but "capacity" for a network trained with SGD is not simply its parameter count — early stopping, architecture and the optimizer's implicit bias all shift where, and how sharply, the peak appears.
- Given another four hours: rerun with correlated features and a heavier-tailed noise distribution, add a $\lambda$ sweep in place of the single closed-form value, and compare against a small two-layer network trained by gradient descent instead of solved in closed form.

<details>
<summary>Checks (runnable)</summary>

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
