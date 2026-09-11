# Cross-cutting things — namespace, backends, GPUs, transparency, spaces

## What is on `fpl.`

```
Figure  NDWidget  loop  IMGUI
LineGraphic  InfLineGraphic  ScatterGraphic  ImageGraphic  ImageYUVGraphic
ImageVolumeGraphic  MeshGraphic  SurfaceGraphic  PolygonGraphic  VectorsGraphic  TextGraphic
LineCollection  LineStack  ScatterCollection  ScatterStack  ImageCollection  ImageGrid
LinearSelector  LinearRegionSelector  RectangleSelector
LinearSelectors  LinearRegionSelectors  RectangleSelectors  PolygonSelectors  SelectorCollection
PositionsHighlightSelector  CollectionHighlightSelector  ImageHighlightSelector  HighlightSelector
VisibilitySelector  ImageVisibilitySelector  SelectionVector
Cursor  Tooltip  TextBox  Legend  GraphicFeatureEvent
pause_events  get_nearest_graphics  get_nearest_graphics_indices
enumerate_adapters  select_adapter  print_wgpu_report
utils  tools  ui  enums  protocols  axes  widgets  layouts  graphics
```

Two traps:

- **`fpl.PolygonSelector` does not exist** (it is missing from the selectors `__all__`). Create one
  with `graphic.add_polygon_selector()`.
- **`fpl.ImageWidget` does not exist.** Use `fpl.NDWidget`.

## Backends and notebooks

The canvas is chosen automatically from what is running. The same code works everywhere; only the
show/run pattern differs:

| environment | pattern |
|---|---|
| script | `figure.show()`, then `fpl.loop.run()` under `if __name__ == "__main__":` |
| Jupyter | `figure.show()` as the **last line of the cell**; never `fpl.loop.run()` |
| Jupyter, several figures | `ipywidgets.VBox([fig1.show(), fig2.show()])` as the last line |
| Jupyter, in a sidecar | `figure.show(sidecar=True)` |
| Jupyter, in a Qt window | `%gui qt` **before** importing fastplotlib |
| Qt app | embed `figure.show()` as a widget |

In Jupyter, rendering happens server-side and only a JPEG stream reaches the browser, so a
visualization of a huge dataset on a remote machine works without moving the data. Pick a display
mode once per kernel; switching needs a restart.

## Picking the GPU

On a machine with more than one adapter (a laptop with integrated + discrete, a workstation with
several cards), wgpu may not choose the one you want. Select it **before creating any figure**:

```python
adapter = fpl.enumerate_adapters()[0]
print(adapter.info)          # check you got the right one
fpl.select_adapter(adapter)
```

`fpl.print_wgpu_report()` dumps the full diagnostic. If fastplotlib warns at import that no adapter
was enumerated, nothing will render — that is a driver/environment problem, and
the [GPU guide](https://www.fastplotlib.org/ver/dev/user_guide/gpu.html) covers it.

## Transparency

Two properties, and both matter:

```python
graphic.alpha = 0.5
graphic.alpha_mode = "blend"
```

`alpha` alone is often not enough: the default `alpha_mode="auto"` writes to the depth buffer, so a
semi-transparent graphic can still occlude what is behind it. For overlapping translucent things
use:

- `"blend"` — classic alpha blending, does not write depth. The usual choice for a 2D overlay.
- `"weighted_blend"` — order-independent; better when many translucent things overlap.
- `"dither"` — stochastic; handles order-independence very well but looks slightly noisy.
- `"add"` — additive, for glow/accumulation effects.
- `"solid"` — ignore alpha entirely.

Anti-aliasing on lines and points is enabled only for `"blend"` and `"weighted_blend"`.

## Spaces and transforms

Three spaces:

- **world** — the 3D space graphics live in
- **model / data** — world plus that graphic's own `offset`, `rotation` and `scale`
- **screen** — canvas pixels

```python
pos = subplot.map_screen_to_world(ev)          # a pointer event -> world (x, y, z)
px  = subplot.map_world_to_screen(pos)
xyz = graphic.map_world_to_model(pos)          # -> that graphic's data coordinates
pos = graphic.map_model_to_world(xyz)
```

**If a graphic has an `offset` or `scale` — which every stack and grid does — you must map through
`map_world_to_model` before using a click position to index its data.** Skipping this is the usual
cause of "the click selects the wrong point".

## Tooltips

Hovering a graphic shows a tooltip with the value under the cursor, by default. Customize it:

```python
graphic.tooltip_format = lambda pick_info: f"cell {pick_info['index'][1]}"
subplot.tooltip.enabled = False        # turn it off for a subplot
```

`pick_info` keys depend on the graphic: `"index"` (`(col, row)` for images, a vertex index for
lines/scatters), `"vertex_index"`, `"rgba"`, `"face_index"`. Stash whatever the formatter needs in
`graphic.metadata` at construction time.

## Colormaps

From the [`cmap`](https://cmap-docs.readthedocs.io/en/stable/catalog/) library — every matplotlib
name works, plus many more.

- A bare name can be ambiguous across catalogues and will warn (`"coolwarm"`). Use the namespaced
  form (`"matlab:jet"`, `"bids:plasma"`) when the exact colors matter.
- `graphic.cmap` returns a `cmap.Colormap`, not the string you passed. `cmap.num_colors` is the
  number of colors, which is what you want for `cmap_range` on a qualitative map.
- `fpl.utils.COLORMAP_NAMES` groups the catalogue into sequential / diverging / cyclic /
  qualitative / miscellaneous. Pick from the right group: sequential for magnitudes, diverging for
  signed values around a midpoint, **qualitative for labels**.

## Other useful pieces

```python
cursor = fpl.Cursor()                      # a crosshair at the same world position in every subplot
cursor.add_subplot(figure[0, 0])
cursor.add_subplot(figure[0, 1])

fpl.get_nearest_graphics(pos, collection)  # sorted nearest-first, works on a collection directly
with fpl.pause_events(g1, g2): ...         # suppress property events for a coordinated change
```

## Performance checklist

Before handing over code that updates at frame rate:

1. Data is `float32`.
2. Updates are slice writes (`graphic.data[:, 1] = ys`), not whole-array assignments.
3. Similar things are in one collection, not N graphics.
4. Uniform values (`colors="w"`, `sizes=5`) where the value is the same everywhere.
5. Large data goes through `NDWidget` with a `display_window`, not subsampled by hand.
6. The animation function does the minimum per call; everything constant is precomputed.
7. `mode="simple"` on scatters and `thin=True` on lines when there are very many of them.
8. `compute_histogram=False` on an `add_nd_image` whose reader is codec-backed.
