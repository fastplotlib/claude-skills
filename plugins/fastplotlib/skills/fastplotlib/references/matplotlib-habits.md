# matplotlib habits — what to write instead

Read this when the request or the user's existing code is matplotlib-shaped: `plt.*` / `ax.*` calls,
or a question phrased as "how do I set xlim / label the axes / add a colorbar / make a 3D plot / set
rcParams".

The model is different, and that is where the wrong code comes from. A `Figure` is a live object on a
canvas that is rendered continuously, and every `Graphic` in it stays mutable for as long as it
exists. You change a visualization by setting properties on the graphic that is already there, and
the changed values are updated in the visualization on the next rendering cycle. There is nothing to
redraw, and no pyplot state machine — no current figure, no current subplot, no `gca()`.

This is fundamentally different from a matplotlib animation, where each frame comes from a callback
that clears the axes and plots again, or that returns the artists to be redrawn. Write with numpy
intuition instead: every aspect of a `Graphic` is an array. `data`, `colors`, `sizes`, `thickness`
and the rest are indexed, sliced, and assigned into the way a numpy array is, and each assignment
writes into the GPU buffer behind that property.

**matplotlib habits also cost the user performance.** Modify the properties of graphics that are
already in the scene, do not create new ones. Creating a graphic allocates GPU buffers, and clearing
a subplot to plot into it again throws those buffers away and uploads all of the data from scratch.

## Translation table

| matplotlib | fastplotlib |
|---|---|
| `fig, ax = plt.subplots()` | `figure = fpl.Figure()`, the subplot is `figure[0, 0]` |
| `plt.subplots(2, 3)` | `fpl.Figure(shape=(2, 3))` |
| `fig.add_axes([x, y, w, h])` | `fpl.Figure(rects=...)` or `extents=...` |
| `plt.gca()` | `figure[0, 0]`, or `names=` and `figure["temperature"]` |
| `fig.set_size_inches(...)` | `fpl.Figure(size=(900, 600))`, in pixels |
| `plt.tight_layout()` | nothing, the layout is computed |
| `plt.show()` | `figure.show()`, plus `fpl.loop.run()` outside notebooks — see below |
| `plt.ion()`, `%matplotlib widget` | nothing, figures are always interactive |
| `ax.plot(y)` | `subplot.add_line(y)` |
| `ax.plot(x, y)` | `subplot.add_line(np.column_stack([x, y]))` |
| `ax.plot(..., "--", lw=3, color="r")` | `add_line(..., dash_pattern="--", thickness=3, colors="r")` |
| `ax.scatter(x, y, s=8, c="cyan", marker="^")` | `add_scatter(xy, sizes=8, colors="cyan", markers="^")` |
| `ax.imshow(img, vmin=, vmax=, cmap=)` | `subplot.add_image(img, vmin=, vmax=, cmap=)` |
| `ax.pcolormesh(values)` | `subplot.add_image(values)` |
| `ax.axvline(x)` / `ax.axhline(y)` | `subplot.add_inf_line([x], axis="x")` / `axis="y"` |
| `ax.bar`, `ax.stem` | **out of scope in fastplotlib** — say so, do not improvise one out of line segments |
| `ax.fill_between`, `ax.axvspan` | `subplot.add_polygon(vertices)` — filled mesh, takes `alpha` |
| `ax.quiver` | `subplot.add_vectors(positions, directions)` |
| `ax.plot_surface` | `subplot.add_surface(heights)` |
| `Poly3DCollection` | `subplot.add_mesh(positions, indices)` |
| `ax.text(x, y, s)` | `subplot.add_text(s, offset=(x, y, 0))` — screen space by default, `screen_space=False` scales with the world |
| `ax.set_title("t")` | `subplot.title = "t"` — reading it back gives the `TextGraphic` |
| `ax.set_xlabel("t")` | `subplot.axes.x.label.set_text("t")` |
| `ax.set_xticks` + labels | `subplot.axes.x.ticks = {0: "a", 30: "b"}`, a list of values, or `None` for auto |
| tick formatting | `subplot.axes.x.tick_format`, `.min_tick_distance` (px), `.tick_size` |
| `ax.set_xlim(a, b)`, `ax.get_xlim()` | `subplot.x_range = (a, b)`, `subplot.x_range` |
| `ax.autoscale()` | `subplot.auto_scale(maintain_aspect=False, zoom=0.9)` |
| `ax.set_aspect("equal")` | `subplot.camera.maintain_aspect = True` — locks x, y **and** z; `False` lets them scale independently |
| `ax.invert_yaxis()` | `subplot.camera.local.scale_y = -1` |
| `ax.grid(False)` | `subplot.axes.grids.visible = False` |
| `ax.axis("off")` | `subplot.axes.visible = False` |
| `ax.set_facecolor("k")` | `subplot.background_color = ["k"]` — a **sequence**, a bare string raises |
| `line.set_ydata(ys)` | `line.data[:, 1] = ys` |
| `im.set_data(frame)` | `image.data[:] = frame` |
| `im.set_clim(a, b)` | `image.vmin, image.vmax = a, b` |
| `ax.cla()` then re-plot | mutate the existing graphic |
| `fig.canvas.draw()` | nothing |
| `FuncAnimation(fig, fn)` | `subplot.add_animations(fn)` |
| `plt.subplots(sharex=True)` | `fpl.Figure(controller_ids="sync")`, or `include_state` (below) |
| `plt.colorbar(im)` | `subplot.add_imgui_window(ImguiColorbar(images=im), location="right", size=80)` |
| `plt.rcParams`, `plt.style.use` | `LineGraphic.config.init.colors = ...`, `fpl.style.light()` |
| `plt.savefig("f.png")` | `figure.export("f.png")` — needs `imageio`, raster only, no serialization yet |
| a `LineCollection` to color a line by a value | `cmap=` + `cmap_transform=` on **one** line |
| `Axes3D`, `projection="3d"` | nothing — every subplot is already 3D |
| ipywidgets sliders over an nD array | `fpl.NDWidget` |
| `SpanSelector`, `RectangleSelector` | `graphic.add_linear_region_selector()`, `add_rectangle_selector()` |

matplotlib's linestyle strings (`"-"`, `"--"`, `"-."`, `":"`) work for `dash_pattern`, and its marker
strings (`"o"`, `"s"`, `"D"`, `"+"`, `"x"`, `"^"`, `"<"`, `">"`, `"v"`, `"*"`) work for `markers`.
`add_image` draws row 0 at the top, like `imshow` — the camera's `scale_y` is set to `-1` at
`figure.show()` for any subplot holding an image.

`maintain_aspect=False` is what you want when the data in each dimension are of a different
magnitude, such as a timeseries, and for most large heatmaps. `True` is usually right for images.

## What actually produces wrong code

### 1. `x_range` / `y_range` are world space, and orthographic-only

```python
subplot.x_range = (0, 100)
xmin, xmax = subplot.y_range
```

The units are **world space**, not data space — if the graphic has an `offset` or `scale` (every
stack and grid does) they are not the same thing. And they are only valid for an orthographic
projection of the xy plane, i.e. `camera.fov == 0`. The setter does not raise for `fov > 0`, it
multiplies the zoom and gives you something you did not ask for. For a perspective camera drive the
camera instead:

```python
state = subplot.camera.get_state()      # position, rotation, scale, reference_up, fov,
subplot.camera.set_state(state)         # width, height, depth, zoom, maintain_aspect, depth_range
subplot.camera.set_state({"x": 10, "fov": 50})   # x/y/z set one position component
```

### 2. A multi-colored line is one line, not a collection

In matplotlib you build a `LineCollection` of segments with a norm and a cmap. Do **not** reach for
`add_line_collection` here — that is for N separate lines. In fastplotlib you set a colormap on the
line, or set per-datapoint colors like any other array:

```python
line = subplot.add_line(xy, cmap="viridis", cmap_transform=speed, cmap_range=(0, 10))
```

`cmap_transform` holds the per-datapoint values the colors are looked up from. `cmap_range` is the
`(min, max)` of that transform mapped onto the colormap, and defaults to the range of the transform.

A **qualitative** colormap takes integer labels as its `cmap_transform`, which is the idiom for
cluster/class colors: you do not build a color per datapoint, you give the class of each datapoint.
Pass `cmap_range` to the constructor, `(0, n_colors_in_the_cmap)`, so label *k* always gets color *k*:

```python
# tab10 has 10 colors
scatter = subplot.add_scatter(xy, cmap="tab10", cmap_transform=cluster_labels, cmap_range=(0, 10))
```

On a collection, `collection.cmap = "tab10"` instead spreads the colormap **across the graphics**, one
color each. A list, `cmap=["jet"] * n_lines`, gives every graphic its own colormap along its datapoints.
A qualitative `cmap_transform` on a collection labels the graphics rather than the datapoints, and
there you pass **no** `cmap_range` — the labels index the colors directly, and a range raises.

Per-datapoint colors are an `[n_points, 4]` RGBA array. Pass one to the constructor to start out
per-datapoint, or assign one later to switch a graphic over. A single color is stored as one value
rather than a buffer, so it cannot be sliced until you do:

```python
scatter = subplot.add_scatter(xy, colors="r")    # one color for every point, not sliceable
scatter.colors = np.random.rand(n_points, 4)     # now one color per point
scatter.colors[mask] = "w"
```

**Prefer `cmap` + `cmap_transform` over an RGBA array whenever you can.** An RGBA array stores four
float values for every datapoint, so it takes up far more GPU RAM than a colormap and a transform do.

### 3. There is no Axes3D and no 2D/3D split

Every subplot is a 3D scene and every graphic takes `[n_points, 3]`. How it looks is entirely the
camera: `fov = 0` is an orthographic projection (what people call "2D"), `fov > 0` is perspective.

```python
figure = fpl.Figure(cameras="3d", controller_types="orbit")   # "3d" is fov 50
subplot.camera.fov = 0                                        # back to orthographic
```

Never tell a user to "switch to 3D plotting" — there is nothing to switch.

### 4. Pan and zoom are the controller, and it is already on

No `plt.ion()`, no `%matplotlib widget`. One controller per subplot, chosen from the camera's fov:

- `fov == 0` → `PanZoomController`: left drag pans, right drag zooms, wheel zooms to the pointer.
- `fov > 0` → `FlyController`: first-person-game controls — `wasd` moves, space/shift up and down,
  `q`/`e` roll, left drag looks, wheel changes speed.

```python
fpl.Figure(controller_types="orbit")    # "panzoom", "orbit", "trackball", "fly"
subplot.controller = "trackball"
subplot.controller.enabled = False      # freeze the view
```

`OrbitController`/`TrackballController`: left drag rotates, right drag pans, wheel zooms.

The camera and the controller are mutable properties on a `Subplot`, like most other properties in
fastplotlib. Assigning one subplot's controller to another shares it, and the two then pan and zoom
together.

Linked views, in increasing specificity:

```python
fpl.Figure(shape=(2, 2), controller_ids="sync")                       # sharex and sharey, all
fpl.Figure(shape=(1, 3), names=names, controller_ids=[("a", "b")])    # groups

figure["b"].controller = figure["a"].controller                       # share one controller

# one axis only — the idiom for stacked timeseries, x linked, y independent
figure[0, 0].controller.add_camera(figure[1, 0].camera, include_state={"x", "width"})
figure[1, 0].controller.add_camera(figure[0, 0].camera, include_state={"x", "width"})
```

Add the **other** subplot's camera to each controller; each subplot's own camera is already on its
own controller.

### 5. How a Figure is shown depends on where it runs

`fpl.loop.run()` is not "the script case", it is the normal case: applications, scripts, and most
other use-cases start the event loop with it. It blocks, so it is never used in a notebook or IPython.

- **notebooks**: `figure.show()` as the last line of the cell, or wrapped in `IPython.display.display()`
- **applications, scripts, most other use-cases**: `figure.show()` then `fpl.loop.run()`
- **an interactive Qt window from a notebook or IPython**: `%gui qt` **before** importing fastplotlib

### 6. Defaults are attributes on the class, not a string-keyed dict

There is no rcParams and no `style.use()`. Do not build one.

```python
fpl.LineGraphic.config.init.colors = "magenta"    # grouped by the method that takes the argument
fpl.Figure.config.init.size = (900, 700)
fpl.Figure.config.show.axes_visible = False
fpl.style.light(); fpl.style.compact()            # presets merge; light/dark/spaced/default/
                                                  # compact/very_compact/flynn
```

Config is read at construction, so it affects objects created after it and nothing that exists.

## Legends

It is a WIP. Do not offer `plt.legend()`'s equivalent.

## Colors that raise

`"C0"` and `"tab:blue"` are not valid colors. Single letters (`"r"`), named colors (`"cyan"`), hex
(`"#ff0000"`) and RGBA sequences are. There is no color cycle either — lines and scatters are white
unless you pass `colors`. To color several differently, put them in a collection and set
`cmap="tab10"` across it.

## Not there yet, or out of scope

Say which of the two it is. Do not present a planned feature as a permanent gap, and do not
improvise an out-of-scope one.

| matplotlib | status, and what to do |
|---|---|
| `ax.set_xscale("log")` | **not implemented yet.** For now plot transformed values and set `axes.x.ticks` labels explicitly |
| `ax.twinx()` | **not implemented yet**, planned as reference-spaces. For now use a second subplot and link the x range with `include_state={"x", "width"}` |
| `ax.bar`, `ax.stem` | **out of scope** |
| vector output (PDF/SVG) | **out of scope.** fastplotlib is for live interactive visualization, not publication figures or videos. `figure.export()` gives you a png |
| `ax.contour` | compute contours elsewhere, draw with `add_line_collection`, or mark them on the image with `ImageHighlightSelector` |
| `fig.suptitle` | `Figure` has no `title`; use subplot titles |

## Anti-patterns

| Do not | Do instead |
|---|---|
| `add_line_collection` for a line colored by a value | one `add_line(..., cmap=, cmap_transform=)` |
| an `[n_points, 4]` RGBA array when the colors come from a value or a class | `cmap` + `cmap_transform`, far less GPU RAM |
| a loop of `add_line` / `add_image` for N similar things | a collection |
| `subplot.clear()` then re-add, to refresh | `graphic.data[:] = ...` |
| a new `Figure` per update | mutate the graphics |
| build an rcParams-style dict or `style.use()` shim | `Class.config.<method>.<arg> = value` |
| `subplot.background_color = "black"` | `subplot.background_color = ["black"]` |
| improvise a bar chart or stem plot out of line segments | say it is out of scope |
| call a planned feature (log axes, twin axes) permanently missing | say it is not implemented yet |
| hand-rolled ipywidgets/imgui sliders over an nD array | `fpl.NDWidget` |
| suggest fastplotlib for a static publication figure or a video | say matplotlib is the better tool |
