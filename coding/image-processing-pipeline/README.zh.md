# 图像处理流水线

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| 题型 | 优先级 | 难度 | 岗位 | 考点 | 形式 | 轮次 |
| --- | --- | --- | --- | --- | --- | --- |
| 编程 · 并发，图像处理 | ★★★★★ | 中等 | SWE · MLE · Infra Eng | image-processing, pillow, multiprocessing, concurrency, caching | 3 个部分 / 55 分钟 | 技术电面 · 现场面（VO） |
<!-- meta:end -->

## 题目

预先给定三个文件夹和一个辅助函数，均为绝对路径。`image_dir` 存放源图片：其中每个文件都能被 Pillow
解码为 `"RGB"` 图片，其文件名（含扩展名）就是该图片的*图片名*。`transformation_dir` 存放*流水线文件*：
其中每个文件都是一个 JSON 文件，文件名就是该文件的*json 名*，形状为

```text
{"transformations": [{"type": "<type>", ...params}, ...]}
```

—— 一个有序的*步骤*列表，每个步骤都是一个 JSON 对象，其 `"type"` 字段选定六种变换之一，并带上该类型
所需的参数；某个 json 名按顺序列出的这些步骤，就是它的*流水线*。`output_dir` 是结果的存放位置，通过
给定的辅助函数写入：

```py
def get_output_path(image_name: str, json_name: str) -> str: ...
```

该函数返回的绝对路径，就是这张图片在这条流水线下的结果应写入的位置；该路径所在的目录已经存在。

对 `image_dir` 中的每张图片和 `transformation_dir` 中的每条流水线——也就是 $|\text{图片}| \times
|\text{流水线}|$ 种组合中的每一种——都要按顺序对该图片执行该流水线的步骤，并把结果以 PNG 格式保存到
`get_output_path(image_name, json_name)`。流水线的第一个步骤以刚解码出的源图片为输入；之后每个步骤都
以上一步骤的输出为输入。流水线可以一个步骤都不列出，此时其输出就是源图片本身，不作改动，以 PNG 格式
保存。

共支持六种步骤类型，由步骤的 `"type"` 字段命名。`grayscale` 以及两种翻转不带其他参数；`scale`、
`blur`、`rotate` 各带一个参数，见下文。

- `grayscale` 把图片转换为 Pillow 的单通道模式 `"L"`（`image.convert("L")`）。
- `flip_horizontal` 把图片左右镜像（`image.transpose(Image.Transpose.FLIP_LEFT_RIGHT)`）。
- `flip_vertical` 把图片上下镜像（`image.transpose(Image.Transpose.FLIP_TOP_BOTTOM)`）。
- `scale`——参数 `"factor"`，一个正浮点数——把图片缩放到
  `(max(1, round(width * factor)), max(1, round(height * factor)))`，其中 `width`、`height` 是图片
  当前的尺寸，`round` 是 Python 的四舍五入到偶数，重采样滤波器为 `Image.Resampling.BILINEAR`。
- `blur`——参数 `"radius"`，一个非负整数——施加该半径的高斯模糊
  （`image.filter(ImageFilter.GaussianBlur(radius=radius))`）。
- `rotate`——参数 `"angle"`，一个以度为单位的浮点数——让图片绕其中心逆时针旋转 `angle` 度，并扩展画布
  （`expand=True`）以容纳整张旋转后的图片，新露出的四角填充为黑色（模式 `"L"` 下为 `0`，模式 `"RGB"`
  下为 `(0, 0, 0)`）；重采样滤波器同样是 `Image.Resampling.BILINEAR`。

`grayscale` 是唯一会改变图片模式的步骤；`scale` 和 `rotate` 是唯二会改变图片尺寸的步骤；其余情况下，
模式和尺寸都保持原样。

例如，`sample.png` 解码后是一张尺寸为 `(60, 40)` 的 `"RGB"` 图片。`p.json` 中的流水线，

```json
{"transformations": [{"type": "grayscale"}, {"type": "scale", "factor": 0.5}, {"type": "rotate", "angle": 90}]}
```

执行过程如下：

```text
decode                            mode "RGB", size (60, 40)
grayscale                      -> mode "L",   size (60, 40)   # 模式改变；尺寸不变
scale(factor=0.5)              -> mode "L",   size (30, 20)   # round(60*0.5)=30, round(40*0.5)=20
rotate(angle=90, expand=True)  -> mode "L",   size (20, 30)   # 90 的倍数会让宽高互换
```

结果保存在 `get_output_path("sample.png", "p.json")`。

### Part 1 —— 顺序流水线

实现 `process_images`。

```py
def process_images(
    image_dir: str,
    transformation_dir: str,
    output_dir: str,
    get_output_path: Callable[[str, str], str],
) -> None: ...
```

```text
image_dir:           a.png, b.png
transformation_dir:  x.json, y.json
process_images 恰好写出 4 个文件，每个 (image, pipeline) 组合一个：
  get_output_path("a.png", "x.json")
  get_output_path("a.png", "y.json")
  get_output_path("b.png", "x.json")
  get_output_path("b.png", "y.json")
```

### Part 2 —— 并行批处理

现在 `image_dir` 中的图片可能大到、`transformation_dir` 中的流水线可能多到，让 `process_images`
按顺序逐一处理慢到无法接受。请实现 `process_images_parallel`：对每一个输入，输出都与 `process_images`
相同，但要分摊到多个 CPU 核心上执行。它不能让同一张源图片按用到它的流水线数量重复解码，而且跨进程
边界传递的只能是文件路径和 JSON 大小的数据——绝不能是解码后的图片或其原始像素字节。

```py
def process_images_parallel(
    image_dir: str,
    transformation_dir: str,
    output_dir: str,
    get_output_path: Callable[[str, str], str],
    max_workers: int | None = None,
) -> None: ...
```

`max_workers` 限定同时运行的工作进程数；省略或为 `None` 时使用 `os.cpu_count()`。

```text
big_images/ 里有 200 张图片；transformations/ 里有 5 个流水线文件。
process_images_parallel 仍须写出全部 200 * 5 = 1,000 个输出，
但总共只解码 200 张图片——每张一次——而不是 1,000 次。
```

### Part 3 —— 共享前缀复用

流水线的*前缀*是它的前 $k$ 个步骤，$`0 \le k \le`$ 其长度；两个前缀*相同*，当且仅当二者长度相同，且
逐步对应的 `"type"` 与参数值都相同。请实现 `process_images_shared_prefix`：签名、输出和跨进程规则都
与 `process_images_parallel` 相同，但对同一张图片而言，被两条或更多流水线共享的每个前缀都只计算
一次——某条流水线在该前缀之后继续执行，不能对共享它的另一条流水线产生可见影响。共享只发生在应用于
*同一张*图片的流水线之间；两条作用于不同图片的流水线永远不共享计算，哪怕它们的步骤完全相同。不要把
一个共享前缀的图片一直留在内存里：一旦所有需要它的流水线都用过了它，就可以释放。

```py
def process_images_shared_prefix(
    image_dir: str,
    transformation_dir: str,
    output_dir: str,
    get_output_path: Callable[[str, str], str],
    max_workers: int | None = None,
) -> None: ...
```

对同一张图片，给定如下流水线：

```text
p1: grayscale -> scale(factor=0.5) -> rotate(angle=90)
p2: grayscale -> scale(factor=0.5) -> blur(radius=2)
p3: grayscale -> scale(factor=0.5) -> rotate(angle=90)      # identical to p1
p4: flip_horizontal -> blur(radius=2)
```

若独立执行每条流水线，`grayscale` 和 `scale(factor=0.5)` 各会执行三次（p1、p2、p3），
`rotate(angle=90)` 执行两次（p1、p3），`flip_horizontal` 和 `blur(radius=2)` 各为 p4 执行一次：共 11
次步骤执行。共享之后，`grayscale` 和 `scale(factor=0.5)` 各只执行一次，被 p1、p2、p3 共用；
`rotate(angle=90)` 只执行一次，其结果同时作为 p1 和 p3 的输出保存；`flip_horizontal` 之后的
`blur(radius=2)`，与 p2 中 `scale(factor=0.5)` 之后的那次 `blur(radius=2)`，是两次独立的执行，因为
它们是经由不同的前缀到达的。总计执行六次：`grayscale`、`scale(factor=0.5)`、`rotate(angle=90)`、
`blur(radius=2)`（在 `scale` 之后）、`flip_horizontal`、`blur(radius=2)`（在 `flip_horizontal` 之后）。

## 参考解答

<details>
<summary>展开参考解答</summary>

开始之前有两点值得与面试官确认：是否可以假定每张图片和每个流水线文件都是良构的（这里三个部分都这样
假定；损坏的图片、未知的步骤类型、非法的参数则留给“追问”部分讨论），以及评测是否按某个具体库的输出
逐像素比对——本解法通篇使用 Pillow 自身的操作，滤波器和取整方式都已在题目中精确给出，因此只要确定
用这个库，答案就是确定的。

### Part 1

`apply_step` 是唯一知道六种 `"type"` 各自含义的地方，后面每个部分都直接复用它，不作修改。Pillow 的
`convert`、`transpose`、`resize`、`filter`、`rotate` 都会返回一个新的 `Image`，而不会修改调用者
自身，所以 `run_pipeline` 可以在每次迭代时放心地重新赋值 `image`，不必担心别名问题；`process_images`
也因此可以只解码一次图片，再把它喂给它的每一条流水线：没有哪条流水线能看到、更不用说撤销，更早的
流水线对这同一个起点做过的事。

```python
import json
import os

from PIL import Image, ImageFilter


def apply_step(image, step):
    kind = step["type"]
    if kind == "grayscale":
        return image.convert("L")
    if kind == "flip_horizontal":
        return image.transpose(Image.Transpose.FLIP_LEFT_RIGHT)
    if kind == "flip_vertical":
        return image.transpose(Image.Transpose.FLIP_TOP_BOTTOM)
    if kind == "scale":
        factor = step["factor"]
        # NOTE: round(), not int() -- truncating would bias every scaled dimension down
        new_size = (max(1, round(image.width * factor)), max(1, round(image.height * factor)))
        return image.resize(new_size, resample=Image.Resampling.BILINEAR)
    if kind == "blur":
        return image.filter(ImageFilter.GaussianBlur(radius=step["radius"]))
    if kind == "rotate":
        return image.rotate(step["angle"], resample=Image.Resampling.BILINEAR, expand=True)
    raise ValueError(f"unknown transformation type: {kind!r}")


def run_pipeline(image, steps):
    for step in steps:
        image = apply_step(image, step)
    return image


def _load_transformations(transformation_dir):
    """json_name -> its ordered list of steps."""
    steps_by_json = {}
    for json_name in sorted(os.listdir(transformation_dir)):
        with open(os.path.join(transformation_dir, json_name)) as f:
            steps_by_json[json_name] = json.load(f)["transformations"]
    return steps_by_json


def process_images(image_dir, transformation_dir, output_dir, get_output_path):
    steps_by_json = _load_transformations(transformation_dir)
    for image_name in sorted(os.listdir(image_dir)):
        with Image.open(os.path.join(image_dir, image_name)) as base:
            base.load()  # decode once per image
            for json_name, steps in steps_by_json.items():
                # NOTE: base is reused for every json_name -- safe since apply_step never mutates it
                run_pipeline(base, steps).save(get_output_path(image_name, json_name), format="PNG")
```

对于 $N$ 张图片和长度总和为 $S$ 步的 $M$ 条流水线，这一实现恰好调用 $N$ 次 `Image.open`、$`N \times S`$
次 `apply_step`，全部在一个进程、一个核心上完成。

### Part 2

`process_images` 从结构上就是单核的：它的 $N \times S$ 个步骤全都在同一个解释器里依次执行。Pillow
的变换是 CPU 密集型的——每一步都在计算新的像素值，不等待任何东西——所以用满多个核心的自然做法是
`ProcessPoolExecutor`：每个工作进程都是独立的操作系统进程，有自己的解释器、自己的 GIL，因此 $W$ 个
工作进程能真正同时跑 $W$ 个步骤。共享同一个解释器的 `ThreadPoolExecutor` 对普通 Python 代码做不到
这一点，因为 GIL 规定同一时刻只有一个线程能执行 Python 字节码——但 Pillow 的 C 扩展不是普通的 Python
代码：它会在自己最重的若干逐像素循环上释放 GIL，包括解码、`resize`，以及 `GaussianBlur` 最终编译成
的那个基于卷积的滤波器。恰好调用这些操作的线程确实能在不同核心上重叠执行，但其余部分——`json.load`、
字典相关的簿记，以及每次 `apply_step` 调用本身的解释器开销——仍然是串行的；进程池则会无条件地把整个
工作单元并行化，这就是 `process_images_parallel` 选用进程池的原因（追问部分会讨论线程池何时变得有
竞争力）。

跨越进程边界本身也有代价：传给工作进程的每个参数、它返回的每个值，都要经过 pickle 序列化、通过管道
发送。一个已解码的 `Image` 会把完整的像素缓冲区也序列化进去，所以把它传给工作进程——或者让工作进程
处理完后传回来，而不是就地保存——花的代价可能比变换本身还大；因此这里传给工作进程的是图片的*路径*。
把一张图片和作用于它的每一条流水线归并成一个任务，而不是拆成 $N \times M$ 个只含一条流水线的独立
任务，就意味着一个工作进程对同一路径调用 `Image.open` 恰好一次，不论后面跟着多少条流水线。
`get_output_path` 本身也从不跨越这条边界：它是一个任意的普通可调用对象，并不保证可被 pickle，所以
每个输出路径都在主进程里解析好，跨边界传递的只是解析出的字符串。

```python
from concurrent.futures import ProcessPoolExecutor


def _process_one_image(task):
    image_path, steps_by_json, output_paths = task
    with Image.open(image_path) as base:  # NOTE: decode from a path, inside the worker -- not a pickled Image
        base.load()
        for json_name, steps in steps_by_json.items():
            run_pipeline(base, steps).save(output_paths[json_name], format="PNG")


def _dispatch(image_dir, transformation_dir, output_dir, get_output_path, worker, max_workers):
    steps_by_json = _load_transformations(transformation_dir)
    tasks = []
    for image_name in sorted(os.listdir(image_dir)):
        image_path = os.path.join(image_dir, image_name)
        # NOTE: resolved here, in the main process -- get_output_path itself never has to be picklable
        output_paths = {json_name: get_output_path(image_name, json_name) for json_name in steps_by_json}
        tasks.append((image_path, steps_by_json, output_paths))
    workers = max_workers or os.cpu_count() or 1
    chunksize = max(1, len(tasks) // (4 * workers))  # fewer IPC round-trips when there are many images
    with ProcessPoolExecutor(max_workers=max_workers) as pool:
        return list(pool.map(worker, tasks, chunksize=chunksize))


def process_images_parallel(image_dir, transformation_dir, output_dir, get_output_path, max_workers=None):
    _dispatch(image_dir, transformation_dir, output_dir, get_output_path, _process_one_image, max_workers)
```

`apply_step` 的调用次数与 Part 1 相比并没有变化，仍是 $N \times S$ 次——变的只是它们在核心间的
分布——所以能达到的最大加速比被 $\min(W, N)$ 限定：一张图片自己的所有流水线都在解码它的那一个任务里
运行，因此如果图片数远少于核心数、而每张图片又有很多条流水线，大部分工作进程会闲置。Part 3 并不会
改变这个上限：它减少的是每个任务内部的工作量，而不是任务的数量。把一张图片的流水线拆到多个任务里可以
放宽这个上限，代价是这张图片要被解码不止一次——这笔交易是否划算，取决于解码相对于后续变换有多昂贵。

### Part 3

把每条流水线都归并到同一个任务里——Part 2 已经这样做了——正是让共享成为可能、且不需要在工作进程之间
增加任何通信的原因：一张图片的各条流水线，只会在已经持有它们全部、以及那一份解码好的图片的那一个任务
内部互相比较，所以除非同一个工作进程恰好同时拥有两条流水线，否则它们永远发现不了共享的前缀——而按
图片分组恰好保证了这一点。

两个步骤在字典树里对应同一个节点，当且仅当它们的 `"type"` 和每个参数都一致，这里用比较
`sorted(step.items())` 来判断，而不是直接比较步骤字典自身的键顺序：JSON 并不保证一个步骤的键是按
什么顺序写的，两个描述同一个步骤、只是键的顺序被交换过的文件，仍须能够共享。`_build_trie` 从根节点
开始遍历每条流水线的步骤，只要当前节点下已经存在下一个步骤对应的键就复用那个子节点，并记录哪些 json
名恰好在某个节点终止——当一条流水线的步骤正好是另一条更长流水线的前缀时，它终止的那个节点也可以同时
带有子节点。

`_run_trie` 就是一次朴素的深度优先遍历：把当前节点自身的步骤应用到父节点算出的图片上，为所有在这里
终止的流水线各保存一份输出，然后在返回之前递归进入每个子节点。`image` 是一个局部变量，只被根节点到
当前所在节点之间的那些栈帧持有——每一层深度最多一张图片，上限由共享这张图片的最长流水线决定，而不是
整棵字典树里每个不同前缀各留一份。这里在分叉进入多个子节点之前，不需要防御性地拷贝 `image`：因为
`apply_step` 从不修改自己的输入（见 Part 1），同一个 `image` 对象可以原样交给某个节点的每一个子
节点，一个子节点后续的操作不会影响另一个——这是恰好这六种变换才有的性质，一旦加入用 `ImageDraw` 或
`Image.paste` 这类原地修改的 Pillow 调用实现的步骤，就需要重新审视这一点。

```python
def _step_key(step):
    # NOTE: sorted() so a step's own JSON key order never matters for matching it against another
    return tuple(sorted(step.items()))


class _TrieNode:
    __slots__ = ("step", "children", "leaves")

    def __init__(self, step=None):
        self.step = step      # the step that produced this node's image from its parent's; None at the root
        self.children = {}    # _step_key(step) -> _TrieNode
        self.leaves = []      # json_names whose pipeline ends exactly at this node


def _build_trie(steps_by_json):
    root = _TrieNode()
    for json_name, steps in steps_by_json.items():
        node = root
        for step in steps:
            key = _step_key(step)
            if key not in node.children:  # NOTE: children is per-node, so the same step reached via a
                node.children[key] = _TrieNode(step)  # different prefix still gets its own node here
            node = node.children[key]
        node.leaves.append(json_name)
    return root


def _run_trie(node, image, output_paths):
    if node.step is not None:
        image = apply_step(image, node.step)
    applications = 0 if node.step is None else 1
    for json_name in node.leaves:
        image.save(output_paths[json_name], format="PNG")
    for child in node.children.values():
        applications += _run_trie(child, image, output_paths)
    return applications


def _process_one_image_shared_prefix(task):
    image_path, steps_by_json, output_paths = task
    trie = _build_trie(steps_by_json)
    with Image.open(image_path) as base:
        base.load()
        return _run_trie(trie, base, output_paths)


def process_images_shared_prefix(image_dir, transformation_dir, output_dir, get_output_path, max_workers=None):
    _dispatch(image_dir, transformation_dir, output_dir, get_output_path, _process_one_image_shared_prefix,
              max_workers)
```

对于长度分别为 $L_1, \dots, L_J$ 的多条流水线（作用于同一张图片），共享把 Part 2 里 `apply_step` 的
$\sum_j L_j$ 次调用，变成了恰好等于字典树中不同节点个数的次数——上限是 $\sum_j L_j$（任意两条流水线
都不共享任何前缀时），下限是 $\max_j L_j$（每条流水线都是最长那条的前缀时）。`_run_trie` 的返回值
直接给出这个数字，每次真正应用了一个步骤就计一次。

### 追问

- 损坏的图片直接整张跳过——它涉及的所有组合都不产生输出，因为关于它的一切都不可信；而未知的
  `"type"` 或缺失/非法的参数，只会让它所属的那一个 `(image, json)` 组合失败。两种情况都只记录下来，
  而不是一路抛出到最外层，这样一个坏文件就不会中断整批任务。
- 用同样方式（每张图片一个任务）搭建的 `ThreadPoolExecutor`，一旦性能分析表明工作量确实以那些会
  释放 GIL 的 Pillow 调用为主，或者图片数量多到 $W$ 个进程各自的固定内存开销已经超过了残余 GIL 争用
  的代价，就值得一试；线程还不需要 pickle 序列化，并且可以在所有任务之间共享同一份只读的已解码图片
  缓存，这是进程池若不借助额外机制做不到的。
- 普通的 JPEG 或 PNG，第一次访问像素时就会被整张解码，所以超出内存大小的图片必须避免在全分辨率下
  走到这一步：`Image.draft(mode, size)` 能让 Pillow 的 JPEG 解码器直接产出一张更小的图片；而真正按
  瓦片存储的格式，通过为其设计的库（`pyvips`、`rasterio`）读取，可以逐条带处理——`grayscale`、两种
  翻转以及 `blur` 都足够局部，可以这样处理（`blur` 需要每条带多读大约 `3 * radius` 像素的边距，这样
  卷积核在接缝处才有真实的邻居），但 `rotate` 会混合整张图片各处的像素，不论用什么库都很难被干净地
  分块处理。
- 要定位瓶颈，可以用两种方式为一批有代表性的任务计时：一种保留全部步骤，另一种跳过所有变换、只保留
  解码和重新编码。两者之间的差距若随核心数扩大而扩大，说明这部分时间花在变换本身的 CPU 计算上；如果
  增加工作进程后墙钟时间并不缩短，同时每个工作进程的 CPU 占用都不高，则说明瓶颈在磁盘或网络 I/O 上。
- 跨多台机器时，按图片名的哈希值对图片（而不是单个组合）分区，让每台机器分到的份额互不相交，各自在
  自己那一份上原样运行 Part 2 或 Part 3；前缀共享仍应局限在每台机器自己的工作进程内部，因为把解码好
  的中间结果通过网络传输，几乎总是比重新计算它本身更贵。

<details>
<summary>验证代码（可运行）</summary>

```python
import random
import tempfile

# ---------- setup helpers ----------


def make_get_output_path(base_out):
    def get_output_path(image_name, json_name):
        path = os.path.join(base_out, f"{image_name}__{json_name}.png")
        os.makedirs(os.path.dirname(path), exist_ok=True)
        return path

    return get_output_path


def pixels(path):
    """(mode, size, raw bytes) -- PNG is lossless, so this is an exact pixel comparison."""
    with Image.open(path) as im:
        return (im.mode, im.size, im.tobytes())


def brute_force_pipeline(image_path, steps, out_path):
    """Independent of apply_step / run_pipeline: the simplest possible per-pipeline implementation."""
    img = Image.open(image_path)
    img.load()
    for step in steps:
        kind = step["type"]
        if kind == "grayscale":
            img = img.convert("L")
        elif kind == "flip_horizontal":
            img = img.transpose(Image.Transpose.FLIP_LEFT_RIGHT)
        elif kind == "flip_vertical":
            img = img.transpose(Image.Transpose.FLIP_TOP_BOTTOM)
        elif kind == "scale":
            w = max(1, round(img.width * step["factor"]))
            h = max(1, round(img.height * step["factor"]))
            img = img.resize((w, h), resample=Image.Resampling.BILINEAR)
        elif kind == "blur":
            img = img.filter(ImageFilter.GaussianBlur(radius=step["radius"]))
        elif kind == "rotate":
            img = img.rotate(step["angle"], resample=Image.Resampling.BILINEAR, expand=True)
        else:
            raise ValueError(kind)
    img.save(out_path, format="PNG")


def independent_distinct_prefix_count(steps_by_json):
    """Counts distinct (image-local) prefixes directly from the JSON steps -- no trie, no solution code."""
    seen = set()
    for steps in steps_by_json.values():
        prefix = ()
        for step in steps:
            prefix = prefix + (tuple(sorted(step.items())),)
            seen.add(prefix)
    return len(seen)


def write_transformations(transformation_dir, steps_by_json):
    for json_name, steps in steps_by_json.items():
        with open(os.path.join(transformation_dir, json_name), "w") as f:
            json.dump({"transformations": steps}, f)


def make_random_image(rng, path, min_side=3, max_side=10):
    w, h = rng.randint(min_side, max_side), rng.randint(min_side, max_side)
    data = bytes(rng.randrange(256) for _ in range(w * h * 3))
    Image.frombytes("RGB", (w, h), data).save(path, format="PNG")


def compare_all(image_dir, transformation_dir, steps_by_json, image_names, root, worker_counts):
    """Runs process_images, process_images_parallel and process_images_shared_prefix (each worker
    count in worker_counts), compares every output against the brute force, and returns the
    shared-prefix application counts, one per worker count (must all agree)."""
    out_seq = os.path.join(root, "out_seq")
    out_brute = os.path.join(root, "out_brute")
    os.makedirs(out_seq, exist_ok=True)
    os.makedirs(out_brute, exist_ok=True)
    process_images(image_dir, transformation_dir, out_seq, make_get_output_path(out_seq))
    for image_name in image_names:
        for json_name, steps in steps_by_json.items():
            brute_force_pipeline(os.path.join(image_dir, image_name), steps,
                                  os.path.join(out_brute, f"{image_name}__{json_name}.png"))

    par_dirs, shared_dirs, shared_counts = {}, {}, {}
    for workers in worker_counts:
        out_par = os.path.join(root, f"out_par{workers}")
        os.makedirs(out_par, exist_ok=True)
        process_images_parallel(image_dir, transformation_dir, out_par, make_get_output_path(out_par),
                                 max_workers=workers)
        par_dirs[workers] = out_par

        out_shared = os.path.join(root, f"out_shared{workers}")
        os.makedirs(out_shared, exist_ok=True)
        counts = _dispatch(image_dir, transformation_dir, out_shared, make_get_output_path(out_shared),
                            _process_one_image_shared_prefix, workers)
        shared_dirs[workers] = out_shared
        shared_counts[workers] = sum(counts)

    for image_name in image_names:
        for json_name in steps_by_json:
            fn = f"{image_name}__{json_name}.png"
            want = pixels(os.path.join(out_brute, fn))
            assert pixels(os.path.join(out_seq, fn)) == want, ("sequential", fn)
            for workers in worker_counts:
                assert pixels(os.path.join(par_dirs[workers], fn)) == want, ("parallel", workers, fn)
                assert pixels(os.path.join(shared_dirs[workers], fn)) == want, ("shared-prefix", workers, fn)

    assert len(set(shared_counts.values())) == 1, shared_counts  # worker count must not change the work done
    return shared_counts[worker_counts[0]]


# ---------- the worked examples given in the Problem, exactly as stated ----------

with tempfile.TemporaryDirectory() as example_root:
    Image.new("RGB", (60, 40)).save(os.path.join(example_root, "sample.png"), format="PNG")
    with Image.open(os.path.join(example_root, "sample.png")) as im:
        assert (im.mode, im.size) == ("RGB", (60, 40))
        step1 = apply_step(im, {"type": "grayscale"})
        assert (step1.mode, step1.size) == ("L", (60, 40))
        step2 = apply_step(step1, {"type": "scale", "factor": 0.5})
        assert (step2.mode, step2.size) == ("L", (30, 20))
        step3 = apply_step(step2, {"type": "rotate", "angle": 90})
        assert (step3.mode, step3.size) == ("L", (20, 30))

with tempfile.TemporaryDirectory() as root:
    image_dir = os.path.join(root, "images")
    transformation_dir = os.path.join(root, "transformations")
    out_dir = os.path.join(root, "out")
    os.makedirs(image_dir)
    os.makedirs(transformation_dir)
    os.makedirs(out_dir)
    for name in ("a.png", "b.png"):
        Image.new("RGB", (8, 6), (5, 5, 5)).save(os.path.join(image_dir, name), format="PNG")
    write_transformations(transformation_dir, {"x.json": [{"type": "grayscale"}],
                                                "y.json": [{"type": "flip_horizontal"}]})
    process_images(image_dir, transformation_dir, out_dir, make_get_output_path(out_dir))
    written = set(os.listdir(out_dir))
    assert written == {"a.png__x.json.png", "a.png__y.json.png", "b.png__x.json.png", "b.png__y.json.png"}

PART3_EXAMPLE = {
    "p1.json": [{"type": "grayscale"}, {"type": "scale", "factor": 0.5}, {"type": "rotate", "angle": 90}],
    "p2.json": [{"type": "grayscale"}, {"type": "scale", "factor": 0.5}, {"type": "blur", "radius": 2}],
    "p3.json": [{"type": "grayscale"}, {"type": "scale", "factor": 0.5}, {"type": "rotate", "angle": 90}],
    "p4.json": [{"type": "flip_horizontal"}, {"type": "blur", "radius": 2}],
}
assert sum(len(steps) for steps in PART3_EXAMPLE.values()) == 11  # naive, stated in the Problem
assert independent_distinct_prefix_count(PART3_EXAMPLE) == 6  # with sharing, stated in the Problem

with tempfile.TemporaryDirectory() as root:
    image_dir = os.path.join(root, "images")
    transformation_dir = os.path.join(root, "transformations")
    os.makedirs(image_dir)
    os.makedirs(transformation_dir)
    rng = random.Random(0)
    make_random_image(rng, os.path.join(image_dir, "leaf.png"))
    write_transformations(transformation_dir, PART3_EXAMPLE)
    shared_total = compare_all(image_dir, transformation_dir, PART3_EXAMPLE, ["leaf.png"], root, (1, 2))
    assert shared_total == 6
    out_shared1 = os.path.join(root, "out_shared1")
    p1 = pixels(os.path.join(out_shared1, "leaf.png__p1.json.png"))
    p3 = pixels(os.path.join(out_shared1, "leaf.png__p3.json.png"))
    assert p1 == p3  # p1 and p3 are identical pipelines

# ---------- edge cases: empty pipeline, a pipeline that is a strict prefix of another, upscale,
# the minimum-size-1 clamp, and a step whose JSON keys are written in a different order ----------

with tempfile.TemporaryDirectory() as root:
    image_dir = os.path.join(root, "images")
    transformation_dir = os.path.join(root, "transformations")
    os.makedirs(image_dir)
    os.makedirs(transformation_dir)
    rng = random.Random(1)
    image_names = [f"img{i}.png" for i in range(4)]
    sizes = [(12, 8), (5, 5), (20, 6), (7, 7)]
    for name, (w, h) in zip(image_names, sizes):
        data = bytes(rng.randrange(256) for _ in range(w * h * 3))
        Image.frombytes("RGB", (w, h), data).save(os.path.join(image_dir, name), format="PNG")

    edge_cases = {
        "empty1.json": [],
        "empty2.json": [],
        "short.json": [{"type": "grayscale"}],
        "long.json": [{"type": "grayscale"}, {"type": "scale", "factor": 0.5}],  # short.json is its prefix
        "upscale.json": [{"type": "scale", "factor": 2.0}],
        "shrink_to_one.json": [{"type": "scale", "factor": 0.1}],  # forces the minimum-size-1 clamp
        "odd_angle.json": [{"type": "scale", "factor": 0.7}, {"type": "flip_vertical"},
                            {"type": "rotate", "angle": 33.5}],
        "key_order_a.json": [{"type": "blur", "radius": 3}],
        "key_order_b.json": [{"radius": 3, "type": "blur"}],  # same step, keys written in a different order
    }
    write_transformations(transformation_dir, edge_cases)
    edge_total = compare_all(image_dir, transformation_dir, edge_cases, image_names, root, (1, 3))
    assert edge_total == independent_distinct_prefix_count(edge_cases) * len(image_names)

    out_shared1 = os.path.join(root, "out_shared1")
    with Image.open(os.path.join(image_dir, "img0.png")) as src:
        empty_pixels = pixels(os.path.join(out_shared1, "img0.png__empty1.json.png"))
        assert empty_pixels == (src.mode, src.size, src.tobytes())
    assert pixels(os.path.join(out_shared1, "img0.png__empty1.json.png")) == \
        pixels(os.path.join(out_shared1, "img0.png__empty2.json.png"))
    with Image.open(os.path.join(out_shared1, "img1.png__shrink_to_one.json.png")) as im:
        assert im.size == (1, 1)  # 5x5 * 0.1 rounds to 0 in each dimension, clamped up to 1
    assert pixels(os.path.join(out_shared1, "img0.png__key_order_a.json.png")) == \
        pixels(os.path.join(out_shared1, "img0.png__key_order_b.json.png"))

# ---------- randomized trials: fresh images and pipelines each time, some sharing common segments ----------


def random_step(rng):
    kind = rng.choice(["grayscale", "flip_horizontal", "flip_vertical", "scale", "blur", "rotate"])
    if kind == "scale":
        return {"type": "scale", "factor": rng.choice([0.5, 0.75, 1.5, 2.0])}
    if kind == "blur":
        return {"type": "blur", "radius": rng.choice([0, 1, 2, 3])}
    if kind == "rotate":
        return {"type": "rotate", "angle": rng.choice([45, 90, 180, 270])}
    return {"type": kind}


def random_pipeline(rng, segments):
    steps = list(rng.choice(segments)) if segments and rng.random() < 0.6 else []
    steps += [random_step(rng) for _ in range(rng.randint(0, 3))]
    return steps


total_naive, total_shared = 0, 0
for trial in range(5):
    rng = random.Random(trial)
    with tempfile.TemporaryDirectory() as root:
        image_dir = os.path.join(root, "images")
        transformation_dir = os.path.join(root, "transformations")
        os.makedirs(image_dir)
        os.makedirs(transformation_dir)
        image_names = [f"img{i}.png" for i in range(3)]
        for name in image_names:
            make_random_image(rng, os.path.join(image_dir, name))
        segments = [[random_step(rng) for _ in range(rng.randint(1, 2))] for _ in range(3)]
        steps_by_json = {f"pl{i}.json": random_pipeline(rng, segments) for i in range(6)}
        write_transformations(transformation_dir, steps_by_json)

        shared_total = compare_all(image_dir, transformation_dir, steps_by_json, image_names, root, (1, 3))
        assert shared_total == independent_distinct_prefix_count(steps_by_json) * len(image_names)
        total_naive += sum(len(steps) for steps in steps_by_json.values()) * len(image_names)
        total_shared += shared_total

assert total_shared < total_naive  # the random segments must have produced genuine sharing

print("all checks passed")
```

</details>

</details>
