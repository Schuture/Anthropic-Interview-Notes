# 调试 GRPO 训练循环

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 调试 · PyTorch，强化学习 | ★★☆☆☆ | 困难 | RE · RS · MLE | grpo, ppo-clipping, policy-gradient, advantage-normalization, reinforcement-learning | 3 个 bug + 2 个部分 / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

下面的文件 `grpo_train.py` 用*组相对策略优化*（Group Relative Policy Optimization，GRPO）训练一个很小的策略，
任务是一个奖励可验证的合成任务：给定两个数字 $a, b \in \{0,\dots,9\}$ 作为 prompt，策略必须给出单个 token
$(a+b) \bmod 10$ 作为回答。

*策略*（policy）$\pi_\theta$ 把一个 prompt $(a,b)$ 映射为在 $V = 10$ 个候选答案 token
$\{0,\dots,9\}$ 上的一个类别分布：$\pi_\theta(\cdot \mid a,b) = \operatorname{softmax}(f_\theta(a,b))$，
其中 $f_\theta(a,b) \in \mathbb{R}^V$ 是一个小网络给出的 logits。*响应*（response）$o$ 是从
$\pi_\theta(\cdot\mid a,b)$ 中采样出的单个 token，它的*奖励*（reward）为
$r(o,a,b) = \mathbb{1}[\,o = (a+b) \bmod 10\,]$。

对每个 prompt，都从同一个分布 $\pi_{\theta_{\mathrm{old}}}(\cdot\mid a,b)$ 中独立采样出一*组*（group）
$G$ 个响应 $o_1,\dots,o_G$，其中 $\theta_{\mathrm{old}}$ 是在下文所有更新开始之前取的一份参数快照。
记 $r_i = r(o_i,a,b)$，响应 $i$ 的*组内相对优势*（group-relative advantage）为

```math
A_i = \frac{r_i - \bar r}{\operatorname{std}(r) + \epsilon_{\mathrm{adv}}}, \qquad
\bar r = \frac{1}{G}\sum_{j=1}^G r_j, \qquad
\operatorname{std}(r) = \sqrt{\frac{1}{G-1}\sum_{j=1}^G (r_j - \bar r)^2},
```

其中 $\epsilon_{\mathrm{adv}} > 0$ 是一个很小的常数。例如，一组 $G=4$ 个响应，奖励为 $(1,0,1,0)$，
则 $\bar r = 0.5$，$\operatorname{std}(r) = 1/\sqrt3 \approx 0.5774$（$\epsilon_{\mathrm{adv}}$ 取
$10^{-6}$ 时这一步几乎没有影响），于是

```text
A = (0.8660, -0.8660, 0.8660, -0.8660)   # (r_i - 0.5) / 0.5774, elementwise
```

响应 $i$ 的*比率*（ratio）比较的是当前正在被优化的策略 $\pi_\theta$，与生成该样本的策略
$\pi_{\theta_{\mathrm{old}}}$：

```math
\rho_i = \exp\!\big(\log \pi_\theta(o_i \mid a,b) - \log \pi_{\theta_{\mathrm{old}}}(o_i \mid a,b)\big).
```

对由 $P$ 个 prompt、每个 $G$ 个响应组成、共 $P \cdot G$ 个样本的一个展平（flattened）批次而言，
训练损失为

```math
L(\theta) = \frac{1}{PG}\sum_{i=1}^{PG} -\min\!\big(\rho_i A_i,\ \operatorname{clip}(\rho_i,\, 1-\varepsilon,\, 1+\varepsilon)\, A_i\big)
\;+\; \beta \cdot \frac{1}{PG}\sum_{i=1}^{PG} D_{\mathrm{KL}}\!\big(\pi_\theta(\cdot\mid a_i,b_i) \,\Vert\, \pi_{\mathrm{ref}}(\cdot\mid a_i,b_i)\big),
```

其中 $\operatorname{clip}(x, l, u) = \min(\max(x,l),u)$，$\varepsilon$ 是裁剪范围，$\beta$ 是一个固定的系数，
$\pi_{\mathrm{ref}}$ 是策略的另一份冻结副本，固定为训练开始之前的取值（不会被优化器更新，也与
$\theta_{\mathrm{old}}$ 无关）。例如，当 $\varepsilon = 0.2$ 时，四组
$(\rho_i, A_i) \in \{(0.7,1), (1.5,1), (0.7,-1), (1.5,-1)\}$ 对应的裁剪值为
$\operatorname{clip}(\rho_i,0.8,1.2) \in \{0.8, 1.2, 0.8, 1.2\}$，于是

```text
rho_i * A_i          =  0.7,  1.5, -0.7, -1.5
clip(rho_i) * A_i    =  0.8,  1.2, -0.8, -1.2
min(...)             =  0.7,  1.2, -0.8, -1.5     # the surrogate term, before the leading minus sign
-> loss contribution = -0.7, -1.2,  0.8,  1.5      # mean over the four: 0.1
```

由于这里的响应只有一个 token，比率通常按 token 逐个定义与按整条响应定义是重合的；如果响应有多个 token，
$\log \pi_\theta(o \mid a,b)$ 就会是各 token 对数概率之和，而 $A_i$ 则是同一个值广播到响应 $i$ 的每个 token 上。

`grpo_train.py` 交替执行两类步骤。在每个*外层*（outer）步，它先采样一批 $P$ 个 prompt，再为每个 prompt 用
当前策略采样一组 $G$ 个响应；这份参数快照在接下来的所有计算中都作为 $\theta_{\mathrm{old}}$。它把每个
奖励 $r_i$ 和优势 $A_i$ 都只计算一次。随后，在一个长度为 `inner_steps` 的*内层*（inner）步循环中，
它反复针对这同一批 $PG$ 个已采样响应计算 $L(\theta)$ 并做一次梯度更新——于是 $\theta$ 在每个内层步都会变化，
而 $\theta_{\mathrm{old}}$（从而 $\rho_i$ 中用到的每个 $\log \pi_{\theta_{\mathrm{old}}}(o_i\mid\cdot)$）
始终固定在本次外层步开始时的取值。只有等全部 `inner_steps` 次更新做完，下一个外层步才会重新采样一批新的
prompt，并取一份新的 $\theta_{\mathrm{old}}$。

### Part 1 —— 找出并修复三个 bug

文件中有三个 bug：一个会立即导致崩溃，一个在损失变成 `nan` 之前都悄无声息，还有一个从不崩溃、也从不产生
`nan`。对每一个 bug，指出它的位置，解释它对采样或训练造成了什么影响、原因是什么，然后修复它。三个都修好之后，
`python grpo_train.py` 训练 200 个外层步（CPU 上几秒钟），在一批 2000 个新采样的留出（held-out）prompt 上，
用贪心地取 $\arg\max_o \pi_\theta(o\mid a,b)$（而不是采样）来评估，平均奖励能超过 $0.5$（随机水平是 $0.1$）。
`sample_prompts`、`reward`、网络结构，以及每一个超参数都是正确的，不需要改动。

```py
import copy

import torch
import torch.nn as nn
import torch.nn.functional as F

V = 10                          # digits 0-9: both the two prompt digits and the possible answer tokens
D_MODEL = 16
HIDDEN = 32
CLIP_EPS = 0.2                   # epsilon: the clip range is [1 - CLIP_EPS, 1 + CLIP_EPS]
KL_COEF = 0.01                   # beta: weight of the KL penalty against the frozen reference policy
ADV_EPS = 1e-6                   # epsilon_adv: the advantage's standard-deviation denominator


class Policy(nn.Module):
    def __init__(self):
        super().__init__()
        self.embed_a = nn.Embedding(V, D_MODEL)
        self.embed_b = nn.Embedding(V, D_MODEL)
        self.net = nn.Sequential(nn.Linear(D_MODEL, HIDDEN), nn.ReLU(), nn.Linear(HIDDEN, V))

    def forward(self, a, b):                   # a, b: (N,) digit ids -> logits (N, V)
        return self.net(self.embed_a(a) + self.embed_b(b))


def sample_prompts(n, rng):                     # n random (a, b) prompts, each digit uniform in [0, V)
    a = torch.randint(0, V, (n,), generator=rng)
    b = torch.randint(0, V, (n,), generator=rng)
    return a, b


def reward(a, b, answer):                       # verifiable reward: 1.0 iff answer == (a + b) mod 10
    return (answer == (a + b) % V).float()


def group_advantages(rewards):                   # rewards: (P, G) -> advantages (P, G)
    mean = rewards.mean(dim=1, keepdim=True)
    std = rewards.std(dim=1, keepdim=True)
    return (rewards - mean) / std


def train(policy, ref_policy, steps=200, batch_size=100, group_size=16, inner_steps=4, lr=0.02, seed=0):
    opt = torch.optim.Adam(policy.parameters(), lr=lr)
    rng = torch.Generator().manual_seed(seed)
    for step in range(steps):
        a, b = sample_prompts(batch_size, rng)
        a_rep, b_rep = a.repeat_interleave(group_size), b.repeat_interleave(group_size)
        with torch.no_grad():
            logits_old = policy(a_rep, b_rep)
            actions = torch.multinomial(logits_old, 1, generator=rng).squeeze(1)
            logp_old = F.log_softmax(logits_old, dim=-1).gather(1, actions[:, None]).squeeze(1)
            logp_ref_all = F.log_softmax(ref_policy(a_rep, b_rep), dim=-1)
        r = reward(a_rep, b_rep, actions).view(batch_size, group_size)
        adv = group_advantages(r).reshape(-1)
        for _ in range(inner_steps):
            logits_new = policy(a_rep, b_rep)
            logp_new_all = F.log_softmax(logits_new, dim=-1)
            logp_new = logp_new_all.gather(1, actions[:, None]).squeeze(1)
            logp_old = logp_new.detach()
            ratio = torch.exp(logp_new - logp_old)
            clipped = torch.clamp(ratio, 1 - CLIP_EPS, 1 + CLIP_EPS)
            pg_loss = -torch.min(ratio * adv, clipped * adv).mean()
            probs_new_all = logp_new_all.exp()
            kl = (probs_new_all * (logp_new_all - logp_ref_all)).sum(dim=-1).mean()
            loss = pg_loss + KL_COEF * kl
            opt.zero_grad()
            loss.backward()
            opt.step()
    return policy


@torch.no_grad()
def evaluate(policy, n, seed):                   # greedy accuracy on n freshly sampled held-out prompts
    rng = torch.Generator().manual_seed(seed)
    a, b = sample_prompts(n, rng)
    pred = policy(a, b).argmax(dim=-1)
    return reward(a, b, pred).mean().item()


if __name__ == "__main__":
    torch.manual_seed(0)
    policy = Policy()
    ref_policy = copy.deepcopy(policy)
    for p in ref_policy.parameters():
        p.requires_grad_(False)
    train(policy, ref_policy)
    accuracy = evaluate(policy, 2000, seed=999)
    print(f"held-out accuracy: {accuracy:.3f}")
    assert accuracy > 0.5, "the policy did not learn to add single digits"
    print("cleared the reward threshold")
```

### Part 2 —— 目标函数背后的数学

结合上面定义的量，回答下面关于修复后训练器的问题。

**(a)** 在某个外层步的第一个内层步，$\theta$ 恰好等于 $\theta_{\mathrm{old}}$。推导此时的
$\partial(\rho_i A_i)/\partial\theta$，证明它等于朴素策略梯度（REINFORCE）项
$A_i \nabla_\theta \log\pi_\theta(o_i\mid a,b)$，并说明此时 $\rho_i = 1$ 不会引发任何裁剪。
解释为什么代理目标（surrogate）的梯度与 REINFORCE 梯度之间的这种相等关系只能在每个外层步的第一个内层步成立，
以及为什么从第二个内层步开始，$\rho_i$ 一般就不再是 $1$ 了。

**(b)** 当 $\varepsilon = 0.2$ 时，给出一般规律（任意 $\rho$、任意 $A$ 的符号）：什么时候
$\min(\cdot,\cdot)$ 选中的是 $\operatorname{clip}(\rho,1-\varepsilon,1+\varepsilon)A$ 而不是
$\rho A$，以及在那个区域里 $\partial(\text{代理目标})/\partial\rho$ 等于多少。

**(c)** 证明对任意一组都有 $\frac{1}{G}\sum_{i=1}^G A_i = 0$，这是公式本身决定的。用这一点解释组均值
相对于 actor-critic 方法里学出来的基线 $V_\phi(s)$ 扮演了什么角色，以及 GRPO 为什么不需要这样一个网络。

**(d)** 准确说明 `group_advantages` 对一个大小 $G=1$ 的组会返回什么，再说明对一个大小 $G \ge 2$、
但 $G$ 个奖励恰好全部相等的组会返回什么。解释为什么 `ADV_EPS` 能修好其中一种情况，却修不好另一种。

**(e)** $\varepsilon$ 限制的是 $\rho_i$ 相对 $\theta_{\mathrm{old}}$ 能移动多远，而
$\theta_{\mathrm{old}}$ 在每个外层步开始时都会被重新设定；$\beta$ 惩罚的是相对 $\pi_{\mathrm{ref}}$
的距离，而 $\pi_{\mathrm{ref}}$ 在整个训练过程中都固定不变。解释这两项各自限制的是许多个外层步之后的什么量，
以及当 $\beta \to 0$ 和 $\beta \to \infty$ 时，定性地看会发生什么变化。

## 参考解答

<details>
<summary>展开参考解答</summary>

写代码之前有一点值得先确认：任务和网络结构由题目给定，唯一的自由度在于 `train` 的关键字参数默认值
（批大小、组大小、内层步数、学习率）以及奖励阈值；下面用到的取值只是能够较为轻松、快速达到题目所述阈值的
一种设定，并不是唯一可行的设定。

### Part 1

先把脚本跑起来，再细读代码：三个 bug 同时存在时，`python grpo_train.py` 在第一次调用
`torch.multinomial` 时就会崩溃。把 bug 逐个修复并重新运行，会依次看到每个 bug 各自的症状。

| 已修复的 bug | 观察到的现象 | 指向 |
| --- | --- | --- |
| 无 | 第 0 步就出现 `RuntimeError: probability tensor contains either inf, nan or element < 0` | bug 1，采样 |
| 1 | 第 0 步损失为 `nan`，第 1 步出现同样的 `RuntimeError` | bug 2，`group_advantages` |
| 1、2 | 能跑完全程；留出集准确率停在 $0.2$ 左右，远低于 $0.5$ 的阈值 | bug 3，`logp_old` |
| 全部 | 用本页全程所用的随机种子，留出集准确率能达到约 $0.7$ | |

**Bug 1：`actions = torch.multinomial(logits_old, 1, generator=rng)`。** `torch.multinomial`
把输入当成各个类别上一组非负的、未归一化的权重——它并不会先做 softmax。`logits_old` 直接来自一个线性层，
数值集中在 0 附近，因此正负号都有；第一次调用时几乎必然出现负的分量，PyTorch 会直接报错，而不是悄悄把它
截断。修复方法是在采样之前先把 logits 变成一个真正的分布：

```py
probs_old = F.softmax(logits_old, dim=-1)
actions = torch.multinomial(probs_old, 1, generator=rng).squeeze(1)   # was torch.multinomial(logits_old, ...)
```

**Bug 2：`group_advantages` 里的 `return (rewards - mean) / std`。** 当一个组里的每个响应得到的奖励都相同时——
每个响应与组均值的偏差都恰好是 $0$——`std` 并不是“接近”零，而是恰好等于 $0$，而 $0/0$ 得到的是 `nan`，
不是一个很大或很小的数。在 `batch_size=100`、`group_size=16`、策略又是刚初始化、准确率接近 $10\%$ 的情况下，
100 个组全都至少包含一个正确响应的概率微乎其微，所以这个 `nan` 在第一次优化器更新时就会出现。
`loss.backward()` 会把 `nan` 写进每个参数的梯度，`opt.step()` 又会把 `nan` 永久地写进每个参数本身——
等 bug 1 修好之后，下一次 `torch.multinomial` 调用崩溃，其实是*这个* bug 的下游症状，而不是一个新 bug。
修复方法是加上已经定义好的 `ADV_EPS`：

```py
return (rewards - mean) / (std + ADV_EPS)     # was (rewards - mean) / std
```

**Bug 3：`for _ in range(inner_steps):` 循环内部的 `logp_old = logp_new.detach()`。** 在进入内层循环之前，
`logp_old` 已经在 `torch.no_grad()` 下，从 rollout 快照 $\theta_{\mathrm{old}}$ 正确地计算出来了。
被标出的这一行却在每一次内层迭代里，用*当前*策略的对数概率重新覆盖了这个名字，`.detach()` 在这里只是
阻断了梯度流向它，并没有把它的取值锁定在 rollout 快照上。于是 `logp_new` 和这里的 `logp_old` 总是来自
同一次前向计算，`logp_new - logp_old` 恒为 $0$，$\rho_i$ 在每一个内层步都恒等于 $1$，而不仅仅是第一步——
第一步比率恰好为 $1$ 本来是对的（见 Part 2a）。由于 $\rho_i$ 永远不会离开 $[1-\varepsilon, 1+\varepsilon]$，
裁剪从未触发，它本该起到的信任域（trust region）约束也就悄悄失效了：每个内层步实际上都在*当前*参数上做了
一次未经修正的 REINFORCE 更新，重复使用最初 rollout 算出的优势值，而这些优势值随着内层步的推进只会越来越“过时”。
打印 `ratio` 看不到 `nan`，也不会崩溃——训练甚至仍然会让奖励上升，只是远不如三个 bug 都修好时那么多，
因为这些额外的内层步都没有真正被保护、以免它们离生成这批数据时的策略太远。修复方法是删掉这一行，
让循环开始前算好的、不带梯度的 `logp_old` 在每个内层步里原样保留、不被重新赋值：

```py
ratio = torch.exp(logp_new - logp_old)      # logp_old: the fixed value from before the loop; no reassignment here
```

### Part 2

**(a)** 在第一个内层步，还没有发生任何更新，所以 $\theta = \theta_{\mathrm{old}}$，
$\log\pi_\theta(o_i\mid a,b)$（计算图中一个带梯度的节点）在数值上等于
$\log\pi_{\theta_{\mathrm{old}}}(o_i\mid a,b)$（之前存下来的一个已分离的常数），于是
$\rho_i = \exp(0) = 1$。记 $x(\theta) = \log\pi_\theta(o_i\mid a,b)$，$c$ 为那个已分离的常数，则
$\rho_i(\theta) = \exp(x(\theta) - c)$，从而

```math
\frac{\partial \rho_i}{\partial\theta} = \exp(x(\theta) - c)\cdot\frac{\partial x}{\partial\theta}
= \rho_i \cdot \nabla_\theta\log\pi_\theta(o_i\mid a,b),
```

在 $\rho_i = 1$ 处即为 $\partial(\rho_i A_i)/\partial\theta = A_i\nabla_\theta\log\pi_\theta(o_i\mid a,b)$，
恰好就是 REINFORCE 项。由于 $\rho_i = 1 \in [1-\varepsilon, 1+\varepsilon]$，$\min(\cdot,\cdot)$ 的两个分支
在这一点上也相等，所以裁剪在这个具体点上不改变任何东西。这种相等只是因为 $c$ 恰好等于 $x(\theta)$ 在这一瞬间的取值；
从第二个内层步开始，$\theta$ 已经因为第一步的更新而偏离了 $\theta_{\mathrm{old}}$，而
$c = \log\pi_{\theta_{\mathrm{old}}}(o_i\mid a,b)$ 却始终固定在任何更新之前的取值，因此一般有
$x(\theta) \ne c$，$\rho_i \ne 1$：代理目标的梯度这时才真正不同于朴素的 REINFORCE，被策略在这批数据上
已经移动了多远重新加权（并可能被裁剪）。这也解释了为什么 Part 1c 里的那个 bug 在 `inner_steps == 1` 时
是看不出来的：只有一个内层步时，有 bug 的和正确的 `logp_old` 在数值上完全一样，两种实现算出的是同一次更新。

**(b)** 当且仅当 $\rho$ 已经*朝着 $A$ 的符号所偏好的方向*移出了 $[1-\varepsilon,1+\varepsilon]$ 时，
$\min(\cdot,\cdot)$ 选中的才是 $\operatorname{clip}(\rho,1-\varepsilon,1+\varepsilon)A$ 而不是 $\rho A$：
即 $\rho > 1+\varepsilon$ 且 $A>0$（已经把一个好响应的概率提得超出了信任域），或者 $\rho < 1-\varepsilon$
且 $A<0$（已经把一个差响应的概率压得超出了信任域）。题目给出的四组取值恰好覆盖了这两种情况及其反面：

```text
(rho, A) = (0.7, 1):  unclipped 0.7 < clipped 0.8 -> min picks 0.7 (unclipped; rho has NOT overshot the favored direction)
(rho, A) = (1.5, 1):  unclipped 1.5 > clipped 1.2 -> min picks 1.2 (clipped;   rho HAS overshot the favored direction)
(rho, A) = (0.7, -1): unclipped -0.7 > clipped -0.8 -> min picks -0.8 (clipped;   overshot, favored direction is decreasing rho)
(rho, A) = (1.5, -1): unclipped -1.5 < clipped -1.2 -> min picks -1.5 (unclipped; not overshot)
```

裁剪分支被选中时，$\operatorname{clip}(\rho,\dots)$ 在局部相对 $\rho$ 是常数，所以
$\partial(\text{代理目标})/\partial\rho = 0$：没有任何梯度信号继续把 $\rho$ 往那个方向推。未裁剪分支被
选中时——包括 $\rho$ 移动方向与 $A$ 相反的所有情形——梯度就是未经削弱的 $A$：裁剪从不会阻止一次修正，
只会阻止对一次已经被过度利用的移动的延续。

**(c)** $\sum_{i=1}^G (r_i - \bar r) = \sum_i r_i - G\bar r = G\bar r - G\bar r = 0$，而组内每个 $i$
都除以同一个 $\operatorname{std}(r) + \epsilon_{\mathrm{adv}}$，这个等于零的性质自然保留：
$\sum_i A_i = 0/(\operatorname{std}(r)+\epsilon_{\mathrm{adv}}) = 0$。在 actor-critic 的估计量
$\frac1G\sum_i (r_i - b(s))\nabla_\theta\log\pi_\theta(o_i\mid s)$ 中，任何不依赖于 $o_i$ 的基线 $b(s)$
都不会引入偏差，因为对固定的 $s$ 有
$\sum_o \pi_\theta(o\mid s)\nabla_\theta\log\pi_\theta(o\mid s) = \nabla_\theta \sum_o \pi_\theta(o\mid s) = \nabla_\theta 1 = 0$——
减去 $b(s)$ 在期望上贡献恰好是 $0$，而当 $b(s)$ 接近 $\mathbb{E}_{o\sim\pi}[r(o,s)\mid s]$ 时，通常还能降低
估计量的方差。一个学出来的 $V_\phi(s)$ 正是通过回归来逼近这同一个条件期望，代价是要多训练一个网络、
一个额外的优化器，以及它自己的近似误差。组均值 $\bar r$ 则是用这个精确 prompt 的 $G$ 个新样本直接估计出的
同一个量，因此不需要第二个网络就能起到同样的方差缩减作用；再除以 $\operatorname{std}(r)$，则进一步把不同组
之间奖励分散程度不同的问题归一化掉，让各组的优势值可以互相比较。

**(d)** 当 $G=1$ 时，`rewards.std(dim=1)` 要把偏差平方和除以 $G-1=0$：结果是 `nan`，与 `ADV_EPS` 无关，
因为 `nan + ADV_EPS` 仍然是 `nan`——对单个样本而言，这个估计量本身就没有定义，所以 `group_advantages`
根本不能用在 $G=1$ 的情形上。而当 $G\ge2$ 且这 $G$ 个奖励恰好全部相等时，每个偏差都恰好是 $0$，
所以 `std` 是一个有定义的、有限的 $0$（$G$ 个零求和，再除以 $G-1>0$）；真正出问题的是*随后*
`(rewards - mean) / std` 这一步的 $0/0$。`ADV_EPS` 修复的正是这第二种情况，把它变成
$0/(0+\epsilon_{\mathrm{adv}}) = 0$——一个“这个组没有信号”的、有定义的优势值，而不是 `nan`——但它修不好
第一种情况，因为参与除法的分子分母在那之前就已经是 `nan` 了。

**(e)** 裁剪限制的是 $\rho_i$（从而大致上是 $\pi_\theta$）在一个外层步的 `inner_steps` 次更新之内，
能相对 $\theta_{\mathrm{old}}$ 移动多远，而 $\theta_{\mathrm{old}}$ 在每个新的外层步开始时都会重置为
当前的 $\theta$，所以这个约束每一步都会重置，对 $\pi_\theta$ 在许多个外层步之后能漂移多远完全没有约束。
$\beta D_{\mathrm{KL}}(\pi_\theta\Vert\pi_{\mathrm{ref}})$ 则是相对同一个固定不变的 $\pi_{\mathrm{ref}}$
在整个训练过程中持续计算的，所以只有它才限制了累积的漂移。当 $\beta\to0$ 时，这个锚点消失了：单个外层步
仍然受裁剪的信任域约束，但只要外层步足够多，没有什么能阻止 $\pi_\theta$ 相对 $\pi_{\mathrm{ref}}$ 任意远离——
在奖励不像本题这样严格可验证的场景里，这正是 reward hacking（见追问）变得可能的地方。当 $\beta\to\infty$
时，KL 项会主导 $L(\theta)$，$\pi_\theta$ 无论奖励信号如何都会被钉在 $\pi_{\mathrm{ref}}$ 附近，训练也就
不再提升奖励了。

### 追问

- **`ratio = logp_new - logp_old`，不取 `exp`。** 对很小的更新而言，由于 $x$ 接近 $0$ 时
  $\log(1+x)\approx x$，这个近似还算贴近真实比率，模型往往仍然能学到一些东西；问题在于结构性，而不是
  立即显现。此时 $\operatorname{clip}$ 裁剪的是一个*对数*差到 $[-\varepsilon,\varepsilon]$，而不是一个
  概率比率到 $[1-\varepsilon,1+\varepsilon]$——一旦 $|x|$ 不再很小，这在概率意义上是一个窄得多的区间——
  而且真实比率的下界是 $0$（一个策略现在强烈不喜欢的 token，对代理目标的贡献至多是 $-A_i$），原始的
  对数差却没有下界，于是单个被极度压低概率的 token 就可能主导整个 batch 的梯度。
- **Reward hacking（钻奖励的空子）。** 像本题这样严格可验证的奖励，可钻的空子很少，但一般情况下奖励本身就是一个模型或
  一种启发式规则，不受约束的策略完全可能朝着能在这个奖励上打高分、却并未真正变好的输入漂移。KL 项和比率裁剪
  正是把 $\pi_\theta$ 分别约束在离 $\pi_{\mathrm{ref}}$ 足够近、离刚刚被评估过的数据足够近的两种机制。
- **Entropy collapse（熵坍缩）。** 一旦某组的 $G$ 个样本全部落到同一个响应上，每个样本与组均值的偏差就都是
  $0$，于是该组每个 $i$ 的 $A_i = 0$（见 Part 2d）——无论这个共同的响应是对是错：一个策略已经把它变得
  近乎确定性的 prompt，不论那个确定性的答案对不对，都不会再为自己产生任何梯度信号。让 KL 项指向一个熵更高的
  $\pi_{\mathrm{ref}}$ 是常见的缓解办法，因为它直接惩罚策略把概率质量过度集中的行为。
- **按 token 平均还是按序列平均。** 当响应有多个 token 时，$\log\pi_\theta(o\mid a,b)$ 是响应各 token
  对数概率之和，$\rho_i$ 也就是逐 token 相乘累积起来的；如果在整个 batch 里按 token 对代理目标取平均
  （而不是按响应取平均、或者先按响应再按 prompt 取平均），更长的响应仅仅因为包含更多 token，就会在梯度里
  获得更大的总权重，而与它们实际好不好无关。
- **GRPO 相对 PPO。** GRPO 完全去掉了 PPO 里学出来的价值网络（见 Part 2c），少训练一个模型，也去掉了
  GAE 自身的偏差-方差权衡，代价是每个 prompt 需要 $G$ 次完整 rollout 而不是 $1$ 次，而且基线只在同一个
  prompt 的各个响应之间做比较，对哪些 prompt 本身更难没有任何信号。
- **梯度范数裁剪。** 在 `opt.step()` 之前对 $\lVert\nabla_\theta L\rVert$ 施加一个全局上限，是另一种
  常见的稳定手段，和比率裁剪是相互独立的：比率裁剪限制的是单个样本的概率能移动多远，梯度裁剪限制的则是
  聚合之后那一次更新本身的大小，与是哪些样本产生了它无关。

<details>
<summary>验证代码（可运行）</summary>

```python
import copy

import torch
import torch.nn as nn
import torch.nn.functional as F

torch.manual_seed(0)
_PRIOR_TORCH_THREADS = torch.get_num_threads()  # NOTE: restored at the end of the checks below so this
torch.set_num_threads(1)                        # setting cannot leak into a later page's own torch calls

V = 10
D_MODEL = 16
HIDDEN = 32
CLIP_EPS = 0.2
KL_COEF = 0.01
ADV_EPS = 1e-6


class Policy(nn.Module):
    def __init__(self):
        super().__init__()
        self.embed_a = nn.Embedding(V, D_MODEL)
        self.embed_b = nn.Embedding(V, D_MODEL)
        self.net = nn.Sequential(nn.Linear(D_MODEL, HIDDEN), nn.ReLU(), nn.Linear(HIDDEN, V))

    def forward(self, a, b):                   # a, b: (N,) digit ids -> logits (N, V)
        return self.net(self.embed_a(a) + self.embed_b(b))


def sample_prompts(n, rng):                     # n random (a, b) prompts, each digit uniform in [0, V)
    a = torch.randint(0, V, (n,), generator=rng)
    b = torch.randint(0, V, (n,), generator=rng)
    return a, b


def reward(a, b, answer):                       # verifiable reward: 1.0 iff answer == (a + b) mod 10
    return (answer == (a + b) % V).float()


def group_advantages(rewards):                   # rewards: (P, G) -> advantages (P, G)
    mean = rewards.mean(dim=1, keepdim=True)
    std = rewards.std(dim=1, keepdim=True)
    return (rewards - mean) / (std + ADV_EPS)    # fix for bug 2: + ADV_EPS


def train(policy, ref_policy, steps=200, batch_size=100, group_size=16, inner_steps=4, lr=0.02, seed=0):
    opt = torch.optim.Adam(policy.parameters(), lr=lr)
    rng = torch.Generator().manual_seed(seed)
    for step in range(steps):
        a, b = sample_prompts(batch_size, rng)
        a_rep, b_rep = a.repeat_interleave(group_size), b.repeat_interleave(group_size)
        with torch.no_grad():
            logits_old = policy(a_rep, b_rep)
            probs_old = F.softmax(logits_old, dim=-1)                     # fix for bug 1: softmax before sampling
            actions = torch.multinomial(probs_old, 1, generator=rng).squeeze(1)
            logp_old = F.log_softmax(logits_old, dim=-1).gather(1, actions[:, None]).squeeze(1)
            logp_ref_all = F.log_softmax(ref_policy(a_rep, b_rep), dim=-1)
        r = reward(a_rep, b_rep, actions).view(batch_size, group_size)
        adv = group_advantages(r).reshape(-1)
        for _ in range(inner_steps):
            logits_new = policy(a_rep, b_rep)
            logp_new_all = F.log_softmax(logits_new, dim=-1)
            logp_new = logp_new_all.gather(1, actions[:, None]).squeeze(1)
            # fix for bug 3: no reassignment of logp_old here -- it stays the value from before the loop
            ratio = torch.exp(logp_new - logp_old)
            clipped = torch.clamp(ratio, 1 - CLIP_EPS, 1 + CLIP_EPS)
            pg_loss = -torch.min(ratio * adv, clipped * adv).mean()
            probs_new_all = logp_new_all.exp()
            kl = (probs_new_all * (logp_new_all - logp_ref_all)).sum(dim=-1).mean()
            loss = pg_loss + KL_COEF * kl
            opt.zero_grad()
            loss.backward()
            opt.step()
    return policy


@torch.no_grad()
def evaluate(policy, n, seed):                   # greedy accuracy on n freshly sampled held-out prompts
    rng = torch.Generator().manual_seed(seed)
    a, b = sample_prompts(n, rng)
    pred = policy(a, b).argmax(dim=-1)
    return reward(a, b, pred).mean().item()


if __name__ == "__main__":
    torch.manual_seed(0)
    policy = Policy()
    ref_policy = copy.deepcopy(policy)
    for p in ref_policy.parameters():
        p.requires_grad_(False)
    train(policy, ref_policy)
    accuracy = evaluate(policy, 2000, seed=999)
    print(f"held-out accuracy: {accuracy:.3f}")
    assert accuracy > 0.5, "the policy did not learn to add single digits"
    print("cleared the reward threshold")
```

```python
import warnings

# Part 1, statement: the fixed script clears the reward threshold with the seed used throughout this page
torch.manual_seed(0)
policy = Policy()
ref_policy = copy.deepcopy(policy)
for p in ref_policy.parameters():
    p.requires_grad_(False)
train(policy, ref_policy)
accuracy = evaluate(policy, 2000, seed=999)
assert accuracy > 0.5, f"held-out accuracy {accuracy:.3f} did not clear the threshold"

# Bug 1, reinserted in isolation: raw logits (which take both signs) crash torch.multinomial;
# softmax'd logits sample cleanly, exactly as Part 1's fix says
raw_logits = torch.tensor([[1.0, -0.5, 0.3, 2.0], [0.1, 0.2, -0.1, -0.4]])
try:
    torch.multinomial(raw_logits, 1)
    crashed_on_raw_logits = False
except RuntimeError:
    crashed_on_raw_logits = True
assert crashed_on_raw_logits, "sampling from raw logits should raise -- bug 1's exact failure mode"
sampled = torch.multinomial(F.softmax(raw_logits, dim=-1), 1)
assert sampled.shape == (2, 1)

# The advantage formula against the hand-computed example of the statement (G = 4, rewards (1, 0, 1, 0))
rewards_ex = torch.tensor([[1., 0., 1., 0.]])
adv_ex = group_advantages(rewards_ex)
expected_adv = torch.tensor([[3 ** 0.5 / 2, -(3 ** 0.5) / 2, 3 ** 0.5 / 2, -(3 ** 0.5) / 2]])
assert torch.allclose(adv_ex, expected_adv, atol=1e-4)


def buggy_group_advantages(rewards):             # bug 2 reinserted: no + ADV_EPS
    mean = rewards.mean(dim=1, keepdim=True)
    std = rewards.std(dim=1, keepdim=True)
    return (rewards - mean) / std


# Bug 2, reinserted in isolation: a group with a constant reward is nan without ADV_EPS, 0 with it;
# a group with genuine variance is unaffected either way
constant_group = torch.tensor([[1., 1., 1., 1.], [0., 1., 0., 1.]])
assert torch.isnan(buggy_group_advantages(constant_group)[0]).all()
assert not torch.isnan(group_advantages(constant_group)).any()
assert torch.allclose(buggy_group_advantages(constant_group)[1], group_advantages(constant_group)[1], atol=1e-4)

# Part 2d: G = 1 is nan regardless of ADV_EPS (the std estimator itself is 0/0, not merely the division)
with warnings.catch_warnings():
    warnings.simplefilter("ignore")              # torch warns about the degenerate degrees of freedom; expected here
    assert torch.isnan(group_advantages(torch.tensor([[1.0]]))).all()

# The clipped objective against the hand-computed example of the statement, all four clip/sign combinations
ratio_ex = torch.tensor([0.7, 1.5, 0.7, 1.5])
adv_ex2 = torch.tensor([1.0, 1.0, -1.0, -1.0])
clipped_ex = torch.clamp(ratio_ex, 1 - CLIP_EPS, 1 + CLIP_EPS)
surrogate_ex = torch.min(ratio_ex * adv_ex2, clipped_ex * adv_ex2)
assert torch.allclose(surrogate_ex, torch.tensor([0.7, 1.2, -0.8, -1.5]), atol=1e-6)
assert round((-surrogate_ex).mean().item(), 4) == 0.1


def run_inner_steps(policy, bug3, inner_steps=3, seed=1):
    """Three inner steps on one rollout batch, returning the ratio tensor computed at each step."""
    rng = torch.Generator().manual_seed(seed)
    a, b = sample_prompts(20, rng)
    a_rep, b_rep = a.repeat_interleave(8), b.repeat_interleave(8)
    opt = torch.optim.Adam(policy.parameters(), lr=0.05)
    with torch.no_grad():
        logits_old = policy(a_rep, b_rep)
        actions = torch.multinomial(F.softmax(logits_old, dim=-1), 1, generator=rng).squeeze(1)
        logp_old = F.log_softmax(logits_old, dim=-1).gather(1, actions[:, None]).squeeze(1)
    r = reward(a_rep, b_rep, actions).view(20, 8)
    adv = group_advantages(r).reshape(-1)
    ratios = []
    for _ in range(inner_steps):
        logp_new_all = F.log_softmax(policy(a_rep, b_rep), dim=-1)
        logp_new = logp_new_all.gather(1, actions[:, None]).squeeze(1)
        if bug3:
            logp_old = logp_new.detach()          # bug 3 reinserted
        ratio = torch.exp(logp_new - logp_old)
        ratios.append(ratio.detach().clone())
        clipped = torch.clamp(ratio, 1 - CLIP_EPS, 1 + CLIP_EPS)
        loss = -torch.min(ratio * adv, clipped * adv).mean()
        opt.zero_grad()
        loss.backward()
        opt.step()
    return ratios


# Bug 3, reinserted in isolation: both variants start at ratio == 1 exactly (Part 2a); from the second
# inner step on, the correct policy's ratio must move away from 1, while the buggy one stays pinned to it
torch.manual_seed(2)
policy_correct = Policy()
policy_buggy = copy.deepcopy(policy_correct)     # identical starting weights, so only the bug differs
ratios_correct = run_inner_steps(policy_correct, bug3=False)
ratios_buggy = run_inner_steps(policy_buggy, bug3=True)
ones = torch.ones(160)
assert torch.allclose(ratios_correct[0], ones, atol=1e-6)
assert torch.allclose(ratios_buggy[0], ones, atol=1e-6)
for step in (1, 2):
    assert not torch.allclose(ratios_correct[step], ones, atol=1e-3), "correct: ratio should move away from 1"
    assert torch.allclose(ratios_buggy[step], ones, atol=1e-6), "buggy: ratio should stay pinned to 1"

torch.set_num_threads(_PRIOR_TORCH_THREADS)  # NOTE: undo the pin so it cannot leak into another page

print("all checks passed")
```

</details>

</details>

