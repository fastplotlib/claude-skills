# Graphic properties — changing data and reacting to changes

Every mutable property of a graphic (`data`, `colors`, `cmap`, `thickness`, `sizes`, `vmin`,
`visible`, `offset`, ...) is a *graphic feature*: it writes straight into the GPU buffer and it
emits an event. This file is about using them well.

## Updating array data

```python
line.data[:, 1] = new_ys        # write y-values into the existing buffer  ✅
image.data[:] = frame           # whole image, same shape                  ✅
image.data[100:200, :50] = 0    # a region                                 ✅
scatter.sizes[mask] = 20        # fancy index                              ✅

line.data = np.column_stack([xs, ys])   # only when the number of points changed
```

**A slice write uploads only the affected range.** A whole-property assignment with a new length
allocates a new buffer, re-uploads everything, and swaps it into the geometry. Both are supported;
slice write is 100× cheaper.

Keys that work for slicing a feature: an int, any slice (including negative and stepped), a list of
ints, an integer array, a boolean mask over the datapoints, and a tuple for multiple axes
(`image.data[rows, cols]`, `line.data[:, 1]`). Ellipsis is not supported.

## Uniform vs per-datapoint is decided by the value you pass

There is no `uniform_color=` or `color_mode=` argument. What you pass decides:

```python
line = subplot.add_line(data, colors="w")          # one uniform color, no buffer
line = subplot.add_line(data, colors=rgba_n_by_4)  # per-vertex buffer
line.colors = "r"                                  # stays uniform (or broadcasts, if per-vertex)
line.colors = rgba_n_by_4                          # switches to per-vertex
```

The same applies to `sizes`, `markers`, `edge_colors`, and `point_rotations` on a scatter. Prefer
the uniform form whenever the value is the same everywhere: `sizes=5` is one number,
`sizes=np.full(n, 5)` is an `n`-element float32 buffer that says the same thing.

Reading back reflects the mode: `line.colors` is a `pygfx.Color` when uniform, a sliceable
`VertexColors` when per-vertex, and **`None` when a `cmap` is set**.

**So slicing only works in per-datapoint mode.** With a uniform value, `graphic.colors` is a
`pygfx.Color`, `graphic.sizes` is a `float`, `graphic.markers` is a `str` — none subscriptable, and
`graphic.colors[mask] = "r"` raises `TypeError`. Assigning the whole property with an array switches
the mode, after which slicing works:

```python
scatter.colors = np.tile([1, 1, 1, 1], (n, 1)).astype(np.float32)
scatter.colors[mask] = "r"          # fine now
```

If you know up front that you will recolor a subset, construct it per-datapoint.

## Events

```python
@line.add_event_handler("data")
def on_data(ev):
    print(ev.type, ev.info["key"], ev.info["value"])

def handler(ev): ...
line.add_event_handler(handler, "colors", "thickness")   # non-decorator form, many types
line.remove_event_handler(handler, "colors", "thickness")
```

Every event has `type`, `graphic`, `info`, `target`, `time_stamp`. `graphic.supported_events` lists
every valid string for that graphic — every property plus the pointer/key events.

`info` contents:

| kind of property | `info` keys |
|---|---|
| scalar (`thickness`, `vmin`, `cmap`, `visible`, ...) | `value` |
| array (`data`, `colors`, `sizes`, image `data`) | `key` (what you sliced), `value` (the new values) |
| `colors` specifically | also `user_value` (what you passed, before color parsing) |
| selector `selection` | `value`, plus callables `get_selected_indices`, `get_selected_data`, `get_selected_index` |

The [event tables](https://www.fastplotlib.org/ver/dev/user_guide/event_tables.html) document the
exact dict per graphic and property.

## Avoiding event loops

A handler that sets the property it is listening to would recurse. Two mechanisms:

- Re-entrance into a property setter is already blocked internally, so the recursion terminates
  rather than exploding. Do not rely on it for logic.
- To make a coordinated change without firing handlers:

```python
with fpl.pause_events(graphic1, graphic2):
    graphic1.data[:] = a
    graphic2.data[:] = b
# handlers fire again after the block
```

`fpl.pause_events(*graphics, event_handlers=[fn])` blocks only specific handlers.

## Performance notes that matter in animation callbacks

- Slice, do not reassign (above).
- Nothing is built if nothing is listening: the event `info` dict is only constructed when a handler
  is registered. Adding handlers to a property you update every frame is not free — keep them cheap.
- `graphic.data.value` is the live numpy array behind the buffer. Reading it is free; **writing to
  it directly does not mark the range for upload**, so the GPU will not see your change. Always go
  through `graphic.data[...] = ...`.
- `np.asarray(graphic.data)` raises on purpose. Use `.value` if you really want the array.

## Things that are easy to get wrong

| Symptom | Cause |
|---|---|
| the plot does not update | you wrote to `graphic.data.value` instead of `graphic.data[...]` |
| `TypeError: 'Color' object does not support item assignment` | the colors are uniform; assign an `[n, 4]` array first to make them per-datapoint |
| `TypeError` slicing `sizes` or `markers` | same — they are a `float` and a `str` in uniform mode |
| `graphic.colors` is `None` | a `cmap` is set; there is no colors buffer in that mode |
| "casting float64 array to float32" warning | pass `float32` |
| `ValueError` about 64-bit dtypes | wgpu has no 64-bit buffers; cast to 32-bit |
| setting `colors` had no effect on a cmap plot | setting `colors` clears the cmap — check you did not set the cmap again afterwards |
| animation got slower over time | you are reallocating a buffer each frame by assigning a whole array |
| `AttributeError` adding a handler for a property | that property does not exist in the graphic's current mode (e.g. `markers` on a `mode="simple"` scatter) |
