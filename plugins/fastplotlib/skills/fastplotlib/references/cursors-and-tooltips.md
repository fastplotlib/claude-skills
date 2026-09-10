# Tools — cursors, tooltips, text boxes

Three things live here, all reached from the top-level namespace: `fpl.Cursor`, `fpl.Tooltip`,
`fpl.TextBox`. None of them is a `Graphic`.

## Tooltips — you already have one

**Every subplot creates a `Tooltip` for you.** Hovering a graphic shows the value under the pointer.
Do not build a hover text by hand.

```python
subplot.tooltip.enabled = False          # turn it off for a subplot (e.g. a video)
subplot.tooltip.continuous_update = True # re-evaluate on every render, not just on pointer move
subplot.tooltip.font_size = 16
subplot.tooltip.background_color = (0, 0, 0, 0.9)
```

Customize *what it says* on the graphic, not on the tooltip:

```python
image.tooltip_format = lambda pick_info: f"{image.data[pick_info['index'][1], pick_info['index'][0]]:.3f}"
```

`tooltip_format` takes the `pick_info` dict and returns a string. Without it, the graphic's own
`format_pick_info` is used. Note the index order for images: `pick_info["index"]` is `(col, row)`,
so `data[row, col]` means `data[index[1], index[0]]`.

Stash anything the formatter needs in `graphic.metadata` at construction time — a frequency axis, a
list of behavior names, unit ids — rather than closing over globals.

**`continuous_update=True` is the one to know about for animations.** By default the tooltip is
recomputed on pointer move, so if the pointer sits still while the data underneath changes (an
`NDWidget` playing back, an animation), the displayed value goes stale. `continuous_update` re-picks
on every render and keeps it live. It costs a pick per frame, so enable it only when required.

Some graphics never show a tooltip: `TextGraphic`, selectors, and graphic collections all set
`_fpl_support_tooltip = False`. The tooltip comes from the child graphics of a collection.

## Cursor — the same world position across subplots

`fpl.Cursor` draws a crosshair or a marker at one world-space position in every subplot you add to
it, and follows the pointer.

```python
cursor = fpl.Cursor(
    mode="crosshair",     # or "marker"
    size=1.0,             # crosshair line thickness, or marker size (use > 5 for markers)
    color="w",
    alpha=0.7,
    size_space="screen",  # or "world"
    # marker mode only:
    marker="+", edge_color="k", edge_width=0.5,
)

for name in ("cam_left", "cam_right", "movie"):
    cursor.add_subplot(figure[name])

cursor.enabled = False     # stop it following the pointer, without removing it
cursor.position            # (x, y) in world space
```

### Configure it in the constructor and then leave it alone

This is the practical rule, and it is not a style preference — the property setters only touch
subplots that are already attached, and several of them are broken in that state:

| after `add_subplot`, in `"crosshair"` mode | |
|---|---|
| `cursor.color = ...` | ❌ `AttributeError: 'NoneType' object has no attribute 'color'` |
| `cursor.alpha = ...` | ❌ `AttributeError: ... 'opacity'` |
| `cursor.mode = ...` | ❌ `RuntimeError: dictionary changed size during iteration` |
| `cursor.clear()` | ❌ same `RuntimeError` |
| `size`, `size_space`, `marker`, `edge_color`, `edge_width`, `enabled`, `position` | ✅ |

A crosshair cursor is a `pygfx.Group` of two infinite lines, so its `.material` is `None`; the
`color` and `alpha` setters assume a single material and only work in `"marker"` mode. `mode` and
`clear` both iterate the subplot dict while mutating it.

All of them work *before* any subplot is attached, because the loop is empty — which is why passing
everything to the constructor is safe. `remove_subplot(subplot)` works; `clear()` does not.

### One cursor per subplot

`add_subplot` takes over tooltip handling for that subplot: it removes the subplot's own
`pointer_move` tooltip handler and drives the tooltip itself, calling your `tooltip_format` exactly
as before. So cursors and custom tooltips compose fine — but adding a **second** `Cursor` to the same
subplot raises `KeyError`, because the handler it tries to remove is already gone. Adding the *same*
subplot twice to one cursor raises `KeyError` too, by design.

`remove_subplot` hands tooltip control back to the subplot.

### `transform` — draw the cursor somewhere other than the pointer

```python
cursor.add_subplot(subplot, transform=lambda pos: (pos[0], f(pos[0])))
```

The callable takes the cursor's `(x, y)` world position and returns the position at which the cursor
is *drawn* in that subplot. Use it when subplots share an axis but not a coordinate system — a
spectrogram whose y is a frequency row index next to a trace whose y is a voltage, for example. The
reported `cursor.position` is untransformed.

## TextBox — a standalone label

`fpl.TextBox` is the text-on-a-background-plane that `Tooltip` is built from. Use it for text
that is not tied to hovering — a running frame count, a state label, a legend of your own.

```python
box = fpl.TextBox(
    font_size=12,
    text_color="w",
    background_color=(0.1, 0.1, 0.3, 0.95),
    outline_color=(0.8, 0.8, 1.0, 1.0),
    padding=(5, 5),          # pixels of background around the text
)

figure._fpl_overlay_scene.add(box._fpl_world_object)   # attach it to the figure
box.display((20.0, 40.0), "custom text")           # position is SCREEN space, in pixels
box.clear()                                            # hide it
box.visible = True
```

Two things to note: the position is in **screen pixels**, not world coordinates (map with
`subplot.map_world_to_screen` if you have a data position), and attaching it goes through
`figure._fpl_overlay_scene`, which is an internal — it is how the built-in tooltip attaches itself,
and there is no public alternative today. If you only want a label anchored to the data, use
`subplot.add_text(...)` instead; that is a real `TextGraphic` and lives in world space.

`font_size`, `text_color`, `background_color`, `outline_color`, `padding` and `visible` are all
settable at any time.

## Anti-patterns

| Do not | Do instead |
|---|---|
| build a hover text with a `pointer_move` handler and a `TextGraphic` | set `graphic.tooltip_format`; every subplot already has a tooltip |
| set `cursor.color` / `cursor.alpha` / `cursor.mode` after `add_subplot` | pass them to `fpl.Cursor(...)` |
| `cursor.clear()` | `remove_subplot` per subplot, or just drop the reference |
| add two `Cursor` instances to one subplot | one cursor, many subplots |
| a `TextBox` per subplot for a static label | `subplot.add_text(...)`, which is a real graphic in world space |
| leave the tooltip on over a video subplot| `subplot.tooltip.enabled = False` — pixel values are noise there |
| a tooltip that goes stale while a movie plays | `subplot.tooltip.continuous_update = True` |
| close over globals in `tooltip_format` | put what it needs in `graphic.metadata` |
