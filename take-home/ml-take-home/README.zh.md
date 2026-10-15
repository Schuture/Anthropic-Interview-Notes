# Take-home：四小时探索性机器学习研究

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| Take-home · 实验与汇报 | ★★☆☆☆ | 困难 | RE · RS · MLE · Fellows | label-noise, model-capacity, early-stopping, generalization, numpy | 4 小时独立工作 + 40 分钟现场复盘 | Take-home 作业 |
<!-- meta:end -->

## 题目

你会在 4 小时窗口开始时收到以下材料，事先不能针对这道具体题目做准备。可以在自己的本地环境，或者某个托管
notebook 服务中工作，两者皆可，只要能运行装有 NumPy 和 scikit-learn 的 Python 即可。查阅文档是允许且预期的
行为，但不能使用 AI 编程助手，每一行代码、每一条结论都必须出自你自己之手。4 小时结束时，做一次 10 分钟的
展示——用幻灯片还是走读 notebook 都可以，只要结果呈现清楚——随后是 30 分钟的技术讨论，内容是这次实验背后的
设计取舍、数据到底支持哪些结论，以及如果有更多时间你会怎么改。

*对称标签噪声*（symmetric label noise）在速率 $p$ 下，是一种只作用于训练标签的污染过程：对每个训练样本独立地，
以概率 $p$ 把它的标签替换成其余类别中随机均匀选出的一个，以概率 $1 - p$ 保持不变。

研究问题：对于在合成数据上训练的单隐藏层分类器，训练标签中的对称标签噪声如何与训练集规模、网络隐藏层宽度相互
作用，提前停止（early stopping）针对一个小型干净验证集，又能挽回其中多少损失？

### 数据

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

`make_dataset` 在给定 `seed` 时是确定性的：它从一次 `make_classification` 调用中抽取 `n_train + 200 + 1000`
个点（20 个特征——15 个信息特征、3 个冗余特征、2 个纯噪声特征——3 个数量均衡的类别，每类 2 个簇），再把它们切分
为规模为 `n_train` 的训练集、200 个点的验证集，以及 1000 个点的测试集，并只对训练标签按 `noise_rate` 施加对称
噪声。`y_train_true`，即加噪声之前的标签，仅作诊断用途返回，绝不能用来拟合或挑选模型。`X_val`、`y_val`、
`X_test`、`y_test` 则始终是干净的。

### 模型与实验网格

模型是一个单隐藏层分类器：`D` 个输入、宽度为 $H$ 的隐藏层配 ReLU 非线性，以及在 `C` 个类别上的 softmax 输出，
训练目标是最小化（可能带噪声的）训练标签的交叉熵。优化器、学习率、训练轮数，以及权重衰减都由你自己选择；说明
你用了什么、又是怎么选出来的。

依次跑遍以下组合：

- 隐藏层宽度 $`H \in \{8, 128\}`$；
- 训练集规模 $`n_{\text{train}} \in \{100, 400, 1600\}`$；
- 噪声率 $`p \in \{0.0, 0.2, 0.4\}`$，

每个组合至少用 5 个种子（同一个种子同时驱动 `make_dataset` 与模型的权重初始化）。对每个种子，除了记录最后一轮
训练结束后的测试准确率之外，还要记录每一轮结束后的验证准确率，以及验证准确率最高那一轮对应的测试准确率——这
正是针对 `X_val`、`y_val` 做提前停止本该拿到的准确率，而 `X_test`、`y_test` 自始至终都没有被用来做这个选择。

### 需要回答的问题

1. 在每种隐藏层宽度下，测试准确率如何随噪声率和训练集规模变化？
2. 隐藏层宽度是否改变了噪声对测试准确率的损害程度，这种改变又是否随训练集规模而变化？
3. 提前停止能否挽回噪声造成的测试准确率损失？挽回幅度是否取决于噪声率、隐藏层宽度或训练集规模——在完全没有
   噪声时，提前停止是否会带来代价？
4. 你的结论背后，哪些趋势在 5 个种子下是站得住的，哪些站不住？说明你需要看到多大的差距，才会相信一次比较。

交出两张表——一张是最后一轮模型的测试准确率（跨种子的均值与离散程度），一张是提前停止的挽回幅度（最佳验证
轮次的平均测试准确率，减去最后一轮的平均测试准确率，同样取这些种子）——引用这两张表作答的四个问题，以及展示
本身。

## 参考解答

<details>
<summary>展开参考解答</summary>

值得在开始前与面试官确认的两点：所谓“对称”噪声——如上文定义，翻转为随机均匀选出的另一个类别——是否就是预期
的污染模型，还是预期一种特定的混淆模式；以及 5 个种子这个下限是硬性下限还是目标值，因为这决定了四小时更多地
花在扩大网格上，还是花在同一网格上增加种子。

### 四小时的时间安排

| 时间 | 任务 |
| --- | --- |
| 0:00–0:15 | 读题，复述研究问题，定下实验网格与要填的两张表。 |
| 0:15–0:45 | 实现 `make_dataset`；核对实际翻转率与 `noise_rate` 相符，且 `X_val`/`X_test` 保持干净。 |
| 0:45–1:15 | 实现模型并跑一次训练；先确认无噪声条件能收敛，再去相信任何带噪声的结果。 |
| 1:15–2:45 | 跑完全部 18 个条件 × 5 个种子并跟踪提前停止——这是唯一真正耗时的一步。 |
| 2:45–3:30 | 填表、寻找趋势，起草结果解读与局限。 |
| 3:30–4:00 | 搭好十分钟展示的提纲；把常见追问的答案过一遍。 |

### Part 1 —— 数据与模型

隐藏层使用 He 初始化（`std = sqrt(2 / fan_in)`），适合 ReLU 单元；不论 $H$ 取何值，两个权重矩阵都按同样方式
初始化，因此窄网络与宽网络之间只有宽度不同，Part 2 里的比较也就把容量（capacity）和初始化的其他因素隔离
开了。训练用全批量梯度下降（full-batch gradient descent），不用 mini-batch 也不用 Adam：当
$n_{\text{train}} \le 1600$ 时整个训练集就是一次矩阵乘法，全批量去掉了随机梯度这一在数据抽取、标签噪声抽取
之外额外的种子间方差来源，也让 Part 2 的 90 次独立训练（18 个条件 × 5 个种子）能在四小时预算内跑完。
`train_with_early_stopping` 保存的是验证准确率最高那一轮的权重*快照*（snapshot），而不是保存每一轮在
`X_test` 上的预测：测试集在每次训练里恰好被读两次，一次在最后一轮之后，一次在恢复最佳验证快照之后，从不用来
挑选任何东西，只用来报告结果。

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

### Part 2 —— 网格实验：噪声、训练集规模与模型容量

以 `lr=0.5`、权重衰减 `1e-3` 跑 80 轮全批量梯度下降，足以让无噪声（$`p=0.0`$）条件在每种训练集规模下都收敛到
平台期，于是整个网格都沿用这套设置，故意不针对每个格子分别调参——这个取舍会在下文的结果解读里重新讨论。

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

**表 1——最后一轮的测试准确率**（5 个种子上的均值 ± 标准差）：

| $n_{\text{train}}$ | $H$ | $p=0.0$ | $p=0.2$ | $p=0.4$ |
| --- | --- | --- | --- | --- |
| 100 | 8 | 0.869 ± 0.029 | 0.736 ± 0.043 | 0.542 ± 0.072 |
| 100 | 128 | 0.879 ± 0.025 | 0.749 ± 0.036 | 0.594 ± 0.060 |
| 400 | 8 | 0.937 ± 0.008 | 0.833 ± 0.031 | 0.663 ± 0.049 |
| 400 | 128 | 0.949 ± 0.011 | 0.816 ± 0.014 | 0.615 ± 0.069 |
| 1600 | 8 | 0.964 ± 0.009 | 0.930 ± 0.011 | 0.875 ± 0.020 |
| 1600 | 128 | 0.964 ± 0.005 | 0.868 ± 0.021 | 0.724 ± 0.041 |

六个 $(n_{\text{train}}, H)$ 组合里的每一个都满足两条稳健的趋势：准确率随 $p$ 从 0.0 到 0.2 再到 0.4 单调
下降（表中最小的一步是 0.034，出现在 $n_{\text{train}}=1600$、$`H=8`$ 处；其余每一步都更大），并且在有噪声时
$n_{\text{train}}=1600$ 处的准确率高于 $n_{\text{train}}=100$ 处（最小的这类差距是 0.119，出现在 $H=128$、
$p=0.2$ 处）。这两条趋势本身都不意外；更具体的发现在于 $H$ 如何与 $n_{\text{train}}$ 相互作用。在
$n_{\text{train}}=100$ 时，宽网络（$`H=128`$）在噪声下并不比窄网络差——如果说有区别，反而还略微领先，$`p=0.4`$
时 0.594 对 0.542。到了 $n_{\text{train}}=1600$，名次反转，差距也急剧拉大：窄网络在 $p=0.4$ 时达到 0.875，
宽网络只有 0.724，差距 0.151——大约是 $n_{\text{train}}=100$ 处那个*方向相反*的 0.052 差距的三倍。更大的
容量（capacity）在噪声下并非一概有害；它恰恰在训练数据足够多、多余的容量被用来记住错标样本而非拟合共同模式
时才有害。只有 100 个点时，两个网络的参数数量早已远超数据量——光是窄网络两层里的权重就有
$20 \times 8 + 8 \times 3 = 184$ 个——因此谁都没有比对方更多的过拟合空间。

### Part 3 —— 提前停止能否挽回损失？

**表 2——挽回幅度**（最佳验证轮次的平均测试准确率，减去最后一轮的平均测试准确率，同样取 5 个种子）：

| $n_{\text{train}}$ | $H$ | $p=0.0$ | $p=0.2$ | $p=0.4$ |
| --- | --- | --- | --- | --- |
| 100 | 8 | -0.002 | 0.039 | 0.069 |
| 100 | 128 | -0.006 | 0.038 | 0.024 |
| 400 | 8 | -0.005 | 0.025 | 0.064 |
| 400 | 128 | -0.003 | 0.037 | 0.076 |
| 1600 | 8 | -0.006 | 0.000 | -0.015 |
| 1600 | 128 | -0.002 | 0.029 | 0.016 |

在 $p=0.0$ 时，每一个挽回幅度都很小且为负——介于 -0.006 到 -0.002 之间，六个条件的平均值是 -0.004——这正是
一条基于验证集的停止规则，在无需防范任何东西时该有的表现：既没有帮助，也没有明显的损害。在 $p=0.4$ 时，平均
挽回幅度是 0.039，量级大约是前者的十倍，六个条件里有五个是正收益；唯一的例外是 $n_{\text{train}}=1600$、
$H=8$（-0.015），恰好就是那个容量相对于这么多干净数据而言太小、原本就记不住噪声的条件——既然没有什么病态
行为需要纠正，仅凭验证准确率去选一个更早的轮次，反而会丢掉最后几轮本可以带来的一点训练收益。单个挽回幅度
最大的是 +0.076，出现在 $n_{\text{train}}=400$、$`H=128`$、$`p=0.4`$ 处。与 Part 2 的容量故事一致，宽网络在数据
稀缺、噪声又高时验证准确率很早就见顶：在 $n_{\text{train}}=100$、$`H=128`$、$`p=0.4`$ 处，平均最佳轮次是 80 轮
里的第 9.8 轮，而 $n_{\text{train}}=1600$、$`H=8`$、$`p=0.4`$ 处是第 64.4 轮——早了一个数量级。

### 结果解读

有三条结论证据支持得相当充分，因为背后每一次比较都以数倍于整个网格里观察到的种子间波动的余量成立：准确率
随噪声率单调下降，在每种容量、每种训练集规模下都成立；有噪声时 $n_{\text{train}}=1600$ 处的准确率高于
$n_{\text{train}}=100$ 处，在每种容量、两个有噪声条件下都成立；噪声下的容量代价——宽网络比窄网络差多少——在
$n_{\text{train}}=1600$ 处远大于 $n_{\text{train}}=100$ 处，在后者甚至根本算不上代价。

只有一条结论证据支持得比较勉强：准确率是否在*每一个*条件下都随训练集规模单调上升，而不只是在两个端点上升。
在 $H=128$、$`p=0.4`$ 时，从 $n_{\text{train}}=100$ 到 $400$ 这一步是 0.021（0.594 到 0.615），小于两个均值
各自的标准误（standard error，5 个种子下约为 0.027 和 0.031）；从 400 到 1600 这一步是 0.109，数倍于任一
标准误，毫无疑问。5 个种子足以确定这项研究里量级悬殊的比较，却确定不了一个 0.02 量级的差异——这正好印证了
一条一般规律：需要多少个种子，取决于要分辨多小的差异，而不取决于网格有多大。

它不支持一条关于容量的一般性结论。学习率、训练轮数与权重衰减只在 Part 2 里依据无噪声条件确定过一次，此后
两种隐藏层宽度都直接沿用，而没有分别调参。如果 `lr=0.5` 让宽网络相对窄网络训练不足，测得的容量代价里就可能
有一部分是优化层面的假象，而非纯粹的容量效应——这是当前设计本身无法排除的一个混杂因素。它也不支持超出这一
数据生成过程之外的任何结论：只有一个合成数据族、一种类别间隔、只有对称噪声，以及两个容量取值点而非一整条
曲线。

### 局限与后续工作

- 5 个种子能排出相差数倍标准误的效应的先后，排不出没有这个差距的——见上文 $n_{\text{train}}=100 \to 400$
  这一步；两张表里大概率还有别的格子同样接近临界，但没有逐一核实。
- 训练超参数只依据无噪声条件确定过一次，此后处处沿用；上文的结果解读已经指出，这在容量比较里具体可能掩盖
  了什么。
- 只试了两个隐藏层宽度。它们合在一起只能说明在 8 到 128 个单元之间（相对于 $n_{\text{train}} \le 1600$）
  存在一个交叉点，说明不了它具体在哪、有多陡。
- 这里的噪声是对称且只作用于标签的：每个错误标签出现的概率均等，特征本身从未被污染。现实里的标签噪声往往
  是按类别条件的（某些类别之间的混淆远比均匀混淆更常见）或依样本而定的（较难的样本更容易被标错）；这两种
  情形这里都没有体现。
- 用于提前停止的验证集是干净的。更贴近现实的设定——验证标签也来自和训练一样的噪声过程——很可能会显示出更小
  的挽回幅度，因为停止准则本身也被污染了。

如果再给一天：先针对每种隐藏层宽度分别调学习率与训练轮数，再重新跑一遍容量比较；加入两三个中间的隐藏层宽度，
把交叉点画成一条曲线而不是两个点；把对称噪声换成固定的类别混淆矩阵，看看这种相互作用是否还在；把种子数量
有针对性地加到上文指出接近临界的那几个条件上，而不是在整个网格上一律增加。

### 十分钟展示的结构

十分钟，六张幻灯片，掐着点留出完整 30 分钟用于问答：

```text
1. 问题（30 秒）：一句话，标签噪声是否会和训练集规模、模型容量相互作用，提前停止又能不能挽回其中一部分？
2. 背景（1.5 分钟）：数据生成器、单隐藏层模型、三个网格维度、每种设置 5 个种子。
3. 结果一（2 分钟）：表 1，噪声总是有害，数据总是有益；然后是容量交叉效应，n=100 时窄网络不输给宽网络，
   n=1600 时窄网络遥遥领先。
4. 结果二（2 分钟）：表 2，提前停止平均能挽回噪声造成的损失，在 p=0.0 时不带来任何代价，唯一的例外可以用
   同一套容量故事解释。
5. 数据支持什么、不支持什么（2.5 分钟）：H=128 时那个脆弱的 100 到 400 这一步；固定学习率给容量结论带来的
   混杂因素。
6. 如果有更多时间（1.5 分钟）：按条件分别调参、把交叉点画成曲线、换成按类别条件的噪声、把种子加在真正需要
   的地方。
```

### 追问

- 为什么用对称噪声而不是更贴近现实的污染模型？它只有一个参数，其效果可以直接对照设定的翻转率做经验验证，
  而且它正是更贴近现实的噪声模型（按类别条件的、依样本而定的）通常最先拿来对照的标准条件。
- 为什么用全批量梯度下降而不是 Adam 或 mini-batch？在这个规模（$`n_{\text{train}} \le 1600`$）下图的是确定性
  与速度：它去掉了随机梯度这个在数据抽取、标签噪声抽取之外额外的种子间方差来源，也让 90 次独立训练稳稳落在
  时间预算之内。
- 什么会改变容量交叉效应这个结论？给每种隐藏层宽度分别调学习率——目前的比较把学习率固定不变，因此这个效应
  里可能有一部分是优化层面的假象，而不是纯粹的容量效应。
- 为什么是 5 个种子而不是更多？四小时的预算所限；这份解答明确指出了这个种子数量留下的哪一次比较判断不了
  （$`H=128`$、$`p=0.4`$ 时 $n_{\text{train}}=100 \to 400$ 这一步），而不是把表里每条趋势都当作同样确定。
- 如果再给两天你会做什么？按条件分别调超参数、加入中间的隐藏层宽度把交叉点画成曲线、再换成按类别条件的
  噪声重跑一遍。
- 你怎么知道提前停止不是单纯把模型拟合任何东西（包括干净标签）的能力正则化掉了？看 $p=0.0$ 那一行：那里的
  每个挽回幅度都很小且为负，说明同一条在噪声下起作用的停止规则，在无需纠正任何东西时并不会凭空制造收益。

<details>
<summary>验证代码（可运行）</summary>

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
