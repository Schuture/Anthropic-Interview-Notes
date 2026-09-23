# Performance Modelling: Matmuls, Roofline and Sharding

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Estimation · roofline model, no code | ★★☆☆☆ | Hard | Perf Eng · MLE · Infra Eng | roofline, arithmetic-intensity, gpu-performance, parallelism, ffn-sharding | 4 parts / 60 min | Onsite |
<!-- meta:end -->

## Problem

One accelerator has a peak dense bf16 throughput of $P = 312$ TFLOP/s ($312\times10^{12}$ FLOP/s), an
HBM bandwidth of $BW = 1.5$ TB/s ($1.5\times10^{12}$ bytes/s), and $40$ GB of HBM ($1$ GB $=10^9$ bytes
throughout this page). Two or more such accelerators are connected by an interconnect that sustains
$B_{\text{net}} = 400$ GB/s of send traffic and, independently and at the same time, $400$ GB/s of
receive traffic between any pair of devices — a full-duplex link, so a device that is simultaneously
sending and receiving is not throttled by sharing one budget. Every tensor in this problem — a weight,
an activation, or a value in flight on the interconnect — is stored in bf16, $s = 2$ bytes/element,
unless a part says otherwise.

For a matrix multiply $C = AB$ with $A \in \mathbb{R}^{m\times k}$, $B \in \mathbb{R}^{k\times n}$,
$C \in \mathbb{R}^{m\times n}$:

- *FLOPs*: counting one multiply and one add as two FLOPs, $\text{FLOPs} = 2mkn$.
- *Bytes moved*: reading $A$ and $B$ from HBM once each and writing $C$ once, at $s$ bytes/element,
  $\text{bytes} = s(mk+kn+mn)$. Neither operand is re-read and nothing beyond $C$ is written.
- *Arithmetic intensity* $I = \text{FLOPs}/\text{bytes}$, in FLOP/byte.
- The *ridge point* $I^\star = P/BW$ is the intensity at which compute time and transfer time are
  equal. A matmul with $I > I^\star$ is *compute-bound*; one with $I < I^\star$ is *memory-bound*.
- *Predicted time* $t = \max(\text{FLOPs}/P,\ \text{bytes}/BW)$, assuming compute and HBM traffic fully
  overlap inside one kernel.

### Part 1 — Four matmuls

| Shape | $m$ | $k$ | $n$ |
| --- | --- | --- | --- |
| A | 8 | 4,096 | 4,096 |
| B | 2,048 | 4,096 | 4,096 |
| C | 4,096 | 64 | 4,096 |
| D | 4,096 | 4,096 | 4,096 |

For each shape, give FLOPs, bytes moved, arithmetic intensity, the bound regime, and the predicted
time. Then, for a general $(m,k,n)$, say whether growing $m$, $k$ or $n$ alone — holding the other two
fixed — can ever move a matmul from compute-bound to memory-bound, or only ever the other way, and
justify the answer for all three variables, not only the two that shapes A–D exercise directly.

### Part 2 — A 72-layer stack

A network has $L = 72$ layers, each a separate kernel with no fusion between layers. Layer $\ell$ takes
an input $X_\ell$ and a weight $W_\ell$ and produces $X_{\ell+1}$; there is no attention, no
normalisation, no residual connection and no bias, so $X_{\ell+1}$ is $X_\ell$'s only successor and
$X_\ell$'s only source is $X_{\ell-1}$ (or, for $\ell = 1$, the network's own input). $X_{\ell+1}$ is
written to HBM by layer $\ell$ and read back from HBM by layer $\ell + 1$, exactly as an ordinary
matmul's output and next input. Layers alternate between two shapes, both built from
$T = 20{,}480$ (the number of tokens processed together in this pass), $d = 8{,}192$ and
$f = 32{,}768$:

- Odd $\ell$ (*type U*): $X_\ell \in \mathbb{R}^{T\times d}$, $W_\ell \in \mathbb{R}^{d\times f}$,
  $X_{\ell+1} = X_\ell W_\ell \in \mathbb{R}^{T\times f}$.
- Even $\ell$ (*type D*): $X_\ell \in \mathbb{R}^{T\times f}$, $W_\ell \in \mathbb{R}^{f\times d}$,
  $X_{\ell+1} = X_\ell W_\ell \in \mathbb{R}^{T\times d}$.

(A pointwise nonlinearity may sit between $X_\ell W_\ell$ and $X_{\ell+1}$; it is not a matmul and its
cost is not counted.) All 72 weights are resident on one accelerator for the whole pass. Give the total
compute time (every layer's FLOPs, divided by $P$) and total transfer time (every layer's bytes,
divided by $BW$) for one forward pass over all 72 layers. Then determine exactly which tensors must be
resident at once to run this pass, whether their total fits in the accelerator's 40 GB, and by how much
it does or does not.

### Part 3 — Two devices, one input

The same 72-layer stack processes one $T = 20{,}480$-token input, now split across two identical
accelerators of the kind above, connected by the interconnect. Compare two schemes.

*Pipeline parallelism*: device $A$ holds layers 1–36 and their weights; device $B$ holds layers 37–72
and theirs. Device $A$ computes its 36 layers in sequence, sends the resulting activation to device $B$
over the interconnect, and device $B$ computes its 36 layers.

*Tensor parallelism*: every layer's weight is split into 2 equal shards along its output axis — a
type-U layer's $f$ columns, or a type-D layer's $d$ columns — one shard per device; both devices hold a
full, identical copy of the network's input $X_1$. For each layer, a device computes its own output
shard from the full input it already holds, with no communication needed for that; the two devices then
exchange shards over the interconnect so that both again hold the layer's full output, before the next
layer starts.

For each scheme, give the end-to-end latency for this one input, the peak memory used on either device
— stating exactly which tensors are resident — and the total bytes either device sends over the
interconnect during the whole pass. Say which scheme a single request should prefer, and why the other
scheme is used in practice anyway.

### Part 4 — Sharding a feed-forward network

A feed-forward network computes $Y = \operatorname{act}(XW_1)W_2$ with $X \in \mathbb{R}^{T\times d}$,
$W_1 \in \mathbb{R}^{d\times f}$, $\operatorname{act}(XW_1) \in \mathbb{R}^{T\times f}$,
$W_2 \in \mathbb{R}^{f\times d}$, $Y \in \mathbb{R}^{T\times d}$, with $d = 8{,}192$ and $f = 32{,}768$
as in Part 2, and $\operatorname{act}$ a pointwise nonlinearity. Counting multiply-accumulates as two
FLOPs and excluding $\operatorname{act}$ and any bias, this costs $4Tdf$ FLOPs, and the two weights
together occupy $2dfs$ bytes ($s = 2$ for bf16). Consider two ways to split this block across $p$
devices; assume $p$ divides $f$ (tensor sharding) or $T$ (token sharding) exactly, as it does for every
$p$ used below.

*Tensor sharding*: $W_1$'s $f$ columns and $W_2$'s $f$ rows are each split into $p$ equal shards, one
per device; $X$ is replicated on every device. Device $i$ computes $H_i =
\operatorname{act}(XW_{1,i}) \in \mathbb{R}^{T\times f/p}$ locally, then a partial output
$Y_i = H_iW_{2,i} \in \mathbb{R}^{T\times d}$, also locally. The true $Y = \sum_i Y_i$ is formed by an
all-reduce of the $p$ partial outputs, each of logical size $T \times d$ at $s_c$ bytes/element for the
communicated values ($s_c = s = 2$ unless stated otherwise); an ideal ring all-reduce has every device
send $2(p-1)/p$ times that logical size and receive the same amount, and this all-reduce cannot overlap
with any of this block's own compute.

*Token sharding*: $W_1$ and $W_2$ are fully replicated on every device; $T$'s rows are split into $p$
equal groups, one per device, each computing the complete two-matmul block on its own $T/p$ rows, with
no communication of its own inside this block.

Take $T = 2{,}048$ tokens (a batched decode step, smaller than Part 2's prefill). For
$p = 1, 2, 4, 8, 16, 32, 64$, give each scheme's FLOPs, weight bytes, and predicted latency per device,
and, for tensor sharding, the communication volume per device and its time on the
$B_{\text{net}} = 400$ GB/s interconnect of Part 3. For each $p$, state whether tensor sharding is
compute-, memory- or communication-bound, and do the same for token sharding. Repeat the bound
classification for tensor sharding at $p = 2, 4, 16, 64$ on a slower, cross-node interconnect with
$B_{\text{net}}' = 25$ GB/s in place of $B_{\text{net}}$. Finally, say what, if anything, would let an
all-reduce's communication overlap with compute in a real deployment of this block, given that it
cannot for the single block considered above.

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two points worth confirming with the interviewer: whether "bytes moved" prices every matmul on its own
inputs and output even when consecutive layers share a tensor (this page's rule: yes — no fusion across
separate kernels, so a shared activation is written once and read once), and whether the ridge point is
a property of the hardware alone. It is: $I^\star = P/BW = 312\times10^{12} / 1.5\times10^{12} = 208$
FLOP/byte, fixed for every part below.

### Part 1

**Shape A** works the formulas once in full. With $(m,k,n) = (8, 4{,}096, 4{,}096)$:

$$\text{FLOPs} = 2\cdot 8\cdot 4{,}096\cdot 4{,}096 = 268{,}435{,}456,$$

$$\text{bytes} = 2\bigl(8\cdot4{,}096 + 4{,}096\cdot4{,}096 + 8\cdot4{,}096\bigr) = 2\cdot 16{,}842{,}752 = 33{,}685{,}504,$$

$$I = \frac{268{,}435{,}456}{33{,}685{,}504} \approx 7.97 \text{ FLOP/byte} < 208 = I^\star \implies \text{memory-bound},$$

$$t = \max\!\left(\frac{268{,}435{,}456}{312\times10^{12}},\ \frac{33{,}685{,}504}{1.5\times10^{12}}\right) = \max(0.86,\ 22.46)\ \mu s = 22.46\ \mu s.$$

The same formulas, applied to B, C and D:

| Shape | FLOPs | bytes | $I$ (FLOP/byte) | regime | $t$ |
| --- | --- | --- | --- | --- | --- |
| A | $2.684\times10^{8}$ | $3.369\times10^{7}$ | 7.97 | memory | 22.46 μs |
| B | $6.872\times10^{10}$ | $6.711\times10^{7}$ | 1,024.00 | compute | 220.25 μs |
| C | $2.147\times10^{9}$ | $3.460\times10^{7}$ | 62.06 | memory | 23.07 μs |
| D | $1.374\times10^{11}$ | $1.007\times10^{8}$ | 1,365.33 | compute | 440.51 μs |

B is A with $m$ grown from 8 to 2,048 at the same $k=n=4{,}096$: intensity rises from 7.97 to 1,024 and
the regime flips from memory- to compute-bound. C and D show the same flip along $k$ instead, at the
same $m=n=4{,}096$: growing $k$ from 64 to 4,096 raises intensity from 62.06 to 1,365.33. Both are
instances of one fact, proved next for every variable, not only $m$ and $k$.

For fixed $k,n$, intensity as a function of $m$ is $I(m) = \dfrac{2mkn}{s(m(k+n)+kn)} = \dfrac{2kn}{s}\, g(m)$
with $g(m) = \dfrac{m}{m(k+n)+kn}$. Then

$$g'(m) = \frac{\bigl(m(k+n)+kn\bigr) - m(k+n)}{\bigl(m(k+n)+kn\bigr)^{2}} = \frac{kn}{\bigl(m(k+n)+kn\bigr)^{2}} > 0,$$

so $I$ is strictly increasing in $m$, from $I(0) = 0$ towards $\lim_{m\to\infty} I(m) = 2kn/(s(k+n))$ —
the harmonic mean of $k$ and $n$, divided by $s$ — approached but never reached. Bytes
$s(mk+kn+mn)$ and FLOPs $2mkn$ are each a symmetric function of $(m,k,n)$ — the elementary symmetric sum
of the three pairwise products, and twice their triple product — so $I(m,k,n)$ is symmetric too, and the
argument above applies unchanged with any one of $m, k, n$ standing in for "$m$" and the other two for
"$k, n$". Growing any single one of $m, k, n$, holding the other two fixed, strictly increases the
intensity, so it can only ever move a matmul towards compute-bound, never away from it — never the other
way, for any of the three variables. Reaching compute-bound is not guaranteed, though: growing one
variable only drives $I$ towards the ceiling set by the other two, $2xy/(s(x+y))$, and if that ceiling
already sits below $I^\star$ no amount of growth in the third variable alone crosses the ridge. A and C
share the same other-two values $\{4{,}096, 4{,}096\}$, hence the same ceiling
$2\cdot4{,}096^2/(2\cdot 8{,}192) = 2{,}048$ FLOP/byte, comfortably above $I^\star = 208$: growing $m$
past B's 2,048, or growing $k$ further from D's 4,096, both keep pushing further into compute-bound,
towards that same ceiling, since both are instances of the one symmetric fact above.

### Part 2

Every layer, of either type, has the same FLOPs and the same bytes moved: $\text{FLOPs}_\ell = 2Tdf =
2\cdot20{,}480\cdot8{,}192\cdot32{,}768 = 10{,}995{,}116{,}277{,}760$, and
$\text{bytes}_\ell = s(Td+df+Tf) = 2(20{,}480\cdot8{,}192 + 8{,}192\cdot32{,}768 + 20{,}480\cdot32{,}768)
= 2{,}214{,}592{,}512$ — a type-D layer swaps which of the three products is the input and which is the
output, but the set $\{Td, df, Tf\}$ is the same either way. Intensity
$I_\ell = 10{,}995{,}116{,}277{,}760 / 2{,}214{,}592{,}512 \approx 4{,}964.85$ FLOP/byte, far above
$I^\star = 208$, so every layer is compute-bound: its predicted time is set by compute alone,
$t_\ell = \text{FLOPs}_\ell/P \approx 35.24$ ms (transfer alone would take only
$\text{bytes}_\ell/BW \approx 1.48$ ms).

With all 72 layers identical in cost,

$$\text{total compute time} = \frac{72\cdot\text{FLOPs}_\ell}{P} \approx 2{,}537.33\text{ ms}, \qquad
\text{total transfer time} = \frac{72\cdot\text{bytes}_\ell}{BW} \approx 106.30\text{ ms}.$$

Total transfer time is about 4.2% of total compute time, so — as at the single-layer level — the whole
pass is compute-bound, and its predicted latency is the 2,537.33 ms compute figure, not the sum of the
two.

**Memory.** The tensors resident at any instant during this pass are (a) all 72 weight matrices, kept
resident for the whole pass since each is needed again before the pass ends, and (b) the
currently-executing layer's input $X_\ell$ and output $X_{\ell+1}$ — every earlier activation has
already been consumed and freed, and no later one exists yet. (a) totals
$72dfs = 72\cdot8{,}192\cdot32{,}768\cdot2 = 38{,}654{,}705{,}664$ bytes $\approx 38.65$ GB, the same
figure for every layer since a type-U weight and a type-D weight both have $df$ elements. (b) totals
$Ts(d+f) = 20{,}480\cdot2\cdot40{,}960 = 1{,}677{,}721{,}600$ bytes $\approx 1.68$ GB — also the same for
every layer, since a type-U layer's input-plus-output is $T(d+f)$ elements and a type-D layer's is
$T(f+d)$, the same sum. Together, weights plus one layer's live activations come to
$38.65 + 1.68 \approx 40.33$ GB.

That is over the 40 GB budget by about $0.33$ GB — easy to miss by eye, since the weights alone would
fit with $40 - 38.65 \approx 1.35$ GB to spare. It is specifically the one layer's worth of live
activations (1.68 GB, driven by the $T=20{,}480$-token prefill) that exceeds that spare capacity and
tips the total over. This is why Part 3 splits the stack across two devices.

### Part 3

**Pipeline parallelism.** Every layer costs the same $t_\ell \approx 35.24$ ms (Part 2), so each
device's 36 layers take $36\,t_\ell \approx 1{,}268.67$ ms — exactly half of the single-device total,
since the single-device total is $72\,t_\ell$. Layer 36 is even, a type-D layer, so the handed-off
activation has shape $T\times d$: $\text{bytes} = sTd = 2\cdot20{,}480\cdot8{,}192 = 335{,}544{,}320
\approx 335.54$ MB, taking $335{,}544{,}320 / (400\times10^{9}) \approx 0.84$ ms over the interconnect.
End-to-end latency:

$$t_{\text{PP}} = 36\,t_\ell + \frac{sTd}{B_{\text{net}}} + 36\,t_\ell = 72\,t_\ell + \frac{sTd}{B_{\text{net}}}.$$

$72\,t_\ell$ is exactly the Part 2 total compute time, $\approx 2{,}537.33$ ms; adding the one handoff,
$\approx 0.84$ ms, gives $t_{\text{PP}} \approx 2{,}538.17$ ms. Splitting the layers across two devices
does not reduce the latency of one input at all, since the two devices still run one after the other.

Each device holds half the weights, $38.65/2 \approx 19.33$ GB, plus one layer's live activations,
$\approx 1.68$ GB (only one layer is ever mid-flight on a device at a time): $\approx 21.01$ GB per
device, comfortably under 40 GB. A single layer's own input, output and weight together are only
$sTd + sTf + sdf \approx 2.21$ GB, trivially resident alongside the rest. Total interconnect traffic for
the whole pass is the one handoff: $\approx 335.54$ MB per device.

**Tensor parallelism.** A type-U layer, sharded $2$ ways: each device reads the full input ($Td$
elements), its own half of the weight ($df/2$ elements), and writes its own half of the output ($Tf/2$
elements); FLOPs are exactly halved, $Tdf$. Bytes: $s(Td + df/2 + Tf/2) = 2(20{,}480\cdot8{,}192 +
8{,}192\cdot16{,}384 + 20{,}480\cdot16{,}384) = 1{,}275{,}068{,}416$. Both the halved FLOPs and this byte
count give an intensity still far above $I^\star$, so this shard, too, is compute-bound, at
$t = Tdf/P \approx 17.62$ ms. A type-D layer, sharded the same way, has the same FLOPs ($Tfd = Tdf$) and
a different byte count — $s(Tf + fd/2 + Td/2) = 1{,}778{,}384{,}896$ — but is still compute-bound at the
same $t \approx 17.62$ ms (the compute term is identical and already dominates both byte counts). After
each layer, the two devices exchange their output shards: since each device already holds a disjoint,
final slice of the output, this needs only a pure all-gather, not a full all-reduce — no reduce-scatter
step, so half of Part 4's ring volume; at $p=2$ that means each device simply sends its own shard. A
type-U layer's output has $Tf$ elements, so each device sends $sTf/2 = 671{,}088{,}640$ bytes
$\approx 671.09$ MB, taking $\approx 1.68$ ms; a type-D layer's output has $Td$ elements, so each device sends
$sTd/2 = 167{,}772{,}160$ bytes $\approx 167.77$ MB, taking $\approx 0.42$ ms. Summing 36 layers of each type, with the exchange added after each layer's compute since the next
layer needs the gathered result: type-U layers cost $17.62 + 1.68 = 19.298$ ms each, $36$ of them
$\approx 694.73$ ms; type-D layers cost $17.62 + 0.42 = 18.040$ ms each, $36$ of them
$\approx 649.43$ ms; together,

$$t_{\text{TP}} \approx 694.73 + 649.43 = 1{,}344.16\text{ ms}.$$

Tensor parallelism finishes this one input in about $1{,}344.16 / 2{,}537.33 \approx 53.0\%$ of the
single-device time — nearly half, since compute genuinely splits across the two devices — at the cost
of communicating on every one of the 72 layers. Total interconnect traffic per device is
$36\cdot671.09\text{ MB} + 36\cdot167.77\text{ MB} \approx 30.20$ GB — about 90 times pipeline
parallelism's single 335.54 MB handoff. Per-device memory: every weight is halved
($38.65/2 \approx 19.33$ GB total) and, once a layer's exchange completes, both devices hold that
layer's *full* input and output, not a half — so the live-activation figure is the same
$\approx 1.68$ GB as the single-device and pipeline-parallel cases, not half of it. Per-device memory is
therefore $\approx 21.01$ GB — numerically the same total as pipeline parallelism, because both schemes
happen to halve the same weight bytes here and neither shrinks the activation footprint.

**The trade.** For this one input, tensor parallelism dominates pipeline parallelism outright on every
number computed here: lower latency, and the same per-device memory. Pipeline parallelism's real
advantage does not appear for a single input at all: with several inputs in flight, device $B$ can be
finishing input 1's second half while device $A$ is already running input 2's first half, filling the
pipeline and raising throughput without adding communication — tensor parallelism, in contrast, forces
both devices to advance through every layer in lockstep, so it cannot overlap independent inputs the
same way and pays its communication cost on every layer, for every input, regardless of how many are in
flight. A real deployment typically combines both: enough tensor parallelism to fit and speed up one
layer within a fast, single-node interconnect, and pipeline parallelism across that to add throughput
and to reach a model too large for even a tensor-parallel group's combined memory.

### Part 4

**Tensor sharding.** Per-device FLOPs are $4Tdf/p$, split evenly between the two matmuls, each
$2Tdf/p$; per-device weight bytes are $2dfs/p$. At $p=1$ this is just the unsharded block:
$4Tdf/P = 4\cdot2{,}048\cdot8{,}192\cdot32{,}768 / (312\times10^{12}) \approx 7.05$ ms, entirely compute
(intensity is far above $I^\star$ at this size), with $2dfs \approx 1.07$ GB of weights. For $p>1$, each
device's own compute-plus-memory time is the sum of its two matmuls' own roofline times (each still
individually compute-bound at every $p$ used below, since $d,f$ are large), and the all-reduce of the
$T\times d$ partial output — logical size $S = Tds = 2{,}048\cdot8{,}192\cdot2 = 33{,}554{,}432$ bytes —
adds $2(p-1)S/(p\,B_{\text{net}})$ on top, since it cannot overlap with this block's own compute:

| $p$ | FLOPs/device | weight/device | comm/device | $t_{\text{comm}}$ | $t_{\text{total}}$ | bound |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | $2.199\times10^{12}$ | 1.0737 GB | 0 | 0 ms | 7.05 ms | compute |
| 2 | $1.100\times10^{12}$ | 0.5369 GB | 33.55 MB | 0.08 ms | 3.61 ms | compute |
| 4 | $5.498\times10^{11}$ | 0.2684 GB | 50.33 MB | 0.13 ms | 1.89 ms | compute |
| 8 | $2.749\times10^{11}$ | 0.1342 GB | 58.72 MB | 0.15 ms | 1.03 ms | compute |
| 16 | $1.374\times10^{11}$ | 0.0671 GB | 62.91 MB | 0.16 ms | 0.60 ms | compute |
| 32 | $6.872\times10^{10}$ | 0.0336 GB | 65.01 MB | 0.16 ms | 0.38 ms | compute |
| 64 | $3.436\times10^{10}$ | 0.0168 GB | 66.06 MB | 0.17 ms | 0.28 ms | comm |

(comm/device is bytes sent, $2(p-1)S/p$; $t_{\text{comm}}$ uses $B_{\text{net}} = 400$ GB/s.) Compute
time halves with every doubling of $p$ while communication grows only from 0 towards its ceiling
$2S/B_{\text{net}} \approx 0.17$ ms, so compute stays the larger term through $p=32$; only past $p=43$
(found by extending this table) does communication overtake compute-plus-memory, which is why the
table's last row already reads "comm". Latency keeps falling across the whole table, but with sharply
diminishing returns: $p=32\to64$ doubles
the device count and roughly doubles communication yet only takes latency from 0.38 to 0.28 ms, a 28%
cut, even though weight/device is again cut exactly in half (0.0336 GB to 0.0168 GB). A smaller weight
footprint is necessary for a smaller memory budget, but by itself it does not translate into
proportionally smaller latency once communication is a meaningful share of the total.

**Token sharding.** Per-device FLOPs are the same $4Tdf/p$ as tensor sharding (the same total work,
divided the same way), but weight bytes per device stay at the full, unsharded $2dfs \approx 1.07$ GB
for every $p$, since $W_1, W_2$ are replicated rather than split, and there is no communication inside
this block. Per-device time is again each matmul's own roofline time, now with $T/p$ rows through the
*full* weight: at $p=1$ it is identical to tensor sharding's $p=1$ row, 7.05 ms, compute-bound; by
$p=8$ it has fallen to 0.88 ms, still compute-bound; but from $p=16$ on it is memory-bound at 0.73 ms,
barely moving further (0.72 ms at $p=32$, 0.72 ms at $p=64$) — because reading the *full*, unsharded
$W_1$ and $W_2$ from HBM on every device, regardless of how few rows that device owns, has a floor of
$2dfs/BW \approx 0.72$ ms that no amount of extra sharding removes. Tensor sharding has no such floor,
since its weight bytes shrink with $p$ too, which is why it keeps improving (to 0.28 ms at $p=64$) well
past where token sharding has stalled.

**A slower interconnect.** Repeating tensor sharding's bound classification with
$B_{\text{net}}' = 25$ GB/s in place of 400 GB/s changes nothing about compute or weight bytes, only
$t_{\text{comm}} = 2(p-1)S/(p\,B_{\text{net}}')$, which is $16\times$ larger at every $p$:

| $p$ | $t_{\text{comm}}$ | $t_{\text{total}}$ | bound |
| --- | --- | --- | --- |
| 2 | 1.34 ms | 4.87 ms | compute |
| 4 | 2.01 ms | 3.78 ms | comm |
| 16 | 2.52 ms | 2.96 ms | comm |
| 64 | 2.64 ms | 2.75 ms | comm |

Here communication already dominates by $p=4$, and $p=16\to64$ — four times the devices — buys only
2.96 ms $\to$ 2.75 ms, a 7% improvement, almost all of it spent on a shrinking compute term against an
almost-flat communication floor. The lesson is the same as above, sharper: past the point where
communication dominates, adding devices mostly adds communication, not speed.

**Overlap.** For this one block processing one batch of tokens, the all-reduce cannot overlap with
this block's own compute: $Y_i$ needs $H_i$, $H_i$ needs $X$, and whatever consumes $Y$ needs the
all-reduce to have finished, so every step is on one dependency chain. Overlap needs independent work to
fill the gap. Two real sources of it: (a) multiple independent requests or micro-batches in flight —
while one batch's all-reduce is in progress, a device can already be running another, independent
batch's local matmul, exactly as pipeline parallelism overlaps independent inputs in Part 3; (b)
splitting one batch's own reduction into column chunks and pipelining it against compute — start
reducing the first chunk of $Y_i$'s columns while the matmul for the remaining columns is still running
— which shortens the exposed, non-overlapped tail of the collective without changing the total bytes it
moves.

### Follow-ups

- **Batch size and decode.** A decode step's matmul looks like shape A: few tokens ($m$ small) against
  a large weight, so intensity is low and time is set by reading the weight from HBM, almost
  independently of $m$ (bytes are dominated by the $kn$ weight term while $m$ stays small). Batching
  several requests' decode steps together raises $m$ at no extra weight traffic, so FLOPs grow while
  bytes barely do — intensity rises towards the ceiling proved in Part 1 — and throughput improves
  close to linearly with batch size until the ceiling is approached and the step turns compute-bound,
  after which further batching adds latency without adding much throughput.
- **Below the roofline.** Real kernels rarely reach the full $\max(\text{FLOPs}/P,\text{bytes}/BW)$:
  imperfect tiling wastes some FLOPs or bytes on padding when a dimension does not divide the hardware's
  tile size, a kernel launch has fixed overhead that a short matmul cannot amortise, compute and memory
  access do not overlap perfectly inside real hardware, and a network collective pays protocol and
  synchronisation overhead beyond the bytes this page counts. The roofline number is a lower bound on
  time, not a prediction of it.
- **Lower precision.** Hardware that computes fp8 at roughly double the bf16 FLOP/s, while bytes stay
  bf16-sized (only the multiply-accumulate itself drops to fp8, weights and activations still read and
  written at 2 bytes/element), doubles $P$ without changing $BW$ — the ridge point doubles, so some
  matmuls that were comfortably compute-bound in bf16 become memory-bound in fp8, and gain nothing from
  the faster compute. fp8 only delivers close to its full speedup where storage is *also* quantised to 1
  byte/element, halving bytes alongside the doubled FLOP/s and leaving intensity, and the bound regime,
  unchanged.
- **KV-cache reads.** Decode-time attention reads a key/value cache that grows with the sequence
  length seen so far, on top of the weight bytes this page counts; a longer conversation means more
  cache bytes per decode step at essentially the same tiny FLOPs, pushing intensity down further and
  making decode more memory-bound the longer a sequence runs, independently of the batch-size effect
  above.

<details>
<summary>Checks (runnable)</summary>

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
