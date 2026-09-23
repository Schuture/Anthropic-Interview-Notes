# Image Processing Pipeline

[English](README.md) · [中文](README.zh.md)

<!-- meta:begin -->
| Type | Priority | Difficulty | Roles | Topics | Format | Round |
| --- | --- | --- | --- | --- | --- | --- |
| Coding · concurrency, image processing | ★★★★★ | Medium | SWE · MLE · Infra Eng | image-processing, pillow, multiprocessing, concurrency, caching | 3 parts / 55 min | Technical phone screen · Onsite |
<!-- meta:end -->

## Problem

Three folders and a helper are given up front, all as absolute paths. `image_dir` holds the source
images: every file directly inside it decodes, through Pillow, to an `"RGB"` image, and its filename
(with extension) is its *image name*. `transformation_dir` holds *pipeline files*: every file directly
inside it is a JSON file whose filename is its *json name*, with the shape

```text
{"transformations": [{"type": "<type>", ...params}, ...]}
```

— an ordered list of *steps*, each a JSON object whose `"type"` selects one of six transformations
together with whatever parameters that type needs; the steps a json name lists, in order, are its
*pipeline*. `output_dir` is where results go, through the given helper

```py
def get_output_path(image_name: str, json_name: str) -> str: ...
```

which returns the absolute path to write that image's result under that pipeline to; the directory it
lives in already exists.

For every image in `image_dir` and every pipeline in `transformation_dir` — every one of the
$|\text{images}| \times |\text{pipelines}|$ combinations — run that pipeline's steps against that
image, in order, and save the result at `get_output_path(image_name, json_name)` as a PNG. A
pipeline's first step takes the freshly decoded source image as input; every later step takes the
previous step's output. A pipeline may list zero steps, in which case its output is the source image,
unchanged, saved as a PNG.

Six step types are supported, named by the step's `"type"` field. `grayscale` and the two flips take
no other parameters; `scale`, `blur` and `rotate` each take one, named below.

- `grayscale` converts the image to Pillow's single-channel mode `"L"` (`image.convert("L")`).
- `flip_horizontal` mirrors the image left-to-right (`image.transpose(Image.Transpose.FLIP_LEFT_RIGHT)`).
- `flip_vertical` mirrors the image top-to-bottom (`image.transpose(Image.Transpose.FLIP_TOP_BOTTOM)`).
- `scale` — `"factor"`, a positive float — resizes the image to
  `(max(1, round(width * factor)), max(1, round(height * factor)))`, where `width` and `height` are
  its current size, `round` is Python's round-half-to-even, and the resampling filter is
  `Image.Resampling.BILINEAR`.
- `blur` — `"radius"`, a non-negative int — applies a Gaussian blur of that radius
  (`image.filter(ImageFilter.GaussianBlur(radius=radius))`).
- `rotate` — `"angle"`, a float in degrees — rotates the image counter-clockwise about its centre by
  `angle` degrees, expanding the canvas (`expand=True`) so the whole rotated image fits, with the
  newly exposed corners filled in black (`0` in mode `"L"`, `(0, 0, 0)` in mode `"RGB"`); the
  resampling filter is again `Image.Resampling.BILINEAR`.

`grayscale` is the only step that changes an image's mode; `scale` and `rotate` are the only ones that
change its size; every other combination of mode and size is left exactly as it was.

For example, `sample.png` decodes to an `"RGB"` image of size `(60, 40)`. The pipeline in `p.json`,

```json
{"transformations": [{"type": "grayscale"}, {"type": "scale", "factor": 0.5}, {"type": "rotate", "angle": 90}]}
```

is applied like this:

```text
decode                            mode "RGB", size (60, 40)
grayscale                      -> mode "L",   size (60, 40)   # mode changes; size does not
scale(factor=0.5)              -> mode "L",   size (30, 20)   # round(60*0.5)=30, round(40*0.5)=20
rotate(angle=90, expand=True)  -> mode "L",   size (20, 30)   # a multiple of 90 degrees swaps width and height
```

and the result is saved at `get_output_path("sample.png", "p.json")`.

### Part 1 — Sequential pipeline

Implement `process_images`.

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
process_images writes exactly 4 files, one per (image, pipeline) pair:
  get_output_path("a.png", "x.json")
  get_output_path("a.png", "y.json")
  get_output_path("b.png", "x.json")
  get_output_path("b.png", "y.json")
```

### Part 2 — Parallel batch

`image_dir` may now hold images large enough, and `transformation_dir` pipelines numerous enough,
that running them through `process_images` sequentially is too slow to be practical. Implement
`process_images_parallel`: the same outputs as `process_images`, for every input, but spread across
more than one CPU core. It must not decode the same source image once per pipeline that uses it, and
nothing but file paths and JSON-sized data may cross a process boundary — never a decoded image or its
raw pixel bytes.

```py
def process_images_parallel(
    image_dir: str,
    transformation_dir: str,
    output_dir: str,
    get_output_path: Callable[[str, str], str],
    max_workers: int | None = None,
) -> None: ...
```

`max_workers` bounds how many worker processes run at once; omitted or `None` uses `os.cpu_count()`.

```text
big_images/ has 200 images; transformations/ has 5 pipeline files.
process_images_parallel must still write all 200 * 5 = 1,000 outputs,
decoding only 200 images in total -- once each -- rather than 1,000.
```

### Part 3 — Shared-prefix reuse

A *prefix* of a pipeline is its first $k$ steps, for some $0 \le k \le$ its length; two prefixes are
*the same prefix* only if they have the same length and, step by step, the same `"type"` and the same
parameter value(s). Implement `process_images_shared_prefix`: the same signature, outputs and
cross-process rules as `process_images_parallel`, but for a given image, every prefix shared by two or
more of its pipelines is computed only once — continuing one of them past that prefix must not visibly
affect another that shares it. Sharing only ever happens between pipelines applied to the *same*
image; two pipelines for different images never share work, even when their steps are identical. Do
not keep a shared prefix's image alive for the whole run: once every pipeline that needed it has used
it, it may be released.

```py
def process_images_shared_prefix(
    image_dir: str,
    transformation_dir: str,
    output_dir: str,
    get_output_path: Callable[[str, str], str],
    max_workers: int | None = None,
) -> None: ...
```

For one image with the pipelines

```text
p1: grayscale -> scale(factor=0.5) -> rotate(angle=90)
p2: grayscale -> scale(factor=0.5) -> blur(radius=2)
p3: grayscale -> scale(factor=0.5) -> rotate(angle=90)      # identical to p1
p4: flip_horizontal -> blur(radius=2)
```

applying every pipeline independently would apply `grayscale` and `scale(factor=0.5)` three times each
(p1, p2, p3), `rotate(angle=90)` twice (p1, p3), and `flip_horizontal` and `blur(radius=2)` once each
for p4: 11 step applications. With sharing, `grayscale` and `scale(factor=0.5)` are each applied once
and reused by p1, p2 and p3; `rotate(angle=90)` is applied once, and its result is saved as the output
of both p1 and p3; the `blur(radius=2)` after `flip_horizontal` is a separate application from the one
after `scale(factor=0.5)` in p2, since the two are reached by different prefixes. Six step applications
in total: `grayscale`, `scale(factor=0.5)`, `rotate(angle=90)`, `blur(radius=2)` (after `scale`),
`flip_horizontal`, `blur(radius=2)` (after `flip_horizontal`).

## Reference solution

<details>
<summary>Show the reference solution</summary>

Two points worth confirming before starting: whether every image and every pipeline file can be
assumed well-formed (assumed here for all three parts; Follow-ups covers a corrupt image, an unknown
step type and a bad parameter), and whether the checker compares pixels against a specific library's
output bit-for-bit — Pillow's own operations are used throughout, with the exact filters and rounding
stated in the Problem, so the answer is exact once that library choice is settled.

### Part 1

`apply_step` is the single place that knows what each of the six `"type"` values means; every later
part reuses it unchanged. Pillow's `convert`, `transpose`, `resize`, `filter` and `rotate` all return a
new `Image` rather than modifying their receiver, so `run_pipeline` can reassign `image` on every
iteration without worrying about aliasing, and `process_images` can decode one image and feed it to
every one of its pipelines: none of them can see, let alone undo, what an earlier pipeline did to that
same starting point.

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

For $N$ images and $M$ pipelines whose lengths sum to $S$ steps, this makes exactly $N$ calls to
`Image.open` and $N \times S$ calls to `apply_step`, all in one process, one core.

### Part 2

`process_images` is single-core by construction: every one of its $N \times S$ steps runs on the same
interpreter, one after another. Pillow's transformations are CPU-bound — each spends its time computing
new pixel values, not waiting on anything — so the natural way to use more than one core is
`ProcessPoolExecutor`: each worker is a separate operating-system process with its own interpreter and
its own GIL, so $W$ workers really do run $W$ steps at once. A `ThreadPoolExecutor` sharing one
interpreter cannot do that for ordinary Python code, since the GIL lets only one thread run Python
bytecode at a time — but Pillow's C extension is not ordinary Python code: it releases the GIL around
several of its heaviest per-pixel loops, including decoding, `resize`, and the convolution-based filter
that `GaussianBlur` compiles down to. Threads calling into exactly those operations do overlap on
separate cores, but everything else — `json.load`, dictionary bookkeeping, the interpreter overhead of
every `apply_step` call — stays serialized; a process pool parallelizes the whole worker unconditionally
instead, which is why `process_images_parallel` uses one (Follow-ups covers when a thread pool becomes
competitive).

Crossing a process boundary costs something too: every argument passed to a worker, and every value it
returns, is pickled and sent through a pipe. A decoded `Image` pickles its full pixel buffer, so handing
one to a worker — or handing a processed one back instead of saving it there — can cost more than the
transformation itself; workers are given the image's *path* instead. Grouping one image with every
pipeline that applies to it into a single task, rather than submitting $N \times M$ separate
one-pipeline tasks, means a worker calls `Image.open` on that path exactly once no matter how many
pipelines follow. `get_output_path` itself never crosses the boundary either: it is a plain, arbitrary
callable that is not guaranteed to be picklable, so every output path is resolved once in the main
process and only the resulting strings travel to the worker.

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

The $N \times S$ calls to `apply_step` are unchanged in count from Part 1 — only their distribution
across cores changes — so the best possible speed-up is bounded by $\min(W, N)$: an image's own
pipelines all run inside the one task that decoded it, so if there are far fewer images than cores and
each has many pipelines, most of the pool sits idle. Part 3 does not change that cap; it reduces how
much work is inside each task, not how many tasks there are. Splitting one image's pipelines across
several tasks would widen the cap at the cost of decoding that image more than once — which trade wins
depends on how expensive decoding is relative to the transformations that follow it.

### Part 3

Grouping every pipeline under one task, as Part 2 already does, is what makes sharing possible without
adding any communication between workers: an image's pipelines are only ever compared against each
other inside the single task that already holds all of them and its one decoded copy of the image, so
two pipelines never discover a shared prefix unless the same worker happens to own both of them — which
grouping by image guarantees.

Two steps are the same node in the trie only if their `"type"` and every parameter match, checked by
comparing `sorted(step.items())` rather than the step dictionaries' own key order: JSON does not
promise which order a step's keys were written in, and two files describing the identical step with
their keys swapped must still share. `_build_trie` walks every pipeline's steps from the root, reusing
a child whenever the next step's key already exists at the current node, and records which json names
end at each node — a pipeline may end at a node that also has children, when its steps are an exact
prefix of a longer pipeline's.

`_run_trie` is a plain depth-first walk: apply this node's own step to the image its parent computed,
save an output for every pipeline ending here, then recurse into every child before returning. `image`
is a local variable, so it is kept alive only by the stack frames between the root and whichever node
is currently being visited — at most one image per depth, bounded by the longest pipeline sharing this
image, never one per distinct prefix in the whole trie at once. No step here needs to defensively copy
`image` before branching into more than one child: since `apply_step` never mutates its input (Part 1),
the same `image` object can be handed to every child of a node without one child's continuation
affecting another's — a property of exactly these six transformations, worth re-examining before adding
a step implemented with an in-place Pillow call such as `ImageDraw` or `Image.paste`.

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

For one image with pipelines of lengths $L_1, \dots, L_J$, sharing turns $\sum_j L_j$ applications of
`apply_step` (Part 2's count) into exactly the number of distinct nodes in the trie — at most
$\sum_j L_j$, when no two pipelines share anything, and as low as $\max_j L_j$, when every pipeline is a
prefix of the longest one. `_run_trie`'s own return value counts that number directly, one per call that
actually applied a step.

### Follow-ups

- A corrupt image is skipped entirely — no output for any of its pairs, since nothing about it can be
  trusted — while a step with an unknown `"type"` or a missing or invalid parameter fails only the one
  `(image, json)` pair it belongs to; both are recorded rather than raised all the way up, so one bad
  file never aborts the batch.
- A `ThreadPoolExecutor` built the same way (one task per image) is worth it once profiling shows the
  workload is dominated by the GIL-releasing Pillow calls, or once there are enough images that $W$
  processes' fixed memory overhead outweighs the residual GIL contention; threads also need no pickling
  and can share one read-only cache of decoded images across every task, which a process pool cannot do
  without extra machinery.
- An ordinary JPEG or PNG is decoded whole the first time a pixel is touched, so an image larger than
  memory has to avoid ever reaching that point at full resolution: `Image.draft(mode, size)` lets
  Pillow's JPEG decoder produce a smaller image directly, and a genuinely tiled format read through a
  library built for it (`pyvips`, `rasterio`) can be processed strip by strip — grayscale, the flips and
  blur are local enough for that (blur needs each strip read with a margin of about `3 * radius` pixels
  so the kernel has real neighbours at the seam), but rotate mixes pixels from across the whole image and
  resists clean tiling regardless of the library.
- To find the bottleneck, time a representative batch two ways: with every step, and with the
  transformations skipped so only decode and re-encode remain. A gap between the two that scales with
  core count is CPU time inside the transformations; a wall-clock time that does not shrink as workers
  are added, while each worker's CPU usage stays low, points at disk or network I/O instead.
- Across several machines, partition images — not individual pairs — by a hash of the image name, so
  each machine's share is disjoint and it can run Part 2 or Part 3 unchanged against its own share; keep
  prefix sharing local to each machine's workers, since shipping a decoded intermediate over the network
  almost always costs more than recomputing it.

<details>
<summary>Checks (runnable)</summary>

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
