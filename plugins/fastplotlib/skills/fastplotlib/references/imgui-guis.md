# GUI controls with imgui

imgui draws **on the same canvas as the plot**, so a UI written this way works unchanged in Jupyter,
Qt, glfw and wx. That makes it the right choice for controls the user will actually interact with.
ipywidgets only work in Jupyter; Qt widgets only in Qt.

Needs `imgui-bundle` (`pip install "fastplotlib[imgui]"`). `fpl.Figure` is an `ImguiFigure`
whenever it is installed.

Two companion files carry the detail, both from the fastplotlib documentation:

- **`references/imgui-guide.md`** — the full guide: floating, fixed and edge windows, subplot
  toolbars, appending and removing windows, subclassing `ImguiWindow` and `ImguiPopup`, right-click
  menus, and the built-in UIs. Read it before writing a UI.
- **`references/imgui-elements.md`** — every imgui element with its real signature, arguments and
  flags, plus example code. Read it rather than recalling what an element's arguments are.

## The one thing to understand first

The function is redrawn **every frame** and each element returns its current value, so there are no
callbacks. Read the current value off the graphic, draw the element with it, and write back only
when `changed` is true:

```python
from imgui_bundle import imgui

@figure.add_imgui_window(location="right", size=200, title="controls")
def gui(fig):
    line = fig[0, 0]["sine"]

    changed, thickness = imgui.slider_float("thickness", v=line.thickness, v_min=1.0, v_max=50.0)
    if changed:
        line.thickness = thickness
```

The decorated function may take **one positional argument** (the `Figure`, `Subplot`, or `Graphic`
it was attached to) or none. Do not add an unused parameter to satisfy the API.

Because it runs on every render, anything expensive inside it costs frame rate directly.

## Placement

The valid `location` values depend on what you attach to:

| attached to | `location` |
|---|---|
| a `Figure` | `"left"`, `"right"`, `"top"`, `"bottom"`, `"floating"` |
| a `Subplot` | `"left"`, `"right"`, `"top"`, `"bottom"`, `"toolbar"` |

`"toolbar"` is **subplot only** and `"floating"` is **figure only**; the wrong one raises
`ValueError`. A `Figure` window can also be fixed to a `rect` or an `extent` instead of a location.

- The four edges and `"toolbar"` **reserve canvas space**: `size` is the thickness in pixels and the
  plot area shrinks. An edge window without a `size` raises.
- `"floating"` is auto-sized and draggable and reserves nothing.
- One window per location per host, and `add_imgui_window` **replaces** the window already at that
  location. `append_imgui_window` adds to it instead, and `remove_imgui_window(location)` removes
  and returns it.

## Contrast controls for images

Do not build vmin/vmax sliders by hand. `fastplotlib.ui.ImguiColorbar` draws a colormap bar with
draggable vmin/vmax handles, a colormap picker, a gamma slider, and an optional precomputed
histogram, and it stays in sync with the images it manages:

```python
from fastplotlib.ui import ImguiColorbar

cbar = ImguiColorbar(images=[image], histogram=(counts, edges), data_range=(0, 255))
figure[0, 0].add_imgui_window(cbar, location="right", size=120, title=None)
```

`NDWidget` adds one for you when `compute_histogram=True`.

`HistogramLUTTool` and `EdgeWindow` no longer exist; do not use those names.

## Anti-patterns

| Do not | Do instead |
|---|---|
| use ipywidgets for controls the user might run outside Jupyter | imgui — it works on every backend |
| store UI state in module globals | an `ImguiWindow` subclass with instance attributes |
| override `draw()` for an ordinary UI | implement `update()`; `draw()` is for what must be set when the window is created, such as a menu bar flag, or for drawing something that outlives a popup |
| write vmin/vmax/gamma sliders by hand | `ImguiColorbar` |
| do heavy computation inside `update()` | it runs every frame — precompute, cache, or gate on `changed` |
| replace the whole right-click menu to add one item | `append_imgui_right_click` |
| omit `size` for an edge or `"toolbar"` window | they reserve canvas space, so `size` is required and it raises without one |
| recall an imgui element's arguments from memory | `references/imgui-elements.md` has the real signatures |
| rebuild a graphic when a slider moves | write into the existing graphic's properties |
