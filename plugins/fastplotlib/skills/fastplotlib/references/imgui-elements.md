# Elements

The imgui elements as they exist in `imgui_bundle`. Each element is shown with its signature and with example code that runs as it is written. See `references/imgui-guide.md` for adding a UI to a Figure.

An argument typed `ImVec2` or `ImVec4` also takes a tuple or a list.

The examples use `imgui`, `icons_fontawesome_6 as fa` and `numpy as np`.

## Text

Text elements are read-only, they display a value that the user cannot edit.

### text

```python
imgui.text(fmt: str) -> None
```

formatted text

**Parameters**

- `fmt` - the text to draw

```python
n_peaks = 137

imgui.text(f"peaks found: {n_peaks}")
```

### text_colored

```python
imgui.text_colored(col: ImVec4, fmt: str) -> None
```

shortcut for PushStyleColor(ImGuiCol_Text, col); Text(fmt, ...); PopStyleColor();

**Parameters**

- `col` - text color, `(r, g, b, a)` in `0.0` to `1.0`
- `fmt` - the text to draw

```python
vmin, vmax = 180.0, 60.0

if vmin > vmax:
    imgui.text_colored((1.0, 0.3, 0.3, 1.0), f"{fa.ICON_FA_TRIANGLE_EXCLAMATION} vmin > vmax")
```

### text_disabled

```python
imgui.text_disabled(fmt: str) -> None
```

shortcut for PushStyleColor(ImGuiCol_Text, style.Colors[ImGuiCol_TextDisabled]); Text(fmt, ...); PopStyleColor();

**Parameters**

- `fmt` - the text to draw

```python
selected = None

imgui.text("selection:")
imgui.same_line()

if selected is None:
    imgui.text_disabled("none")
else:
    imgui.text(selected)
```

### text_wrapped

```python
imgui.text_wrapped(fmt: str) -> None
```

shortcut for PushTextWrapPos(0.0); Text(fmt, ...); PopTextWrapPos();. Note that this won't work on an auto-resizing window if there's no other widgets to extend the window width, yoy may need to set a size using SetNextWindowSize().

**Parameters**

- `fmt` - the text to draw, wrapped at the right edge of the window

```python
imgui.text_wrapped("the filter runs on the full frame, it can take a few seconds for large images")
```

### label_text

```python
imgui.label_text(label: str, fmt: str) -> None
```

display text+label aligned the same way as value+label widgets

**Parameters**

- `label` - drawn to the right of the value, aligned the same way as the label of a slider or an input
- `fmt` - the value to draw

```python
data = np.random.randint(0, 4096, (512, 512), dtype=np.uint16)

imgui.label_text("shape", str(data.shape))
imgui.label_text("dtype", str(data.dtype))
imgui.label_text("range", f"{data.min()} - {data.max()}")
```

### bullet_text

```python
imgui.bullet_text(fmt: str) -> None
```

shortcut for Bullet()+Text()

**Parameters**

- `fmt` - the text to draw after the bullet

```python
imgui.text("controller:")
imgui.bullet_text("left click drag to pan")
imgui.bullet_text("right click drag to zoom")
imgui.bullet_text("scroll to zoom about the cursor")
```

### separator_text

```python
imgui.separator_text(label: str) -> None
```

currently: formatted text with a horizontal line

**Parameters**

- `label` - the text to draw in the separator

```python
thickness, sigma = 4.0, 1.0

imgui.separator_text("line")
changed, thickness = imgui.slider_float("thickness", v=thickness, v_min=1.0, v_max=20.0)

imgui.separator_text("image")
changed, sigma = imgui.slider_float("gaussian sigma", v=sigma, v_min=0.1, v_max=10.0)
```

## Widgets

### button

```python
imgui.button(label: str, size: ImVec2 | None = None) -> bool
```

button

> **Note:** If size is None, then its default value will be: ImVec2(0, 0)

**Parameters**

- `label` - drawn on the button, `"##hidden"` suppresses it
- `size` - `(width, height)`, a zero component is sized to the label, a negative one fills the available space

**Returns:** `True` on the frame the button is clicked

```python
if imgui.button("autoscale"):
    print("autoscale clicked")

if imgui.button(fa.ICON_FA_TRASH):
    print("trash clicked")
if imgui.is_item_hovered():
    imgui.set_tooltip("remove all graphics")
```

### small_button

```python
imgui.small_button(label: str) -> bool
```

button with (FramePadding.y == 0) to easily embed within text

**Parameters**

- `label` - drawn on the button

**Returns:** `True` on the frame the button is clicked

```python
vmin, vmax = 12.0, 208.0

imgui.text(f"vmin {vmin:.0f}, vmax {vmax:.0f}")
imgui.same_line()

if imgui.small_button("reset"):
    vmin, vmax = 0.0, 255.0
```

### arrow_button

```python
imgui.arrow_button(str_id: str, dir: Dir) -> bool
```

square button with an arrow shape

**Parameters**

- `str_id` - identifies the button, it is not drawn
- `dir` - `imgui.Dir.left`, `right`, `up` or `down`

**Returns:** `True` on the frame the button is clicked

```python
channel, n_channels = 1, 4

if imgui.arrow_button("previous", imgui.Dir.left):
    channel = max(0, channel - 1)

imgui.same_line()
imgui.text(f"channel {channel}")

imgui.same_line()
if imgui.arrow_button("next", imgui.Dir.right):
    channel = min(n_channels - 1, channel + 1)
```

### invisible_button

```python
imgui.invisible_button(str_id: str, size: ImVec2, flags: int = 0) -> bool
```

flexible button behavior without the visuals, frequently useful to build custom behaviors using the public api (along with IsItemActive, IsItemHovered, etc.)

`flags` takes `imgui.ButtonFlags_`

**Parameters**

- `str_id` - identifies the button, nothing is drawn
- `size` - `(width, height)` of the area that responds to the pointer

**Returns:** `True` on the frame the button is clicked

An invisible button gives the pointer behavior of a button to an area that you draw yourself. The pointer is over the
button in the image below, so the bar is drawn in its highlighted color.

```python
draw_list = imgui.get_window_draw_list()
position = imgui.get_cursor_screen_pos()

imgui.invisible_button("threshold-bar", (120, 24))

color = (1.0, 0.8, 0.2, 1.0) if imgui.is_item_hovered() else (0.4, 0.4, 0.4, 1.0)
draw_list.add_rect_filled(
    position, (position.x + 120, position.y + 24), imgui.color_convert_float4_to_u32(color)
)
```

### checkbox

```python
imgui.checkbox(label: str, v: bool) -> tuple[bool, bool]
```

**Parameters**

- `label` - drawn to the right of the box
- `v` - the current state

**Returns:** `(changed, v)`

```python
axes_visible, grid_visible = True, False

changed, axes_visible = imgui.checkbox("axes", axes_visible)
changed, grid_visible = imgui.checkbox("grid", grid_visible)
```

### checkbox_flags

**Overloads**

```python
imgui.checkbox_flags(label: str, flags: int, flags_value: int) -> tuple[bool, int]
```

```python
imgui.checkbox_flags(label: str, flags: int, flags_value: int) -> tuple[bool, int]
```

**Parameters**

- `label` - drawn to the right of the box
- `flags` - the `int` that holds the bits
- `flags_value` - the bit that this checkbox sets and clears

**Returns:** `(changed, flags)`

The box is checked when the bit is set, and is drawn filled when `flags_value` holds several bits and only some of
them are set.

```python
slider_flags = int(imgui.SliderFlags_.logarithmic)

changed, slider_flags = imgui.checkbox_flags(
    "logarithmic", slider_flags, int(imgui.SliderFlags_.logarithmic)
)
changed, slider_flags = imgui.checkbox_flags(
    "no input", slider_flags, int(imgui.SliderFlags_.no_input)
)
```

### radio_button

**Overloads**

```python
imgui.radio_button(label: str, active: bool) -> bool
```

use with e.g. if (RadioButton("one", my_value==1)) { my_value = 1; }

```python
imgui.radio_button(label: str, v: int, v_button: int) -> tuple[bool, int]
```

shortcut to handle the above pattern when value is an integer

**Parameters**

- `label` - drawn to the right of the button
- `active` - whether this button is the selected one
- `v`, `v_button` - the variable that holds the selection, and the value of this button

**Returns:** `True` on the frame the button is clicked, or `(changed, v)` for the second form

Use radio buttons for a small number of options that are all worth showing, a combo box is better for a long list.

```python
mode = 1

for i, label in enumerate(["line", "scatter", "heatmap"]):
    if imgui.radio_button(label, mode == i):
        mode = i
```

### progress_bar

```python
imgui.progress_bar(fraction: float, size_arg: ImVec2 | None = None, overlay: str | None = None) -> None
```

> **Note:** If size_arg is None, then its default value will be: ImVec2(-sys.float_info.min, 0)

**Parameters**

- `fraction` - `0.0` to `1.0`
- `size_arg` - `(width, height)`, the default fills the available width
- `overlay` - text drawn on the bar, the percentage is drawn if it is not given

```python
n_done, n_frames = 317, 500

imgui.progress_bar(n_done / n_frames, overlay=f"{n_done} / {n_frames} frames")
```

### bullet

```python
imgui.bullet() -> None
```

draw a small circle + keep the cursor on the same line. advance cursor x position by GetTreeNodeToLabelSpacing(), same distance that TreeNode() uses

**Parameters**

none

```python
shape = (500, 512, 512)

imgui.bullet()
imgui.text(f"{shape[0]} frames")

imgui.bullet()
imgui.text(f"{shape[1]} x {shape[2]} pixels")
```

## Sliders

A slider is dragged between a lower and an upper bound. A drag has no bound by default and changes its value by how
far the pointer moves, which suits a value with no natural range. Ctrl+click either of them to type a value instead.

`format` is a printf format, it is applied to the value drawn on the element, e.g. `"%.1f px"`.

### slider_float

```python
imgui.slider_float(label: str, v: float, v_min: float, v_max: float, format: str = '%.3f', flags: int = 0) -> tuple[bool, float]
```

adjust format to decorate the value with a prefix or a suffix for in-slider labels or unit display.

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the slider, `"##hidden"` suppresses it
- `v` - the current value
- `v_min`, `v_max` - the bounds, the value is clamped to them
- `format` - printf format of the value drawn on the slider

**Returns:** `(changed, v)`

```python
thickness = 4.0

changed, thickness = imgui.slider_float("thickness", v=thickness, v_min=1.0, v_max=20.0)
```

### slider_float2

```python
imgui.slider_float2(label: str, v: Sequence[float], v_min: float, v_max: float, format: str = '%.3f', flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.SliderFlags_`

Two values on one row, sharing one pair of bounds. Pass a list and use the list that comes back.

**Parameters**

- `label` - drawn to the right of the sliders
- `v` - the current values
- `v_min`, `v_max` - the bounds, applied to both components
- `format` - printf format of the values drawn on the sliders

**Returns:** `(changed, v)`

```python
vmin_vmax = [12.0, 208.0]

changed, vmin_vmax = imgui.slider_float2("vmin / vmax", vmin_vmax, 0.0, 255.0, format="%.0f")
```

### slider_float3

```python
imgui.slider_float3(label: str, v: Sequence[float], v_min: float, v_max: float, format: str = '%.3f', flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the sliders
- `v` - the current values
- `v_min`, `v_max` - the bounds, applied to every component
- `format` - printf format of the values drawn on the sliders

**Returns:** `(changed, v)`

```python
spacing = [1.0, 1.0, 3.0]

changed, spacing = imgui.slider_float3("voxel spacing", spacing, 0.1, 10.0, format="%.2f")
```

### slider_float4

```python
imgui.slider_float4(label: str, v: Sequence[float], v_min: float, v_max: float, format: str = '%.3f', flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the sliders
- `v` - the current values
- `v_min`, `v_max` - the bounds, applied to every component
- `format` - printf format of the values drawn on the sliders

**Returns:** `(changed, v)`

```python
extent = [0.1, 0.9, 0.1, 0.9]

changed, extent = imgui.slider_float4("extent", extent, 0.0, 1.0, format="%.2f")
```

### slider_int

```python
imgui.slider_int(label: str, v: int, v_min: int, v_max: int, format: str = '%d', flags: int = 0) -> tuple[bool, int]
```

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the slider
- `v` - the current value
- `v_min`, `v_max` - the bounds, the value is clamped to them
- `format` - printf format of the value drawn on the slider

**Returns:** `(changed, v)`

```python
n_bins = 100

changed, n_bins = imgui.slider_int("bins", v=n_bins, v_min=10, v_max=500)
```

### slider_int2

```python
imgui.slider_int2(label: str, v: Sequence[int], v_min: int, v_max: int, format: str = '%d', flags: int = 0) -> tuple[bool, list[int]]
```

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the sliders
- `v` - the current values
- `v_min`, `v_max` - the bounds, applied to both components
- `format` - printf format of the values drawn on the sliders

**Returns:** `(changed, v)`

```python
crop = [64, 448]

changed, crop = imgui.slider_int2("crop rows", crop, 0, 512)
```

### slider_int3

```python
imgui.slider_int3(label: str, v: Sequence[int], v_min: int, v_max: int, format: str = '%d', flags: int = 0) -> tuple[bool, list[int]]
```

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the sliders
- `v` - the current values
- `v_min`, `v_max` - the bounds, applied to every component
- `format` - printf format of the values drawn on the sliders

**Returns:** `(changed, v)`

```python
stride = [1, 2, 2]

changed, stride = imgui.slider_int3("stride", stride, 1, 8)
```

### slider_int4

```python
imgui.slider_int4(label: str, v: Sequence[int], v_min: int, v_max: int, format: str = '%d', flags: int = 0) -> tuple[bool, list[int]]
```

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the sliders
- `v` - the current values
- `v_min`, `v_max` - the bounds, applied to every component
- `format` - printf format of the values drawn on the sliders

**Returns:** `(changed, v)`

```python
roi = [64, 64, 256, 256]

changed, roi = imgui.slider_int4("roi", roi, 0, 512)
```

### slider_angle

```python
imgui.slider_angle(label: str, v_rad: float, v_degrees_min: float = -360.0, v_degrees_max: float = 360.0, format: str = '%.0f deg', flags: int = 0) -> tuple[bool, float]
```

`flags` takes `imgui.SliderFlags_`

The value is in radians, the bounds and the value drawn on the slider are in degrees.

**Parameters**

- `label` - drawn to the right of the slider
- `v_rad` - the current angle, in radians
- `v_degrees_min`, `v_degrees_max` - the bounds, in degrees
- `format` - printf format of the angle drawn on the slider

**Returns:** `(changed, v_rad)`

```python
rotation = 0.6

changed, rotation = imgui.slider_angle("rotation", v_rad=rotation, v_degrees_min=-180, v_degrees_max=180)
```

### drag_float

```python
imgui.drag_float(label: str, v: float, v_speed: float = 1.0, v_min: float = 0.0, v_max: float = 0.0, format: str = '%.3f', flags: int = 0) -> tuple[bool, float]
```

If v_min >= v_max we have no bound

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the element
- `v` - the current value
- `v_speed` - how much the value changes per pixel of pointer movement
- `v_min`, `v_max` - the bounds, there is no bound while `v_min >= v_max`
- `format` - printf format of the value drawn on the element

**Returns:** `(changed, v)`

```python
sigma = 1.4

changed, sigma = imgui.drag_float("gaussian sigma", v=sigma, v_speed=0.05, v_min=0.1, v_max=20.0)
```

### drag_float2

```python
imgui.drag_float2(label: str, v: Sequence[float], v_speed: float = 1.0, v_min: float = 0.0, v_max: float = 0.0, format: str = '%.3f', flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the elements
- `v` - the current values
- `v_speed` - how much a value changes per pixel of pointer movement
- `v_min`, `v_max` - the bounds, applied to both components, there is no bound while `v_min >= v_max`
- `format` - printf format of the values drawn on the elements

**Returns:** `(changed, v)`

```python
origin = [0.0, 0.0]

changed, origin = imgui.drag_float2("origin", origin, v_speed=0.5)
```

### drag_float3

```python
imgui.drag_float3(label: str, v: Sequence[float], v_speed: float = 1.0, v_min: float = 0.0, v_max: float = 0.0, format: str = '%.3f', flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the elements
- `v` - the current values
- `v_speed` - how much a value changes per pixel of pointer movement
- `v_min`, `v_max` - the bounds, applied to every component, there is no bound while `v_min >= v_max`
- `format` - printf format of the values drawn on the elements

**Returns:** `(changed, v)`

```python
offset = [0.0, 0.0, 0.0]

changed, offset = imgui.drag_float3("offset", offset, v_speed=0.5)
```

### drag_float4

```python
imgui.drag_float4(label: str, v: Sequence[float], v_speed: float = 1.0, v_min: float = 0.0, v_max: float = 0.0, format: str = '%.3f', flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the elements
- `v` - the current values
- `v_speed` - how much a value changes per pixel of pointer movement
- `v_min`, `v_max` - the bounds, applied to every component, there is no bound while `v_min >= v_max`
- `format` - printf format of the values drawn on the elements

**Returns:** `(changed, v)`

```python
bounds = [0.0, 512.0, 0.0, 512.0]

changed, bounds = imgui.drag_float4("bounds", bounds, v_speed=1.0, format="%.0f")
```

### drag_float_range2

```python
imgui.drag_float_range2(label: str, v_current_min: float, v_current_max: float, v_speed: float = 1.0, v_min: float = 0.0, v_max: float = 0.0, format: str = '%.3f', format_max: str | None = None, flags: int = 0) -> tuple[bool, float, float]
```

`flags` takes `imgui.SliderFlags_`

Two values that cannot cross, the lower one is dragged from the left half and the upper one from the right half.

**Parameters**

- `label` - drawn to the right of the element
- `v_current_min`, `v_current_max` - the current values
- `v_speed` - how much a value changes per pixel of pointer movement
- `v_min`, `v_max` - the bounds, there is no bound while `v_min >= v_max`
- `format` - printf format of the lower value
- `format_max` - printf format of the upper value, `format` is used for both if it is not given

**Returns:** `(changed, v_current_min, v_current_max)`

```python
vmin, vmax = 12.0, 208.0

changed, vmin, vmax = imgui.drag_float_range2(
    "vmin / vmax", vmin, vmax, v_speed=1.0, v_min=0.0, v_max=255.0, format="%.0f"
)
```

### drag_int

```python
imgui.drag_int(label: str, v: int, v_speed: float = 1.0, v_min: int = 0, v_max: int = 0, format: str = '%d', flags: int = 0) -> tuple[bool, int]
```

If v_min >= v_max we have no bound

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the element
- `v` - the current value
- `v_speed` - how much the value changes per pixel of pointer movement
- `v_min`, `v_max` - the bounds, there is no bound while `v_min >= v_max`
- `format` - printf format of the value drawn on the element

**Returns:** `(changed, v)`

```python
window = 30

changed, window = imgui.drag_int("window size", v=window, v_speed=1.0, v_min=1, v_max=500)
```

### drag_int_range2

```python
imgui.drag_int_range2(label: str, v_current_min: int, v_current_max: int, v_speed: float = 1.0, v_min: int = 0, v_max: int = 0, format: str = '%d', format_max: str | None = None, flags: int = 0) -> tuple[bool, int, int]
```

`flags` takes `imgui.SliderFlags_`

**Parameters**

- `label` - drawn to the right of the element
- `v_current_min`, `v_current_max` - the current values, they cannot cross
- `v_speed` - how much a value changes per pixel of pointer movement
- `v_min`, `v_max` - the bounds, there is no bound while `v_min >= v_max`
- `format` - printf format of the lower value
- `format_max` - printf format of the upper value, `format` is used for both if it is not given

**Returns:** `(changed, v_current_min, v_current_max)`

```python
first, last = 40, 260

changed, first, last = imgui.drag_int_range2("frames", first, last, v_min=0, v_max=500)
```

## Input

Input elements are typed into. A slider or a drag is better for a value that is explored by eye, an input is better
for a value that is known.

### input_text

```python
imgui.input_text(label: str, str: str, flags: int = 0, callback: Callable[[InputTextCallbackData], int] | None = None, user_data: capsule | None = None) -> tuple[bool, str]
```

`flags` takes `imgui.InputTextFlags_`

**Parameters**

- `label` - drawn to the right of the field, `"##hidden"` suppresses it
- `str` - the current text
- `callback`, `user_data` - an imgui input callback, for completion or filtering

**Returns:** `(changed, str)` - `changed` is `True` on every keystroke unless
`imgui.InputTextFlags_` asks otherwise

```python
name = "roi-1"

changed, name = imgui.input_text("graphic name", name)
```

### input_text_multiline

```python
imgui.input_text_multiline(label: str, str: str, size: ImVec2 | None = None, flags: int = 0, callback: Callable[[InputTextCallbackData], int] | None = None, user_data: capsule | None = None) -> tuple[bool, str]
```

`flags` takes `imgui.InputTextFlags_`

> **Note:** If size is None, then its default value will be: ImVec2(0, 0)

**Parameters**

- `label` - drawn to the right of the field
- `str` - the current text
- `size` - `(width, height)` of the field, a zero component is a default size
- `callback`, `user_data` - an imgui input callback

**Returns:** `(changed, str)`

```python
notes = "frame 42\nsaturated pixels\nrecheck vmax"

changed, notes = imgui.input_text_multiline("notes", notes, (220, 70))
```

### input_text_with_hint

```python
imgui.input_text_with_hint(label: str, hint: str, str: str, flags: int = 0, callback: Callable[[InputTextCallbackData], int] | None = None, user_data: capsule | None = None) -> tuple[bool, str]
```

`flags` takes `imgui.InputTextFlags_`

The hint is drawn in the field while it is empty, use it instead of a label when there is no room for one.

**Parameters**

- `label` - drawn to the right of the field
- `hint` - drawn in the field while `str` is empty
- `str` - the current text
- `callback`, `user_data` - an imgui input callback

**Returns:** `(changed, str)`

```python
pattern = ""

changed, pattern = imgui.input_text_with_hint("##filter", "filter graphics", pattern)
```

### input_float

```python
imgui.input_float(label: str, v: float, step: float = 0.0, step_fast: float = 0.0, format: str = '%.3f', flags: int = 0) -> tuple[bool, float]
```

`flags` takes `imgui.InputTextFlags_`

**Parameters**

- `label` - drawn to the right of the field
- `v` - the current value
- `step` - amount the `-` and `+` buttons change the value by, they are not drawn while it is `0.0`
- `step_fast` - amount used while ctrl is held
- `format` - printf format of the value in the field

**Returns:** `(changed, v)`

```python
threshold = 0.75

changed, threshold = imgui.input_float("threshold", v=threshold, step=0.05, step_fast=0.5)
```

### input_float2

```python
imgui.input_float2(label: str, v: Sequence[float], format: str = '%.3f', flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.InputTextFlags_`

Two, three, and four fields on one row. Pass a list and use the list that comes back.

**Parameters**

- `label` - drawn to the right of the fields
- `v` - the current values
- `format` - printf format of the values in the fields

**Returns:** `(changed, v)`

```python
pixel_size = [0.325, 0.325]

changed, pixel_size = imgui.input_float2("pixel size (um)", pixel_size, format="%.3f")
```

### input_float3

```python
imgui.input_float3(label: str, v: Sequence[float], format: str = '%.3f', flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.InputTextFlags_`

**Parameters**

- `label` - drawn to the right of the fields
- `v` - the current values
- `format` - printf format of the values in the fields

**Returns:** `(changed, v)`

```python
origin = [0.0, 0.0, 0.0]

changed, origin = imgui.input_float3("origin", origin, format="%.1f")
```

### input_float4

```python
imgui.input_float4(label: str, v: Sequence[float], format: str = '%.3f', flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.InputTextFlags_`

**Parameters**

- `label` - drawn to the right of the fields
- `v` - the current values
- `format` - printf format of the values in the fields

**Returns:** `(changed, v)`

```python
bounds = [0.0, 512.0, 0.0, 512.0]

changed, bounds = imgui.input_float4("bounds", bounds, format="%.0f")
```

### input_int

```python
imgui.input_int(label: str, v: int, step: int = 1, step_fast: int = 100, flags: int = 0) -> tuple[bool, int]
```

`flags` takes `imgui.InputTextFlags_`

**Parameters**

- `label` - drawn to the right of the field
- `v` - the current value
- `step` - amount the `-` and `+` buttons change the value by
- `step_fast` - amount used while ctrl is held

**Returns:** `(changed, v)`

```python
n_components = 8

changed, n_components = imgui.input_int("components", v=n_components, step=1, step_fast=10)
```

### input_int2

```python
imgui.input_int2(label: str, v: Sequence[int], flags: int = 0) -> tuple[bool, list[int]]
```

`flags` takes `imgui.InputTextFlags_`

**Parameters**

- `label` - drawn to the right of the fields
- `v` - the current values

**Returns:** `(changed, v)`

```python
shape = [512, 512]

changed, shape = imgui.input_int2("output shape", shape)
```

### input_int3

```python
imgui.input_int3(label: str, v: Sequence[int], flags: int = 0) -> tuple[bool, list[int]]
```

`flags` takes `imgui.InputTextFlags_`

**Parameters**

- `label` - drawn to the right of the fields
- `v` - the current values

**Returns:** `(changed, v)`

```python
chunks = [1, 256, 256]

changed, chunks = imgui.input_int3("chunks", chunks)
```

### input_int4

```python
imgui.input_int4(label: str, v: Sequence[int], flags: int = 0) -> tuple[bool, list[int]]
```

`flags` takes `imgui.InputTextFlags_`

**Parameters**

- `label` - drawn to the right of the fields
- `v` - the current values

**Returns:** `(changed, v)`

```python
roi = [64, 64, 256, 256]

changed, roi = imgui.input_int4("roi", roi)
```

### input_double

```python
imgui.input_double(label: str, v: float, step: float = 0.0, step_fast: float = 0.0, format: str = '%.6f', flags: int = 0) -> tuple[bool, float]
```

`flags` takes `imgui.InputTextFlags_`

**Parameters**

- `label` - drawn to the right of the field
- `v` - the current value
- `step` - amount the `-` and `+` buttons change the value by, they are not drawn while it is `0.0`
- `step_fast` - amount used while ctrl is held
- `format` - printf format of the value in the field

**Returns:** `(changed, v)`

```python
exposure = 0.008

changed, exposure = imgui.input_double("exposure (s)", v=exposure, step=0.001, format="%.4f")
```

## Selection

### combo

**Overloads**

```python
imgui.combo(label: str, current_item: int, items: Sequence[str], popup_max_height_in_items: int = -1) -> tuple[bool, int]
```

```python
imgui.combo(label: str, current_item: int, items_separated_by_zeros: str, popup_max_height_in_items: int = -1) -> tuple[bool, int]
```

Separate items with \\0 within a string, end item-list with \\0\\0. e.g. "One\\0Two\\0Three\\0"

**Parameters**

- `label` - drawn to the right of the box, `"##hidden"` suppresses it
- `current_item` - index of the selected item
- `items` - the items, as a sequence of strings
- `popup_max_height_in_items` - how many items the open list shows before it scrolls

**Returns:** `(changed, current_item)`

```python
mode, modes = 1, ["mip", "minip", "iso", "slice"]

changed, mode = imgui.combo("render mode", mode, modes)
```

The list is drawn while the box is open:

```python
mode, modes = 1, ["mip", "minip", "iso", "slice"]

changed, mode = imgui.combo("render mode", mode, modes)
```

### begin_combo

```python
imgui.begin_combo(label: str, preview_value: str, flags: int = 0) -> bool
```

`flags` takes `imgui.ComboFlags_`

Use these instead of `combo` when the items are not plain strings, the body draws whatever it likes. Call
`end_combo` only when `begin_combo` returned `True`.

**Parameters**

- `label` - drawn to the right of the box
- `preview_value` - drawn in the box while it is closed

```python
selected, graphics = "line-1", ["line-1", "line-2", "scatter-1"]

if imgui.begin_combo("graphic", selected):
    for name in graphics:
        clicked, _ = imgui.selectable(name, name == selected)
        if clicked:
            selected = name

    imgui.end_combo()
```

### end_combo

```python
imgui.end_combo() -> None
```

only call EndCombo() if BeginCombo() returns True!

Call it only when the matching `begin_combo` returned `True`.

**Parameters**

none

### list_box

```python
imgui.list_box(label: str, current_item: int, items: Sequence[str], height_in_items: int = -1) -> tuple[bool, int]
```

A list box shows several items at once, a combo box hides them until it is opened.

**Parameters**

- `label` - drawn to the right of the box
- `current_item` - index of the selected item
- `items` - the items, as a sequence of strings
- `height_in_items` - how many items are visible before the box scrolls

**Returns:** `(changed, current_item)`

```python
selected, graphics = 0, ["line-1", "line-2", "scatter-1", "image-1"]

changed, selected = imgui.list_box("graphics", selected, graphics, height_in_items=4)
```

### begin_list_box

```python
imgui.begin_list_box(label: str, size: ImVec2 | None = None) -> bool
```

open a framed scrolling region

> **Note:** If size is None, then its default value will be: ImVec2(0, 0)

**Parameters**

- `label` - drawn to the right of the box
- `size` - `(width, height)`, a zero component is a default size

```python
selected, graphics = "line-1", ["line-1", "line-2", "scatter-1"]

if imgui.begin_list_box("graphics", (160, 70)):
    for name in graphics:
        clicked, _ = imgui.selectable(name, name == selected)
        if clicked:
            selected = name

    imgui.end_list_box()
```

### end_list_box

```python
imgui.end_list_box() -> None
```

only call EndListBox() if BeginListBox() returned True!

Call it only when the matching `begin_list_box` returned `True`.

**Parameters**

none

### selectable

```python
imgui.selectable(label: str, p_selected: bool, flags: int = 0, size: ImVec2 | None = None) -> tuple[bool, bool]
```

"bool\* p_selected" point to the selection state (read-write), as a convenient helper.

`flags` takes `imgui.SelectableFlags_`

> **Note:** If size is None, then its default value will be: ImVec2(0, 0)

A row of text that can be selected, and the item to build lists out of.

**Parameters**

- `label` - drawn in the row
- `p_selected` - whether this row is drawn as selected
- `size` - `(width, height)`, a zero component fills the available width

**Returns:** `(clicked, p_selected)`

```python
selected = "scatter-1"

for name in ["line-1", "line-2", "scatter-1"]:
    clicked, _ = imgui.selectable(name, name == selected)
    if clicked:
        selected = name
```

## Color

A color is a list of floats in `0.0` to `1.0`, three of them for RGB and four for RGBA. The `3` and `4`
variants differ only in whether they include alpha.

### color_edit3

```python
imgui.color_edit3(label: str, col: Sequence[float], flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.ColorEditFlags_`

A row of numeric fields with a color square at its right end. Clicking the square opens a picker, right-clicking it
opens a menu of display options.

**Parameters**

- `label` - drawn to the right of the fields, `"##hidden"` suppresses it
- `col` - the current color

**Returns:** `(changed, col)`

```python
color = [0.9, 0.3, 0.2]

changed, color = imgui.color_edit3("line color", color)
```

### color_edit4

```python
imgui.color_edit4(label: str, col: Sequence[float], flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.ColorEditFlags_`

`color_edit3` with an alpha field.

**Parameters**

- `label` - drawn to the right of the fields
- `col` - the current color

**Returns:** `(changed, col)`

```python
color = [0.9, 0.3, 0.2, 0.5]

changed, color = imgui.color_edit4("fill color", color)
```

### color_picker3

```python
imgui.color_picker3(label: str, col: Sequence[float], flags: int = 0) -> tuple[bool, list[float]]
```

`flags` takes `imgui.ColorEditFlags_`

The full picker, drawn inline. `color_edit3` is the compact element and opens this in a popup when its square is
clicked.

**Parameters**

- `label` - drawn above the picker
- `col` - the current color

**Returns:** `(changed, col)`

```python
color = [0.2, 0.6, 0.95]

changed, color = imgui.color_picker3("##picker", color)
```

### color_picker4

```python
imgui.color_picker4(label: str, col: Sequence[float], flags: int = 0, ref_col: float | None = None) -> tuple[bool, list[float]]
```

`flags` takes `imgui.ColorEditFlags_`

`color_picker3` with an alpha bar.

**Parameters**

- `label` - drawn to the right of the picker
- `col` - the current color
- `ref_col` - a second color drawn beside the current one, to compare against

**Returns:** `(changed, col)`

```python
color = [0.2, 0.6, 0.95, 0.7]

changed, color = imgui.color_picker4("##picker4", color)
```

### color_button

```python
imgui.color_button(desc_id: str, col: ImVec4, flags: int = 0, size: ImVec2 | None = None) -> bool
```

display a color square/button, hover for details, return True when pressed.

`flags` takes `imgui.ColorEditFlags_`

> **Note:** If size is None, then its default value will be: ImVec2(0, 0)

**Parameters**

- `desc_id` - identifies the button, and is shown in its tooltip
- `col` - the color to draw, `(r, g, b, a)`
- `size` - `(width, height)`, a zero component is a square the height of one row

**Returns:** `True` on the frame the button is clicked

```python
for name, color in [("magenta", (1.0, 0.0, 1.0, 1.0)), ("cyan", (0.0, 1.0, 1.0, 1.0))]:
    if imgui.color_button(name, color, size=(40, 20)):
        print(f"{name} clicked")

    imgui.same_line()
    imgui.text(name)
```

### set_color_edit_options

```python
imgui.set_color_edit_options(flags: int) -> None
```

initialize current options (generally on application startup) if you want to select a default format, picker type, etc. User will be able to change many settings, unless you pass the _NoOptions flag to your calls.

`flags` takes `imgui.ColorEditFlags_`

Sets the defaults for every color element that follows, so each one does not have to pass the same flags. Call it once
when the UI is created.

**Parameters**

- `flags` - the options to apply

```python
imgui.set_color_edit_options(int(imgui.ColorEditFlags_.float) | int(imgui.ColorEditFlags_.display_hsv))

color = [0.9, 0.3, 0.2]
changed, color = imgui.color_edit3("line color", color)
```

## Trees and tabs

### tree_node

**Overloads**

```python
imgui.tree_node(label: str) -> bool
```

```python
imgui.tree_node(str_id: str, fmt: str) -> bool
```

helper variation to easily decorrelate the id from the displayed string. Read the FAQ about why and how to use ID. to align arbitrary text at the same level as a TreeNode() you can use Bullet().

```python
imgui.tree_node(ptr_id: capsule, fmt: str) -> bool
```

"

Returns `True` while the node is open, in which case its contents are drawn and `tree_pop` must be called. The
node is opened and closed by the user, clicking the arrow.

**Parameters**

- `label` - drawn next to the arrow, and used as the id
- `str_id`, `ptr_id` - an id given separately, for when the label is not unique or changes between frames
- `fmt` - the text to draw when an id is given separately

```python
if imgui.tree_node("image-1"):
    imgui.text("512 x 512, uint16")
    imgui.text("vmin 12, vmax 208")
    imgui.tree_pop()
```

### tree_node_ex

**Overloads**

```python
imgui.tree_node_ex(label: str, flags: int = 0) -> bool
```

```python
imgui.tree_node_ex(str_id: str, flags: int, fmt: str) -> bool
```

```python
imgui.tree_node_ex(ptr_id: capsule, flags: int, fmt: str) -> bool
```

`flags` takes `imgui.TreeNodeFlags_`

`tree_node` with flags, e.g. to have the node start open, or to draw it without an arrow.

**Parameters**

- `label` - drawn next to the arrow, and used as the id
- `str_id`, `ptr_id` - an id given separately
- `fmt` - the text to draw when an id is given separately

```python
if imgui.tree_node_ex("image-1", flags=imgui.TreeNodeFlags_.default_open):
    imgui.text("512 x 512, uint16")
    imgui.tree_pop()
```

### tree_pop

```python
imgui.tree_pop() -> None
```

~ Unindent()+PopID()

**Parameters**

none

### collapsing_header

**Overloads**

```python
imgui.collapsing_header(label: str, flags: int = 0) -> bool
```

if returning 'True' the header is open. doesn't indent nor push on ID stack. user doesn't have to call TreePop().

```python
imgui.collapsing_header(label: str, p_visible: bool, flags: int = 0) -> tuple[bool, bool]
```

when 'p_visible != None': if '\*p_visible==True' display an additional small close button on upper right of the header which will set the bool to False when clicked, if '\*p_visible==False' don't display the header.

`flags` takes `imgui.TreeNodeFlags_`

A header that shows and hides a section. Unlike a tree node it does not indent its contents and needs no
`tree_pop`, which makes it the element for grouping controls.

**Parameters**

- `label` - drawn in the header
- `p_visible` - when given, a close button is drawn and this is set to `False` when it is clicked

**Returns:** `True` while the header is open, or `(open, p_visible)` for the second form

```python
sigma = 1.4

if imgui.collapsing_header("filter", flags=imgui.TreeNodeFlags_.default_open):
    changed, sigma = imgui.slider_float("sigma", v=sigma, v_min=0.1, v_max=10.0)

if imgui.collapsing_header("export"):
    imgui.text("not shown while the header is closed")
```

### set_next_item_open

```python
imgui.set_next_item_open(is_open: bool, cond: int = 0) -> None
```

set next TreeNode/CollapsingHeader open state.

Opens or closes the next tree node or collapsing header from code, rather than waiting for the user to click it.

**Parameters**

- `is_open` - the state to set
- `cond` - an `imgui.Cond_` value, e.g. `once` to set it only the first time

```python
imgui.set_next_item_open(True, imgui.Cond_.once)

if imgui.tree_node("image-1"):
    imgui.text("open because set_next_item_open was called")
    imgui.tree_pop()
```

### begin_tab_bar

```python
imgui.begin_tab_bar(str_id: str, flags: int = 0) -> bool
```

create and append into a TabBar

`flags` takes `imgui.TabBarFlags_`

**Parameters**

- `str_id` - identifies the tab bar, it is not drawn

```python
if imgui.begin_tab_bar("panels"):
    if imgui.begin_tab_item("image")[0]:
        imgui.text("512 x 512, uint16")
        imgui.end_tab_item()

    if imgui.begin_tab_item("filter")[0]:
        imgui.text("gaussian, sigma 1.4")
        imgui.end_tab_item()

    imgui.end_tab_bar()
```

### end_tab_bar

```python
imgui.end_tab_bar() -> None
```

only call EndTabBar() if BeginTabBar() returns True!

Call it only when the matching `begin_tab_bar` returned `True`.

**Parameters**

none

### begin_tab_item

```python
imgui.begin_tab_item(label: str, p_open: bool | None = None, flags: int = 0) -> tuple[bool, bool | None]
```

create a Tab. Returns True if the Tab is selected.

`flags` takes `imgui.TabItemFlags_`

**Parameters**

- `label` - drawn on the tab
- `p_open` - when given, a close button is drawn on the tab and this is set to `False` when it is clicked

**Returns:** `(selected, p_open)`, draw the contents and call `end_tab_item` while `selected`

```python
if imgui.begin_tab_bar("panels"):
    for label in ["image", "filter", "export"]:
        selected, _ = imgui.begin_tab_item(label)
        if selected:
            imgui.text(f"{label} panel")
            imgui.end_tab_item()

    imgui.end_tab_bar()
```

### end_tab_item

```python
imgui.end_tab_item() -> None
```

only call EndTabItem() if BeginTabItem() returns True!

Call it only when the matching `begin_tab_item` returned `True`.

**Parameters**

none

### tab_item_button

```python
imgui.tab_item_button(label: str, flags: int = 0) -> bool
```

create a Tab behaving like a button. return True when clicked. cannot be selected in the tab bar.

`flags` takes `imgui.TabItemFlags_`

**Parameters**

- `label` - drawn on the tab

**Returns:** `True` on the frame the tab is clicked

```python
if imgui.begin_tab_bar("panels"):
    if imgui.begin_tab_item("image")[0]:
        imgui.end_tab_item()

    if imgui.tab_item_button("+"):
        print("add panel")

    imgui.end_tab_bar()
```

## Menus

A menu bar belongs to a window, so the window has to be created with `imgui.WindowFlags_.menu_bar`.

### begin_menu_bar

```python
imgui.begin_menu_bar() -> bool
```

append to menu-bar of current window (requires ImGuiWindowFlags_MenuBar flag set on parent window).

**Parameters**

none

```python
imgui.set_next_window_pos((0, 0))
imgui.set_next_window_size((220, 120))
imgui.begin("controls", flags=imgui.WindowFlags_.menu_bar)

if imgui.begin_menu_bar():
    if imgui.begin_menu("File"):
        imgui.menu_item("Open", "Ctrl+O", False)
        imgui.menu_item("Save", "Ctrl+S", False)
        imgui.end_menu()

    imgui.end_menu_bar()

imgui.end()
```

### end_menu_bar

```python
imgui.end_menu_bar() -> None
```

only call EndMenuBar() if BeginMenuBar() returns True!

Call it only when the matching `begin_menu_bar` returned `True`.

**Parameters**

none

### begin_main_menu_bar

```python
imgui.begin_main_menu_bar() -> bool
```

create and append to a full screen menu-bar.

A bar pinned across the top of the canvas, it is not part of any window.

**Parameters**

none

```python
if imgui.begin_main_menu_bar():
    if imgui.begin_menu("File"):
        imgui.menu_item("Open", "Ctrl+O", False)
        imgui.end_menu()

    if imgui.begin_menu("Help"):
        imgui.menu_item("Version", "", False)
        imgui.end_menu()

    imgui.end_main_menu_bar()
```

### end_main_menu_bar

```python
imgui.end_main_menu_bar() -> None
```

only call EndMainMenuBar() if BeginMainMenuBar() returns True!

Call it only when the matching `begin_main_menu_bar` returned `True`.

**Parameters**

none

### begin_menu

```python
imgui.begin_menu(label: str, enabled: bool = True) -> bool
```

create a sub-menu entry. only call EndMenu() if this returns True!

Returns `True` while the menu is open, in which case its items are drawn and `end_menu` must be called. A
`begin_menu` inside another one is a submenu.

**Parameters**

- `label` - drawn on the menu
- `enabled` - a disabled menu is drawn greyed out and cannot be opened

```python
imgui.set_next_window_pos((0, 0))
imgui.set_next_window_size((240, 130))
imgui.begin("controls", flags=imgui.WindowFlags_.menu_bar)

if imgui.begin_menu_bar():
    if imgui.begin_menu("Graphics"):
        imgui.menu_item("Add line", "", False)

        if imgui.begin_menu("Add image"):
            imgui.menu_item("from file", "", False)
            imgui.menu_item("from array", "", False)
            imgui.end_menu()

        imgui.end_menu()

    imgui.end_menu_bar()

imgui.end()
```

### end_menu

```python
imgui.end_menu() -> None
```

only call EndMenu() if BeginMenu() returns True!

Call it only when the matching `begin_menu` returned `True`.

**Parameters**

none

### menu_item

```python
imgui.menu_item(label: str, shortcut: str, p_selected: bool, enabled: bool = True) -> tuple[bool, bool]
```

return True when activated + toggle (\*p_selected) if p_selected != None

**Parameters**

- `label` - drawn on the item
- `shortcut` - drawn right-aligned on the item, it is a label only and does not bind the key
- `p_selected` - when `True` a check mark is drawn, pass it a bool to make the item a toggle
- `enabled` - a disabled item is drawn greyed out and cannot be clicked

**Returns:** `(clicked, p_selected)`

```python
show_fps = True

imgui.set_next_window_pos((0, 0))
imgui.set_next_window_size((230, 120))
imgui.begin("controls", flags=imgui.WindowFlags_.menu_bar)

if imgui.begin_menu_bar():
    if imgui.begin_menu("View"):
        clicked, show_fps = imgui.menu_item("Show fps", "", show_fps)
        imgui.menu_item("Autoscale", "A", False)
        imgui.menu_item("Reset camera", "", False, enabled=False)
        imgui.end_menu()

    imgui.end_menu_bar()

imgui.end()
```

## Popups and tooltips

A popup is opened by `open_popup` and drawn by `begin_popup`, which returns `True` only while it is open. Both
have to be called for the same window, so calling `open_popup` from inside a menu does not open a popup that
`begin_popup` draws outside of it.

### open_popup

**Overloads**

```python
imgui.open_popup(str_id: str, popup_flags: int = 0) -> None
```

call to mark popup as open (don't call every frame!).

```python
imgui.open_popup(id_: int, popup_flags: int = 0) -> None
```

id overload to facilitate calling from nested stacks

`popup_flags` takes `imgui.PopupFlags_`

**Parameters**

- `str_id` - identifies the popup, `begin_popup` is called with the same id
- `id_` - an integer id instead of a string one
- `popup_flags` - options such as not opening over a popup that is already open

```python
if imgui.button("options"):
    imgui.open_popup("options")

if imgui.begin_popup("options"):
    imgui.menu_item("reset vmin / vmax", "", False)
    imgui.menu_item("reset gamma", "", False)
    imgui.end_popup()
```

### begin_popup

```python
imgui.begin_popup(str_id: str, flags: int = 0) -> bool
```

return True if the popup is open, and you can start outputting to it.

`flags` takes `imgui.WindowFlags_`

Call `end_popup` only when `begin_popup` returned `True`. The popup closes when the user clicks outside it, or
when a menu item inside it is clicked.

**Parameters**

- `str_id` - the id that `open_popup` was called with

```python
sigma = 1.4

if imgui.button("filter"):
    imgui.open_popup("filter")

if imgui.begin_popup("filter"):
    changed, sigma = imgui.slider_float("sigma", v=sigma, v_min=0.1, v_max=10.0)
    imgui.end_popup()
```

### end_popup

```python
imgui.end_popup() -> None
```

only call EndPopup() if BeginPopupXXX() returns True!

Call it only when the matching `begin_popup` returned `True`.

**Parameters**

none

### begin_popup_modal

```python
imgui.begin_popup_modal(name: str, p_open: bool | None = None, flags: int = 0) -> tuple[bool, bool | None]
```

return True if the modal is open, and you can start outputting to it.

`flags` takes `imgui.WindowFlags_`

A modal has a title bar and blocks everything behind it until it is closed. Passing `p_open` draws a close button in
its title bar.

**Parameters**

- `name` - the id that `open_popup` was called with, and the title
- `p_open` - when given, a close button is drawn and imgui closes the modal when it is clicked

**Returns:** `(open, p_open)`

```python
if imgui.button("about"):
    imgui.open_popup("About")

if imgui.begin_popup_modal("About", True)[0]:
    imgui.text("fastplotlib")
    imgui.end_popup()
```

### close_current_popup

```python
imgui.close_current_popup() -> None
```

manually close the popup we have begin-ed into.

Closes the popup being drawn, for a control that should dismiss it. A menu item already does this on its own.

**Parameters**

none

```python
if imgui.button("options"):
    imgui.open_popup("options")

if imgui.begin_popup("options"):
    imgui.text("apply the filter to every frame?")

    if imgui.button("cancel"):
        imgui.close_current_popup()

    imgui.end_popup()
```

### begin_popup_context_item

```python
imgui.begin_popup_context_item(str_id: str | None = None, popup_flags: int = 0) -> bool
```

open+begin popup when clicked on last item. Use str_id==None to associate the popup to previous item. If you want to use that on a non-interactive item such as Text() you need to pass in an explicit ID here. read comments in .cpp!

`popup_flags` takes `imgui.PopupFlags_`

Opens on a right-click on the element that precedes it, so a right-click menu needs no `open_popup` of its own.

**Parameters**

- `str_id` - identifies the popup, the preceding element is used when it is not given
- `popup_flags` - which mouse button opens it, right by default

```python
imgui.button("line-1")

if imgui.begin_popup_context_item():
    imgui.menu_item("hide", "", False)
    imgui.menu_item("delete", "", False)
    imgui.end_popup()
```

### begin_popup_context_window

```python
imgui.begin_popup_context_window(str_id: str | None = None, popup_flags: int = 0) -> bool
```

open+begin popup when clicked on current window.

`popup_flags` takes `imgui.PopupFlags_`

Opens on a right-click anywhere in the window that is not over an element.

**Parameters**

- `str_id` - identifies the popup
- `popup_flags` - which mouse button opens it, right by default

```python
imgui.text("right click the window")

if imgui.begin_popup_context_window():
    imgui.menu_item("add line", "", False)
    imgui.menu_item("add image", "", False)
    imgui.end_popup()
```

### is_popup_open

```python
imgui.is_popup_open(str_id: str, flags: int = 0) -> bool
```

return True if the popup is open.

`flags` takes `imgui.PopupFlags_`

**Parameters**

- `str_id` - the id the popup was opened with
- `flags` - use `imgui.PopupFlags_.any_popup_id` to ask about any popup

**Returns:** `True` while the popup is open

```python
if imgui.button("options"):
    imgui.open_popup("options")

imgui.same_line()
imgui.text(f"open: {imgui.is_popup_open('options')}")

if imgui.begin_popup("options"):
    imgui.menu_item("reset", "", False)
    imgui.end_popup()
```

### set_tooltip

```python
imgui.set_tooltip(fmt: str) -> None
```

set a text-only tooltip. Often used after a ImGui::IsItemHovered() check. Override any previous call to SetTooltip().

**Parameters**

- `fmt` - the text to draw in the tooltip

```python
imgui.button(fa.ICON_FA_MAXIMIZE)

if imgui.is_item_hovered():
    imgui.set_tooltip("autoscale scene")
```

### set_item_tooltip

```python
imgui.set_item_tooltip(fmt: str) -> None
```

set a text-only tooltip if preceding item was hovered. override any previous call to SetTooltip().

The same as `set_tooltip` behind an `is_item_hovered` check, for the common case of a tooltip on the element that
precedes it.

**Parameters**

- `fmt` - the text to draw in the tooltip

```python
imgui.button(fa.ICON_FA_ALIGN_CENTER)
imgui.set_item_tooltip("center scene")
```

### begin_tooltip

```python
imgui.begin_tooltip() -> bool
```

begin/append a tooltip window.

A tooltip that holds any elements, not only text. Call `end_tooltip` only when `begin_tooltip` returned `True`.

**Parameters**

none

```python
imgui.button("image-1")

if imgui.is_item_hovered() and imgui.begin_tooltip():
    imgui.text("image-1")
    imgui.separator()
    imgui.label_text("shape", "(512, 512)")
    imgui.label_text("dtype", "uint16")
    imgui.end_tooltip()
```

### end_tooltip

```python
imgui.end_tooltip() -> None
```

only call EndTooltip() if BeginTooltip()/BeginItemTooltip() returns True!

Call it only when the matching `begin_tooltip` returned `True`.

**Parameters**

none

## Layout

Elements are stacked vertically in the order they are called. These change where the next element goes, so most of them
draw nothing by themselves and are shown here between elements that do.

### same_line

```python
imgui.same_line(offset_from_start_x: float = 0.0, spacing: float = -1.0) -> None
```

call between widgets or groups to layout them horizontally. X position given in window coordinates.

**Parameters**

- `offset_from_start_x` - x position in window coordinates, the default continues after the previous element
- `spacing` - gap in pixels, the default uses the style spacing

```python
imgui.button("apply")
imgui.same_line()
imgui.button("reset")
```

### new_line

```python
imgui.new_line() -> None
```

undo a SameLine() or force a new line when in a horizontal-layout context.

**Parameters**

none

```python
imgui.button("apply")
imgui.same_line()
imgui.new_line()
imgui.button("reset")
```

### separator

```python
imgui.separator() -> None
```

separator, generally horizontal. inside a menu bar or in horizontal layout mode, this becomes a vertical separator.

**Parameters**

none

```python
imgui.text("filter")
imgui.separator()
imgui.text("export")
```

### spacing

```python
imgui.spacing() -> None
```

add vertical spacing.

**Parameters**

none

```python
imgui.button("apply")
imgui.spacing()
imgui.spacing()
imgui.button("reset")
```

### dummy

```python
imgui.dummy(size: ImVec2) -> None
```

add a dummy item of given size. unlike InvisibleButton(), Dummy() won't take the mouse click or be navigable into.

An empty element of a given size, to leave a gap that spacing cannot make. It takes no pointer input, unlike
`invisible_button`.

**Parameters**

- `size` - `(width, height)` of the gap

```python
imgui.button("apply")
imgui.same_line()
imgui.dummy((40, 0))
imgui.same_line()
imgui.button("delete")
```

### indent

```python
imgui.indent(indent_w: float = 0.0) -> None
```

move content position toward the right, by indent_w, or style.IndentSpacing if indent_w <= 0

**Parameters**

- `indent_w` - width in pixels, the default uses the style indent

```python
imgui.text("filter")
imgui.indent()
imgui.text("gaussian, sigma 1.4")
imgui.text("applied to every frame")
imgui.unindent()
imgui.text("export")
```

### unindent

```python
imgui.unindent(indent_w: float = 0.0) -> None
```

move content position back to the left, by indent_w, or style.IndentSpacing if indent_w <= 0

**Parameters**

- `indent_w` - width in pixels, the default uses the style indent

### begin_group

```python
imgui.begin_group() -> None
```

lock horizontal starting position

Everything between them becomes one item, so `same_line` places the whole group and `is_item_hovered` covers all of
it.

**Parameters**

none

```python
imgui.begin_group()
imgui.text("vmin")
imgui.text("12")
imgui.end_group()

imgui.same_line()
imgui.dummy((20, 0))
imgui.same_line()

imgui.begin_group()
imgui.text("vmax")
imgui.text("208")
imgui.end_group()
```

### end_group

```python
imgui.end_group() -> None
```

unlock horizontal starting position + capture the whole group bounding box into one "item" (so you can use IsItemHovered() or layout primitives such as SameLine() on whole group, etc.)

Ends the group, and makes everything in it one item for `same_line` and the item queries.

**Parameters**

none

### align_text_to_frame_padding

```python
imgui.align_text_to_frame_padding() -> None
```

vertically align upcoming text baseline to FramePadding.y so that it will align properly to regularly framed items (call if you have text on a line before a framed item)

Text is drawn without a frame, so on a row shared with a slider or a button it sits too high. Call this before the text
to line them up.

**Parameters**

none

```python
sigma = 1.4

imgui.align_text_to_frame_padding()
imgui.text("sigma")
imgui.same_line()
changed, sigma = imgui.slider_float("##sigma", v=sigma, v_min=0.1, v_max=10.0)
```

### set_next_item_width

```python
imgui.set_next_item_width(item_width: float) -> None
```

set width of the _next_ common large "item+label" widget. >0.0: width in pixels, <0.0 align xx pixels to the right of window (so -FLT_MIN always align width to the right side)

**Parameters**

- `item_width` - width in pixels, a negative value leaves that many pixels between the element and the right edge

```python
vmin, vmax = 12.0, 208.0

imgui.set_next_item_width(80)
changed, vmin = imgui.slider_float("vmin", v=vmin, v_min=0.0, v_max=255.0, format="%.0f")

imgui.set_next_item_width(80)
changed, vmax = imgui.slider_float("vmax", v=vmax, v_min=0.0, v_max=255.0, format="%.0f")
```

### push_item_width

```python
imgui.push_item_width(item_width: float) -> None
```

push width of items for common large "item+label" widgets. >0.0: width in pixels, <0.0 align xx pixels to the right of window (so -FLT_MIN always align width to the right side).

The same as `set_next_item_width` but for every element until `pop_item_width`.

**Parameters**

- `item_width` - width in pixels, a negative value leaves that many pixels between the element and the right edge

```python
vmin, vmax = 12.0, 208.0

imgui.push_item_width(80)
changed, vmin = imgui.slider_float("vmin", v=vmin, v_min=0.0, v_max=255.0, format="%.0f")
changed, vmax = imgui.slider_float("vmax", v=vmax, v_min=0.0, v_max=255.0, format="%.0f")
imgui.pop_item_width()
```

### pop_item_width

```python
imgui.pop_item_width() -> None
```

Pops the width that `push_item_width` pushed.

**Parameters**

none

### calc_text_size

```python
imgui.calc_text_size(text: str, text_end: str | None = None, hide_text_after_double_hash: bool = False, wrap_width: float = -1.0) -> ImVec2
```

Text Utilities

**Parameters**

- `text` - the text to measure
- `text_end` - measure up to this substring
- `hide_text_after_double_hash` - ignore everything after `##`, as the elements do with their labels
- `wrap_width` - measure as if the text were wrapped at this width

**Returns:** the size, use `.x` and `.y`

```python
label = "vmin / vmax"
size = imgui.calc_text_size(label)

imgui.text(label)
imgui.text(f"that text is {size.x:.0f} x {size.y:.0f} px")
```

### get_content_region_avail

```python
imgui.get_content_region_avail() -> ImVec2
```

available space from current position. THIS IS YOUR BEST FRIEND.

The space left in the window from the current position, which is how an element is sized to fill the window.

**Parameters**

none

**Returns:** the available size, use `.x` and `.y`

```python
available = imgui.get_content_region_avail()

imgui.text(f"{available.x:.0f} x {available.y:.0f} px left")
imgui.button("fill the width", (available.x, 0))
```

### get_cursor_pos

```python
imgui.get_cursor_pos() -> ImVec2
```

[window-local] cursor position in window-local coordinates. This is not your best friend.

Where the next element goes, in window coordinates.

**Parameters**

- `local_pos` - `(x, y)` in window coordinates

```python
imgui.set_cursor_pos((60, 30))
imgui.button("moved")
```

### set_cursor_pos

```python
imgui.set_cursor_pos(local_pos: ImVec2) -> None
```

[window-local] "

Moves the position of the next element, in window coordinates.

**Parameters**

- `local_pos` - `(x, y)` in window coordinates

```python
imgui.set_cursor_pos((60, 30))
imgui.button("moved")
```

### get_cursor_screen_pos

```python
imgui.get_cursor_screen_pos() -> ImVec2
```

cursor position, absolute coordinates. THIS IS YOUR BEST FRIEND (prefer using this rather than GetCursorPos(), also more useful to work with ImDrawList API).

The same position in canvas coordinates, which is what a draw list takes.

**Parameters**

- `pos` - `(x, y)` in canvas coordinates

```python
draw_list = imgui.get_window_draw_list()
position = imgui.get_cursor_screen_pos()

draw_list.add_rect_filled(
    position,
    (position.x + 60, position.y + 20),
    imgui.color_convert_float4_to_u32((0.2, 0.6, 0.95, 1.0)),
)
imgui.dummy((60, 20))
```

### set_cursor_screen_pos

```python
imgui.set_cursor_screen_pos(pos: ImVec2) -> None
```

cursor position, absolute coordinates. THIS IS YOUR BEST FRIEND.

Moves the position of the next element, in canvas coordinates.

**Parameters**

- `pos` - `(x, y)` in canvas coordinates

### get_text_line_height

```python
imgui.get_text_line_height() -> float
```

~ FontSize

The height of a line of text, and the height of an element that has a frame such as a button or a slider. Use them to
size something you draw yourself so that it lines up with the elements around it.

**Parameters**

none

```python
imgui.text(f"text line: {imgui.get_text_line_height():.0f} px")
imgui.text(f"framed element: {imgui.get_frame_height():.0f} px")
```

### get_frame_height

```python
imgui.get_frame_height() -> float
```

~ FontSize + style.FramePadding.y \* 2

The height of an element that has a frame, such as a button or a slider.

**Parameters**

none

**Returns:** the height in pixels

```python
imgui.text(f"framed element: {imgui.get_frame_height():.0f} px")
```

## Windows

In fastplotlib the window is created for you, `ImguiWindow.update()` draws into it. These are for a window you create
yourself, inside an overridden `ImguiWindow.draw()`.

### begin

```python
imgui.begin(name: str, p_open: bool | None = None, flags: int = 0) -> tuple[bool, bool | None]
```

`flags` takes `imgui.WindowFlags_`

`end` is called whether or not `begin` returned `True`. `begin` returns `False` when the window is collapsed,
in which case its contents can be skipped.

**Parameters**

- `name` - the title, and the id of the window, `"title##id"` separates the two
- `p_open` - when given, a close button is drawn in the title bar and this is set to `False` when it is clicked

**Returns:** `(expanded, p_open)`

```python
expanded, open_ = imgui.begin("filter", True)

if expanded:
    imgui.text("gaussian")

imgui.end()
```

### end

```python
imgui.end() -> None
```

Called whether or not `begin` returned `True`.

**Parameters**

none

### begin_child

**Overloads**

```python
imgui.begin_child(str_id: str, size: ImVec2 | None = None, child_flags: int = 0, window_flags: int = 0) -> bool
```

```python
imgui.begin_child(id_: int, size: ImVec2 | None = None, child_flags: int = 0, window_flags: int = 0) -> bool
```

`child_flags` takes `imgui.ChildFlags_`

`window_flags` takes `imgui.WindowFlags_`

> **Note:** If size is None, then its default value will be: ImVec2(0, 0)

A region within a window, with its own scrolling and clipping. Use it for a list that should scroll on its own.

**Parameters**

- `str_id`, `id_` - identifies the region
- `size` - `(width, height)`, a zero component fills the available space, a negative one leaves that many pixels

```python
if imgui.begin_child("graphics", (160, 80), child_flags=imgui.ChildFlags_.borders):
    for i in range(8):
        imgui.text(f"line-{i}")

    imgui.end_child()
```

### end_child

```python
imgui.end_child() -> None
```

Call it only when the matching `begin_child` returned `True`.

**Parameters**

none

### set_next_window_pos

```python
imgui.set_next_window_pos(pos: ImVec2, cond: int = 0, pivot: ImVec2 | None = None) -> None
```

set next window position. call before Begin(). use pivot=(0.5,0.5) to center on given point, etc.

> **Note:** If pivot is None, then its default value will be: ImVec2(0, 0)

**Parameters**

- `pos` - `(x, y)` in canvas coordinates
- `cond` - an `imgui.Cond_` value, e.g. `appearing` to place it only when it first appears so the user can move it
- `pivot` - which point of the window lands on `pos`, `(0.5, 0.5)` centers it there

```python
imgui.set_next_window_pos((40, 30))
imgui.set_next_window_size((160, 60))
imgui.begin("filter")
imgui.text("placed at 40, 30")
imgui.end()
```

### set_next_window_size

```python
imgui.set_next_window_size(size: ImVec2, cond: int = 0) -> None
```

set next window size. set axis to 0.0 to force an auto-fit on this axis. call before Begin()

**Parameters**

- `size` - `(width, height)`, a zero component makes that axis fit its contents
- `cond` - an `imgui.Cond_` value

```python
imgui.set_next_window_size((150, 0))
imgui.begin("filter")
imgui.text("fixed width, auto height")
imgui.end()
```

### set_next_window_collapsed

```python
imgui.set_next_window_collapsed(collapsed: bool, cond: int = 0) -> None
```

set next window collapsed state. call before Begin()

**Parameters**

- `collapsed` - the state to set
- `cond` - an `imgui.Cond_` value

```python
imgui.set_next_window_collapsed(True)
imgui.begin("filter")
imgui.text("not drawn while collapsed")
imgui.end()
```

### get_window_pos

```python
imgui.get_window_pos() -> ImVec2
```

get current window position in screen space (IT IS UNLIKELY YOU EVER NEED TO USE THIS. Consider always using GetCursorScreenPos() and GetContentRegionAvail() instead)

The position and size of the window being drawn. For laying out contents, `get_content_region_avail` is what you
want, since it accounts for padding and for the position within the window.

**Parameters**

none

```python
size = imgui.get_window_size()

imgui.text(f"window: {size.x:.0f} x {size.y:.0f} px")
```

### get_window_size

```python
imgui.get_window_size() -> ImVec2
```

get current window size (IT IS UNLIKELY YOU EVER NEED TO USE THIS. Consider always using GetCursorScreenPos() and GetContentRegionAvail() instead)

**Parameters**

none

**Returns:** the size, use `.x` and `.y`

```python
size = imgui.get_window_size()

imgui.text(f"window: {size.x:.0f} x {size.y:.0f} px")
```

### get_window_width

```python
imgui.get_window_width() -> float
```

get current window width (IT IS UNLIKELY YOU EVER NEED TO USE THIS). Shortcut for GetWindowSize().x.

**Parameters**

none

**Returns:** the width in pixels

### get_window_height

```python
imgui.get_window_height() -> float
```

get current window height (IT IS UNLIKELY YOU EVER NEED TO USE THIS). Shortcut for GetWindowSize().y.

**Parameters**

none

**Returns:** the height in pixels

### get_window_draw_list

```python
imgui.get_window_draw_list() -> ImDrawList
```

get draw list associated to the current window, to append your own drawing primitives

The draw list of the window, for drawing shapes and text yourself. Positions are in canvas coordinates, so they start
from `get_cursor_screen_pos`.

**Parameters**

none

**Returns:** an `imgui.ImDrawList`

```python
draw_list = imgui.get_window_draw_list()
position = imgui.get_cursor_screen_pos()

white = imgui.color_convert_float4_to_u32((1.0, 1.0, 1.0, 1.0))
blue = imgui.color_convert_float4_to_u32((0.2, 0.6, 0.95, 1.0))

draw_list.add_rect_filled(position, (position.x + 120, position.y + 8), blue)
draw_list.add_circle_filled((position.x + 30, position.y + 30), 8, white)
draw_list.add_text((position.x + 50, position.y + 22), white, "drawn by hand")

imgui.dummy((120, 45))
```

### set_scroll_here_y

```python
imgui.set_scroll_here_y(center_y_ratio: float = 0.5) -> None
```

adjust scrolling amount to make current cursor position visible. center_y_ratio=0.0: top, 0.5: center, 1.0: bottom. When using to make a "default/current item" visible, consider using SetItemDefaultFocus() instead.

`set_scroll_here_y` scrolls to the element that was just drawn, which is how a list follows a selection.

**Parameters**

- `center_y_ratio` - where the element ends up, `0.0` top, `0.5` center, `1.0` bottom
- `scroll_y` - the scroll amount in pixels

```python
if imgui.begin_child("graphics", (160, 70), child_flags=imgui.ChildFlags_.borders):
    for i in range(10):
        imgui.text(f"line-{i}")

        if i == 6:
            imgui.set_scroll_here_y(0.5)

    imgui.end_child()
```

### get_scroll_y

```python
imgui.get_scroll_y() -> float
```

get scrolling amount [0 .. GetScrollMaxY()]

**Parameters**

none

**Returns:** the scroll amount in pixels

### set_scroll_y

```python
imgui.set_scroll_y(scroll_y: float) -> None
```

set scrolling amount [0 .. GetScrollMaxY()]

**Parameters**

- `scroll_y` - the scroll amount in pixels

## Style and ids

Every push has a matching pop. A push that is not popped leaks into everything drawn afterwards, including elements
that fastplotlib draws.

### push_id

**Overloads**

```python
imgui.push_id(str_id: str) -> None
```

push string into the ID stack (will hash string).

```python
imgui.push_id(str_id_begin: str, str_id_end: str) -> None
```

push string into the ID stack (will hash string).

```python
imgui.push_id(ptr_id: capsule) -> None
```

push pointer into the ID stack (will hash pointer).

```python
imgui.push_id(int_id: int) -> None
```

push integer into the ID stack (will hash integer).

imgui identifies an element by its label, so two elements with the same label are the same element and share their
state. Push an id around them to keep them apart, which is what a loop over graphics needs.

**Parameters**

- `str_id`, `int_id`, `ptr_id` - the value to push, it is hashed and is not drawn
- `str_id_begin`, `str_id_end` - a substring to push

```python
thickness = {"line-1": 4.0, "line-2": 9.0}

for name in thickness:
    imgui.push_id(name)

    imgui.text(name)
    imgui.same_line()
    changed, thickness[name] = imgui.slider_float("##thickness", v=thickness[name], v_min=1.0, v_max=20.0)

    imgui.pop_id()
```

### pop_id

```python
imgui.pop_id() -> None
```

pop from the ID stack.

Pops the id that `push_id` pushed.

**Parameters**

none

### push_style_color

**Overloads**

```python
imgui.push_style_color(idx: int, col: int) -> None
```

modify a style color. always use this if you modify the style after NewFrame().

```python
imgui.push_style_color(idx: int, col: ImVec4) -> None
```

**Parameters**

- `idx` - which color, an `imgui.Col_` value
- `col` - the color, `(r, g, b, a)` or a packed `int`
- `count` - how many pushes to pop

```python
imgui.push_style_color(imgui.Col_.button, (0.6, 0.15, 0.15, 1.0))
imgui.push_style_color(imgui.Col_.button_hovered, (0.75, 0.2, 0.2, 1.0))

imgui.button("delete graphic")

imgui.pop_style_color(2)

imgui.button("keep graphic")
```

### pop_style_color

```python
imgui.pop_style_color(count: int = 1) -> None
```

**Parameters**

- `count` - how many pushed colors to pop

### push_style_var

**Overloads**

```python
imgui.push_style_var(idx: int, val: float) -> None
```

modify a style float variable. always use this if you modify the style after NewFrame()!

```python
imgui.push_style_var(idx: int, val: ImVec2) -> None
```

modify a style ImVec2 variable. "

**Parameters**

- `idx` - which variable, an `imgui.StyleVar_` value
- `val` - a float, or `(x, y)` for the variables that are a pair
- `count` - how many pushes to pop

```python
imgui.push_style_var(imgui.StyleVar_.frame_rounding, 10.0)
imgui.button("rounded")
imgui.pop_style_var()

imgui.button("default")
```

### pop_style_var

```python
imgui.pop_style_var(count: int = 1) -> None
```

**Parameters**

- `count` - how many pushed variables to pop

### get_style_color_vec4

```python
imgui.get_style_color_vec4(idx: int) -> ImVec4
```

retrieve style color as stored in ImGuiStyle structure. use to feed back into PushStyleColor(), otherwise use GetColorU32() to get style color with style alpha baked in.

**Parameters**

- `idx` - which color, an `imgui.Col_` value

**Returns:** the color, use `.x`, `.y`, `.z`, `.w` for r, g, b, a

```python
color = imgui.get_style_color_vec4(imgui.Col_.text)

imgui.text(f"text color: {color.x:.2f}, {color.y:.2f}, {color.z:.2f}")
```

### get_color_u32

**Overloads**

```python
imgui.get_color_u32(idx: int, alpha_mul: float = 1.0) -> int
```

retrieve given style color with style alpha applied and optional extra alpha multiplier, packed as a 32-bit value suitable for ImDrawList

```python
imgui.get_color_u32(col: ImVec4) -> int
```

retrieve given color with style alpha applied, packed as a 32-bit value suitable for ImDrawList

```python
imgui.get_color_u32(col: int, alpha_mul: float = 1.0) -> int
```

retrieve given color with style alpha applied, packed as a 32-bit value suitable for ImDrawList

A draw list takes a packed 32-bit color, not a tuple. `get_color_u32` packs a style color or your own color and
applies the global style alpha, `color_convert_float4_to_u32` packs a color as it is.

**Parameters**

- `idx` - which style color, an `imgui.Col_` value
- `col` - a color, `(r, g, b, a)` or a packed `int`
- `alpha_mul` - multiplies the alpha
- `in_` - the color to pack, `(r, g, b, a)`

**Returns:** the packed color

```python
draw_list = imgui.get_window_draw_list()
position = imgui.get_cursor_screen_pos()

draw_list.add_rect_filled(
    position, (position.x + 60, position.y + 20), imgui.get_color_u32(imgui.Col_.button)
)
draw_list.add_rect_filled(
    (position.x + 70, position.y),
    (position.x + 130, position.y + 20),
    imgui.color_convert_float4_to_u32((1.0, 0.8, 0.2, 1.0)),
)

imgui.dummy((130, 20))
```

### color_convert_float4_to_u32

```python
imgui.color_convert_float4_to_u32(in_: ImVec4) -> int
```

Packs a color as it is, without applying the style alpha.

**Parameters**

- `in_` - the color to pack, `(r, g, b, a)`

**Returns:** the packed color

### get_font_size

```python
imgui.get_font_size() -> float
```

get current scaled font size (= height in pixels). AFTER global scale factors applied. \*IMPORTANT\* DO NOT PASS THIS VALUE TO PushFont()! Use ImGui::GetStyle().FontSizeBase to get value before global scale factors.

**Parameters**

none

**Returns:** the height of the font in pixels

```python
imgui.text(f"font size: {imgui.get_font_size():.0f} px")
```

### begin_disabled

```python
imgui.begin_disabled(disabled: bool = True) -> None
```

Everything between them is greyed out and takes no input, for a control that does not apply yet.

**Parameters**

- `disabled` - pass `False` to leave the elements enabled, so the call can be made unconditionally

```python
apply_filter, sigma = False, 1.4

changed, apply_filter = imgui.checkbox("gaussian filter", apply_filter)

imgui.begin_disabled(not apply_filter)
changed, sigma = imgui.slider_float("sigma", v=sigma, v_min=0.1, v_max=10.0)
imgui.end_disabled()
```

### end_disabled

```python
imgui.end_disabled() -> None
```

Ends the block that `begin_disabled` started.

**Parameters**

none

## Queries

These ask about the element that was drawn last, about the window, or about the mouse and keyboard. The item queries
refer to the element immediately above them, so they go straight after the element they ask about.

The examples below print what they return, and the images were captured with the pointer over the element or a button
held down, which is why they read `True`.

### is_item_hovered

```python
imgui.is_item_hovered(flags: int = 0) -> bool
```

is the last item hovered? (and usable, aka not blocked by a popup, etc.). See ImGuiHoveredFlags for more options.

`flags` takes `imgui.HoveredFlags_`

```python
imgui.button("autoscale")
imgui.text(f"hovered: {imgui.is_item_hovered()}")
```

### is_item_active

```python
imgui.is_item_active() -> bool
```

is the last item active? (e.g. button being held, text field being edited. This will continuously return True while holding mouse button on an item. Items that don't interact will always return False)

```python
imgui.button("autoscale")
imgui.text(f"active: {imgui.is_item_active()}")
```

### is_item_clicked

```python
imgui.is_item_clicked(mouse_button: int = 0) -> bool
```

is the last item hovered and mouse clicked on? (\*\*)  == IsMouseClicked(mouse_button) && IsItemHovered()Important. (\*\*) this is NOT equivalent to the behavior of e.g. Button(). Read comments in function definition.

**Parameters**

- `mouse_button` - `0` left, `1` right, `2` middle

```python
imgui.button("autoscale")
imgui.text(f"clicked: {imgui.is_item_clicked()}")
```

### is_item_edited

```python
imgui.is_item_edited() -> bool
```

did the last item modify its underlying value this frame? or was pressed? This is generally the same as the "bool" return value of many widgets.

`is_item_deactivated_after_edit` is the one to use for work that is too expensive to run while a slider is being
dragged, since it is `True` only on the frame the drag ends.

```python
sigma = 1.4

changed, sigma = imgui.slider_float("sigma", v=sigma, v_min=0.1, v_max=10.0)

imgui.text(f"edited: {imgui.is_item_edited()}")
imgui.text(f"activated: {imgui.is_item_activated()}")
imgui.text(f"finished: {imgui.is_item_deactivated_after_edit()}")
```

### is_item_activated

```python
imgui.is_item_activated() -> bool
```

was the last item just made active (item was previously inactive).

`True` on the frame the element became active, e.g. the frame a drag started.

**Parameters**

none

```python
sigma = 1.4

changed, sigma = imgui.slider_float("sigma", v=sigma, v_min=0.1, v_max=10.0)
imgui.text(f"activated: {imgui.is_item_activated()}")
```

### is_item_deactivated_after_edit

```python
imgui.is_item_deactivated_after_edit() -> bool
```

was the last item just made inactive and made a value change when it was active? (e.g. Slider/Drag moved). Useful for Undo/Redo patterns with widgets that require continuous editing. Note that you may get False positives (some widgets such as Combo()/ListBox()/Selectable() will return True even when clicking an already selected item).

`True` only on the frame an edit ends, which is what to use for work that is too expensive to run while a
slider is being dragged.

**Parameters**

none

```python
sigma = 1.4

changed, sigma = imgui.slider_float("sigma", v=sigma, v_min=0.1, v_max=10.0)
imgui.text(f"finished: {imgui.is_item_deactivated_after_edit()}")
```

### is_any_item_hovered

```python
imgui.is_any_item_hovered() -> bool
```

is any item hovered?

```python
imgui.button("autoscale")
imgui.button("center")

imgui.text(f"any hovered: {imgui.is_any_item_hovered()}")
```

### is_window_hovered

```python
imgui.is_window_hovered(flags: int = 0) -> bool
```

is current window hovered and hoverable (e.g. not blocked by a popup/modal)? See ImGuiHoveredFlags_ for options. IMPORTANT: If you are trying to check whether your mouse should be dispatched to Dear ImGui or to your underlying app, you should not use this function! Use the 'io.WantCaptureMouse' boolean for that! Refer to FAQ entry "How can I tell whether to dispatch mouse/keyboard to Dear ImGui or my application?" for details.

`flags` takes `imgui.HoveredFlags_`

```python
imgui.text(f"window hovered: {imgui.is_window_hovered()}")
```

### is_window_focused

```python
imgui.is_window_focused(flags: int = 0) -> bool
```

is current window focused? or its root/child, depending on flags. see flags for options.

`flags` takes `imgui.FocusedFlags_`

```python
imgui.text(f"window focused: {imgui.is_window_focused()}")
```

### is_window_appearing

```python
imgui.is_window_appearing() -> bool
```

`True` on the first frame the window is drawn, for setup that should happen once, such as sizing a table column.

**Parameters**

none

```python
imgui.text(f"appearing: {imgui.is_window_appearing()}")
```

### is_mouse_down

```python
imgui.is_mouse_down(button: int) -> bool
```

is mouse button held?

These ask about the mouse anywhere, not about an element. A right-click that should open something belongs in
`begin_popup_context_item` instead.

**Parameters**

- `button` - `0` left, `1` right, `2` middle
- `repeat` - report repeats while the button is held

```python
imgui.text(f"left down: {imgui.is_mouse_down(0)}")
imgui.text(f"left clicked: {imgui.is_mouse_clicked(0)}")
imgui.text(f"right down: {imgui.is_mouse_down(1)}")
```

### is_mouse_clicked

```python
imgui.is_mouse_clicked(button: int, repeat: bool = False) -> bool
```

did mouse button clicked? (went from !Down to Down). Same as GetMouseClickedCount() == 1.

`True` on the frame the button goes down.

**Parameters**

- `button` - `0` left, `1` right, `2` middle
- `repeat` - report repeats while the button is held

```python
imgui.text(f"left clicked: {imgui.is_mouse_clicked(0)}")
```

### is_mouse_released

```python
imgui.is_mouse_released(button: int) -> bool
```

did mouse button released? (went from Down to !Down)

`True` on the frame the button goes up.

**Parameters**

- `button` - `0` left, `1` right, `2` middle

```python
imgui.text(f"left released: {imgui.is_mouse_released(0)}")
```

### is_mouse_double_clicked

```python
imgui.is_mouse_double_clicked(button: int) -> bool
```

did mouse button double-clicked? Same as GetMouseClickedCount() == 2. (note that a double-click will also report IsMouseClicked() == True)

`True` on the frame of the second click of a double click.

**Parameters**

- `button` - `0` left, `1` right, `2` middle

```python
imgui.text(f"double clicked: {imgui.is_mouse_double_clicked(0)}")
```

### is_mouse_dragging

```python
imgui.is_mouse_dragging(button: int, lock_threshold: float = -1.0) -> bool
```

is mouse dragging? (uses io.MouseDraggingThreshold if lock_threshold < 0.0)

The delta is measured from where the button went down. Reset it each frame to get the movement since the last frame,
which is what a drag handle needs.

**Parameters**

- `button` - `0` left, `1` right, `2` middle
- `lock_threshold` - how far the pointer must move before it counts as a drag, the default uses the style threshold

```python
delta = imgui.get_mouse_drag_delta(0)

imgui.text(f"dragging: {imgui.is_mouse_dragging(0)}")
imgui.text(f"delta: {delta.x:.0f}, {delta.y:.0f}")
```

### get_mouse_drag_delta

```python
imgui.get_mouse_drag_delta(button: int = 0, lock_threshold: float = -1.0) -> ImVec2
```

return the delta from the initial clicking position while the mouse button is pressed or was just released. This is locked and return 0.0 until the mouse moves past a distance threshold at least once (uses io.MouseDraggingThreshold if lock_threshold < 0.0)

The movement since the button went down, in pixels.

**Parameters**

- `button` - `0` left, `1` right, `2` middle
- `lock_threshold` - how far the pointer must move before it counts as a drag

**Returns:** the delta, use `.x` and `.y`

```python
delta = imgui.get_mouse_drag_delta(0)

imgui.text(f"delta: {delta.x:.0f}, {delta.y:.0f}")
```

### reset_mouse_drag_delta

```python
imgui.reset_mouse_drag_delta(button: int = 0) -> None
```

Sets the delta back to zero, call it each frame to get the movement since the last frame rather than since the
button went down.

**Parameters**

- `button` - `0` left, `1` right, `2` middle

### get_mouse_pos

```python
imgui.get_mouse_pos() -> ImVec2
```

shortcut to ImGui::GetIO().MousePos provided by user, to be consistent with other calls

**Parameters**

none

**Returns:** the pointer position in canvas coordinates, use `.x` and `.y`

```python
position = imgui.get_mouse_pos()

imgui.text(f"pointer: {position.x:.0f}, {position.y:.0f}")
```

### is_key_pressed

```python
imgui.is_key_pressed(key: Key, repeat: bool = True) -> bool
```

was key pressed (went from !Down to Down)? Repeat rate uses io.KeyRepeatDelay / KeyRepeatRate.

**Parameters**

- `key` - an `imgui.Key` member, e.g. `imgui.Key.right_arrow`
- `repeat` - report repeats while the key is held

```python
index = 42

if imgui.is_key_pressed(imgui.Key.right_arrow):
    index += 1

if imgui.is_key_pressed(imgui.Key.left_arrow):
    index -= 1

imgui.text(f"index: {index}")
```

### is_key_down

```python
imgui.is_key_down(key: Key) -> bool
```

is key being held.

`True` while the key is held, rather than only on the frame it goes down.

**Parameters**

- `key` - an `imgui.Key` member

```python
imgui.text(f"shift held: {imgui.is_key_down(imgui.Key.left_shift)}")
```

### get_io

```python
imgui.get_io() -> IO
```

access the ImGuiIO structure (mouse/keyboard/gamepad inputs, time, various configuration options/flags)

The imgui io structure. `want_capture_mouse` is the field to know about: it is `True` while imgui is using the
pointer, and fastplotlib relies on it to keep clicks on a UI from reaching the plot.

**Parameters**

none

**Returns:** an `imgui.IO`

```python
io = imgui.get_io()

imgui.text(f"framerate: {io.framerate:.0f}")
imgui.text(f"capture mouse: {io.want_capture_mouse}")
```

## Plots

These draw a small line plot or histogram from an array of values, for a preview next to the controls. They are not a
plotting library, a fastplotlib subplot is.

`values` must be a contiguous `float32` array.

### plot_lines

```python
imgui.plot_lines(label: str, values: numpy.ndarray, values_offset: int = 0, overlay_text: str | None = None, scale_min: float = 3.4028234663852886e+38, scale_max: float = 3.4028234663852886e+38, graph_size: ImVec2 | None = None, stride: int = -1) -> None
```

> **Note:** If graph_size is None, then its default value will be: ImVec2(0, 0)

**Parameters**

- `label` - drawn to the right of the plot, `"##hidden"` suppresses it
- `values` - the values to plot
- `values_offset` - index to start from, for a ring buffer
- `overlay_text` - text drawn over the plot
- `scale_min`, `scale_max` - the y range, the default fits the values
- `graph_size` - `(width, height)`, a zero component is a default size
- `stride` - byte stride between values, for a column of a 2d array

```python
values = np.sin(np.linspace(0, 4 * np.pi, 100)).astype(np.float32)

imgui.plot_lines("##trace", values, graph_size=(220, 60), overlay_text="channel 0")
```

### plot_histogram

```python
imgui.plot_histogram(label: str, values: numpy.ndarray, values_offset: int = 0, overlay_text: str | None = None, scale_min: float = 3.4028234663852886e+38, scale_max: float = 3.4028234663852886e+38, graph_size: ImVec2 | None = None, stride: int = -1) -> None
```

> **Note:** If graph_size is None, then its default value will be: ImVec2(0, 0)

**Parameters**

- `label` - drawn to the right of the plot
- `values` - the bin counts
- `values_offset` - index to start from
- `overlay_text` - text drawn over the plot
- `scale_min`, `scale_max` - the y range, the default fits the values
- `graph_size` - `(width, height)`, a zero component is a default size
- `stride` - byte stride between values

```python
data = np.random.normal(loc=120, scale=30, size=100_000)
counts = np.histogram(data, bins=64)[0].astype(np.float32)

imgui.plot_histogram("##histogram", counts, graph_size=(220, 60))
```

### image

```python
imgui.image(tex_ref: ImTextureRef, image_size: ImVec2, uv0: ImVec2 | None = None, uv1: ImVec2 | None = None) -> None
```

\* uv0: ImVec2(0, 0) \* uv1: ImVec2(1, 1)

> **Note:** If any of the params below is None, then its default value below will be used:

Draws a texture that you have uploaded to the GPU and registered with the imgui renderer, which is how
`ImguiColorbar` draws its colormap bar. There is no example here because the texture has to come from the wgpu
device of the Figure:

```python
texture_ref = figure.imgui_renderer.backend.register_texture(texture.create_view())
imgui.image(texture_ref, (24, 200))
```

**Parameters**

- `tex_ref` - an `imgui.ImTextureRef` from `register_texture`
- `image_size` - `(width, height)` to draw it at
- `uv0`, `uv1` - the region of the texture to draw, `(0, 0)` to `(1, 1)` by default

### image_button

```python
imgui.image_button(str_id: str, tex_ref: ImTextureRef, image_size: ImVec2, uv0: ImVec2 | None = None, uv1: ImVec2 | None = None, bg_col: ImVec4 | None = None, tint_col: ImVec4 | None = None) -> bool
```

\* uv0: ImVec2(0, 0) \* uv1: ImVec2(1, 1) \* bg_col: ImVec4(0, 0, 0, 0) \* tint_col: ImVec4(1, 1, 1, 1)

> **Note:** If any of the params below is None, then its default value below will be used:

`image` that responds to a click.

**Parameters**

- `str_id` - identifies the button
- `tex_ref` - an `imgui.ImTextureRef` from `register_texture`
- `image_size` - `(width, height)` to draw it at
- `uv0`, `uv1` - the region of the texture to draw
- `bg_col`, `tint_col` - background drawn behind the image, and a color the image is multiplied by

**Returns:** `True` on the frame the button is clicked

## Tables

A table is opened with `begin_table`, and `end_table` is called only when it returned `True`. Cells are filled by
walking rows and columns, either with `table_next_column` or by setting the column index.

### begin_table

```python
imgui.begin_table(str_id: str, columns: int, flags: int = 0, outer_size: ImVec2 | None = None, inner_width: float = 0.0) -> bool
```

`flags` takes `imgui.TableFlags_`

> **Note:** If outer_size is None, then its default value will be: ImVec2(0.0, 0.0)

**Parameters**

- `str_id` - identifies the table
- `columns` - how many columns
- `outer_size` - `(width, height)` of the table, a zero height fits the rows
- `inner_width` - width of the scrolling region when the table scrolls horizontally

```python
graphics = [("line-1", "LineGraphic", True), ("image-1", "ImageGraphic", False)]

if imgui.begin_table("graphics", 3, flags=imgui.TableFlags_.borders):
    for name, kind, visible in graphics:
        imgui.table_next_row()

        imgui.table_next_column()
        imgui.text(name)

        imgui.table_next_column()
        imgui.text(kind)

        imgui.table_next_column()
        imgui.text("visible" if visible else "hidden")

    imgui.end_table()
```

### end_table

```python
imgui.end_table() -> None
```

only call EndTable() if BeginTable() returns True!

Call it only when the matching `begin_table` returned `True`.

**Parameters**

none

### table_next_row

```python
imgui.table_next_row(row_flags: int = 0, min_row_height: float = 0.0) -> None
```

append into the first cell of a new row. 'min_row_height' include the minimum top and bottom padding aka CellPadding.y \* 2.0.

`row_flags` takes `imgui.TableRowFlags_`

**Parameters**

- `min_row_height` - minimum height of the row in pixels

```python
if imgui.begin_table("frames", 2, flags=imgui.TableFlags_.borders):
    for index in range(3):
        imgui.table_next_row(min_row_height=24)

        imgui.table_next_column()
        imgui.text(f"frame {index}")

        imgui.table_next_column()
        imgui.text(f"{index * 40} ms")

    imgui.end_table()
```

### table_next_column

```python
imgui.table_next_column() -> bool
```

append into the next column (or first column of next row if currently in last column). Return True when column is visible.

`table_next_column` moves to the next cell, wrapping to the first column of the next row. Use
`table_set_column_index` to fill cells out of order.

**Parameters**

- `column_n` - the column to move to

**Returns:** `True` when the column is visible, a clipped or hidden column can be skipped

```python
if imgui.begin_table("stats", 2, flags=imgui.TableFlags_.borders):
    for label, value in [("vmin", "12"), ("vmax", "208")]:
        imgui.table_next_row()

        imgui.table_set_column_index(0)
        imgui.text(label)

        imgui.table_set_column_index(1)
        imgui.text(value)

    imgui.end_table()
```

### table_set_column_index

```python
imgui.table_set_column_index(column_n: int) -> bool
```

append into the specified column. Return True when column is visible.

Fills a cell out of order, rather than moving to the next one.

**Parameters**

- `column_n` - the column to move to

**Returns:** `True` when the column is visible

### table_setup_column

```python
imgui.table_setup_column(label: str, flags: int = 0, init_width_or_weight: float = 0.0, user_id: int = 0) -> None
```

`flags` takes `imgui.TableColumnFlags_`

Declare the columns before any row, then `table_headers_row` draws one row with their labels.

**Parameters**

- `label` - the column header
- `init_width_or_weight` - a starting width in pixels, or a share of the table width for a stretched column.
  imgui rejects it unless the sizing policy is explicit, so pass `imgui.TableColumnFlags_.width_fixed` or
  `width_stretch` with it
- `user_id` - an id you can read back when sorting

```python
if imgui.begin_table("graphics", 2, flags=imgui.TableFlags_.borders):
    imgui.table_setup_column("name", flags=imgui.TableColumnFlags_.width_fixed, init_width_or_weight=90)
    imgui.table_setup_column("type")
    imgui.table_headers_row()

    for name, kind in [("line-1", "LineGraphic"), ("image-1", "ImageGraphic")]:
        imgui.table_next_row()

        imgui.table_next_column()
        imgui.text(name)

        imgui.table_next_column()
        imgui.text(kind)

    imgui.end_table()
```

### table_headers_row

```python
imgui.table_headers_row() -> None
```

submit a row with headers cells based on data provided to TableSetupColumn() + submit context menu

Draws one row of headers from the labels given to `table_setup_column`.

**Parameters**

none

## Flags

Flags are passed as `int`. The values are `enum.IntFlag` members of the classes below and can be
combined with `|`:

```python
imgui.slider_float(
    "gamma", v=gamma, v_min=0.1, v_max=5.0,
    flags=imgui.SliderFlags_.logarithmic | imgui.SliderFlags_.no_input,
)
```

`Col_`, `Cond_`, `StyleVar_` hold single values rather than flags, they are listed here because the
elements take them.

### imgui.ButtonFlags_

Flags for InvisibleButton() [extended in imgui_internal.h]

| member | description |
|---|---|
| `ButtonFlags_.none` |  |
| `ButtonFlags_.mouse_button_left` | React on left mouse button (default) |
| `ButtonFlags_.mouse_button_right` | React on right mouse button |
| `ButtonFlags_.mouse_button_middle` | React on center mouse button |
| `ButtonFlags_.mouse_button_mask_` | [Internal] |
| `ButtonFlags_.enable_nav` | InvisibleButton(): do not disable navigation/tabbing. Otherwise disabled by default. |
| `ButtonFlags_.allow_overlap` | Hit testing will allow subsequent widgets to overlap this one. Require previous frame HoveredId to match before being usable. Shortcut to calling SetNextItemAllowOverlap(). |

### imgui.ChildFlags_

Flags for ImGui::BeginChild()

| member | description |
|---|---|
| `ChildFlags_.none` |  |
| `ChildFlags_.borders` | Show an outer border and enable WindowPadding. (IMPORTANT: this is always == 1 == True for legacy reason) |
| `ChildFlags_.always_use_window_padding` | Pad with style.WindowPadding even if no border are drawn (no padding by default for non-bordered child windows because it makes more sense) |
| `ChildFlags_.resize_x` | Allow resize from right border (layout direction). Enable .ini saving (unless ImGuiWindowFlags_NoSavedSettings passed to window flags) |
| `ChildFlags_.resize_y` | Allow resize from bottom border (layout direction). " |
| `ChildFlags_.auto_resize_x` | Enable auto-resizing width. Read "IMPORTANT: Size measurement" details above. |
| `ChildFlags_.auto_resize_y` | Enable auto-resizing height. Read "IMPORTANT: Size measurement" details above. |
| `ChildFlags_.always_auto_resize` | Combined with AutoResizeX/AutoResizeY. Always measure size even when child is hidden, always return True, always disable clipping optimization! NOT RECOMMENDED. |
| `ChildFlags_.frame_style` | Style the child window like a framed item: use FrameBg, FrameRounding, FrameBorderSize, FramePadding instead of ChildBg, ChildRounding, ChildBorderSize, WindowPadding. |
| `ChildFlags_.nav_flattened` | [BETA] Share focus scope, allow keyboard/gamepad navigation to cross over parent border to this child or between sibling child windows. |

### imgui.Col_

Enumeration for PushStyleColor() / PopStyleColor()

| member | description |
|---|---|
| `Col_.text` |  |
| `Col_.text_disabled` |  |
| `Col_.window_bg` | Background of normal windows |
| `Col_.child_bg` | Background of child windows |
| `Col_.popup_bg` | Background of popups, menus, tooltips windows |
| `Col_.border` |  |
| `Col_.border_shadow` |  |
| `Col_.frame_bg` | Background of checkbox, radio button, plot, slider, text input |
| `Col_.frame_bg_hovered` |  |
| `Col_.frame_bg_active` |  |
| `Col_.title_bg` | Title bar |
| `Col_.title_bg_active` | Title bar when focused |
| `Col_.title_bg_collapsed` | Title bar when collapsed |
| `Col_.menu_bar_bg` |  |
| `Col_.scrollbar_bg` |  |
| `Col_.scrollbar_grab` |  |
| `Col_.scrollbar_grab_hovered` |  |
| `Col_.scrollbar_grab_active` |  |
| `Col_.check_mark` | Checkbox tick and RadioButton circle |
| `Col_.checkbox_selected_bg` | Checkbox background when Selected, otherwise use FrameBg |
| `Col_.slider_grab` |  |
| `Col_.slider_grab_active` |  |
| `Col_.button` |  |
| `Col_.button_hovered` |  |
| `Col_.button_active` |  |
| `Col_.header` | Header\* colors are used for CollapsingHeader, TreeNode, Selectable, MenuItem |
| `Col_.header_hovered` |  |
| `Col_.header_active` |  |
| `Col_.separator` |  |
| `Col_.separator_hovered` |  |
| `Col_.separator_active` |  |
| `Col_.resize_grip` | Resize grip in lower-right and lower-left corners of windows. |
| `Col_.resize_grip_hovered` |  |
| `Col_.resize_grip_active` |  |
| `Col_.input_text_cursor` | InputText cursor/caret |
| `Col_.tab_hovered` | Tab background, when hovered |
| `Col_.tab` | Tab background, when tab-bar is focused & tab is unselected |
| `Col_.tab_selected` | Tab background, when tab-bar is focused & tab is selected |
| `Col_.tab_selected_overline` | Tab horizontal overline, when tab-bar is focused & tab is selected |
| `Col_.tab_dimmed` | Tab background, when tab-bar is unfocused & tab is unselected |
| `Col_.tab_dimmed_selected` | Tab background, when tab-bar is unfocused & tab is selected |
| `Col_.tab_dimmed_selected_overline` | ..horizontal overline, when tab-bar is unfocused & tab is selected |
| `Col_.docking_preview` | Preview overlay color when about to docking something |
| `Col_.docking_empty_bg` | Background color for empty node (e.g. CentralNode with no window docked into it) |
| `Col_.plot_lines` |  |
| `Col_.plot_lines_hovered` |  |
| `Col_.plot_histogram` |  |
| `Col_.plot_histogram_hovered` |  |
| `Col_.table_header_bg` | Table header background |
| `Col_.table_border_strong` | Table outer and header borders (prefer using Alpha=1.0 here) |
| `Col_.table_border_light` | Table inner borders (prefer using Alpha=1.0 here) |
| `Col_.table_row_bg` | Table row background (even rows) |
| `Col_.table_row_bg_alt` | Table row background (odd rows) |
| `Col_.text_link` | Hyperlink color |
| `Col_.text_selected_bg` | Selected text inside an InputText |
| `Col_.tree_lines` | Tree node hierarchy outlines when using ImGuiTreeNodeFlags_DrawLines |
| `Col_.drag_drop_target` | Rectangle border highlighting a drop target |
| `Col_.drag_drop_target_bg` | Rectangle background highlighting a drop target |
| `Col_.unsaved_marker` | Unsaved Document marker (in window title and tabs) |
| `Col_.nav_cursor` | Color of keyboard/gamepad navigation cursor/rectangle, when visible |
| `Col_.nav_windowing_highlight` | Highlight window when using Ctrl+Tab |
| `Col_.nav_windowing_dim_bg` | Darken/colorize entire screen behind the Ctrl+Tab window list, when active |
| `Col_.modal_window_dim_bg` | Darken/colorize entire screen behind a modal window, when one is active |
| `Col_.count` |  |

### imgui.ColorEditFlags_

Flags for ColorEdit3() / ColorEdit4() / ColorPicker3() / ColorPicker4() / ColorButton()

| member | description |
|---|---|
| `ColorEditFlags_.none` |  |
| `ColorEditFlags_.no_alpha` | // ColorEdit, ColorPicker, ColorButton: ignore Alpha component (will only read 3 components from the input pointer). |
| `ColorEditFlags_.no_picker` | // ColorEdit: disable picker when clicking on color square. |
| `ColorEditFlags_.no_options` | // ColorEdit: disable toggling options menu when right-clicking on inputs/small preview. |
| `ColorEditFlags_.no_small_preview` | // ColorEdit, ColorPicker: disable color square preview next to the inputs. (e.g. to show only the inputs) |
| `ColorEditFlags_.no_inputs` | // ColorEdit, ColorPicker: disable inputs sliders/text widgets (e.g. to show only the small preview color square). |
| `ColorEditFlags_.no_tooltip` | // ColorEdit, ColorPicker, ColorButton: disable tooltip when hovering the preview. |
| `ColorEditFlags_.no_label` | // ColorEdit, ColorPicker: disable display of inline text label (the label is still forwarded to the tooltip and picker). |
| `ColorEditFlags_.no_side_preview` | // ColorPicker: disable bigger color preview on right side of the picker, use small color square preview instead. |
| `ColorEditFlags_.no_drag_drop` | // ColorEdit: disable drag and drop target/source. ColorButton: disable drag and drop source. |
| `ColorEditFlags_.no_border` | // ColorButton: disable border (which is enforced by default) |
| `ColorEditFlags_.no_color_markers` | // ColorEdit: disable rendering R/G/B/A color marker. May also be disabled globally by setting style.ColorMarkerSize = 0. |
| `ColorEditFlags_.alpha_opaque` | // ColorEdit, ColorPicker, ColorButton: disable alpha in the preview,. Contrary to _NoAlpha it may still be edited when calling ColorEdit4()/ColorPicker4(). For ColorButton() this does the same as _NoAlpha. |
| `ColorEditFlags_.alpha_no_bg` | // ColorEdit, ColorPicker, ColorButton: disable rendering a checkerboard background behind transparent color. |
| `ColorEditFlags_.alpha_preview_half` | // ColorEdit, ColorPicker, ColorButton: display half opaque / half transparent preview. |
| `ColorEditFlags_.alpha_bar` | // ColorEdit, ColorPicker: show vertical alpha bar/gradient in picker. |
| `ColorEditFlags_.hdr` | // (WIP) ColorEdit: Currently only disable 0.0..1.0 limits in RGBA edition (note: you probably want to use ImGuiColorEditFlags_Float flag as well). |
| `ColorEditFlags_.display_rgb` | [Display]    // ColorEdit: override _display_ type among RGB/HSV/Hex. ColorPicker: select any combination using one or more of RGB/HSV/Hex. |
| `ColorEditFlags_.display_hsv` | [Display]    // " |
| `ColorEditFlags_.display_hex` | [Display]    // " |
| `ColorEditFlags_.uint8` | [DataType]   // ColorEdit, ColorPicker, ColorButton: _display_ values formatted as 0..255. |
| `ColorEditFlags_.float` | [DataType]   // ColorEdit, ColorPicker, ColorButton: _display_ values formatted as 0.0..1.0 floats instead of 0..255 integers. No round-trip of value via integers. |
| `ColorEditFlags_.picker_hue_bar` | [Picker]     // ColorPicker: bar for Hue, rectangle for Sat/Value. |
| `ColorEditFlags_.picker_hue_wheel` | [Picker]     // ColorPicker: wheel for Hue, triangle for Sat/Value. |
| `ColorEditFlags_.input_rgb` | [Input]      // ColorEdit, ColorPicker: input and output data in RGB format. |
| `ColorEditFlags_.input_hsv` | [Input]      // ColorEdit, ColorPicker: input and output data in HSV format. |
| `ColorEditFlags_.default_options_` |  |
| `ColorEditFlags_.alpha_mask_` |  |
| `ColorEditFlags_.display_mask_` |  |
| `ColorEditFlags_.data_type_mask_` |  |
| `ColorEditFlags_.picker_mask_` |  |
| `ColorEditFlags_.input_mask_` |  |

### imgui.ComboFlags_

Flags for ImGui::BeginCombo()

| member | description |
|---|---|
| `ComboFlags_.none` |  |
| `ComboFlags_.popup_align_left` | Align the popup toward the left by default |
| `ComboFlags_.height_small` | Max ~4 items visible. Tip: If you want your combo popup to be a specific size you can use SetNextWindowSizeConstraints() prior to calling BeginCombo() |
| `ComboFlags_.height_regular` | Max ~8 items visible (default) |
| `ComboFlags_.height_large` | Max ~20 items visible |
| `ComboFlags_.height_largest` | As many fitting items as possible |
| `ComboFlags_.no_arrow_button` | Display on the preview box without the square arrow button |
| `ComboFlags_.no_preview` | Display only a square arrow button |
| `ComboFlags_.width_fit_preview` | Width dynamically calculated from preview contents |
| `ComboFlags_.height_mask_` |  |

### imgui.Cond_

Enumeration for ImGui::SetNextWindow***(), SetWindow***(), SetNextItem***() functions

| member | description |
|---|---|
| `Cond_.none` | No condition (always set the variable), same as _Always |
| `Cond_.always` | No condition (always set the variable), same as _None |
| `Cond_.once` | Set the variable once per runtime session (only the first call will succeed) |
| `Cond_.first_use_ever` | Set the variable if the object/window has no persistently saved data (no entry in .ini file) |
| `Cond_.appearing` | Set the variable if the object/window is appearing after being hidden/inactive (or the first time) |

### imgui.FocusedFlags_

Flags for ImGui::IsWindowFocused()

| member | description |
|---|---|
| `FocusedFlags_.none` |  |
| `FocusedFlags_.child_windows` | Return True if any children of the window is focused |
| `FocusedFlags_.root_window` | Test from root window (top most parent of the current hierarchy) |
| `FocusedFlags_.any_window` | Return True if any window is focused. Important: If you are trying to tell how to dispatch your low-level inputs, do NOT use this. Use 'io.WantCaptureMouse' instead! Please read the FAQ! |
| `FocusedFlags_.no_popup_hierarchy` | Do not consider popup hierarchy (do not treat popup emitter as parent of popup) (when used with _ChildWindows or _RootWindow) |
| `FocusedFlags_.dock_hierarchy` | Consider docking hierarchy (treat dockspace host as parent of docked window) (when used with _ChildWindows or _RootWindow) |
| `FocusedFlags_.root_and_child_windows` |  |

### imgui.HoveredFlags_

Flags for ImGui::IsItemHovered(), ImGui::IsWindowHovered()

| member | description |
|---|---|
| `HoveredFlags_.none` | Return True if directly over the item/window, not obstructed by another window, not obstructed by an active popup or modal blocking inputs under them. |
| `HoveredFlags_.child_windows` | IsWindowHovered() only: Return True if any children of the window is hovered |
| `HoveredFlags_.root_window` | IsWindowHovered() only: Test from root window (top most parent of the current hierarchy) |
| `HoveredFlags_.any_window` | IsWindowHovered() only: Return True if any window is hovered |
| `HoveredFlags_.no_popup_hierarchy` | IsWindowHovered() only: Do not consider popup hierarchy (do not treat popup emitter as parent of popup) (when used with _ChildWindows or _RootWindow) |
| `HoveredFlags_.dock_hierarchy` | IsWindowHovered() only: Consider docking hierarchy (treat dockspace host as parent of docked window) (when used with _ChildWindows or _RootWindow) |
| `HoveredFlags_.allow_when_blocked_by_popup` | Return True even if a popup window is normally blocking access to this item/window |
| `HoveredFlags_.allow_when_blocked_by_active_item` | Return True even if an active item is blocking access to this item/window. Useful for Drag and Drop patterns. |
| `HoveredFlags_.allow_when_overlapped_by_item` | IsItemHovered() only: Return True even if the item uses AllowOverlap mode and is overlapped by another hoverable item. |
| `HoveredFlags_.allow_when_overlapped_by_window` | IsItemHovered() only: Return True even if the position is obstructed or overlapped by another window. |
| `HoveredFlags_.allow_when_disabled` | IsItemHovered() only: Return True even if the item is disabled |
| `HoveredFlags_.no_nav_override` | IsItemHovered() only: Disable using keyboard/gamepad navigation state when active, always query mouse |
| `HoveredFlags_.allow_when_overlapped` |  |
| `HoveredFlags_.rect_only` |  |
| `HoveredFlags_.root_and_child_windows` |  |
| `HoveredFlags_.for_tooltip` | Shortcut for standard flags when using IsItemHovered() + SetTooltip() sequence. |
| `HoveredFlags_.stationary` | Require mouse to be stationary for style.HoverStationaryDelay (~0.15 sec) _at least one time_. After this, can move on same item/window. Using the stationary test tends to reduces the need for a long delay. |
| `HoveredFlags_.delay_none` | IsItemHovered() only: Return True immediately (default). As this is the default you generally ignore this. |
| `HoveredFlags_.delay_short` | IsItemHovered() only: Return True after style.HoverDelayShort elapsed (~0.15 sec) (shared between items) + requires mouse to be stationary for style.HoverStationaryDelay (once per item). |
| `HoveredFlags_.delay_normal` | IsItemHovered() only: Return True after style.HoverDelayNormal elapsed (~0.40 sec) (shared between items) + requires mouse to be stationary for style.HoverStationaryDelay (once per item). |
| `HoveredFlags_.no_shared_delay` | IsItemHovered() only: Disable shared delay system where moving from one item to the next keeps the previous timer for a short time (standard for tooltips with long delays) |

### imgui.InputTextFlags_

Flags for ImGui::InputText()

| member | description |
|---|---|
| `InputTextFlags_.none` |  |
| `InputTextFlags_.chars_decimal` | Allow 0123456789.+-\*/ |
| `InputTextFlags_.chars_hexadecimal` | Allow 0123456789ABCDEFabcdef |
| `InputTextFlags_.chars_scientific` | Allow 0123456789.+-\*/eE (Scientific notation input) |
| `InputTextFlags_.chars_uppercase` | Turn a..z into A..Z |
| `InputTextFlags_.chars_no_blank` | Filter out spaces, tabs |
| `InputTextFlags_.allow_tab_input` | Pressing TAB input a '\\t' character into the text field |
| `InputTextFlags_.enter_returns_true` | Return 'True' when Enter is pressed (as opposed to every time the value was modified). Consider using IsItemDeactivatedAfterEdit() instead! |
| `InputTextFlags_.escape_clears_all` | Escape key clears content if not empty, and deactivate otherwise (contrast to default behavior of Escape to revert) |
| `InputTextFlags_.ctrl_enter_for_new_line` | In multi-line mode: validate with Enter, add new line with Ctrl+Enter (default is opposite: validate with Ctrl+Enter, add line with Enter). Note that Shift+Enter always enter a new line either way. |
| `InputTextFlags_.read_only` | Read-only mode |
| `InputTextFlags_.password` | Password mode, display all characters as '\*', disable copy |
| `InputTextFlags_.always_overwrite` | Overwrite mode |
| `InputTextFlags_.auto_select_all` | Select entire text when first taking mouse focus |
| `InputTextFlags_.parse_empty_ref_val` | InputFloat(), InputInt(), InputScalar() etc. only: parse empty string as zero value. |
| `InputTextFlags_.display_empty_ref_val` | InputFloat(), InputInt(), InputScalar() etc. only: when value is zero, do not display it. Generally used with ImGuiInputTextFlags_ParseEmptyRefVal. |
| `InputTextFlags_.no_horizontal_scroll` | Disable following the cursor horizontally |
| `InputTextFlags_.no_undo_redo` | Disable undo/redo. Note that input text owns the text data while active, if you want to provide your own undo/redo stack you need e.g. to call ClearActiveID(). |
| `InputTextFlags_.elide_left` | When text doesn't fit, elide left side to ensure right side stays visible. Useful for path/filenames. Single-line only! |
| `InputTextFlags_.callback_completion` | Callback on pressing TAB (for completion handling) |
| `InputTextFlags_.callback_history` | Callback on pressing Up/Down arrows (for history handling) |
| `InputTextFlags_.callback_always` | Callback on each iteration. User code may query cursor position, modify text buffer. |
| `InputTextFlags_.callback_char_filter` | Callback on character inputs to replace or discard them. Modify 'EventChar' to replace or discard, or return 1 in callback to discard. |
| `InputTextFlags_.callback_resize` | Callback on buffer capacity changes request (beyond 'buf_size' parameter value), allowing the string to grow. Notify when the string wants to be resized (for string types which hold a cache of their Size). You will be provided a new BufSize in the callback and NEED to honor it. (see misc/cpp/imgui_stdlib.h for an example of using this) |
| `InputTextFlags_.callback_edit` | Callback on any edit. Note that InputText() already returns True on edit + you can always use IsItemEdited(). The callback is useful to manipulate the underlying buffer while focus is active. |
| `InputTextFlags_.word_wrap` | InputTextMultiline(): word-wrap lines that are too long. |

### imgui.PopupFlags_

Flags for OpenPopup*(), BeginPopupContext*(), IsPopupOpen() functions.

| member | description |
|---|---|
| `PopupFlags_.none` |  |
| `PopupFlags_.mouse_button_left` | For BeginPopupContext\*(): open on Left Mouse release. Only one button allowed! |
| `PopupFlags_.mouse_button_right` | For BeginPopupContext\*(): open on Right Mouse release. Only one button allowed! (default) |
| `PopupFlags_.mouse_button_middle` | For BeginPopupContext\*(): open on Middle Mouse release. Only one button allowed! |
| `PopupFlags_.no_reopen` | For OpenPopup\*(), BeginPopupContext\*(): don't reopen same popup if already open (won't reposition, won't reinitialize navigation) |
| `PopupFlags_.no_open_over_existing_popup` | For OpenPopup\*(), BeginPopupContext\*(): don't open if there's already a popup at the same level of the popup stack |
| `PopupFlags_.no_open_over_items` | For BeginPopupContextWindow(): don't return True when hovering items, only when hovering empty space |
| `PopupFlags_.any_popup_id` | For IsPopupOpen(): ignore the ImGuiID parameter and test for any popup. |
| `PopupFlags_.any_popup_level` | For IsPopupOpen(): search/test at any level of the popup stack (default test in the current level) |
| `PopupFlags_.any_popup` |  |
| `PopupFlags_.mouse_button_shift_` | [Internal] |
| `PopupFlags_.mouse_button_mask_` | [Internal] |
| `PopupFlags_.invalid_mask_` | [Internal] Reserve legacy bits 0-1 to detect incorrectly passing 1 or 2 to the function. |

### imgui.SelectableFlags_

Flags for ImGui::Selectable()

| member | description |
|---|---|
| `SelectableFlags_.none` |  |
| `SelectableFlags_.no_auto_close_popups` | Clicking this doesn't close parent popup window (overrides ImGuiItemFlags_AutoClosePopups) |
| `SelectableFlags_.span_all_columns` | Frame will span all columns of its container table (text will still fit in current column) |
| `SelectableFlags_.allow_double_click` | Generate press events on double clicks too |
| `SelectableFlags_.disabled` | Cannot be selected, display grayed out text |
| `SelectableFlags_.allow_overlap` | Hit testing will allow subsequent widgets to overlap this one. Require previous frame HoveredId to match before being usable. Shortcut to calling SetNextItemAllowOverlap(). |
| `SelectableFlags_.highlight` | Make the item be displayed as if it is hovered |
| `SelectableFlags_.select_on_nav` | Auto-select when moved into, unless Ctrl is held. Automatic when in a BeginMultiSelect() block. |

### imgui.SliderFlags_

Flags for DragFloat(), DragInt(), SliderFloat(), SliderInt() etc.

| member | description |
|---|---|
| `SliderFlags_.none` |  |
| `SliderFlags_.logarithmic` | Make the widget logarithmic (linear otherwise). Consider using ImGuiSliderFlags_NoRoundToFormat with this if using a format-string with small amount of digits. |
| `SliderFlags_.no_round_to_format` | Disable rounding underlying value to match precision of the display format string (e.g. %.3 values are rounded to those 3 digits). |
| `SliderFlags_.no_input` | Disable Ctrl+Click or Enter key allowing to input text directly into the widget. |
| `SliderFlags_.wrap_around` | Enable wrapping around from max to min and from min to max. Only supported by DragXXX() functions for now. |
| `SliderFlags_.clamp_on_input` | Clamp value to min/max bounds when input manually with Ctrl+Click. By default Ctrl+Click allows going out of bounds. |
| `SliderFlags_.clamp_zero_range` | Clamp even if min==max==0.0. Otherwise due to legacy reason DragXXX functions don't clamp with those values. When your clamping limits are dynamic you almost always want to use it. |
| `SliderFlags_.no_speed_tweaks` | Disable keyboard modifiers altering tweak speed. Useful if you want to alter tweak speed yourself based on your own logic. |
| `SliderFlags_.color_markers` | DragScalarN(), SliderScalarN(): Draw R/G/B/A color markers on each component. |
| `SliderFlags_.always_clamp` |  |
| `SliderFlags_.invalid_mask_` | [Internal] We treat using those bits as being potentially a 'float power' argument from legacy API (obsoleted 2020-08) that has got miscast to this enum, and will trigger an assert if needed. |

### imgui.StyleVar_

Enumeration for PushStyleVar() / PopStyleVar() to temporarily modify the ImGuiStyle structure.

| member | description |
|---|---|
| `StyleVar_.alpha` | float     Alpha |
| `StyleVar_.disabled_alpha` | float     DisabledAlpha |
| `StyleVar_.window_padding` | ImVec2    WindowPadding |
| `StyleVar_.window_rounding` | float     WindowRounding |
| `StyleVar_.window_border_size` | float     WindowBorderSize |
| `StyleVar_.window_min_size` | ImVec2    WindowMinSize |
| `StyleVar_.window_title_align` | ImVec2    WindowTitleAlign |
| `StyleVar_.child_rounding` | float     ChildRounding |
| `StyleVar_.child_border_size` | float     ChildBorderSize |
| `StyleVar_.popup_rounding` | float     PopupRounding |
| `StyleVar_.popup_border_size` | float     PopupBorderSize |
| `StyleVar_.frame_padding` | ImVec2    FramePadding |
| `StyleVar_.frame_rounding` | float     FrameRounding |
| `StyleVar_.frame_border_size` | float     FrameBorderSize |
| `StyleVar_.item_spacing` | ImVec2    ItemSpacing |
| `StyleVar_.item_inner_spacing` | ImVec2    ItemInnerSpacing |
| `StyleVar_.indent_spacing` | float     IndentSpacing |
| `StyleVar_.cell_padding` | ImVec2    CellPadding |
| `StyleVar_.scrollbar_size` | float     ScrollbarSize |
| `StyleVar_.scrollbar_rounding` | float     ScrollbarRounding |
| `StyleVar_.scrollbar_padding` | float     ScrollbarPadding |
| `StyleVar_.grab_min_size` | float     GrabMinSize |
| `StyleVar_.grab_rounding` | float     GrabRounding |
| `StyleVar_.image_rounding` | float     ImageRounding |
| `StyleVar_.image_border_size` | float     ImageBorderSize |
| `StyleVar_.layout_align` | float     LayoutAlign |
| `StyleVar_.tab_rounding` | float     TabRounding |
| `StyleVar_.tab_border_size` | float     TabBorderSize |
| `StyleVar_.tab_min_width_base` | float     TabMinWidthBase |
| `StyleVar_.tab_min_width_shrink` | float     TabMinWidthShrink |
| `StyleVar_.tab_bar_border_size` | float     TabBarBorderSize |
| `StyleVar_.tab_bar_overline_size` | float     TabBarOverlineSize |
| `StyleVar_.table_angled_headers_angle` | float     TableAngledHeadersAngle |
| `StyleVar_.table_angled_headers_text_align` | ImVec2  TableAngledHeadersTextAlign |
| `StyleVar_.tree_lines_size` | float     TreeLinesSize |
| `StyleVar_.tree_lines_rounding` | float     TreeLinesRounding |
| `StyleVar_.drag_drop_target_rounding` | float     DragDropTargetRounding |
| `StyleVar_.button_text_align` | ImVec2    ButtonTextAlign |
| `StyleVar_.selectable_text_align` | ImVec2    SelectableTextAlign |
| `StyleVar_.separator_size` | float     SeparatorSize |
| `StyleVar_.separator_text_border_size` | float     SeparatorTextBorderSize |
| `StyleVar_.separator_text_align` | ImVec2    SeparatorTextAlign |
| `StyleVar_.separator_text_padding` | ImVec2    SeparatorTextPadding |
| `StyleVar_.docking_separator_size` | float     DockingSeparatorSize |
| `StyleVar_.count` |  |

### imgui.TabBarFlags_

Flags for ImGui::BeginTabBar()

| member | description |
|---|---|
| `TabBarFlags_.none` |  |
| `TabBarFlags_.reorderable` | Allow manually dragging tabs to re-order them + New tabs are appended at the end of list |
| `TabBarFlags_.auto_select_new_tabs` | Automatically select new tabs when they appear |
| `TabBarFlags_.tab_list_popup_button` | Disable buttons to open the tab list popup |
| `TabBarFlags_.no_close_with_middle_mouse_button` | Disable behavior of closing tabs (that are submitted with p_open != None) with middle mouse button. You may handle this behavior manually on user's side with if (IsItemHovered() && IsMouseClicked(2)) \*p_open = False. |
| `TabBarFlags_.no_tab_list_scrolling_buttons` | Disable scrolling buttons (apply when fitting policy is ImGuiTabBarFlags_FittingPolicyScroll) |
| `TabBarFlags_.no_tooltip` | Disable tooltips when hovering a tab |
| `TabBarFlags_.draw_selected_overline` | Draw selected overline markers over selected tab |
| `TabBarFlags_.fitting_policy_mixed` | Shrink down tabs when they don't fit, until width is style.TabMinWidthShrink, then enable scrolling. Setting TabMinWidthShrink to FLT_MAX makes this behave like ImGuiTabBarFlags_FittingPolicyScroll. |
| `TabBarFlags_.fitting_policy_shrink` | Shrink down tabs when they don't fit |
| `TabBarFlags_.fitting_policy_scroll` | Enable scrolling buttons when tabs don't fit |
| `TabBarFlags_.fitting_policy_mask_` |  |
| `TabBarFlags_.fitting_policy_default_` |  |

### imgui.TabItemFlags_

Flags for ImGui::BeginTabItem()

| member | description |
|---|---|
| `TabItemFlags_.none` |  |
| `TabItemFlags_.unsaved_document` | Display a dot next to the title + set ImGuiTabItemFlags_NoAssumedClosure. |
| `TabItemFlags_.set_selected` | Trigger flag to programmatically make the tab selected when calling BeginTabItem() |
| `TabItemFlags_.no_close_with_middle_mouse_button` | Disable behavior of closing tabs (that are submitted with p_open != None) with middle mouse button. You may handle this behavior manually on user's side with if (IsItemHovered() && IsMouseClicked(2)) \*p_open = False. |
| `TabItemFlags_.no_push_id` | Don't call PushID()/PopID() on BeginTabItem()/EndTabItem() |
| `TabItemFlags_.no_tooltip` | Disable tooltip for the given tab |
| `TabItemFlags_.no_reorder` | Disable reordering this tab or having another tab cross over this tab |
| `TabItemFlags_.leading` | Enforce the tab position to the left of the tab bar (after the tab list popup button) |
| `TabItemFlags_.trailing` | Enforce the tab position to the right of the tab bar (before the scrolling buttons) |
| `TabItemFlags_.no_assumed_closure` | Tab is selected when trying to close + closure is not immediately assumed (will wait for user to stop submitting the tab). Otherwise closure is assumed when pressing the X, so if you keep submitting the tab may reappear at end of tab bar. |

### imgui.TableColumnFlags_

Flags for ImGui::TableSetupColumn()

| member | description |
|---|---|
| `TableColumnFlags_.none` |  |
| `TableColumnFlags_.disabled` | Overriding/master disable flag: hide column, won't show in context menu (unlike calling TableSetColumnEnabled() which manipulates the user accessible state) |
| `TableColumnFlags_.default_hide` | Default as a hidden/disabled column. |
| `TableColumnFlags_.default_sort` | Default as a sorting column. |
| `TableColumnFlags_.width_stretch` | Column will stretch. Preferable with horizontal scrolling disabled (default if table sizing policy is _SizingStretchSame or _SizingStretchProp). |
| `TableColumnFlags_.width_fixed` | Column will not stretch. Preferable with horizontal scrolling enabled (default if table sizing policy is _SizingFixedFit and table is resizable). |
| `TableColumnFlags_.no_resize` | Disable manual resizing. |
| `TableColumnFlags_.no_reorder` | Disable manual reordering this column, this will also prevent other columns from crossing over this column. |
| `TableColumnFlags_.no_hide` | Disable ability to hide/disable this column. |
| `TableColumnFlags_.no_clip` | Disable clipping for this column (all NoClip columns will render in a same draw command). |
| `TableColumnFlags_.no_sort` | Disable ability to sort on this field (even if ImGuiTableFlags_Sortable is set on the table). |
| `TableColumnFlags_.no_sort_ascending` | Disable ability to sort in the ascending direction. |
| `TableColumnFlags_.no_sort_descending` | Disable ability to sort in the descending direction. |
| `TableColumnFlags_.no_header_label` | TableHeadersRow() will submit an empty label for this column. Convenient for some small columns. Name will still appear in context menu or in angled headers. You may append into this cell by calling TableSetColumnIndex() right after the TableHeadersRow() call. |
| `TableColumnFlags_.no_header_width` | Disable header text width contribution to automatic column width. |
| `TableColumnFlags_.prefer_sort_ascending` | Make the initial sort direction Ascending when first sorting on this column (default). |
| `TableColumnFlags_.prefer_sort_descending` | Make the initial sort direction Descending when first sorting on this column. |
| `TableColumnFlags_.indent_enable` | Use current Indent value when entering cell (default for column 0). |
| `TableColumnFlags_.indent_disable` | Ignore current Indent value when entering cell (default for columns > 0). Indentation changes _within_ the cell will still be honored. |
| `TableColumnFlags_.angled_header` | TableHeadersRow() will submit an angled header row for this column. Note this will add an extra row. |
| `TableColumnFlags_.is_enabled` | Status: is enabled == not hidden by user/api (referred to as "Hide" in _DefaultHide and _NoHide) flags. |
| `TableColumnFlags_.is_visible` | Status: is visible == is enabled AND not clipped by scrolling. |
| `TableColumnFlags_.is_sorted` | Status: is currently part of the sort specs |
| `TableColumnFlags_.is_hovered` | Status: is hovered by mouse |
| `TableColumnFlags_.width_mask_` |  |
| `TableColumnFlags_.indent_mask_` |  |
| `TableColumnFlags_.status_mask_` |  |
| `TableColumnFlags_.no_direct_resize_` | [Internal] Disable user resizing this column directly (it may however we resized indirectly from its left edge) |

### imgui.TableFlags_

Flags for ImGui::BeginTable()

| member | description |
|---|---|
| `TableFlags_.none` |  |
| `TableFlags_.resizable` | Enable resizing columns. |
| `TableFlags_.reorderable` | Enable reordering columns in header row. (Need calling TableSetupColumn() + TableHeadersRow() to display headers, or using ImGuiTableFlags_ContextMenuInBody to access context-menu without headers). |
| `TableFlags_.hideable` | Enable hiding/disabling columns in context menu. |
| `TableFlags_.sortable` | Enable sorting. Call TableGetSortSpecs() to obtain sort specs. Also see ImGuiTableFlags_SortMulti and ImGuiTableFlags_SortTristate. |
| `TableFlags_.no_saved_settings` | Disable persisting columns order, width, visibility and sort settings in the .ini file. |
| `TableFlags_.context_menu_in_body` | Right-click on columns body/contents will also display table context menu. By default it is available in TableHeadersRow(). |
| `TableFlags_.row_bg` | Set each RowBg color with ImGuiCol_TableRowBg or ImGuiCol_TableRowBgAlt (equivalent of calling TableSetBgColor with ImGuiTableBgFlags_RowBg0 on each row manually) |
| `TableFlags_.borders_inner_h` | Draw horizontal borders between rows. |
| `TableFlags_.borders_outer_h` | Draw horizontal borders at the top and bottom. |
| `TableFlags_.borders_inner_v` | Draw vertical borders between columns. |
| `TableFlags_.borders_outer_v` | Draw vertical borders on the left and right sides. |
| `TableFlags_.borders_h` | Draw horizontal borders. |
| `TableFlags_.borders_v` | Draw vertical borders. |
| `TableFlags_.borders_inner` | Draw inner borders. |
| `TableFlags_.borders_outer` | Draw outer borders. |
| `TableFlags_.borders` | Draw all borders. |
| `TableFlags_.no_borders_in_body` | [ALPHA] Disable vertical borders in columns Body (borders will always appear in Headers). -> May move to style |
| `TableFlags_.no_borders_in_body_until_resize` | [ALPHA] Disable vertical borders in columns Body until hovered for resize (borders will always appear in Headers). -> May move to style |
| `TableFlags_.sizing_fixed_fit` | Columns default to _WidthFixed or _WidthAuto (if resizable or not resizable), matching contents width. |
| `TableFlags_.sizing_fixed_same` | Columns default to _WidthFixed or _WidthAuto (if resizable or not resizable), matching the maximum contents width of all columns. Implicitly enable ImGuiTableFlags_NoKeepColumnsVisible. |
| `TableFlags_.sizing_stretch_prop` | Columns default to _WidthStretch with default weights proportional to each columns contents widths. |
| `TableFlags_.sizing_stretch_same` | Columns default to _WidthStretch with default weights all equal, unless overridden by TableSetupColumn(). |
| `TableFlags_.no_host_extend_x` | Make outer width auto-fit to columns, overriding outer_size.x value. Only available when ScrollX/ScrollY are disabled and Stretch columns are not used. |
| `TableFlags_.no_host_extend_y` | Make outer height stop exactly at outer_size.y (prevent auto-extending table past the limit). Only available when ScrollX/ScrollY are disabled. Data below the limit will be clipped and not visible. |
| `TableFlags_.no_keep_columns_visible` | Disable keeping column always minimally visible when ScrollX is off and table gets too small. Not recommended if columns are resizable. |
| `TableFlags_.precise_widths` | Disable distributing remainder width to stretched columns (width allocation on a 100-wide table with 3 columns: Without this flag: 33,33,34. With this flag: 33,33,33). With larger number of columns, resizing will appear to be less smooth. |
| `TableFlags_.no_clip` | Disable clipping rectangle for every individual columns (reduce draw command count, items will be able to overflow into other columns). Generally incompatible with TableSetupScrollFreeze(). |
| `TableFlags_.pad_outer_x` | Default if BordersOuterV is on. Enable outermost padding. Generally desirable if you have headers. |
| `TableFlags_.no_pad_outer_x` | Default if BordersOuterV is off. Disable outermost padding. |
| `TableFlags_.no_pad_inner_x` | Disable inner padding between columns (double inner padding if BordersOuterV is on, single inner padding if BordersOuterV is off). |
| `TableFlags_.scroll_x` | Enable horizontal scrolling. Require 'outer_size' parameter of BeginTable() to specify the container size. Changes default sizing policy. Because this creates a child window, ScrollY is currently generally recommended when using ScrollX. |
| `TableFlags_.scroll_y` | Enable vertical scrolling. Require 'outer_size' parameter of BeginTable() to specify the container size. |
| `TableFlags_.sort_multi` | Hold shift when clicking headers to sort on multiple column. TableGetSortSpecs() may return specs where (SpecsCount > 1). |
| `TableFlags_.sort_tristate` | Allow no sorting, disable default sorting. TableGetSortSpecs() may return specs where (SpecsCount == 0). |
| `TableFlags_.highlight_hovered_column` | Highlight column headers when hovered (may evolve into a fuller highlight) |
| `TableFlags_.sizing_mask_` |  |

### imgui.TableRowFlags_

Flags for ImGui::TableNextRow()

| member | description |
|---|---|
| `TableRowFlags_.none` |  |
| `TableRowFlags_.headers` | Identify header row (set default background color + width of its contents accounted differently for auto column width) |

### imgui.TreeNodeFlags_

Flags for ImGui::TreeNodeEx(), ImGui::CollapsingHeader*()

| member | description |
|---|---|
| `TreeNodeFlags_.none` |  |
| `TreeNodeFlags_.selected` | Draw as selected |
| `TreeNodeFlags_.framed` | Draw frame with background (e.g. for CollapsingHeader) |
| `TreeNodeFlags_.allow_overlap` | Hit testing will allow subsequent widgets to overlap this one. Require previous frame HoveredId to match before being usable. Shortcut to calling SetNextItemAllowOverlap(). |
| `TreeNodeFlags_.no_tree_push_on_open` | Don't do a TreePush() when open (e.g. for CollapsingHeader) = no extra indent nor pushing on ID stack |
| `TreeNodeFlags_.no_auto_open_on_log` | Don't automatically and temporarily open node when Logging is active (by default logging will automatically open tree nodes) |
| `TreeNodeFlags_.default_open` | Default node to be open |
| `TreeNodeFlags_.open_on_double_click` | Open on double-click instead of simple click (default for multi-select unless any _OpenOnXXX behavior is set explicitly). Both behaviors may be combined. |
| `TreeNodeFlags_.open_on_arrow` | Open when clicking on the arrow part (default for multi-select unless any _OpenOnXXX behavior is set explicitly). Both behaviors may be combined. |
| `TreeNodeFlags_.leaf` | No collapsing, no arrow (use as a convenience for leaf nodes). Note: will always open a tree/id scope and return True. If you never use that scope, add ImGuiTreeNodeFlags_NoTreePushOnOpen. |
| `TreeNodeFlags_.bullet` | Display a bullet instead of arrow. IMPORTANT: node can still be marked open/close if you don't set the _Leaf flag! |
| `TreeNodeFlags_.frame_padding` | Use FramePadding (even for an unframed text node) to vertically align text baseline to regular widget height. Equivalent to calling AlignTextToFramePadding() before the node. |
| `TreeNodeFlags_.span_avail_width` | Extend hit box to the right-most edge, even if not framed. This is not the default in order to allow adding other items on the same line without using AllowOverlap mode. |
| `TreeNodeFlags_.span_full_width` | Extend hit box to the left-most and right-most edges (cover the indent area). |
| `TreeNodeFlags_.span_label_width` | Narrow hit box + narrow hovering highlight, will only cover the label text. |
| `TreeNodeFlags_.span_all_columns` | Frame will span all columns of its container table (label will still fit in current column) |
| `TreeNodeFlags_.label_span_all_columns` | Label will span all columns of its container table |
| `TreeNodeFlags_.nav_left_jumps_to_parent` | Nav: left arrow moves back to parent. This is processed in TreePop() when there's an unfulfilled Left nav request remaining. |
| `TreeNodeFlags_.collapsing_header` |  |
| `TreeNodeFlags_.draw_lines_none` | No lines drawn |
| `TreeNodeFlags_.draw_lines_full` | Horizontal lines to child nodes. Vertical line drawn down to TreePop() position: cover full contents. Faster (for large trees). |
| `TreeNodeFlags_.draw_lines_to_nodes` | Horizontal lines to child nodes. Vertical line drawn down to bottom-most child node. Slower (for large trees). |

### imgui.WindowFlags_

Flags for ImGui::Begin()

| member | description |
|---|---|
| `WindowFlags_.none` |  |
| `WindowFlags_.no_title_bar` | Disable title-bar |
| `WindowFlags_.no_resize` | Disable user resizing with the lower-right grip |
| `WindowFlags_.no_move` | Disable user moving the window |
| `WindowFlags_.no_scrollbar` | Disable scrollbars (window can still scroll with mouse or programmatically) |
| `WindowFlags_.no_scroll_with_mouse` | Disable user vertically scrolling with mouse wheel. On child window, mouse wheel will be forwarded to the parent unless NoScrollbar is also set. |
| `WindowFlags_.no_collapse` | Disable user collapsing window by double-clicking on it. Also referred to as Window Menu Button (e.g. within a docking node). |
| `WindowFlags_.always_auto_resize` | Resize every window to its content every frame |
| `WindowFlags_.no_background` | Disable drawing background color (WindowBg, etc.) and outside border. Similar as using SetNextWindowBgAlpha(0.0). |
| `WindowFlags_.no_saved_settings` | Never load/save settings in .ini file |
| `WindowFlags_.no_mouse_inputs` | Disable catching mouse, hovering test with pass through. |
| `WindowFlags_.menu_bar` | Has a menu-bar |
| `WindowFlags_.horizontal_scrollbar` | Allow horizontal scrollbar to appear (off by default). You may use SetNextWindowContentSize(ImVec2(width,0.0)); prior to calling Begin() to specify width. Read code in imgui_demo in the "Horizontal Scrolling" section. |
| `WindowFlags_.no_focus_on_appearing` | Disable taking focus when transitioning from hidden to visible state |
| `WindowFlags_.no_bring_to_front_on_focus` | Disable bringing window to front when taking focus (e.g. clicking on it or programmatically giving it focus) |
| `WindowFlags_.always_vertical_scrollbar` | Always show vertical scrollbar (even if ContentSize.y < Size.y) |
| `WindowFlags_.always_horizontal_scrollbar` | Always show horizontal scrollbar (even if ContentSize.x < Size.x) |
| `WindowFlags_.no_nav_inputs` | No keyboard/gamepad navigation within the window |
| `WindowFlags_.no_nav_focus` | No focusing toward this window with keyboard/gamepad navigation (e.g. skipped by Ctrl+Tab) |
| `WindowFlags_.unsaved_document` | Display a dot next to the title. When used in a tab/docking context, tab is selected when clicking the X + closure is not assumed (will wait for user to stop submitting the tab). Otherwise closure is assumed when pressing the X, so if you keep submitting the tab may reappear at end of tab bar. |
| `WindowFlags_.no_docking` | Disable docking of this window |
| `WindowFlags_.no_nav` |  |
| `WindowFlags_.no_decoration` |  |
| `WindowFlags_.no_inputs` |  |
| `WindowFlags_.dock_node_host` | Don't use! For internal use by Begin()/NewFrame() |
| `WindowFlags_.child_window` | Don't use! For internal use by BeginChild() |
| `WindowFlags_.tooltip` | Don't use! For internal use by BeginTooltip() |
| `WindowFlags_.popup` | Don't use! For internal use by BeginPopup() |
| `WindowFlags_.modal` | Don't use! For internal use by BeginPopupModal() |
| `WindowFlags_.child_menu` | Don't use! For internal use by BeginMenu() |
