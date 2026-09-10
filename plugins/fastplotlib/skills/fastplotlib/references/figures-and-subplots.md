# Figures, subplots, cameras — laying out and navigating a plot

## Creating a figure

```python
figure = fpl.Figure(size=(700, 560))                      # one subplot
figure = fpl.Figure(shape=(2, 3), size=(900, 600))        # a 2x3 grid
figure = fpl.Figure(shape=(2, 2), names=[["a", "b"], ["c", "d"]])
figure = fpl.Figure(extents=extents, names=names)         # arbitrary layout
figure = fpl.Figure(rects=rects, names=names)
```

`size` is the initial canvas size in pixels; the window is resizable afterwards.

**Name your subplots** whenever there are more than two. `figure["traces"]` survives a layout
change; `figure[1, 2]` does not, and is unreadable.

### Non-grid layouts

- `extents` — `(xmin, xmax, ymin, ymax)` per subplot
- `rects` — `(x, y, width, height)` per subplot

Both accept **fractions of the canvas** (all values `<= 1`) or **absolute pixels** (values `> 1`),
and you cannot mix the two in one entry. Fractions are almost always what you want, because the
layout then survives a window resize:

```python
extents = {                              # a dict: keys become subplot names
    "video":   (0,    0.4, 0,    1),
    "traces":  (0.4,  1,   0,    0.5),
    "raster":  (0.4,  1,   0.5,  1),
}
figure = fpl.Figure(extents=extents, size=(1300, 800))
```

With `rects`/`extents` the user can also drag and resize the subplots at runtime; with `shape` they
stay on the grid.

## Showing it

```python
figure.show(maintain_aspect=False, autoscale=True)
```

- **Notebook**: `figure.show()` must be the **last line of the cell**, or wrap it in
  `display(...)`. Never call `fpl.loop.run()`. For several figures use
  `ipywidgets.VBox([fig1.show(), fig2.show()])` as the last line.
- **Script**: `figure.show()` then `fpl.loop.run()` inside `if __name__ == "__main__":`.
- `maintain_aspect=False` for any plot where x and y are different quantities. Leave it `True` for
  images and spatial data or they will be distorted.
- `figure.show(sidecar=True)` opens it in a jupyter sidecar.

## Cameras and controllers

One camera and one controller per subplot. A "2d" camera is an orthographic projection; a "3d"
camera has perspective.

```python
figure = fpl.Figure(cameras="3d", controller_types="orbit")   # or "panzoom", "fly", "trackball"
figure[0, 0].camera.maintain_aspect = False
figure[0, 0].auto_scale(maintain_aspect=False)
figure[0, 0].center_graphic(graphic)
figure[0, 0].x_range = (0, 100)          # get or set the visible x range
figure[0, 0].y_range = (-1, 1)
```

`cameras` and `controller_types` accept one value for the whole figure or a per-subplot iterable.

### Linking views — the three ways

```python
# 1. everything linked
figure = fpl.Figure(shape=(2, 2), controller_ids="sync")

# 2. groups linked, by subplot name (or by integer ids in a nested list)
figure = fpl.Figure(
    shape=(2, 3), names=names,
    controller_ids=[("traces", "raster"), ("video_left", "video_right")],
)

# 3. link only some axes — the idiom for time series
subplot.controller.add_camera(subplot.camera, include_state={"x", "width"})
```

Form 3 is what you want for stacked time-series subplots: x pans and zooms together while each subplot
keeps its own y scale. Add each subplot's camera to each controller you want it to follow. Note the
call in `add_nd_timeseries` code is on the subplot's *own* camera — that registers the subplot with
the controller for the restricted state.

## Working with subplots

```python
subplot = figure[0, 0]
subplot.add_line(...)                # every add_* method lives here
subplot["graphic_name"]              # a graphic by name
subplot.graphics                     # all graphics
subplot.selectors                    # all selectors
"name" in subplot                    # membership by name or by graphic
subplot.delete_graphic(graphic)
subplot.clear()                      # delete everything in this subplot
subplot.title = "dataset 1"
subplot.axes.visible = False         # hide axes/ticks
subplot.axes.grids.visible = False   # keep axes, drop the grid
subplot.toolbar = False              # hide the imgui toolbar strip
subplot.background_color = "black"
for subplot in figure: ...           # iterate
```

`subplot.map_screen_to_world(ev)` and `map_world_to_screen(pos)` convert between a pointer event and
data coordinates. `graphic.map_model_to_world` / `map_world_to_model` additionally apply that
graphic's own offset/scale/rotation — needed whenever a graphic has been moved.

## Animations

```python
def update(subplot):
    subplot["sine"].data[:, 1] = next_ys()

figure[0, 0].add_animations(update)          # called with the subplot
figure.add_animations(other_fn)              # called with the figure
figure[0, 0].remove_animation(update)
```

Called once per render cycle. `add_animations(..., pre_render=True, post_render=False)` chooses when.
Exceptions inside an animation function are swallowed by rendercanvas — if a plot silently stops
updating, look for a traceback in the output.

## Exporting

```python
figure.export("plot.png")                # via imageio
arr = figure.export_numpy(rgb=True)      # for tests or compositing
```

## Anti-patterns

| Do not | Do instead |
|---|---|
| `figure[1, 2]` in a script with many subplots | name the subplots and use `figure["subplot-name"]` |
| absolute-pixel `extents` for a window the user will resize | fractional extents |
| `maintain_aspect=True` (the default) for a time series | `figure.show(maintain_aspect=False)` |
| build a new `Figure` to update a plot | mutate the graphics |
| `subplot.clear()` then re-add to refresh | write into the existing graphic's `data` |
| a `while` loop or a thread to animate | `subplot.add_animations(fn)` |
| `fpl.loop.run()` in a notebook | `figure.show()` as the last line |
| pairwise event handlers to sync panning | `controller_ids`, or `controller.add_camera(..., include_state=...)` |
