# Graphics — reference for writing user code

Every graphic is created through a subplot: `figure[0, 0].add_<name>(...)`. The method returns the
graphic, and every constructor argument is also a settable property afterwards.

Arguments **every** graphic accepts, on top of its own: `name` (then `subplot["name"]` finds it),
`offset` `(x, y, z)`, `rotation` (quaternion), `scale` `(x, y, z)`, `alpha`, `alpha_mode`,
`visible`, `metadata` (anything you want to hang on it).

## Constructors, with real defaults

```python
add_image(data, vmin=None, vmax=None, cmap="plasma", gamma=1.0,
          interpolation="nearest", cmap_interpolation="linear",
          colorspace="srgb", cpu_buffer=True)

add_image_volume(data, mode="mip", vmin=None, vmax=None, cmap="plasma", gamma=1.0,
                 interpolation="linear", cmap_interpolation="linear",
                 plane=(0, 0, -1, 0), threshold=0.5, step_size=1.0,
                 substep_size=0.1, emissive=(0, 0, 0), shininess=30)

add_line(data, thickness=2.0, colors="w", cmap=None, cmap_transform=None,
         cmap_range=None, size_space="screen", dash_pattern=(), thin=False)

add_inf_line(data, axis=None, thickness=2.0, colors="w", cmap=None, cmap_transform=None,
             cmap_range=None, start_is_infinite=True, end_is_infinite=True,
             dash_pattern=(), size_space="screen")

add_scatter(data, colors="w", cmap=None, cmap_transform=None, cmap_range=None,
            mode="markers", markers="o", custom_sdf=None, edge_colors="black",
            edge_width=1.0, image=None, point_rotations=None, sizes=5,
            size_space="screen")

add_mesh(positions, indices, mode="phong", plane=(0., 0., 1., 0.), colors="w",
         mapcoords=None, cmap=None, clim=None)
add_surface(data, mode="phong", colors="w", mapcoords=None, cmap=None, clim=None)
add_polygon(data, mode="basic", colors="w", mapcoords=None, cmap=None, clim=None)

add_vectors(positions, directions, color="w", size=None, vector_shape_options=None)

add_text(text, font_size=14, face_color="w", outline_color="w", outline_thickness=0.0,
         screen_space=True, offset=(0, 0, 0), anchor="middle-center")
```

Collections take the same arguments as their child graphic, each accepting either one value for all
graphics or one per graphic, plus plural forms of the common arguments (`names`, `offsets`,
`visibles`, `metadatas`, ...):

```python
add_line_collection(data, thickness=2.0, colors="w", cmap=None, cmap_transform=None, ...)
add_line_stack(data, separation=(0., 0., 0.), separation_axis="y", steps=None, ...)
add_scatter_collection(data, ...)      add_scatter_stack(data, ...)
add_image_collection(data, offsets=None, ...)
add_image_grid(data, shape=None, separation=(0., 0.), offsets=None, ...)
```

`add_vectors` takes **`color`, singular** — it is the one graphic that does not use `colors`.

## Images

```python
image = figure[0, 0].add_image(frame, cmap="gray", vmin=0, vmax=255)
image.data[:] = next_frame          # in place; the shape must match
image.vmin, image.vmax = 10, 200
image.reset_vmin_vmax()             # re-estimate from the data
```

- `vmin`/`vmax` are in the **data's own units**, not normalized. A `uint8` image uses `0..255`.
  Omitting them estimates from a subsample, which is an estimate, not the true min/max.
- `[rows, cols]` is grayscale and uses `cmap`. `[rows, cols, 3|4]` is RGB(A) and **ignores `cmap`**
  — `image.cmap` is `None` and setting it raises.
- `interpolation="linear"` smooths pixels; `"nearest"` (default) keeps them sharp. Keep `"nearest"`
  for anything where a pixel is a measurement.
- Rows are y and columns are x. `image.data[row, col]`, but a click's `pick_info["index"]` is
  `(col, row)`.
- Images added to the same subplot are automatically depth-ordered so later ones draw on top. If you
  set an image's z offset yourself, use a **non-integer** value or it will be overwritten.
- `cpu_buffer=False` keeps no copy in system RAM and sends data straight to the GPU: much faster.
  A regular `ImageGraphic` can use it, and `add_video`/`ImageYUVGraphic` are bufferless always.
  Set the whole array — `image.data = new_data`, which must be the same shape as the original;
  slice writes (`image.data[...] = ...`) raise. Some features aren't available: you must pass
  `vmin`/`vmax` (no estimation is done locally in host RAM), `reset_vmin_vmax()` is not supported,
  selectors cannot retrieve the data values under the selection, RGB `[rows, cols, 3]` is rejected
  (wgpu has no RGB textures — use RGBA, or grayscale), and grayscale tooltip values are estimated
  by inverting the colormap LUT rather than read from the data.

Volumes: `mode` is `"mip"` (max intensity projection, default), `"minip"`, `"iso"` (isosurface,
uses `threshold`/`step_size`/`emissive`/`shininess`), or `"slice"` (uses `plane`, the `(a, b, c, d)`
of `ax + by + cz + d = 0`). Mode is settable after creation.

## Lines

```python
line = figure[0, 0].add_line(np.column_stack([xs, ys]), thickness=2, colors="w")
line.data[:, 1] = new_ys            # y-values only, in place
line.data[100:200, 1] += 5          # slice writes work
line.colors[bool_mask] = "r"        # only when colors is per-vertex
line.thickness = 5
line.dash_pattern = "--"            # "-", "--", "-.", ":" or a float sequence
```

- 1D input is treated as y-values with x set to `arange`. Pass `np.column_stack([xs, ys])` whenever
  x is meaningful — this is a frequent source of silently wrong plots.
- `thin=True` switches to a one-pixel material that is much faster for very many lines, and ignores
  `thickness`, `dash_pattern` and anti-aliasing.
- `size_space="screen"` (default) keeps the line the same visual thickness at any zoom; `"world"`
  scales it with the data.

`add_inf_line` draws lines that extend forever — thresholds, event markers, stimulus onsets:

```python
figure[0, 0].add_inf_line(np.array([0.0, np.pi, 2 * np.pi]), axis="x", colors="r")
figure[0, 0].add_inf_line(np.array([-1.0, 1.0]), axis="y", dash_pattern="--")
```

With `axis="x"|"y"|"z"` the data is a **1D array of positions** and `colors` is **one per line**.
Its `data` and `colors` are per-line, not per-vertex. It does not support selectors.

## Scatters

```python
scatter = figure[0, 0].add_scatter(points, sizes=5, colors="w")
scatter.sizes = 10                       # uniform
scatter.sizes = per_point_sizes          # switches to per-point

# to recolor a subset, the colors must already be per-point:
scatter = figure[0, 0].add_scatter(points, colors=np.tile([1, 1, 1, 1], (len(points), 1)))
scatter.colors[labels == 2] = "r"
```

**Slicing a property only works when it is per-datapoint.** With a uniform value `scatter.colors` is
a `pygfx.Color`, `scatter.sizes` is a `float` and `scatter.markers` is a `str` — none of them are
subscriptable, and `scatter.colors[mask] = "r"` raises `TypeError`. Assign the whole property once
with an array to switch modes, then slice.

- `mode` cannot be changed after creation. `"simple"` is the fastest for very many points;
  `"markers"` gives shapes and edges; `"gaussian"` gives soft blobs; `"image"` renders a sprite
  image at each point.
- `markers` accepts matplotlib characters (`"osD+x^v<>*"`), unicode (`"●■♦"`), emoji, a
  `pygfx.MarkerShape` name, or `"custom"` with `custom_sdf`. One string is uniform; a sequence is
  per-point.
- `edge_colors=None` means no edge. `edge_width=0` is the cheapest option when you do not want
  edges at all.
- `point_rotations=None` (default) makes each marker follow the curve of the data; a float rotates
  all of them; an array rotates each.
- `sizes` is in screen pixels by default (`size_space="screen"`).

## Meshes, surfaces, polygons

- `add_surface(height_map)` for `[m, n]` heights, or `[m, n, 3]` to give x and y explicitly. With a
  `cmap` and no `mapcoords`, the colormap is applied to z automatically.
- `add_polygon(vertices)` for `[n_vertices, 2]`; it is triangulated for you and always lies in the
  xy plane — set `rotation` to place it elsewhere. This is what to use for ROI outlines and filled
  regions. **Needs at least 4 vertices** — a 3-vertex polygon currently raises
  `ValueError: indices must be of shape [n_vertices, 3] or [n_vertices, 4]`. For a triangle, use
  `add_mesh` with explicit `indices=[[0, 1, 2]]`.
- `add_mesh(positions, indices)` for an arbitrary triangle mesh.
- These use `mapcoords` + `clim` for colormapping rather than `cmap_transform` + `cmap_range`:
  `mapcoords` is the per-vertex value, `clim` is the `(min, max)` mapped onto the colormap.
- Adding a mesh installs scene lighting automatically. `mode="basic"` is flat/ambient only,
  `"phong"` is shaded.

## Collections

A collection holds many graphics of one type and exposes **every property of the child across all of
them**, indexed numpy-style: the first axis is the graphic, the rest index within each graphic.

```python
stack = figure[0, 0].add_line_stack(traces, separation=(0, 2, 0), cmap="tab10", thickness=2)

stack.colors[:10] = "r"            # first ten lines
stack.colors[mask] = "w"           # boolean mask over graphics
stack.data[3, :, 1] = new_ys       # y-values of line 3
stack.data[:, ::10, 1] += 1        # every 10th point of every line
stack.thickness[stack.thickness < 3] = 5
stack.visibles = False             # plural: one per graphic
stack.visible = False              # singular: the whole collection
stack.graphics[0]                  # the underlying LineGraphic
for line in stack: ...             # iteration works
len(stack)
```

- **Singular vs plural**: a property the collection also has as a graphic in its own right is
  plural for the per-graphic version. `collection.offset` moves the whole collection;
  `collection.offsets` moves each graphic. Same for `names`, `rotations`, `scales`, `alphas`,
  `alpha_modes`, `visibles`, `metadatas`.
- **`collection[0]` does not work.** Use `collection.graphics[0]` or iterate.
- Graphics may have different numbers of datapoints (jagged). Reading across graphics returns an
  object array of views, so `collection.data[:]` is not a rectangular array.
- Comparison and arithmetic work elementwise for masking:
  `collection.colors[collection.thickness < 3] = "r"`.
- `add_graphic()` / `remove_graphic()` grow and shrink a collection at runtime. Every graphic must
  use the same uniform-vs-per-vertex mode for a given property as the first one.

`add_line_stack` / `add_scatter_stack` additionally offset each graphic so they do not overlap:
`separation` is the extra `(x, y, z)` gap, `separation_axis` is any combination of `"x"`, `"y"`,
`"z"`, and `steps` lets you space graphics individually. Set `separation` again to restack after
changing the data.

`add_image_grid(images, shape=(2, 3), separation=(10, 10))` lays images out row-major;
`add_image_collection(images, offsets=...)` places them wherever you want.

### Colormaps on collections

On a `LineCollection`/`ScatterCollection`, `cmap` is a **collection-level property**, not an
accessor you can index:

```python
collection.cmap = "viridis"                          # one color per graphic, spread over the cmap
collection.cmap = "tab10"; collection.cmap_transform = labels   # color per graphic by label
collection.cmap = ["jet"] * len(collection)          # each graphic gets its own cmap along its points
collection.graphics[3].cmap = "plasma"               # one graphic's own cmap
```

`collection.cmap[3] = "plasma"` does **not** work — `collection.cmap` returns whatever you assigned
(a `str` or a `list`), not a `Colormap` and not an indexable accessor.

A qualitative colormap with an integer `cmap_transform` gives every graphic with the same label the
same color — this is the idiom for cluster labels, cell types, or trial conditions.

## Reading values back

`graphic.data`, `graphic.colors` and other array-valued properties return a buffer wrapper, not an
array. Slice it (`graphic.data[10:20]`) or use `graphic.data.value` for the whole numpy array.
`np.asarray(graphic.data)` deliberately raises to stop you from silently copying a GPU buffer.

`graphic.colors` is `None` whenever a `cmap` is set.

## Deleting

`subplot.delete_graphic(graphic)` frees VRAM and unregisters handlers. `subplot.remove_graphic()`
only takes it out of the scene, and `subplot.add_graphic()` puts it back — that pair is how you
hide and restore something expensive. `del graphic` alone leaks.
