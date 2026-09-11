# Selection tools — letting the user pick things

## What actually works right now

Verified against the current checkout. Check this table before promising a user an interaction.

| tool | status |
|---|---|
| `add_linear_selector`, `add_linear_region_selector`, `add_rectangle_selector`, `add_polygon_selector` | ✅ |
| `ImageHighlightSelector` — options mode, and free mode with `rows` or `pixels` | ✅ |
| `ImageVisibilitySelector` | ✅ |
| `VisibilitySelector` on a `LineCollection` / `ScatterCollection` | ✅ |
| `SelectionVector` with a bare selector, `(sel, 1-D array)`, `(sel, dict)`, or `(sel, fwd, inv)` | ✅ |
| `Cursor.add_subplot`, `cursor.remove_subplot`, `cursor.position` | ✅ |
| `VisibilitySelector` on a `LineStack` / `ScatterStack` | ❌ `ValueError` in `_restack` (the whole 3-vector `separation` is added to a scalar distance). Use a `LineCollection` with explicit `offsets`. |
| `PositionsHighlightSelector` on a line or scatter | ❌ `TypeError` — no positions graphic is built with a `Highlightable*` material, so there is nothing for it to write into |
| `CollectionHighlightSelector` | ❌ silently does nothing, same cause: `_update_highlight_buffers` skips every sub-graphic whose material is not `Highlightable*` |
| `ImageHighlightSelector` free mode with `cols` alone | ❌ `TypeError`; `rows` works. Mixing `rows` and `cols` requires equal lengths |
| `SelectionVector` with `(selector, single_callable)` | ❌ `ValueError` — needs both directions |

To highlight lines or scatters until the material situation is fixed, write the colors directly
(`collection.colors[selected] = "w"`) or toggle visibility with a `VisibilitySelector` on a
collection. Do not try to fix the materials without asking.

## Three different tools, for three different jobs

| the user wants to | use |
|---|---|
| pick a point / time along an axis | `graphic.add_linear_selector()` |
| pick a range along an axis | `graphic.add_linear_region_selector()` |
| pick a rectangular region | `graphic.add_rectangle_selector()` |
| draw a freehand region / lasso | `graphic.add_polygon_selector()` |
| mark specific datapoints, lines, or pixels as "selected" | `fpl.PositionsHighlightSelector`, `fpl.CollectionHighlightSelector`, `fpl.ImageHighlightSelector` |
| show only a subset of a collection | `fpl.VisibilitySelector`, `fpl.ImageVisibilitySelector` |
| keep several selectors in sync across subplots/sessions | `fpl.SelectionVector` |
| many independent selectors of one kind on one graphic | `fpl.LinearSelectors`, `fpl.LinearRegionSelectors`, `fpl.RectangleSelectors`, `fpl.PolygonSelectors` |

## Spatial selectors

Always create them **from the graphic** — the bounds and size are computed from that graphic's data
for you:

```python
selector = line.add_linear_selector()                      # defaults to the first datapoint
region  = line.add_linear_region_selector(padding=5)
rect    = image.add_rectangle_selector()
lasso   = collection.add_polygon_selector()
```

Read and drive them through `selection`:

```python
selector.selection            # x (or y) value, in data space
selector.selection = 200      # moves it programmatically
region.selection              # (min, max)
rect.selection                # (xmin, xmax, ymin, ymax)
```

React to dragging with the `"selection"` event. The event carries callables so you do not have to
work out the indexing yourself:

```python
@selector.add_event_handler("selection")
def on_move(ev):
    index = ev.get_selected_index()          # linear selector
    ys = line.data[index, 1]

@region.add_event_handler("selection")
def on_region(ev):
    data = ev.get_selected_data()            # the data under the selection
    ixs = ev.get_selected_indices()
```

Outside a handler, call them on the selector with the graphic:
`region.get_selected_data(some_graphic)`. A selector can be read against **any** graphic, not just
its parent, which is how you drive several plots from one selector.

On a collection, `get_selected_indices()` returns one entry per graphic. The common idiom for
"which lines are under the selection" is:

```python
ixs = ev.get_selected_indices()
selected = [i for i in range(len(ixs)) if ixs[i].size > 0]
collection.colors[selected] = "w"
```

Notes:

- `InfLineGraphic` does not support selectors.
- Selectors are graphics, so `subplot.delete_graphic(selector)` removes them.
- Selector coordinates are in world space. If the parent graphic has an `offset` or `scale`, map
  through `graphic.map_world_to_model(...)` before indexing data.
- `add_linear_region_selector(padding=...)` widens the grab area along the other axis; useful on a
  flat trace where the selector would otherwise be only a few pixels tall.

## Highlight selectors — marking things without touching your data

`HighlightSelector` variants highlight on the GPU using a separate buffer, so **your colors,
colormap and data are untouched** and the highlight can be cleared instantly. They are **not**
graphics: construct them, then attach graphics.

```python
sel = fpl.ImageHighlightSelector(color="w", alpha=0.7)
sel.add_graphic(image)
sel.selection = {"rows": [10, 11, 12]}
sel.append({"rows": [50]})
sel.clear()
```

| class | selects | status |
|---|---|---|
| `ImageHighlightSelector` | pixel regions of an `ImageGraphic` | ✅ |
| `PositionsHighlightSelector` | individual datapoints of a `LineGraphic`/`ScatterGraphic` | ❌ raises |
| `CollectionHighlightSelector` | whole graphics within a collection | ❌ silent no-op |

`ImageHighlightSelector` takes a dict with **one** key: `{"rows": [...]}` or
`{"pixels": [array_of_row_col_pairs, ...]}`. `{"cols": ...}` alone currently raises, and giving both
`rows` and `cols` requires them to be the same length.

Its **options mode** is the idiom for ROIs — preload every candidate region once, then select by
index, which is far cheaper than rebuilding masks per click:

```python
sel = fpl.ImageHighlightSelector(
    lut="tab10",                              # a color per selected item
    lut_wrap="repeat",                        # cycle the colormap past 10 items
    selection_options={"pixels": contours},   # every ROI, preloaded
    options_color="w", options_alpha=0.1,     # how unselected ROIs are drawn
    alpha=0.7,
)
sel.add_graphic(image)
sel.selection = (3, 7)                        # option indices
```

One selector manages one buffer, so every graphic added to it shows the same selection — that is
how you highlight the same pixels across several sessions' movies.

Use `color=` for one color for everything, or `lut=` (a colormap name or an `(n, 4)` RGBA array) for
a color per selected item.

## Visibility selectors — showing a subset

```python
vis = fpl.VisibilitySelector(line_stack, lut="tab10", lut_wrap="repeat")
vis.selection = [0, 4, 9]      # only these lines are visible
vis.append(2)
vis.clear()                    # hide everything
```

`VisibilitySelector` toggles `visible` on the graphics of a collection and can recolor them from a
LUT. `ImageVisibilitySelector(graphic, axis="rows")` hides rows or columns of an image, collapsing
them in the shader rather than blanking them, so auto-scaling still works.

## `SelectionVector` — coordinated selection

When several selectors index the same conceptual thing through different local index spaces
(three sessions of the same cells, an image and its trace heatmap), do not wire N×N event handlers.
Register each selector with a mapping from a master index to that selector's local index:

```python
sv = fpl.SelectionVector()
sv.add_selector(image_selector)                          # identity mapping
sv.add_selector((traces_selector, local_index_array))    # array[master] -> local
sv.add_selector((other_selector, {0: 3, 1: 7}))          # dict master -> local
sv.add_selector((third, master_to_local, local_to_master))   # both directions, as callables

sv.selection = [master_index]   # every registered selector follows
sv.append(master_index)
```

Four accepted forms, and only four: a bare selector, a `(selector, 1-D int array)` where the array
index is the master index and the value is the local index, a `(selector, dict)`, or a
`(selector, forward, inverse)` triple of callables. **A `(selector, single_callable)` 2-tuple
raises** — the inverse cannot be derived.

## Reacting to a highlight/visibility selection

These are not graphics, so they have their own handler API rather than `add_event_handler(fn, type)`:

```python
def on_change(info):
    ...
sel.add_event_handler(on_change)
sel.remove_event_handler(on_change)
```

## Anti-patterns

| Do not | Do instead |
|---|---|
| construct `LinearSelector(...)` directly | `graphic.add_linear_selector()` — it computes limits from the data |
| write into `collection.colors` to show a selection, then restore the old colors | a `HighlightSelector`; it never touches your colors |
| rebuild ROI masks on every click | `ImageHighlightSelector(selection_options=...)` and select by index |
| add a `pointer_move` handler and re-derive the nearest item by hand | `fpl.get_nearest_graphics(pos, collection)` |
| wire selectors to each other pairwise | `fpl.SelectionVector` |
| add 20 `LinearSelector`s in a loop | `fpl.LinearSelectors(parent, limits, ...)` and `.append(...)` |
| forget that `pick_info["index"]` on an image is `(col, row)` | unpack as `col, row = ev.pick_info["index"]` |
