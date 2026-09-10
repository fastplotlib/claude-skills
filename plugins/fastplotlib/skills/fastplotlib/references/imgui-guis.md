# Adding GUI controls with imgui

imgui draws **on the same canvas as the plot**, so a UI written this way works unchanged in Jupyter,
Qt, glfw and wx. That makes it the right choice for controls the user will actually interact with.
ipywidgets only work in Jupyter; Qt widgets only in Qt.

Needs `imgui-bundle` (`pip install "fastplotlib[imgui]"`). `fpl.Figure` is an `ImguiFigure`
whenever it is installed.

## The decorator form — use this unless you need state

```python
from imgui_bundle import imgui

@figure.add_imgui_window(location="right", size=200, title="controls")
def gui(fig):
    line = fig[0, 0]["sine"]

    changed, thickness = imgui.slider_float("thickness", v=line.thickness, v_min=1.0, v_max=50.0)
    if changed:
        line.thickness = thickness

    if imgui.button("randomize"):
        line.data[:, 1] = np.random.rand(100)
```

The function is redrawn **every frame** and each element returns its current value, so there are no
callbacks. Read the current value off the graphic, draw the element with it, and write back only
when `changed` is true.

The decorated function may take **one positional argument** (the `Figure`, `Subplot`, or `Graphic`
it was attached to) or none. Do not add an unused parameter to satisfy the API.

Attach the same way to a subplot, which confines the window to that subplot:

```python
@figure[0, 0].add_imgui_window(location="top", size=36, title=None)
def controls(subplot):
    ...
```

## Placement

`location` is `"left"`, `"right"`, `"top"`, `"bottom"`, `"toolbar"`, or `"floating"`.

- The four edges and `"toolbar"` **reserve canvas space**: `size` is the thickness in pixels and the
  plot area shrinks. An edge window without a `size` raises.
- `"floating"` is auto-sized and draggable and reserves nothing (no `size`). Figure-level only.
- A subplot accepts only the edges and `"toolbar"`.
- One window per location per host, and adding a second at the same location **silently replaces**
  the first. Call `remove_imgui_window(location)` if you mean to swap.

## The class form — when the UI has its own state

```python
from fastplotlib.ui import ImguiWindow

class Controls(ImguiWindow):
    def __init__(self, line):
        super().__init__()
        self._line = line
        self._amplitude = 1.0

    def update(self):                       # draw elements only
        changed, amp = imgui.slider_float("amplitude", v=self._amplitude, v_min=0.1, v_max=10)
        if changed:
            self._amplitude = amp
            self._line.data[:, 1] = amp * np.sin(xs)

figure.add_imgui_window(Controls(line), location="right", size=220, title="controls")
```

Implement `update()`, never `draw()` — the base class handles the window frame, positioning and
imgui IDs. Keep per-UI state on the instance, not in globals.

`fastplotlib.ui.ChangeFlag` accumulates "did anything change" across many elements in a frame; use
it instead of chaining `or`.

## Right-click popups

```python
@figure[0, 0].set_imgui_right_click()
def menu(subplot):
    if imgui.menu_item("reset view", "", False)[0]:
        subplot.auto_scale()

@graphic.set_imgui_right_click()            # per-graphic menu
def graphic_menu(graphic):
    ...

# same menu for multiple graphics, e.g. a custom menu for all images
graphic1.set_imgui_right_click(common_graphic_menu)
graphic2.set_imgui_right_click(common_graphic_menu)

figure.append_imgui_right_click(extra_gui)  # add to the standard menu instead of replacing it
```

Precedence is graphic → subplot → figure. A `StandardRightClickMenu` is already on every figure, so
prefer `append_imgui_right_click` when you just want to add an item.

## Contrast controls for images

Do not build vmin/vmax sliders by hand. `fastplotlib.ui.ImguiColorbar` draws a colormap bar with
draggable vmin/vmax handles, an optional precomputed histogram, and a gamma slider, and it stays in
sync with the images it manages:

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
| override `draw()` unless you need full-control or highly customized control of imgui | implement `update()` |
| write vmin/vmax/gamma sliders by hand | `ImguiColorbar` |
| do heavy computation inside `update()` | it runs every frame — precompute, cache, or gate on `changed` |
| replace the whole right-click menu to add one item | `append_imgui_right_click` |
| pass `size` to a `"floating"` window, or omit it for an edge window | edges need `size`; floating takes none |
| rebuild a graphic when a slider moves | write into the existing graphic's properties |
