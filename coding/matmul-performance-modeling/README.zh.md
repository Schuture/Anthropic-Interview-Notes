# 性能建模：矩阵乘法、Roofline 与切分

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 估算 · Roofline 模型，无代码 | ★★☆☆☆ | 困难 | Perf Eng · MLE · Infra Eng | roofline, arithmetic-intensity, gpu-performance, parallelism, ffn-sharding | 4 个部分 / 60 分钟 | 现场面（VO） |
<!-- meta:end -->

## 题目

一块加速卡的峰值密集 bf16 算力为 $P = 312$ TFLOP/s（即 $312\times10^{12}$ FLOP/s），HBM 带宽为
$BW = 1.5$ TB/s（即 $1.5\times10^{12}$ 字节/秒），HBM 容量为 $40$ GB（本页全文约定 $1$ GB $=10^9$
字节）。两块或更多这样的加速卡之间由一条互连（interconnect）连接：任意两块设备之间，该互连能同时、独立
地支持 $B_{\text{net}} = 400$ GB/s 的发送流量和 $400$ GB/s 的接收流量——即一条全双工（full-duplex）链
路，一块设备同时发送和接收时，不会因为共用同一份带宽预算而被限速。本题中出现的每一个张量——无论是权重、
激活值，还是正在互连上传输的值——都以 bf16 存储，每元素 $s = 2$ 字节，除非某一部分另有说明。

对矩阵乘法 $C = AB$，其中 $A \in \mathbb{R}^{m\times k}$，$B \in \mathbb{R}^{k\times n}$，
$C \in \mathbb{R}^{m\times n}$：

- *FLOPs*：把一次乘法和一次加法算作两次 FLOP，$\text{FLOPs} = 2mkn$。
- *搬运的字节数*（bytes moved）：从 HBM 各读一次 $A$ 和 $B$，写一次 $C$，每元素 $s$ 字节，
  $\text{bytes} = s(mk+kn+mn)$。两个操作数都不会被重复读取，除 $C$ 之外也不再写任何内容。
- *计算强度*（arithmetic intensity）$I = \text{FLOPs}/\text{bytes}$，单位 FLOP/byte。
- *拐点*（ridge point）$I^\star = P/BW$ 是计算时间与搬运时间相等时的强度。$I > I^\star$ 的矩阵乘法是
  *计算受限*（compute-bound）；$I < I^\star$ 的是*显存带宽受限*（memory-bound）。
- *预测时间* $t = \max(\text{FLOPs}/P,\ \text{bytes}/BW)$，假定一个 kernel 内部计算与 HBM 流量完全重叠。

### Part 1 —— 四个矩阵乘法

| 形状 | $m$ | $k$ | $n$ |
| --- | --- | --- | --- |
| A | 8 | 4,096 | 4,096 |
| B | 2,048 | 4,096 | 4,096 |
| C | 4,096 | 64 | 4,096 |
| D | 4,096 | 4,096 | 4,096 |

对每个形状，给出 FLOPs、搬运的字节数、计算强度、瓶颈类型（bound regime），以及预测时间。然后，对一般的
$(m,k,n)$，说明单独增大 $m$、$k$ 或 $n$（固定另外两个）是否可能把一个矩阵乘法从计算受限变成显存带宽受
限，还是只可能反过来，并对三个变量都给出论证，不要只论证 A—D 四个形状直接体现的那两个。

### Part 2 —— 72 层堆叠

一个网络有 $L = 72$ 层，每一层都是独立的 kernel，层与层之间不做融合（fusion）。第 $\ell$ 层接收输入
$X_\ell$ 和权重 $W_\ell$，产生 $X_{\ell+1}$；这里没有注意力、没有归一化、没有残差连接、也没有偏置，所以
$X_{\ell+1}$ 是 $X_\ell$ 唯一的后继，$X_\ell$ 唯一的来源是 $X_{\ell-1}$（对 $\ell = 1$ 而言，来源就是网络
自身的输入）。$X_{\ell+1}$ 由第 $\ell$ 层写入 HBM，再由第 $\ell+1$ 层从 HBM 读回——和一次普通矩阵乘法的
输出、下一次的输入完全一样。各层在两种形状之间交替，两者都由 $T = 20{,}480$（这一次前向传播中一起处理的
token 数）、$d = 8{,}192$ 和 $f = 32{,}768$ 构成：

- 奇数 $\ell$（*U 型*）：$X_\ell \in \mathbb{R}^{T\times d}$，$W_\ell \in \mathbb{R}^{d\times f}$，
  $X_{\ell+1} = X_\ell W_\ell \in \mathbb{R}^{T\times f}$。
- 偶数 $\ell$（*D 型*）：$X_\ell \in \mathbb{R}^{T\times f}$，$W_\ell \in \mathbb{R}^{f\times d}$，
  $X_{\ell+1} = X_\ell W_\ell \in \mathbb{R}^{T\times d}$。

（$X_\ell W_\ell$ 和 $X_{\ell+1}$ 之间可能夹着一个逐元素的非线性；它不是矩阵乘法，其开销不计入。）全部
72 个权重在整个前向传播期间都常驻在一块加速卡上。给出这 72 层一次前向传播的总计算时间（每层的 FLOPs 除
以 $P$ 之和）和总搬运时间（每层的字节数除以 $BW$ 之和）。然后确定运行这次传播、任意时刻必须同时常驻的张
量究竟是哪些，它们的总量能否放进这块加速卡的 40 GB，超出或留有多少余量。

### Part 3 —— 两块设备，一个输入

同一个 72 层堆叠处理一个 $T = 20{,}480$ token 的输入，现在切到两块相同的、由互连相连的加速卡上。比较
两种方案。

*流水线并行*（pipeline parallelism）：设备 $A$ 持有第 1—36 层及其权重；设备 $B$ 持有第 37—72 层及其权
重。设备 $A$ 依次算完它的 36 层，把得到的激活值通过互连发给设备 $B$，设备 $B$ 再算它的 36 层。

*张量并行*（tensor parallelism）：每一层的权重都沿其输出轴切成 2 份相等的分片——U 型层沿它的 $f$ 列切，
D 型层沿它的 $d$ 列切——每块设备一份；两块设备都持有网络输入 $X_1$ 完整、相同的一份副本。对每一层，设备
用自己已经持有的完整输入算出自己那一份输出分片，这一步不需要通信；然后两块设备通过互连交换各自的分片，
使双方重新都持有这一层完整的输出，才开始下一层。

对每种方案，给出这一个输入的端到端延迟、任一设备上的峰值显存占用——说清楚具体是哪些张量常驻——以及整
个过程中任一设备通过互连发送的总字节数。说明单个请求应该偏好哪种方案，以及为什么实践中另一种方案仍然会
被使用。

### Part 4 —— 切分前馈网络

一个前馈网络计算 $Y = \operatorname{act}(XW_1)W_2$，其中 $X \in \mathbb{R}^{T\times d}$，
$W_1 \in \mathbb{R}^{d\times f}$，$\operatorname{act}(XW_1) \in \mathbb{R}^{T\times f}$，
$W_2 \in \mathbb{R}^{f\times d}$，$Y \in \mathbb{R}^{T\times d}$，$d = 8{,}192$、$f = 32{,}768$ 与
Part 2 相同，$\operatorname{act}$ 是逐元素的非线性。把每次乘加算作两次 FLOP，不计 $\operatorname{act}$
和偏置，这一步的开销是 $4Tdf$ FLOPs；两个权重合计占用 $2dfs$ 字节（bf16 下 $s = 2$）。考虑两种把这一块
切到 $p$ 块设备上的方式；假定 $p$ 能整除 $f$（张量切分）或 $T$（token 切分），下面用到的每个 $p$ 都满
足这一点。

*张量切分*（tensor sharding）：$W_1$ 的 $f$ 列和 $W_2$ 的 $f$ 行各自切成 $p$ 份相等的分片，每块设备一
份；$X$ 在每块设备上都有副本。设备 $i$ 在本地算出 $H_i = \operatorname{act}(XW_{1,i}) \in
\mathbb{R}^{T\times f/p}$，再在本地算出一个部分输出 $Y_i = H_iW_{2,i} \in \mathbb{R}^{T\times d}$。真正
的 $Y = \sum_i Y_i$ 由对这 $p$ 个部分输出做一次 all-reduce 得到，每个部分输出的逻辑大小是 $T \times d$，
通信用的元素占 $s_c$ 字节（除非另有说明，$s_c = s = 2$）；理想的环形（ring）all-reduce 中，每块设备发
送 $2(p-1)/p$ 倍于该逻辑大小的数据，接收同样多，这次 all-reduce 不能与这一块自身的计算重叠。

*token 切分*（token sharding）：$W_1$、$W_2$ 在每块设备上都各有一份完整副本；$T$ 的行切成 $p$ 组相等
的分组，每块设备一组，各自在自己的 $T/p$ 行上算完整的两次矩阵乘法，这一块内部不需要通信。

取 $T = 2{,}048$ 个 token（一次批量 decode 步，比 Part 2 的 prefill 小）。对
$p = 1, 2, 4, 8, 16, 32, 64$，给出两种方案每块设备的 FLOPs、权重字节数和预测延迟，并给出张量切分下每块
设备的通信量及其在 Part 3 那条 $B_{\text{net}} = 400$ GB/s 互连上所需的时间。对每个 $p$，说明张量切分是
计算受限、显存带宽受限还是通信受限，并对 token 切分做同样的判断。在一条更慢的跨节点互连上
（$B_{\text{net}}' = 25$ GB/s 取代 $B_{\text{net}}$）对张量切分在 $p = 2, 4, 16, 64$ 重复这一判断。最后
说明，既然上面这一块自身无法与 all-reduce 重叠，在真实部署中什么（如果有的话）能让一次 all-reduce 的通
信与计算重叠起来。

## 参考解答

<details>
<summary>展开参考解答</summary>

先向面试官确认两点：当连续两层共享同一个张量时，“搬运的字节数”是否仍按每个矩阵乘法各自的输入和输出计
价（本页的约定：是——kernel 之间不做融合，一个被共享的激活值写一次、读一次），以及拐点是否只由硬件决
定。确实如此：$I^\star = P/BW = 312\times10^{12} / 1.5\times10^{12} = 208$ FLOP/byte，下面每一部分都
固定用这个数。

### Part 1

**形状 A** 把公式完整地走一遍。取 $(m,k,n) = (8, 4{,}096, 4{,}096)$：

$$\text{FLOPs} = 2\cdot 8\cdot 4{,}096\cdot 4{,}096 = 268{,}435{,}456,$$

$$\text{bytes} = 2\bigl(8\cdot4{,}096 + 4{,}096\cdot4{,}096 + 8\cdot4{,}096\bigr) = 2\cdot 16{,}842{,}752 = 33{,}685{,}504,$$

$$I = \frac{268{,}435{,}456}{33{,}685{,}504} \approx 7.97 \text{ FLOP/byte} < 208 = I^\star \implies \text{显存带宽受限},$$

$$t = \max\!\left(\frac{268{,}435{,}456}{312\times10^{12}},\ \frac{33{,}685{,}504}{1.5\times10^{12}}\right) = \max(0.86,\ 22.46)\ \mu s = 22.46\ \mu s.$$

同样的公式，用在 B、C、D 上：

| 形状 | FLOPs | bytes | $I$（FLOP/byte） | 瓶颈 | $t$ |
| --- | --- | --- | --- | --- | --- |
| A | $2.684\times10^{8}$ | $3.369\times10^{7}$ | 7.97 | 显存带宽 | 22.46 μs |
| B | $6.872\times10^{10}$ | $6.711\times10^{7}$ | 1,024.00 | 计算 | 220.25 μs |
| C | $2.147\times10^{9}$ | $3.460\times10^{7}$ | 62.06 | 显存带宽 | 23.07 μs |
| D | $1.374\times10^{11}$ | $1.007\times10^{8}$ | 1,365.33 | 计算 | 440.51 μs |

B 是把 A 的 $m$ 从 8 增大到 2,048、同时保持 $k=n=4{,}096$ 不变：强度从 7.97 升到 1,024，瓶颈从显存带宽
翻转为计算。C 和 D 展示的是沿 $k$ 的同一种翻转，保持 $m=n=4{,}096$ 不变：$k$ 从 64 增大到 4,096，强度
从 62.06 升到 1,365.33。这两次翻转都是同一个事实的实例，下面对每一个变量（不只是 $m$ 和 $k$）证明这一
点。

固定 $k,n$，强度作为 $m$ 的函数是 $I(m) = \dfrac{2mkn}{s(m(k+n)+kn)} = \dfrac{2kn}{s}\, g(m)$，其中
$g(m) = \dfrac{m}{m(k+n)+kn}$。于是

$$g'(m) = \frac{\bigl(m(k+n)+kn\bigr) - m(k+n)}{\bigl(m(k+n)+kn\bigr)^{2}} = \frac{kn}{\bigl(m(k+n)+kn\bigr)^{2}} > 0,$$

所以 $I$ 关于 $m$ 严格递增，从 $I(0) = 0$ 一路升向 $\lim_{m\to\infty} I(m) = 2kn/(s(k+n))$——即 $k$ 与
$n$ 的调和平均数，再除以 $s$——只能无限逼近，达不到。字节数 $s(mk+kn+mn)$ 和 FLOPs $2mkn$ 都是
$(m,k,n)$ 的对称函数——前者是三个两两乘积的初等对称和，后者是三者乘积的两倍——所以 $I(m,k,n)$ 也是对
称的，上面的论证原封不动地适用：把 $m, k, n$ 中的任意一个当作“$m$”，另外两个当作“$k, n$”都成立。单独
增大 $m, k, n$ 中的任意一个、固定另外两个，都会严格提高强度，因此只可能把一个矩阵乘法推向计算受限，绝
不会推向相反的方向——这一点对三个变量中的每一个都成立。但达到计算受限并没有保证：单独增大一个变量，只
是把 $I$ 推向由另外两个变量决定的上限 $2xy/(s(x+y))$，如果这个上限本身已经低于 $I^\star$，那么无论第
三个变量增大多少都跨不过拐点。A 和 C 的“另外两个”都是 $\{4{,}096, 4{,}096\}$，所以上限相同，都是
$2\cdot4{,}096^2/(2\cdot 8{,}192) = 2{,}048$ FLOP/byte，比 $I^\star = 208$ 高出不少：$m$ 从 B 的 2,048
继续增大，或者 $k$ 从 D 的 4,096 继续增大，都会继续推向计算受限，逼近同一个上限，因为两者都是上面这条
对称事实的实例。

### Part 2

不论哪一种类型，每一层的 FLOPs 和搬运字节数都相同：$\text{FLOPs}_\ell = 2Tdf =
2\cdot20{,}480\cdot8{,}192\cdot32{,}768 = 10{,}995{,}116{,}277{,}760$，
$\text{bytes}_\ell = s(Td+df+Tf) = 2(20{,}480\cdot8{,}192 + 8{,}192\cdot32{,}768 +
20{,}480\cdot32{,}768) = 2{,}214{,}592{,}512$——D 型层只是把这三个乘积里哪个是输入、哪个是输出对调了
一下，$\{Td, df, Tf\}$ 这个集合两种情况下是一样的。强度 $I_\ell = 10{,}995{,}116{,}277{,}760 /
2{,}214{,}592{,}512 \approx 4{,}964.85$ FLOP/byte，远高于 $I^\star = 208$，所以每一层都是计算受限的：
预测时间只由计算决定，$t_\ell = \text{FLOPs}_\ell/P \approx 35.24$ ms（单看搬运只需要
$\text{bytes}_\ell/BW \approx 1.48$ ms）。

72 层的开销完全相同，所以

$$\text{总计算时间} = \frac{72\cdot\text{FLOPs}_\ell}{P} \approx 2{,}537.33\text{ ms}, \qquad
\text{总搬运时间} = \frac{72\cdot\text{bytes}_\ell}{BW} \approx 106.30\text{ ms}.$$

总搬运时间只占总计算时间的约 4.2%，所以——和单层的情形一样——整个前向传播也是计算受限的，预测延迟就
是这个 2,537.33 ms 的计算数字，而不是两者之和。

**显存。** 这次传播中任意时刻必须常驻的张量是：(a) 全部 72 个权重矩阵，整个传播期间都要常驻，因为传播
结束前每一个都还会再用到；(b) 当前正在执行那一层的输入 $X_\ell$ 和输出 $X_{\ell+1}$——更早的激活值已
经被消费并释放，更晚的还没产生。(a) 合计 $72dfs = 72\cdot8{,}192\cdot32{,}768\cdot2 =
38{,}654{,}705{,}664$ 字节 $\approx 38.65$ GB，每一层这个数字都相同，因为 U 型和 D 型的权重都是 $df$
个元素。(b) 合计 $Ts(d+f) = 20{,}480\cdot2\cdot40{,}960 = 1{,}677{,}721{,}600$ 字节 $\approx 1.68$
GB——每一层也都相同，因为 U 型层的输入加输出是 $T(d+f)$ 个元素，D 型层的是 $T(f+d)$ 个元素，两者的和
相等。加在一起，权重加上某一层的存活激活值共 $38.65 + 1.68 \approx 40.33$ GB。

这超出 40 GB 的预算约 $0.33$ GB——肉眼很容易漏看，因为单看权重的话还有 $40 - 38.65 \approx 1.35$ GB 的
余量。真正把总量推过预算的，恰恰是某一层那 1.68 GB 的存活激活值（由 $T=20{,}480$ 个 token 的 prefill
决定），超过了那份余量。这正是 Part 3 要把这个堆叠切到两块设备上的原因。

### Part 3

**流水线并行。** 每一层的开销都是同一个 $t_\ell \approx 35.24$ ms（Part 2），所以每块设备的 36 层共需
$36\,t_\ell \approx 1{,}268.67$ ms——正好是单设备总时间的一半，因为单设备总时间就是 $72\,t_\ell$。第
36 层是偶数层，即 D 型层，所以被转交的激活值形状是 $T\times d$：$\text{bytes} = sTd =
2\cdot20{,}480\cdot8{,}192 = 335{,}544{,}320 \approx 335.54$ MB，通过互连传输需要
$335{,}544{,}320 / (400\times10^{9}) \approx 0.84$ ms。端到端延迟：

$$t_{\text{PP}} = 36\,t_\ell + \frac{sTd}{B_{\text{net}}} + 36\,t_\ell = 72\,t_\ell + \frac{sTd}{B_{\text{net}}}.$$

$72\,t_\ell$ 正好就是 Part 2 的总计算时间，$\approx 2{,}537.33$ ms；再加上这一次转交，
$\approx 0.84$ ms，得到 $t_{\text{PP}} \approx 2{,}538.17$ ms。把层切到两块设备上，完全没有降低单个输
入的延迟，因为两块设备仍然是先后依次运行的。

每块设备持有一半的权重，$38.65/2 \approx 19.33$ GB，加上某一层的存活激活值 $\approx 1.68$ GB（同一时刻
每块设备上只有一层在执行）：合计 $\approx 21.01$ GB，远低于 40 GB。单独一层自己的输入、输出和权重加在
一起只有 $sTd + sTf + sdf \approx 2.21$ GB，随手就能和其余部分一起常驻。整个传播期间通过互连的总流量
就是那一次转交：每块设备 $\approx 335.54$ MB。

**张量并行。** 一个 U 型层，切成 2 份：每块设备读完整的输入（$Td$ 个元素）、自己那一半权重（$df/2$ 个
元素），写自己那一半输出（$Tf/2$ 个元素）；FLOPs 正好减半，为 $Tdf$。字节数：$s(Td + df/2 + Tf/2) =
2(20{,}480\cdot8{,}192 + 8{,}192\cdot16{,}384 + 20{,}480\cdot16{,}384) = 1{,}275{,}068{,}416$。减半后的
FLOPs 和这个字节数给出的强度仍然远高于 $I^\star$，所以这个分片依然是计算受限的，
$t = Tdf/P \approx 17.62$ ms。D 型层用同样的方式切分，FLOPs 相同（$Tfd = Tdf$），字节数不同——
$s(Tf + fd/2 + Td/2) = 1{,}778{,}384{,}896$——但同样在 $t \approx 17.62$ ms 处计算受限（计算这一项两
种情形下相同，且已经超过两个字节数对应的搬运时间）。每一层结束后，两块设备交换各自的输出分片：由于每
块设备本来就已经持有输出里互不重叠的最终一部分，这里只需要一次纯粹的 all-gather，而不是完整的
all-reduce——不需要 reduce-scatter 那一步，通信量只有 Part 4 环形 all-reduce 的一半；$p=2$ 时，这意味
着每块设备只需发送自己那一份分片。U 型层的输出有 $Tf$ 个元素，每块设备发送 $sTf/2 = 671{,}088{,}640$
字节 $\approx 671.09$ MB，需要 $\approx 1.68$ ms；D 型层的输出有 $Td$ 个元素，每块设备发送
$sTd/2 = 167{,}772{,}160$ 字节 $\approx 167.77$ MB，需要 $\approx 0.42$ ms。对两种类型各 36 层求和，交换的时间要加在每一层自己的计算之后，因为下一层需要
汇总完的结果：U 型层每层耗时 $17.62 + 1.68 = 19.298$ ms，36 层合计 $\approx 694.73$ ms；D 型层每层耗
时 $17.62 + 0.42 = 18.040$ ms，36 层合计 $\approx 649.43$ ms；两者相加，

$$t_{\text{TP}} \approx 694.73 + 649.43 = 1{,}344.16\text{ ms}.$$

张量并行处理这一个输入的时间约为单设备时间的 $1{,}344.16 / 2{,}537.33 \approx 53.0\%$——接近一半，因为
计算确实被两块设备真正分担了——代价是在全部 72 层的每一层都要通信。每块设备通过互连的总流量是
$36\cdot671.09\text{ MB} + 36\cdot167.77\text{ MB} \approx 30.20$ GB——约为流水线并行那一次 335.54 MB
转交的 90 倍。显存方面：每份权重都减半（$38.65/2 \approx 19.33$ GB），而每一层的交换一旦完成，两块设
备持有的都是这一层完整的输入和输出，不是一半——所以存活激活值的数字和单设备、流水线并行时一样，仍是
$\approx 1.68$ GB，不是它的一半。每块设备的显存因而也是 $\approx 21.01$ GB——数值上和流水线并行完全相
同，因为这里两种方案恰好把同样的权重字节数减了半，激活值的占用谁都没有压缩。

**权衡。** 对这一个输入而言，张量并行在这里算出的每一个数字上都全面压过流水线并行：延迟更低，显存占用
相同。流水线并行真正的优势，在单个输入的场景下根本不会体现出来：如果有好几个输入同时在系统里，设备
$B$ 可以在算完输入 1 的后半段的同时，就开始算输入 2 的前半段，让流水线始终满载，提高吞吐而不增加任何
通信——相比之下，张量并行强迫两块设备在每一层上步调一致地推进，所以它无法用这种方式重叠不同的输入，而
且不论同时有多少个输入在跑，每一层都要付出通信的代价。实际部署通常两者都用：用适度的张量并行让一层在
一条快速的单节点互连内既放得下又算得快，再在此基础上叠加流水线并行来提升吞吐，并支撑一个连张量并行组
的总显存都放不下的模型。

### Part 4

**张量切分。** 每块设备的 FLOPs 是 $4Tdf/p$，平均分给两次矩阵乘法，各占 $2Tdf/p$；每块设备的权重字节
数是 $2dfs/p$。$p=1$ 时就是不切分的那一块：$4Tdf/P = 4\cdot2{,}048\cdot8{,}192\cdot32{,}768 /
(312\times10^{12}) \approx 7.05$ ms，完全是计算（在这个规模下强度远高于 $I^\star$），权重
$2dfs \approx 1.07$ GB。$p>1$ 时，每块设备自身的计算加显存时间，是它两次矩阵乘法各自 roofline 时间之
和（在下面用到的每个 $p$ 上，两次矩阵乘法都依然各自计算受限，因为 $d,f$ 都很大），再加上对
$T\times d$ 部分输出做 all-reduce 的开销——逻辑大小 $S = Tds = 2{,}048\cdot8{,}192\cdot2 =
33{,}554{,}432$ 字节——这部分开销要叠加在上面，因为它无法与这一块自身的计算重叠：
$2(p-1)S/(p\,B_{\text{net}})$。

| $p$ | 每设备 FLOPs | 每设备权重 | 每设备通信量 | $t_{\text{comm}}$ | $t_{\text{total}}$ | 瓶颈 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | $2.199\times10^{12}$ | 1.0737 GB | 0 | 0 ms | 7.05 ms | 计算 |
| 2 | $1.100\times10^{12}$ | 0.5369 GB | 33.55 MB | 0.08 ms | 3.61 ms | 计算 |
| 4 | $5.498\times10^{11}$ | 0.2684 GB | 50.33 MB | 0.13 ms | 1.89 ms | 计算 |
| 8 | $2.749\times10^{11}$ | 0.1342 GB | 58.72 MB | 0.15 ms | 1.03 ms | 计算 |
| 16 | $1.374\times10^{11}$ | 0.0671 GB | 62.91 MB | 0.16 ms | 0.60 ms | 计算 |
| 32 | $6.872\times10^{10}$ | 0.0336 GB | 65.01 MB | 0.16 ms | 0.38 ms | 计算 |
| 64 | $3.436\times10^{10}$ | 0.0168 GB | 66.06 MB | 0.17 ms | 0.28 ms | 通信 |

（每设备通信量是发送的字节数 $2(p-1)S/p$；$t_{\text{comm}}$ 用 $B_{\text{net}} = 400$ GB/s 算出。）
$p$ 每翻一倍，计算时间就减半；通信则只是从 0 慢慢升向它的上限 $2S/B_{\text{net}} \approx 0.17$ ms，所
以直到 $p=32$，计算都还是更大的一项；只有过了 $p=43$（把这张表继续往下延伸就能找到）通信才会超过计算
加显存，这也是为什么表格最后一行已经写着“通信”。延迟在整张表里一直在下降，但收益递减得很明显：
$p=32\to64$ 把设备数翻倍，通信量也大致翻倍，延迟却只从 0.38 降到 0.28 ms，降幅 28%，即便每设备权重同
样精确地减了半（0.0336 GB 到 0.0168 GB）。更小的权重占用对更小的显存预算是必要的，但一旦通信在总时间
里占了不小的比例，它本身并不能带来成比例的延迟下降。

**token 切分。** 每块设备的 FLOPs 和张量切分一样，都是 $4Tdf/p$（总工作量相同，分法也相同），但每块
设备的权重字节数始终是完整、不切分的 $2dfs \approx 1.07$ GB，不随 $p$ 变化，因为 $W_1, W_2$ 是复制而
不是切分的，这一块内部也没有通信。每块设备的时间依然是它两次矩阵乘法各自的 roofline 时间，只不过现在
过的是 $T/p$ 行数据，经过的却是*完整*的权重：$p=1$ 时和张量切分的 $p=1$ 那一行完全相同，7.05 ms，计算
受限；到 $p=8$ 时已经降到 0.88 ms，仍然计算受限；但从 $p=16$ 开始就变成显存带宽受限，0.73 ms，之后几
乎不再下降（$p=32$ 是 0.72 ms，$p=64$ 也是 0.72 ms）——原因是不论一块设备只分到多少行，它都要把*完整*
的 $W_1$、$W_2$ 从 HBM 读一遍，这就有一个 $2dfs/BW \approx 0.72$ ms 的下限，再怎么加大切分也去不掉。张
量切分没有这样的下限，因为它的权重字节数也会随 $p$ 一起缩小，这正是它能一路改善到 $p=64$ 的 0.28 ms、
远远超过 token 切分停滞之处的原因。

**更慢的互连。** 把张量切分的瓶颈判断换成 $B_{\text{net}}' = 25$ GB/s（取代 400 GB/s）重新算一遍，计
算和权重字节数都不变，只有 $t_{\text{comm}} = 2(p-1)S/(p\,B_{\text{net}}')$ 变了——每个 $p$ 上都变成
原来的 16 倍：

| $p$ | $t_{\text{comm}}$ | $t_{\text{total}}$ | 瓶颈 |
| --- | --- | --- | --- |
| 2 | 1.34 ms | 4.87 ms | 计算 |
| 4 | 2.01 ms | 3.78 ms | 通信 |
| 16 | 2.52 ms | 2.96 ms | 通信 |
| 64 | 2.64 ms | 2.75 ms | 通信 |

这里到 $p=4$ 通信就已经占了主导，$p=16\to64$——设备数翻了四倍——延迟却只从 2.96 ms 降到 2.75 ms，改
善 7%，而且这点改善几乎全部来自一个还在缩小的计算项，通信本身几乎是一条平线。结论和上面一样，只是更
尖锐：一旦越过通信占主导的那个点，再加设备主要加的是通信，不是速度。

**重叠。** 对处理一批 token 的这一个模块而言，all-reduce 无法与这一块自身的计算重叠：$Y_i$ 需要
$H_i$，$H_i$ 需要 $X$，而下游要用 $Y$ 就必须等 all-reduce 做完，每一步都在同一条依赖链上。要重叠，需
要有独立的工作来填补空档。两种真实存在的来源：(a) 同时有多个互相独立的请求或 micro-batch 在跑——某个
batch 的 all-reduce 还在进行时，设备已经可以开始跑另一个独立 batch 的本地矩阵乘法，和 Part 3 里流水线
并行重叠不同输入是一回事；(b) 把一个 batch 自己的归约按列切成若干块，让它和计算交错进行——$Y_i$ 剩下
的列还在算的时候，就开始归约已经算完的那部分列——这样能缩短这次 collective 暴露在外、无法重叠的那一
段尾巴，但不会改变它总共要搬运的字节数。

### 追问

- **批大小与 decode。** decode 一步的矩阵乘法形状很像形状 A：token 数（$m$）很少，权重却很大，所以强
  度很低，时间主要花在从 HBM 读权重上，几乎和 $m$ 无关（字节数被 $kn$ 那一项——权重——主导，$m$ 太小
  影响不了它）。把多个请求的 decode 步一起打包，会在不增加额外权重流量的情况下提高 $m$，于是 FLOPs 跟
  着涨、字节数却几乎不动——强度朝着 Part 1 证明过的那个上限升去——吞吐随批大小近乎线性地提高，直到逼
  近上限、这一步变成计算受限为止，再往后继续加大批大小只会增加延迟，换不来多少吞吐。
- **为什么达不到 roofline。** 真实 kernel 很少能打满整个 $\max(\text{FLOPs}/P,\text{bytes}/BW)$：某个
  维度除不尽硬件的 tile 大小时，不完美的分块（tiling）会在 FLOPs 或字节数上浪费一部分；kernel 启动本
  身有固定开销，一次很短的矩阵乘法摊不掉；真实硬件里计算和显存访问也做不到完全重叠；一次网络
  collective 还要在这页算的字节数之外，另外付出协议和同步开销。roofline 给出的是时间的下界，不是对时
  间的预测。
- **更低的精度。** 如果硬件让 fp8 的算力大致是 bf16 的两倍、字节数却仍按 bf16 计（只有乘加本身降到
  fp8，权重和激活值依然按每元素 2 字节读写），那就是只让 $P$ 翻倍、$BW$ 不变——拐点也跟着翻倍，一些在
  bf16 下稳稳计算受限的矩阵乘法，到了 fp8 下反倒变成显存带宽受限，快不了多少。只有当存储也一并量化到
  每元素 1 字节时，fp8 才能拿到接近完整的加速：字节数跟着算力一起减半，强度和瓶颈类型都不变。
- **KV cache 的读取。** decode 阶段的注意力要读一个随已生成序列长度增长的 key/value 缓存，这是在本页
  已经计入的权重字节数之外的开销；对话越长，每个 decode 步要读的缓存字节就越多，FLOPs 却几乎不变，把
  强度进一步往下压，使 decode 随序列变长而越来越偏向显存带宽受限——这和上面批大小的效应是两回事，各自
  独立发挥作用。

<details>
<summary>验证代码（可运行）</summary>

```python
import itertools
import math
import random

# ---- hardware constants ----
P = 312e12          # peak dense bf16 FLOP/s
BW = 1.5e12         # HBM bandwidth, bytes/s
CAP = 40e9          # HBM capacity, bytes
Bnet = 400e9        # interconnect, bytes/s per direction
Bnet_slow = 25e9    # slower, cross-node interconnect
s = 2               # bf16 bytes/element

ridge = P / BW
assert ridge == 208.0


def matmul_stats(m, k, n, elem=2):
    flops = 2 * m * k * n
    bytes_ = elem * (m * k + k * n + m * n)
    return flops, bytes_, flops / bytes_, max(flops / P, bytes_ / BW), flops / P, bytes_ / BW


# ---------------------------------------------------------------- Part 1 ----
shapes = {"A": (8, 4096, 4096), "B": (2048, 4096, 4096), "C": (4096, 64, 4096), "D": (4096, 4096, 4096)}
expected_p1 = {
    "A": (268_435_456, 33_685_504, 7.97, "memory", 22.46),
    "B": (68_719_476_736, 67_108_864, 1024.00, "compute", 220.25),
    "C": (2_147_483_648, 34_603_008, 62.06, "memory", 23.07),
    "D": (137_438_953_472, 100_663_296, 1365.33, "compute", 440.51),
}
for name, (m, k, n) in shapes.items():
    flops, bytes_, I, t, tc, tm = matmul_stats(m, k, n)
    e_flops, e_bytes, e_I, e_regime, e_t_us = expected_p1[name]
    assert flops == e_flops and bytes_ == e_bytes
    assert round(I, 2) == e_I
    assert ("compute" if I > ridge else "memory") == e_regime
    assert round(t * 1e6, 2) == e_t_us

flopsA, bytesA, IA, tA, tcA, tmA = matmul_stats(*shapes["A"])
assert round(tcA * 1e6, 2) == 0.86 and round(tmA * 1e6, 2) == 22.46

# the general lemma: I(m,k,n) is symmetric and strictly increasing in each argument
rng = random.Random(0)


def intensity(m, k, n, elem=2):
    return 2 * m * k * n / (elem * (m * k + k * n + m * n))


for _ in range(300):
    m, k, n = (rng.uniform(1, 5000) for _ in range(3))
    I0 = intensity(m, k, n)
    delta = rng.uniform(1e-3, 100)
    for which in range(3):
        vals = [m, k, n]
        vals[which] += delta
        assert intensity(*vals) > I0

for _ in range(100):
    m, k, n = (rng.uniform(1, 5000) for _ in range(3))
    values = {round(intensity(*perm), 6) for perm in itertools.permutations([m, k, n])}
    assert len(values) == 1

# A and C share the ceiling 2*4096^2 / (s*8192) = 2048, approached but never reached as the free variable grows
k4, n4 = 4096, 4096
ceiling = 2 * k4 * n4 / (s * (k4 + n4))
assert ceiling == 2048
prev_gap = math.inf
for big in (1e6, 1e9, 1e12):
    gap = (ceiling - intensity(big, k4, n4)) / ceiling
    assert 0 < gap < prev_gap
    prev_gap = gap
assert prev_gap < 1e-6

# ---------------------------------------------------------------- Part 2 ----
L, T, d, f = 72, 20_480, 8_192, 32_768
flops_layer = 2 * T * d * f
bytes_layer = s * (T * d + d * f + T * f)
assert flops_layer == 10_995_116_277_760
assert bytes_layer == 2_214_592_512
I_layer = flops_layer / bytes_layer
assert round(I_layer, 2) == 4964.85
assert I_layer > ridge
t_layer = flops_layer / P            # compute-bound, so this is the layer's predicted time
tm_layer = bytes_layer / BW
assert round(t_layer * 1e3, 2) == 35.24
assert round(tm_layer * 1e3, 2) == 1.48

total_flops = L * flops_layer
total_bytes = L * bytes_layer
total_compute_time = total_flops / P
total_transfer_time = total_bytes / BW
assert round(total_compute_time * 1e3, 2) == 2537.33
assert round(total_transfer_time * 1e3, 2) == 106.30
assert round(100 * total_transfer_time / total_compute_time, 1) == 4.2

weight_bytes_total = L * d * f * s
assert weight_bytes_total == 38_654_705_664
assert round(weight_bytes_total / 1e9, 2) == 38.65
act_alive = s * T * (d + f)
assert act_alive == 1_677_721_600
assert round(act_alive / 1e9, 2) == 1.68
total_mem = weight_bytes_total + act_alive
assert round(total_mem / 1e9, 2) == 40.33
assert total_mem > CAP
assert round((total_mem - CAP) / 1e9, 2) == 0.33
assert round((CAP - weight_bytes_total) / 1e9, 2) == 1.35

# a type-U and a type-D layer give the same activation-alive and weight-element counts
assert T * (d + f) == T * (f + d)
assert d * f == f * d

# ---------------------------------------------------------------- Part 3 ----
half = L // 2
t_pp_dev = half * t_layer
assert round(t_pp_dev * 1e3, 2) == 1268.67
handoff_bytes = s * T * d
assert handoff_bytes == 335_544_320
assert round(handoff_bytes / 1e6, 2) == 335.54
t_handoff = handoff_bytes / Bnet
assert round(t_handoff * 1e3, 2) == 0.84
t_pp_total = 2 * t_pp_dev + t_handoff
assert round(t_pp_total * 1e3, 2) == 2538.17
assert math.isclose(t_pp_total, total_compute_time + t_handoff, rel_tol=1e-9)  # PP = single-device total + one handoff

w_pp_dev = weight_bytes_total / 2
assert round(w_pp_dev / 1e9, 2) == 19.33
assert round((w_pp_dev + act_alive) / 1e9, 2) == 21.01
one_layer_ws = s * T * d + s * T * f + s * d * f
assert round(one_layer_ws / 1e9, 2) == 2.21


def tp_layer(width_in, width_out, p=2):
    flops = 2 * T * width_in * (width_out // p)
    bytes_ = s * (T * width_in + width_in * (width_out // p) + T * (width_out // p))
    t = max(flops / P, bytes_ / BW)
    gather_full = s * T * width_out
    comm = gather_full * (p - 1) / p          # bytes sent per device for the all-gather
    return t, comm, comm / Bnet


t_up, comm_up, tcomm_up = tp_layer(d, f)
t_down, comm_down, tcomm_down = tp_layer(f, d)
assert round(t_up * 1e3, 2) == 17.62 and round(t_down * 1e3, 2) == 17.62
assert comm_up == 671_088_640 and round(comm_up / 1e6, 2) == 671.09
assert comm_down == 167_772_160 and round(comm_down / 1e6, 2) == 167.77
assert round(tcomm_up * 1e3, 2) == 1.68 and round(tcomm_down * 1e3, 2) == 0.42

up_total, down_total = t_up + tcomm_up, t_down + tcomm_down
assert round(up_total * 1e3, 3) == 19.298
assert round(down_total * 1e3, 3) == 18.040
assert round(36 * up_total * 1e3, 2) == 694.73
assert round(36 * down_total * 1e3, 2) == 649.43
t_tp_total = 36 * up_total + 36 * down_total
assert round(t_tp_total * 1e3, 2) == 1344.16
assert round(100 * t_tp_total / total_compute_time, 1) == 53.0

w_tp_dev = weight_bytes_total / 2
act_tp = s * T * (d + f)
assert w_tp_dev == w_pp_dev and act_tp == act_alive       # PP and TP happen to tie on per-device memory here
assert round((w_tp_dev + act_tp) / 1e9, 2) == 21.01
total_comm_tp_dev = 36 * comm_up + 36 * comm_down
assert round(total_comm_tp_dev / 1e9, 2) == 30.20
assert round(total_comm_tp_dev / handoff_bytes, 1) == 90.0

# ---------------------------------------------------------------- Part 4 ----
d4, f4, T4 = 8_192, 32_768, 2_048
weight_total_p1 = 2 * d4 * f4 * s
assert weight_total_p1 == 1_073_741_824
assert round(weight_total_p1 / 1e9, 4) == 1.0737
total_flops_p1 = 4 * T4 * d4 * f4
assert total_flops_p1 == 2_199_023_255_552 == 2 * (2 * T4 * d4 * f4)


def sig4(x):
    return float(f"{x:.3e}")


def tensor_sharded(p, Bn):
    flops1, flops2 = 2 * T4 * d4 * (f4 // p), 2 * T4 * (f4 // p) * d4
    bytes1 = s * (T4 * d4 + d4 * (f4 // p) + T4 * (f4 // p))
    bytes2 = s * (T4 * (f4 // p) + (f4 // p) * d4 + T4 * d4)
    t1c, t1m = flops1 / P, bytes1 / BW
    t2c, t2m = flops2 / P, bytes2 / BW
    t1, t2 = max(t1c, t1m), max(t2c, t2m)
    t_compmem = t1 + t2
    buf = T4 * d4 * s
    comm = buf * 2 * (p - 1) / p
    t_comm = comm / Bn
    return dict(flops_dev=flops1 + flops2, w_dev=2 * d4 * f4 * s / p, comm=comm, t_comm=t_comm,
                t_total=t_compmem + t_comm, bound=("comm" if t_comm > t_compmem else "compute"),
                each_matmul_compute_bound=(t1c >= t1m and t2c >= t2m))


fast_expected = {
    1: (2.199e12, 1.0737, 0.0, 0.0, 7.05, "compute"),
    2: (1.100e12, 0.5369, 33.55, 0.08, 3.61, "compute"),
    4: (5.498e11, 0.2684, 50.33, 0.13, 1.89, "compute"),
    8: (2.749e11, 0.1342, 58.72, 0.15, 1.03, "compute"),
    16: (1.374e11, 0.0671, 62.91, 0.16, 0.60, "compute"),
    32: (6.872e10, 0.0336, 65.01, 0.16, 0.38, "compute"),
    64: (3.436e10, 0.0168, 66.06, 0.17, 0.28, "comm"),
}
for p, (e_flops, e_w, e_comm_mb, e_tcomm, e_ttot, e_bound) in fast_expected.items():
    r = tensor_sharded(p, Bnet)
    assert sig4(r["flops_dev"]) == e_flops
    assert round(r["w_dev"] / 1e9, 4) == e_w
    assert round(r["comm"] / 1e6, 2) == e_comm_mb
    assert round(r["t_comm"] * 1e3, 2) == e_tcomm
    assert round(r["t_total"] * 1e3, 2) == e_ttot
    assert r["bound"] == e_bound
    assert r["each_matmul_compute_bound"]

comm_ceiling = 2 * (T4 * d4 * s) / Bnet
assert round(comm_ceiling * 1e3, 2) == 0.17
assert tensor_sharded(43, Bnet)["bound"] == "compute"
assert tensor_sharded(44, Bnet)["bound"] == "comm"
p32, p64 = tensor_sharded(32, Bnet), tensor_sharded(64, Bnet)
assert round(100 * (p32["t_total"] - p64["t_total"]) / p32["t_total"]) == 28
assert p32["w_dev"] == 2 * p64["w_dev"]


def token_sharded(p):
    Tp = T4 / p
    flops1, flops2 = 2 * Tp * d4 * f4, 2 * Tp * f4 * d4
    bytes1 = s * (Tp * d4 + d4 * f4 + Tp * f4)
    bytes2 = s * (Tp * f4 + f4 * d4 + Tp * d4)
    t1c, t1m = flops1 / P, bytes1 / BW
    t2c, t2m = flops2 / P, bytes2 / BW
    t_compmem = max(t1c, t1m) + max(t2c, t2m)
    return dict(t_compmem=t_compmem, w_dev=2 * d4 * f4 * s,
                bound=("compute" if (t1c + t2c) >= (t1m + t2m) else "memory"))


token_expected = {1: (7.05, "compute"), 2: (3.52, "compute"), 4: (1.76, "compute"), 8: (0.88, "compute"),
                  16: (0.73, "memory"), 32: (0.72, "memory"), 64: (0.72, "memory")}
for p, (e_t, e_bound) in token_expected.items():
    r = token_sharded(p)
    assert round(r["t_compmem"] * 1e3, 2) == e_t
    assert r["bound"] == e_bound
    assert r["w_dev"] == weight_total_p1

floor = 2 * s * d4 * f4 / BW
assert round(floor * 1e3, 2) == 0.72
assert token_sharded(64)["t_compmem"] > floor
assert (token_sharded(64)["t_compmem"] - floor) / floor < 0.01

slow_expected = {2: (1.34, 4.87, "compute"), 4: (2.01, 3.78, "comm"), 16: (2.52, 2.96, "comm"),
                 64: (2.64, 2.75, "comm")}
for p, (e_tcomm, e_ttot, e_bound) in slow_expected.items():
    r = tensor_sharded(p, Bnet_slow)
    assert round(r["t_comm"] * 1e3, 2) == e_tcomm
    assert round(r["t_total"] * 1e3, 2) == e_ttot
    assert r["bound"] == e_bound
assert tensor_sharded(3, Bnet_slow)["bound"] == "compute"
assert tensor_sharded(4, Bnet_slow)["bound"] == "comm"
r16, r64 = tensor_sharded(16, Bnet_slow), tensor_sharded(64, Bnet_slow)
assert round(100 * (r16["t_total"] - r64["t_total"]) / r16["t_total"]) == 7

# ------------------------------------------------------------- Follow-ups ---
# batching a decode step: FLOPs grow linearly with m while bytes barely move, for m tiny next to k, n
k, n = 8192, 8192
_, bytes_m1, *_ = matmul_stats(1, k, n)
_, bytes_m8, *_ = matmul_stats(8, k, n)
assert bytes_m8 / bytes_m1 < 1.01
assert (2 * 8 * k * n) / (2 * 1 * k * n) == 8

# fp8: compute-only speedup doubles the ridge without moving intensity, so it can create a new
# memory-bound regime; quantising storage too keeps intensity and the bound regime unchanged
P_fp8, ridge_fp8 = 2 * P, 2 * ridge
assert ridge_fp8 == 2 * ridge
_, _, I_bf16, *_ = matmul_stats(4096, 4096, 4096, elem=2)
_, _, I_fp8_storage, *_ = matmul_stats(4096, 4096, 4096, elem=1)
assert I_fp8_storage == 2 * I_bf16
assert math.isclose(I_bf16 / ridge, I_fp8_storage / ridge_fp8, rel_tol=1e-12)

print("all checks passed")
```

</details>

</details>
