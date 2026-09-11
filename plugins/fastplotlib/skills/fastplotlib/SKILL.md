---
name: fastplotlib
description: Write fastplotlib code for interactive scientific visualization. Use when the user asks to plot, visualize, view, or browse data with fastplotlib or fpl; mentions Figure, Subplot, NDWidget, LineCollection, LineStack, ScatterCollection, ImageGraphic, add_image, add_line, add_scatter, add_nd_image or add_nd_timeseries; wants an interactive viewer, a movie or video player, sliders to scroll through an n-dimensional array, a heatmap, a spike raster, calcium imaging traces, pose tracking, an ethogram, or a spectrogram; wants several datasets synchronized on one time axis; or asks about GPU plotting with pygfx or wgpu. Also use for multi-modal neuroscience visualization, including pynapple, spikeinterface, masknmf and NWB data.
license: See LICENSE
metadata:
  project: fastplotlib
  homepage: https://www.fastplotlib.org
---

# Using fastplotlib

Guidance for writing **fastplotlib code for a user**, usually a scientist who wants a visualization
and is not going to debug your output. Get it right the first time, and make it fast and readable.

## Scope, responsibility and scientific integrity

This skill helps write fastplotlib code. It cannot tell anyone whether the result is scientifically
correct.

- **Verifying the scientific integrity of anything you build with LLM/AI tools is your
  responsibility.** It is not the tool's, not this skill's, and not the library's.
- **It is very easy to create visualizations that look right at first glance but are subtly
  wrong.** An axis in the wrong units, two modalities aligned incorrectly, a colormap that
  implies structure the data does not have, a `vmin`/`vmax` that clips the effect you are
  looking for. This is not an exhaustive list. None of these raise an error,
  and _none of them are bugs_.
- **LLMs sound confident.** When real humans communicate they indicate their confidence
  in various ways. LLMs give no measure of that and they are confidently incorrect, which is hard
  to read past even for an expert. LLMs also make judgement calls instead of asking
  you, unless you tell it not to repeatedly. And it will quietly do something subtly different
  from what you asked. It will produce code that runs and _looks_ reasonable,
  which you will not notice unless you already know what the right answer is supposed to look like.
- **LLMs are only tools.** They cannot validate the scientific integrity of your visualization or
  your analysis, and they cannot grasp the full scope of the scientific questions, the experiments,
  or the data you are working with.
- **You must know how to perform sanity checks for your specific datasets and experiments.** Check
  the shapes, units, sampling rates and timebases yourself. Verify any data that has to be
  cross-checked. **If you cannot say how you would detect an error, you are not in a position
  to trust the output**.
- **Garbage in, garbage out.** An LLM is not going to magically produce better analysis or
  visualization from bad data. It will render bad data confidently and attractively.

**Rule of thumb:** if you already know what the code should look like and *typing* or boilerplate is
your real limit, an LLM can be helpful. If you do not know what the answer is even supposed to
look like, proceed with caution and please consult an expert.

When writing code from this skill: state your assumptions about shapes, dtypes, units and sampling
rates explicitly, say which of them you verified and which you assumed, and say plainly what you
could not check.

## Reference material

Read the file for the area you are working in before writing code. Paths are absolute so they
resolve wherever the skill is installed; if `${CLAUDE_SKILL_DIR}` is not substituted, the files are
in the `references/` directory next to this file.

| you are working on | read |
|---|---|
| picking and configuring a graphic, collections | `${CLAUDE_SKILL_DIR}/references/graphics.md` |
| changing data/colors efficiently, events | `${CLAUDE_SKILL_DIR}/references/properties-and-events.md` |
| figures, subplots, cameras, linking views | `${CLAUDE_SKILL_DIR}/references/figures-and-subplots.md` |
| an n-dimensional or multi-modal viewer | `${CLAUDE_SKILL_DIR}/references/ndwidget.md` |
| interactive selection, highlighting | `${CLAUDE_SKILL_DIR}/references/selectors.md` |
| GUI controls (sliders, buttons, colorbars) | `${CLAUDE_SKILL_DIR}/references/imgui-guis.md` |
| writing any imgui UI — windows, toolbars, right-click menus | `${CLAUDE_SKILL_DIR}/references/imgui-guide.md` |
| the arguments or flags of a specific imgui element | `${CLAUDE_SKILL_DIR}/references/imgui-elements.md` |
| cursors, tooltips, text labels | `${CLAUDE_SKILL_DIR}/references/cursors-and-tooltips.md` |
| the `fpl.` namespace, backends, GPUs, transparency, coordinate spaces | `${CLAUDE_SKILL_DIR}/references/namespace-and-backends.md` |
| colormap and array helpers | `${CLAUDE_SKILL_DIR}/references/utils.md` |
| **neuroscience data of any kind** | `${CLAUDE_SKILL_DIR}/references/neuroscience.md` |

Worked examples are in the
[examples gallery](https://www.fastplotlib.org/ver/dev/_gallery/index.html), organised by section
(image, image_volume, heatmap, line, line_collection, scatter, mesh, vectors, text, events,
selection_tools, guis, gridplot, window_layouts, controllers, spaces_transforms, machine_learning,
ipywidgets, qt, misc). The [user guide](https://www.fastplotlib.org/ver/dev/user_guide/index.html)
and [API reference](https://www.fastplotlib.org/ver/dev/api/index.html) are the other references.

## Ground rules

- Go through the fastplotlib documentation and the examples gallery before writing code. When you
  are unsure whether an argument exists, read the class in the installed package — do not guess.
  `python -c "import fastplotlib, pathlib; print(pathlib.Path(fastplotlib.__file__).parent)"`
  prints where it is.
- Use the coding and writing style of fastplotlib. The gallery examples are the reference.
- Do not over-engineer. Do not write low quality code. No wrapper classes, no config dicts, no
  "framework" around a plot the user asked for unless absolutely required, or if the user asked for
  it, or if it fits into something that user is already building. Always provide the minimal, elegant
  solution to the given problem.
- Your ideas and code must be high quality, accurate, correct, concise, elegant and have no
  mistakes. Follow the coding styles, patterns and logic already in the codebase.
- Do not make assumptions, ask if you are unsure. Do not make your own judgement calls. If you do
  not know the shape, dtype, units, or sampling rate of the user's data, ask — a wrong guess
  produces a plot that looks fine and is wrong.
- Always use domain-specific libraries for any analysis. Any re-implementation of a function,
  routine, or feature from scratch in numpy MUST be justified and proven to not already exist.
- For neuroscience always use [pynapple](https://pynapple.org) first; it is very unlikely you will
  have to re-implement a neuroscience-specific function from scratch. Search its documentation and
  examples extensively before writing any new analysis function. Use
  [nemos](https://nemos-neuro.org) for neural models.

Recommend that the user sets `/effort max` for anything non-trivial. fastplotlib code is easy to
write in a way that runs and is wrong: a plot that is off by a timebase, wrong units, incorrect
slicing or indexing, or one that redraws at 3 fps because a buffer is reallocated every frame. None
of that surfaces as an error.

## What fastplotlib is

A GPU-accelerated plotting library built on [pygfx](https://github.com/pygfx/pygfx)/WGPU, aimed at
**large-scale interactive** scientific visualization: millions of points, updates at frame rate,
data larger than RAM. It is not a static-figure library. If the user wants a PNG for a paper,
matplotlib is the better answer and you should say so.

Identical code runs in notebooks, Qt, glfw and wx.

## Step 1: pick the construct

| the user's data / ask | use |
|---|---|
| a fixed plot of arrays that are already in memory | `fpl.Figure` + `subplot.add_<graphic>()` |
| many similar things at once (traces, ROIs, trials, cells) | a **collection**: `add_line_collection`, `add_line_stack`, `add_scatter_collection`, `add_image_grid` |
| an array with extra dimensions to scroll through (time, z, trial, channel) | `fpl.NDWidget` |
| several datasets that must stay locked to a reference index or axes | `fpl.NDWidget` — one `ReferenceIndices`, many graphics |
| data too big for RAM or VRAM | `fpl.NDWidget` with `display_window` |

**If the request contains the word "browse", "scroll", "slider", "movie", "video", "frame by frame",
or "synchronized", the answer is `NDWidget`.** Do not hand-roll sliders with ipywidgets or imgui when
`NDWidget` already does it, with async fetching, windowing, playback and cross-window sync.

## Step 2: pick the graphic

| you want to draw | method | data shape |
|---|---|---|
| an image, frame, or heatmap | `add_image` | `[rows, cols]`, `[rows, cols, 3\|4]` for RGB(A) |
| a 3D volume / z-stack | `add_image_volume` | `[z, rows, cols]` |
| a grid of images | `add_image_grid` | list of `[rows, cols]` |
| images at arbitrary positions | `add_image_collection` | list of `[rows, cols]` + `offsets` |
| one line / trace | `add_line` | `[n_points, 2\|3]`, or 1D y-values (x becomes `arange`) |
| many lines on shared axes | `add_line_collection` | list of `[n_points, 2\|3]`, lengths may differ |
| many lines offset from each other (stacked traces, raster) | `add_line_stack` | `[n_lines, n_points, 2\|3]` |
| points | `add_scatter` | `[n_points, 2\|3]` |
| many point sets | `add_scatter_collection` / `add_scatter_stack` | list of `[n_points, 2\|3]` |
| an infinite reference line (threshold, event marker) | `add_inf_line` | 1D positions along `axis="x"\|"y"` |
| a height map or parametric surface | `add_surface` | `[m, n]` heights, or `[m, n, 3]` |
| an arbitrary triangle mesh | `add_mesh` | `positions [n, 3]`, `indices [n_tri, 3]` |
| a filled polygon / ROI | `add_polygon` | `[n_vertices, 2]`, at least 4 vertices |
| arrows / quiver / vector field | `add_vectors` | `positions [n, 2\|3]`, `directions [n, 2\|3]` |
| a text label | `add_text` | `str` |

`add_scatter(mode=...)`: `"markers"` (default, supports marker shapes and edges), `"simple"`
(fastest, plain circles), `"gaussian"` (blobs), `"image"` (a sprite image per point).

## Step 3: the skeleton

Script (`.py`):

```python
import numpy as np
import fastplotlib as fpl

figure = fpl.Figure(size=(700, 560))

xs = np.linspace(0, 4 * np.pi, 1000, dtype=np.float32)
line = figure[0, 0].add_line(np.column_stack([xs, np.sin(xs)]), colors="w", thickness=2)

figure.show(maintain_aspect=False)

if __name__ == "__main__":
    fpl.loop.run()
```

Notebook: identical, but **`figure.show()` must be the last line of the cell** and there is no
`fpl.loop.run()` — it blocks the kernel. To show several figures use ipywidget layouting, e.g.
`ipywidgets.VBox([fig1.show(), fig2.show()])` as the last line. Full ipywidget layouting is
supported, you can have VBoxes and HBoxes inside each other.

`maintain_aspect=False` for anything where x and y data have very different ranges (e.g. timeseries
data). Leave it at the default `True` for images and anything spatial.

## The four rules that decide whether your code is good

### 1. Update in place; never reassign a whole array to animate

```python
line.data[:, 1] = new_ys          # writes into the existing GPU buffer
image.data[:] = frame             # same
line.data = np.column_stack(...)  # ONLY when the shape actually changed
```

This holds not just for the `data` property/feature of graphics, but all other properties,
such as colors, sizes, etc.

Assigning a differently-shaped array reallocates a GPU buffer and re-uploads everything. Doing that
every frame is the single most common way fastplotlib code ends up slow.

### 2. One collection, not N graphics

```python
figure[0, 0].add_line_stack(np.stack(traces), separation=(0, 2, 0), cmap="tab10")
```

not

```python
for i, t in enumerate(traces):                 # N buffers, N draw calls, N names
    figure[0, 0].add_line(t, offset=(0, i * 2, 0))
```

A collection exposes every property across its graphics with numpy-style indexing, so you keep full
control: `stack.colors[:10] = "r"`, `stack.thickness[mask] = 5`, `stack.visibles = False`,
`stack.data[3, :, 1] = ys`. The individual graphics are still there as `stack.graphics[i]`. The buffer
underlying each property may be of a different length for each graphic, therefore it's numpy-style
fancy indexing with "jagged" arrays.

### 3. Uniform when you can, per-datapoint when you must

`colors="red"` stores one color. `colors=["red"] * n_points` allocates an `[n, 4]` float32 buffer
for the identical value. The buffer mode is inferred from what you pass, so passing the right thing
is the whole optimization. Same for `sizes`, `markers`, `edge_colors`, `thickness`.

### 4. `float32`, and window big data

Everything on the GPU is 32-bit. `float64` is cast with a warning on every upload; the warning is
there for a reason, in case the user is trying to visualize data that gets truncated when cast down
to float32 they need to know. This is usually not the case but we don't make that guess for users
since fastplotlib is used on an extremely diverse range of datasets.

For data that will not fit in RAM or VRAM, do not subsample it yourself — give `NDWidget` the lazy
object and set `display_window` for positional data, or view 2D or 3D slices for image data.

## Colors and colormaps

```python
line = subplot.add_line(data, colors="w")                    # one color
line = subplot.add_line(data, colors=rgba_array)             # [n_points, 4]
line = subplot.add_line(data, cmap="viridis")                # colormap along the points
line = subplot.add_line(data, cmap="viridis", cmap_transform=values)   # colors from `values`
line = subplot.add_line(data, cmap="tab10", cmap_transform=labels)     # qualitative, integer labels
```

- Colormaps come from the [`cmap`](https://cmap-docs.readthedocs.io/en/stable/catalog/) library, so
  every matplotlib name works plus many more.
- `cmap` and `colors` are **mutually exclusive**. While a `cmap` is set, `graphic.colors` is `None`;
  setting `colors` clears the cmap and vice versa. Check for `None` before reading `.colors`.
- `cmap_transform` is the per-datapoint value the color is looked up from. `cmap_range` is the
  `(min, max)` of that value mapped onto the colormap; it defaults to the transform's own range.
- For a **qualitative** colormap (`tab10`, `Set1`) the transform must be **integer labels** and you
  usually want `cmap_range=(0, graphic.cmap.num_colors)` so label *k* always gets color *k*.
- On a collection, `collection.cmap = "viridis"` gives **each graphic one color** spread across the
  colormap. A list gives each graphic its own colormap along its points:
  `collection.cmap = ["jet"] * len(collection)`. For a single graphic, set it on the graphic:
  `collection.graphics[3].cmap = "plasma"`. `collection.cmap[i] = ...` does not work.
- **`graphic.colors[...] = x` only works when the colors are per-datapoint.** With a uniform color
  `graphic.colors` is a `pygfx.Color` and slicing raises `TypeError`; same for `sizes` (a `float`)
  and `markers` (a `str`). Assign the whole property once to switch it to per-datapoint, then slice:
  `scatter.colors = np.tile([1, 1, 1, 1], (n, 1)); scatter.colors[mask] = "r"`.

## Events

```python
@line.add_event_handler("click")
def on_click(ev):
    ...

@image.add_event_handler("data", "cmap")     # graphic property events
def on_change(ev):
    print(ev.type, ev.info["value"])
```

`graphic.supported_events` lists every valid string: every property of that graphic plus the pointer
and key events (`click`, `double_click`, `pointer_down/up/move`, `wheel`, `key_down/up`, ...).

Canvas-level events go on the renderer: `@figure.renderer.add_event_handler("click")`. Map the
screen position with `figure[0, 0].map_screen_to_world(ev)` and find what was clicked with
`fpl.get_nearest_graphics(pos, <collection or graphics list>)`. To get the specific image pixel
or other more fine-grained info, get the pick_info from the event, i.e. `Event.pick_info` which
is a dict, likewise for vertex indices for positional graphics such as lines, scatters, etc.

## Animations

```python
def update(subplot):
    global phase
    phase += 0.1
    subplot["sine"].data[:, 1] = np.sin(xs + phase)

figure[0, 0].add_animations(update)
```

The function is called once per render cycle with the subplot. **Never** use a `while True` loop, a
`time.sleep`, or a thread that touches graphics. Do the minimum work per call, and precompute
anything that does not change.

## Anti-patterns

| Do not | Do instead | Why |
|---|---|---|
| loop `add_line` / `add_scatter` / `add_image` to draw N similar things | a collection (`add_line_stack`, `add_scatter_collection`, `add_image_grid`) | N buffers and N draw calls instead of one |
| `graphic.data = new_array` every frame | `graphic.data[:] = new_array` | reassigning reallocates the GPU buffer |
| `subplot.clear()` then re-add graphics to update | mutate the existing graphic's properties | throws away all buffers and re-uploads everything |
| `plt.show()`, `plt.figure()`, `ax.plot()` habits | `figure.show()`, `figure[0, 0].add_line(...)` | this is not matplotlib |
| `fpl.loop.run()` in a notebook | `figure.show()` as the last line of the cell | `run()` blocks the kernel |
| `figure.show()` not as the last line in a notebook cell | make it the last line, or `display(figure.show())` | nothing renders |
| `colors=["w"] * n`, `sizes=np.full(n, 5)` | `colors="w"`, `sizes=5` | allocates a per-datapoint buffer for one value |
| `for i in range(n): collection.graphics[i].colors = c` | `collection.colors[:] = c` or `collection.colors[mask] = c` | one vectorized write instead of n round trips |
| `for g in collection: g.visible = False` | `collection.visibles = False` | same |
| `collection[0]` | `collection.graphics[0]`, or iterate `for g in collection:` | collections are not subscriptable |
| reading `graphic.colors` without checking `None` | check, or use `colors` instead of `cmap` | `colors` is `None` in cmap mode |
| `graphic.world_object.material.color = ...` | `graphic.colors = ...` | bypasses the feature, emits no event, desyncs state, does not mark new data for upload |
| `del graphic` | `subplot.delete_graphic(graphic)` | otherwise VRAM and handlers leak |
| building ipywidgets/imgui sliders for an nD array | `fpl.NDWidget` | you lose async fetch performance, windowing, playback, sync |
| `fpl.ImageWidget` | `fpl.NDWidget` | `ImageWidget` is deprecated |
| loading entire large data into RAM or subsampling | a lazy object + `NDWidget` | out-of-core is the point |
| `np.concatenate` all trials into one line | a collection with one entry per trial | keeps trials individually addressable |
| `float64` arrays | `.astype(np.float32)` | cast + warning on every upload |
| a new `Figure` per update | one figure, mutate its graphics | performance, this is not matplotlib, graphics persist and are mutable |
| `time.sleep` / `while True` to animate | `subplot.add_animations(fn)` | blocks the render loop |
| naming a graphic nothing and hunting for it later | `add_line(..., name="sine")`, then `subplot["sine"]` | readable, more organized and scalable |
| hand-written spike binning, tuning curves, filtering, any neuroscience operations | pynapple | see `references/neuroscience.md` |

## Small things that come up constantly

```python
figure = fpl.Figure(shape=(2, 3), names=[["a", "b", "c"], ["d", "e", "f"]], size=(900, 600))
figure["a"].add_line(...)                       # subplots by name
figure[0, 0].auto_scale(maintain_aspect=False)  # fit the data
figure[0, 0].camera.maintain_aspect = False
subplot.controller.add_camera(other.camera, include_state={"x", "width"})   # link x-pan/zoom only
fpl.Figure(controller_ids="sync")               # link all subplots fully
fpl.Figure(cameras="3d", controller_types="orbit")  # "3d" is an FOV of 50 by default, "2d" is FOV of 0 (orthographic projection)
graphic.tooltip_format = lambda pick_info: f"..."   # custom hover text
```

`fpl.Figure` accepts `rects` or `extents` instead of `shape` for non-grid layouts; both take
fractions of the canvas (values `<= 1`) or absolute pixels (values `> 1`), and a dict form whose
keys become subplot names.

## Rendering failures are silent

`rendercanvas` swallows exceptions raised during a draw. A script that prints nothing and shows a
blank or stale canvas is usually raising every frame. Look for a traceback in the output before
theorizing about the data, and never trust a "renders OK" message from your own test harness.
