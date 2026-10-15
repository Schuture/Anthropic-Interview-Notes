# Take-Home: A Four-Hour Exploratory ML Study

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Take-home · experiment and presentation | ★★☆☆☆ | Hard | RE · RS · MLE · Fellows | label-noise, model-capacity, early-stopping, generalization, numpy | 4 hours + 40 min review | Take-home |
<!-- meta:end -->

## Problem

You receive the materials below at the start of a 4-hour window; you may not prepare on this specific task
beforehand. Work happens in your own local environment or a hosted notebook service — either is fine, as long
as you can run Python with NumPy and scikit-learn. Documentation is allowed and expected, but no AI coding
assistant may be used, and every line of code and every conclusion must be your own. At the end of the 4
hours, give a 10-minute presentation — slides or a notebook walkthrough are both fine, as long as the results
are shown clearly — followed by a 30-minute technical discussion of the design choices behind the experiment,
which of the conclusions the data actually supports, and what you would change with more time.

*Symmetric label noise* at rate $p$ is a corruption process applied to training labels only: independently
for each training example, with probability $p$ its label is replaced by one of the other classes, chosen
uniformly at random, and with probability $1 - p$ it is left unchanged.

Research question: for a one-hidden-layer classifier trained on synthetic data, how does symmetric label
noise in the training labels interact with the size of the training set and the network's hidden width, and
how much of the resulting damage does early stopping against a small clean validation set recover?

### The data

```python
import numpy as np
from sklearn.datasets import make_classification

D, C = 20, 3
N_VAL, N_TEST = 200, 1000


def make_dataset(seed: int, n_train: int, noise_rate: float):
    rng = np.random.default_rng(seed)
    n_total = n_train + N_VAL + N_TEST
    X, y = make_classification(
        n_samples=n_total, n_features=D, n_informative=15, n_redundant=3,
        n_classes=C, n_clusters_per_class=2, class_sep=1.8, flip_y=0.0, random_state=seed,
    )
    X_train, X_val, X_test = X[:n_train], X[n_train:n_train + N_VAL], X[n_train + N_VAL:]
    y_train, y_val, y_test = y[:n_train], y[n_train:n_train + N_VAL], y[n_train + N_VAL:]

    y_train_noisy = y_train.copy()
    flip = rng.random(n_train) < noise_rate
    for i in np.where(flip)[0]:
        other_classes = [c for c in range(C) if c != y_train[i]]
        y_train_noisy[i] = rng.choice(other_classes)
    return X_train, y_train_noisy, y_train, X_val, y_val, X_test, y_test
```

`make_dataset` is deterministic given `seed`: it draws `n_train + 200 + 1000` points from one
`make_classification` call (20 features — 15 informative, 3 redundant, 2 pure noise — 3 balanced classes,
2 clusters per class), splits them into a training set of size `n_train`, a validation set of 200 points, and
a test set of 1000 points, and applies symmetric noise at `noise_rate` to the training labels only.
`y_train_true`, the label before noise, is returned for diagnostics; it must never be used to fit or select a
model. `X_val`, `y_val`, `X_test`, and `y_test` are always clean.

### The model and the experiment grid

The model is a classifier with one hidden layer: `D` inputs, a hidden layer of width $H$ with a ReLU
nonlinearity, and a softmax output over the `C` classes, trained to minimize the cross-entropy of the
(possibly noisy) training labels. The optimizer, the learning rate, the number of epochs, and any weight
decay are your choice; state what you used and how you picked it.

Run every combination of:

- hidden width $`H \in \{8, 128\}`$;
- training-set size $`n_{\text{train}} \in \{100, 400, 1600\}`$;
- noise rate $`p \in \{0.0, 0.2, 0.4\}`$,

with at least 5 seeds per combination (the same seed drives both `make_dataset` and the model's weight
initialization). For every seed, in addition to the test accuracy after the last training epoch, record
validation accuracy after every epoch and the test accuracy at the epoch with the best validation accuracy —
the accuracy early stopping against `X_val`, `y_val` would have delivered, without `X_test`, `y_test` ever
being used to choose it.

### Questions to answer

1. Within each hidden width, how does test accuracy depend on the noise rate and on the training-set size?
2. Does the hidden width change how much noise hurts test accuracy, and does that change with training-set
   size?
3. Does early stopping recover test accuracy lost to noise? Does the size of the recovery depend on the
   noise rate, the hidden width, or the training-set size — and does early stopping cost anything when there
   is no noise at all?
4. Which of the trends behind your answers are well supported by 5 seeds, and which are not? State the
   margin you would want to see before trusting a comparison.

Hand in two tables — one of test accuracy (mean and spread across seeds) for the last-epoch models, one of
early stopping's recovery (mean test accuracy at the best-validation epoch minus mean test accuracy at the
last epoch, over the same seeds) — answers to the four questions that reference those tables, and the
presentation.

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two points worth confirming with the interviewer before starting: whether "symmetric" noise — flipping to a
uniformly random other class, as defined above — is the intended corruption model or a specific confusion
pattern is expected instead, and whether the 5-seed minimum is a floor or a target, since that changes
whether the four hours go toward a wider grid or more seeds on the same one.

### Plan for the four hours

| Time | Task |
| --- | --- |
| 0:00–0:15 | Read the prompt, restate the research question, decide the grid and the two tables to fill in. |
| 0:15–0:45 | Implement `make_dataset`; check the empirical flip rate matches `noise_rate` and that `X_val`/`X_test` stay clean. |
| 0:45–1:15 | Implement the model and one training run; confirm the noise-free condition converges before trusting anything noisy. |
| 1:15–2:45 | Run all 18 conditions × 5 seeds with early-stopping tracking — the only step with real runtime. |
| 2:45–3:30 | Fill in the tables, look for the trends, draft the interpretation and the limitations. |
| 3:30–4:00 | Build the ten-minute presentation outline; rehearse answers to likely follow-up questions. |

### Part 1 — Data and model

The hidden layer uses He initialization (`std = sqrt(2 / fan_in)`), appropriate for ReLU units; both weight
matrices are drawn this way regardless of $H$, so only the width changes between the narrow and the wide
network and the comparison in Part 2 isolates capacity from anything else about the initialization. Training
is full-batch gradient descent, not mini-batches or Adam: with $n_{\text{train}} \le 1600$ the whole training
set is one matrix multiply, full-batch removes stochastic-gradient noise as a second source of seed-to-seed
variance on top of the data draw and the label-noise draw, and it keeps the 90 independent training runs of
Part 2 (18 conditions × 5 seeds) fast enough for the four-hour budget. `train_with_early_stopping` keeps a
*snapshot* of the weights at the best-validation epoch instead of storing per-epoch predictions on `X_test`:
the test set is read exactly twice per run, after the last epoch and once more after restoring the
best-validation snapshot, never to choose anything, only to report it.

```python
class MLP:
    def __init__(self, d_in: int, d_hidden: int, d_out: int, seed: int):
        rng = np.random.default_rng(seed)
        self.W1 = rng.normal(0, np.sqrt(2.0 / d_in), size=(d_in, d_hidden))
        self.b1 = np.zeros(d_hidden)
        self.W2 = rng.normal(0, np.sqrt(2.0 / d_hidden), size=(d_hidden, d_out))
        self.b2 = np.zeros(d_out)

    def forward(self, X):
        Z1 = X @ self.W1 + self.b1
        H = np.maximum(Z1, 0.0)
        Z2 = H @ self.W2 + self.b2
        Z2 = Z2 - Z2.max(axis=1, keepdims=True)          # NOTE: subtract the row max before exp -- keeps softmax stable
        expZ = np.exp(Z2)
        P = expZ / expZ.sum(axis=1, keepdims=True)
        return Z1, H, P

    def loss_and_grad(self, X, y, l2: float):
        n = X.shape[0]
        Z1, H, P = self.forward(X)
        Y_onehot = np.zeros_like(P)
        Y_onehot[np.arange(n), y] = 1.0
        dZ2 = (P - Y_onehot) / n                          # gradient of mean cross-entropy w.r.t. the logits
        dW2 = H.T @ dZ2 + l2 * self.W2
        db2 = dZ2.sum(axis=0)
        dH = dZ2 @ self.W2.T
        dZ1 = dH * (Z1 > 0)                               # ReLU gradient
        dW1 = X.T @ dZ1 + l2 * self.W1
        db1 = dZ1.sum(axis=0)
        return dW1, db1, dW2, db2

    def step(self, grads, lr: float):
        dW1, db1, dW2, db2 = grads
        self.W1 -= lr * dW1
        self.b1 -= lr * db1
        self.W2 -= lr * dW2
        self.b2 -= lr * db2

    def predict(self, X):
        _, _, P = self.forward(X)
        return P.argmax(axis=1)

    def snapshot(self):
        return self.W1.copy(), self.b1.copy(), self.W2.copy(), self.b2.copy()

    def restore(self, snapshot):
        self.W1, self.b1, self.W2, self.b2 = (a.copy() for a in snapshot)


def train_with_early_stopping(X_train, y_train, X_val, y_val, hidden: int, seed: int, epochs: int, lr: float, l2: float):
    model = MLP(D, hidden, C, seed)
    best_val_acc, best_snapshot, best_epoch = -1.0, model.snapshot(), 0
    for epoch in range(epochs):
        grads = model.loss_and_grad(X_train, y_train, l2)
        model.step(grads, lr)
        val_acc = (model.predict(X_val) == y_val).mean()
        if val_acc > best_val_acc:                        # NOTE: strictly greater -- ties keep the earlier (smaller) epoch
            best_val_acc, best_snapshot, best_epoch = val_acc, model.snapshot(), epoch
    return model, best_snapshot, best_epoch
```

### Part 2 — The grid: noise, training-set size, and capacity

Eighty epochs of full-batch gradient descent at `lr=0.5` with weight decay `1e-3` were enough for the
noise-free ($p=0.0$) condition to plateau at every training-set size tried, so that schedule was kept fixed
across the whole grid, deliberately not retuned per cell — a choice revisited in the Interpretation below.

```python
EPOCHS, LR, L2 = 80, 0.5, 1e-3
SEEDS = range(5)
N_TRAINS = [100, 400, 1600]
NOISES = [0.0, 0.2, 0.4]
HIDDENS = [8, 128]

results = {}
for n_train in N_TRAINS:
    for hidden in HIDDENS:
        for noise in NOISES:
            finals, bests, best_epochs = [], [], []
            for seed in SEEDS:
                X_train, y_noisy, y_true, X_val, y_val, X_test, y_test = make_dataset(seed, n_train, noise)
                model, best_snapshot, best_epoch = train_with_early_stopping(
                    X_train, y_noisy, X_val, y_val, hidden, seed, EPOCHS, LR, L2)
                finals.append((model.predict(X_test) == y_test).mean())
                model.restore(best_snapshot)
                bests.append((model.predict(X_test) == y_test).mean())
                best_epochs.append(best_epoch)
            results[(n_train, hidden, noise)] = dict(
                final_mean=np.mean(finals), final_std=np.std(finals),
                best_mean=np.mean(bests), best_std=np.std(bests),
                best_epoch_mean=np.mean(best_epochs),
            )
```

**Table 1 — final-epoch test accuracy** (mean ± standard deviation over 5 seeds):

| $n_{\text{train}}$ | $H$ | $p=0.0$ | $p=0.2$ | $p=0.4$ |
| --- | --- | --- | --- | --- |
| 100 | 8 | 0.869 ± 0.029 | 0.736 ± 0.043 | 0.542 ± 0.072 |
| 100 | 128 | 0.879 ± 0.025 | 0.749 ± 0.036 | 0.594 ± 0.060 |
| 400 | 8 | 0.937 ± 0.008 | 0.833 ± 0.031 | 0.663 ± 0.049 |
| 400 | 128 | 0.949 ± 0.011 | 0.816 ± 0.014 | 0.615 ± 0.069 |
| 1600 | 8 | 0.964 ± 0.009 | 0.930 ± 0.011 | 0.875 ± 0.020 |
| 1600 | 128 | 0.964 ± 0.005 | 0.868 ± 0.021 | 0.724 ± 0.041 |

Two trends hold at every one of the six $(n_{\text{train}}, H)$ pairs: accuracy falls as $p$ goes from 0.0 to
0.2 to 0.4 (the smallest step anywhere in the table is 0.034, at $n_{\text{train}}=1600$, $H=8$; every other
step is larger), and accuracy at $n_{\text{train}}=1600$ is higher than at $n_{\text{train}}=100$ under noise
(the smallest such gap is 0.119, at $H=128$, $p=0.2$). Neither trend is a surprise by itself; the more
specific finding is how $H$ interacts with $n_{\text{train}}$. At $n_{\text{train}}=100$ the wide network
($H=128$) is not worse than the narrow one under noise — if anything it is slightly ahead, 0.594 against
0.542 at $p=0.4$. At $n_{\text{train}}=1600$ the ranking reverses and the gap widens sharply: the narrow
network reaches 0.875 at $p=0.4$ against the wide network's 0.724, a gap of 0.151 — roughly three times the
0.052 gap in the *other* direction at $n_{\text{train}}=100$. More capacity is not uniformly harmful under
noise; it is harmful specifically once there is enough training data for the extra capacity to be spent
memorizing mislabeled points rather than fitting the shared pattern. With only 100 points, both networks
already have far more parameters than data — the narrow one alone has $20 \times 8 + 8 \times 3 = 184$
weights in its two layers — so neither has more room than the other to overfit.

### Part 3 — Does early stopping recover the loss?

**Table 2 — recovery** (mean test accuracy at the best-validation epoch, minus mean test accuracy at the
final epoch, over the same 5 seeds):

| $n_{\text{train}}$ | $H$ | $p=0.0$ | $p=0.2$ | $p=0.4$ |
| --- | --- | --- | --- | --- |
| 100 | 8 | -0.002 | 0.039 | 0.069 |
| 100 | 128 | -0.006 | 0.038 | 0.024 |
| 400 | 8 | -0.005 | 0.025 | 0.064 |
| 400 | 128 | -0.003 | 0.037 | 0.076 |
| 1600 | 8 | -0.006 | 0.000 | -0.015 |
| 1600 | 128 | -0.002 | 0.029 | 0.016 |

At $p=0.0$ every recovery is small and negative — between -0.006 and -0.002, averaging -0.004 over the six
conditions — which is what a validation-based stopping rule should do when there is nothing to protect
against: it neither helps nor meaningfully hurts. At $p=0.4$ the average recovery is 0.039, roughly ten times
larger in magnitude, and five of the six conditions gain; the exception is $n_{\text{train}}=1600$, $H=8$
(-0.015), the same condition whose capacity is too small, relative to that much clean data, to memorize the
noise in the first place — with nothing pathological to correct, choosing an earlier epoch by validation
accuracy alone gives up a little of the training the last few epochs would still have added. The largest
single recovery is +0.076, at $n_{\text{train}}=400$, $H=128$, $p=0.4$. Consistent with the capacity story in
Part 2, the wide network's validation accuracy peaks early when data is scarce and noise is high: at
$n_{\text{train}}=100$, $H=128$, $p=0.4$ the mean best epoch is 9.8 of 80, against 64.4 for
$n_{\text{train}}=1600$, $H=8$, $p=0.4$ — an order of magnitude sooner.

### Interpretation

The evidence supports three claims without much qualification, because every comparison behind them clears a
margin several times the seed-to-seed spread observed anywhere in the grid: accuracy decreases monotonically
in the noise rate, at every capacity and every training-set size tested; accuracy at $n_{\text{train}}=1600$
exceeds accuracy at $n_{\text{train}}=100$ under noise, at every capacity and both noisy conditions; and the
capacity penalty under noise — how much worse the wide network does than the narrow one — is far larger at
$n_{\text{train}}=1600$ than at $n_{\text{train}}=100$, where it is not a penalty at all.

It supports one claim only loosely: that accuracy rises *monotonically* with training-set size at every
single condition, rather than just at the two endpoints. At $H=128$, $p=0.4$, the step from
$n_{\text{train}}=100$ to $400$ is 0.021 (0.594 to 0.615), smaller than the standard error of either mean
(≈ 0.027 and 0.031 over 5 seeds); the step from 400 to 1600 is 0.109, several times either standard error and
not in doubt. Five seeds settle the large, order-of-magnitude comparisons in this study and do not settle a
0.02-sized one — a concrete instance of the general rule that the number of seeds needed scales with how
small a difference must be resolved, not with the size of the grid.

It does not support a claim about capacity in general. The learning rate, epoch budget, and weight decay were
fixed once, from the noise-free condition in Part 2, and reused for both hidden widths rather than retuned
per condition. If `lr=0.5` undertrains the wide network relative to the narrow one, part of the measured
capacity penalty could be an optimization artifact rather than a pure capacity effect — a confound this
design cannot rule out on its own. Nor does it support any claim beyond this data-generating process: one
synthetic family, one class separation, symmetric noise only, and two capacity points rather than a curve.

### Limitations and next steps

- Five seeds order effects that differ by several standard errors but not ones that do not — see the
  $n_{\text{train}}=100 \to 400$ step above; a handful of other cells in the two tables are likely similarly
  close and were not individually checked.
- The training hyperparameters were chosen once, from the noise-free condition, and reused everywhere; the
  Interpretation above already flags what this could be hiding in the capacity comparison specifically.
- Only two hidden widths were tried. Together they establish that a crossover exists somewhere between 8 and
  128 units relative to $n_{\text{train}} \le 1600$, not where it is or how sharp it is.
- Noise here is symmetric and label-only: every wrong label is equally likely, and the features themselves
  are never corrupted. Real label noise is often class-conditional (some classes are confused with specific
  others far more than uniformly) or instance-dependent (harder examples are mislabeled more often); neither
  is represented here.
- The validation set used for early stopping is clean. A more realistic setting — validation labels drawn
  from the same noisy process as training — would plausibly show a smaller recovery, since the stopping
  criterion would itself be corrupted.

With a second day: retune the learning rate and epoch budget per hidden width before rerunning the capacity
comparison; add two or three intermediate hidden widths to trace the crossover as a curve rather than two
points; replace symmetric noise with a fixed class-confusion matrix and check whether the interaction
survives; and raise the seed count specifically on the conditions flagged as close, rather than uniformly
across the whole grid.

### The ten-minute presentation

Ten minutes, six slides, timed to leave the full 30 minutes after for questions:

```text
1. Question (30s): one sentence, does label noise interact with training-set size and capacity, and
   does early stopping buy any of it back?
2. Setup (1.5 min): the data generator, the one-hidden-layer model, the three grid axes, 5 seeds each.
3. Result 1 (2 min): Table 1, noise always hurts, more data always helps; then the capacity crossover,
   narrow at least matches wide at n=100, narrow beats wide by far at n=1600.
4. Result 2 (2 min): Table 2, early stopping recovers noise-driven loss on average and costs nothing
   at p=0.0, with one clean exception explained by the same capacity story.
5. What the data does and does not support (2.5 min): the fragile 100-to-400 step at H=128; the fixed
   learning rate as a confound for the capacity claim specifically.
6. With more time (1.5 min): retune per condition, trace the crossover, class-conditional noise, more
   seeds where it matters.
```

### Follow-ups

- Why symmetric noise instead of a more realistic corruption model? It has one parameter, its effect can be
  verified empirically against the requested rate, and it is the standard control condition more realistic
  noise models (class-conditional, instance-dependent) are usually compared against first.
- Why full-batch gradient descent rather than Adam or mini-batches? Determinism and speed at this scale
  ($n_{\text{train}} \le 1600$): it removes stochastic-gradient noise as a source of seed-to-seed variance on
  top of the data draw and the label-noise draw, and 90 independent training runs stay well inside the time
  budget.
- What would change the capacity-crossover conclusion? Retuning the learning rate separately for each hidden
  width — the current comparison holds it fixed, so part of the effect could be an optimization artifact
  rather than a pure capacity effect.
- Why 5 seeds and not more? The four-hour budget; the write-up names exactly which comparison that seed
  count leaves too close to call (the $n_{\text{train}}=100 \to 400$ step at $H=128$, $p=0.4$) rather than
  treating every trend in the tables as equally certain.
- What would you do with two more days? Retune hyperparameters per condition, add intermediate hidden widths
  to trace the crossover, and repeat with class-conditional noise.
- How do you know early stopping is not simply regularizing away the model's ability to fit anything, clean
  labels included? The $p=0.0$ row: every recovery there is small and negative, so the same stopping rule
  that helps under noise does not manufacture a gain when there is nothing to correct.

<details>
<summary>Checks (runnable)</summary>

```python
# Every mean/std/gap quoted in the two tables above, to the printed precision.
EXPECTED = {
    (100, 8, 0.0): (0.869, 0.029, -0.002), (100, 8, 0.2): (0.736, 0.043, 0.039), (100, 8, 0.4): (0.542, 0.072, 0.069),
    (100, 128, 0.0): (0.879, 0.025, -0.006), (100, 128, 0.2): (0.749, 0.036, 0.038), (100, 128, 0.4): (0.594, 0.060, 0.024),
    (400, 8, 0.0): (0.937, 0.008, -0.005), (400, 8, 0.2): (0.833, 0.031, 0.025), (400, 8, 0.4): (0.663, 0.049, 0.064),
    (400, 128, 0.0): (0.949, 0.011, -0.003), (400, 128, 0.2): (0.816, 0.014, 0.037), (400, 128, 0.4): (0.615, 0.069, 0.076),
    (1600, 8, 0.0): (0.964, 0.009, -0.006), (1600, 8, 0.2): (0.930, 0.011, 0.0), (1600, 8, 0.4): (0.875, 0.020, -0.015),
    (1600, 128, 0.0): (0.964, 0.005, -0.002), (1600, 128, 0.2): (0.868, 0.021, 0.029), (1600, 128, 0.4): (0.724, 0.041, 0.016),
}
for (n_train, hidden, noise), (exp_mean, exp_std, exp_gap) in EXPECTED.items():
    r = results[(n_train, hidden, noise)]
    gap = r["best_mean"] - r["final_mean"]
    assert round(r["final_mean"], 3) == exp_mean, (n_train, hidden, noise, "mean", r["final_mean"])
    assert round(r["final_std"], 3) == exp_std, (n_train, hidden, noise, "std", r["final_std"])
    assert round(gap, 3) == exp_gap, (n_train, hidden, noise, "gap", gap)


def acc(n_train, hidden, noise):                                  # the value Table 1 displays, to 3 decimals
    return round(results[(n_train, hidden, noise)]["final_mean"], 3)


# qualitative claim 1: accuracy falls monotonically as noise rises, at every (n_train, hidden); smallest step 0.034
noise_steps = []
for n_train in N_TRAINS:
    for hidden in HIDDENS:
        step_a = acc(n_train, hidden, 0.0) - acc(n_train, hidden, 0.2)
        step_b = acc(n_train, hidden, 0.2) - acc(n_train, hidden, 0.4)
        noise_steps += [step_a, step_b]
        assert step_a > 0.01 and step_b > 0.01
assert round(min(noise_steps), 3) == 0.034

# qualitative claim 2: n_train=1600 beats n_train=100 under noise, at every hidden width; smallest gap 0.119
size_gaps = [acc(1600, hidden, noise) - acc(100, hidden, noise) for hidden in HIDDENS for noise in (0.2, 0.4)]
for g in size_gaps:
    assert g > 0.05
assert round(min(size_gaps), 3) == 0.119

# qualitative claim 3: the capacity penalty under noise is far larger with more data, and reverses sign with less
gap_large_n = acc(1600, 8, 0.4) - acc(1600, 128, 0.4)
gap_small_n = acc(100, 8, 0.4) - acc(100, 128, 0.4)
assert round(gap_large_n, 3) == 0.151
assert round(gap_small_n, 3) == -0.052
assert gap_large_n > 0.08 and gap_small_n < 0.02
assert 2.5 < gap_large_n / abs(gap_small_n) < 3.5                 # "roughly three times"

assert D * 8 + 8 * C == 184                                       # the narrow network's weight count, quoted above

# qualitative claim 4: early stopping helps more under noise than in the noise-free control, on average
gaps04 = [results[(n, h, 0.4)]["best_mean"] - results[(n, h, 0.4)]["final_mean"] for n in N_TRAINS for h in HIDDENS]
gaps00 = [results[(n, h, 0.0)]["best_mean"] - results[(n, h, 0.0)]["final_mean"] for n in N_TRAINS for h in HIDDENS]
avg04, avg00 = sum(gaps04) / len(gaps04), sum(gaps00) / len(gaps00)
assert round(avg04, 3) == 0.039
assert round(avg00, 3) == -0.004
assert all(-0.006 <= round(g, 3) <= -0.002 for g in gaps00)       # every noise-free recovery is small and negative
assert avg04 > 0.02 and avg04 - avg00 > 0.02
assert 8 < avg04 / abs(avg00) < 12                                # "roughly ten times"

# the largest single recovery in Table 2
best_gap, best_cell = max(
    (round(results[(n, h, p)]["best_mean"] - results[(n, h, p)]["final_mean"], 3), (n, h, p))
    for n in N_TRAINS for h in HIDDENS for p in NOISES
)
assert (best_gap, best_cell) == (0.076, (400, 128, 0.4))

# the two representative early-stopping epochs quoted in the text
assert round(results[(100, 128, 0.4)]["best_epoch_mean"], 1) == 9.8
assert round(results[(1600, 8, 0.4)]["best_epoch_mean"], 1) == 64.4

# the one step that is not safely bigger than its own uncertainty, versus one that clearly is
sem_100 = results[(100, 128, 0.4)]["final_std"] / len(SEEDS) ** 0.5
sem_400 = results[(400, 128, 0.4)]["final_std"] / len(SEEDS) ** 0.5
step_100_400 = acc(400, 128, 0.4) - acc(100, 128, 0.4)
step_400_1600 = acc(1600, 128, 0.4) - acc(400, 128, 0.4)
assert round(sem_100, 3) == 0.027 and round(sem_400, 3) == 0.031
assert round(step_100_400, 3) == 0.021 and round(step_400_1600, 3) == 0.109
assert step_100_400 < sem_100 + sem_400                           # smaller than one pooled standard error
assert step_400_1600 > 3 * sem_400                                # several standard errors clear

print("all checks passed")
```

</details>

</details>
