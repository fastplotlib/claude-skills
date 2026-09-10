# Helpers worth knowing before you write your own

Reach for these instead of reimplementing them. Available as `fpl.utils.<name>`.

## `heatmap_to_positions(heatmap, xvals)`

`[n_rows, n_datapoints]` → `[n_rows, n_datapoints, 2]` xy data. This is the bridge from anything
naturally shaped as a heatmap (spike counts per unit, traces per cell, a spectrogram, an ethogram)
to a `LineCollection`/`LineStack`/`ScatterCollection`/`NDTimeseries`.

```python
positions = fpl.utils.heatmap_to_positions(counts.T, xvals=bin_centers)
ndw["raster"].add_nd_timeseries(positions, ("unit", "time", "xy"), ("unit", "time", "xy"),
                                graphic_type=fpl.ImageGraphic)
```

Going through this rather than `add_image(heatmap)` directly is what keeps the subplot on the shared
reference index and gives you a real x axis in seconds.

## `quick_min_max(data, max_size=1e6)`

An **estimate** of `(min, max)` by subsampling — this is what `vmin`/`vmax` default to when you omit
them, and what `reset_vmin_vmax()` uses. It returns precomputed `data.min`/`data.max` directly if
the object exposes them as scalars, which lazy readers usually do. Never present its result as the
true range; pass explicit `vmin`/`vmax` when the exact range matters.

## Colors

```python
fpl.utils.make_colors(n_colors, cmap, alpha=1.0)    # (n, 4) RGBA, evenly spaced along a colormap
fpl.utils.make_colors_dict(labels, cmap)            # {label: color}, for categorical data
fpl.utils.COLORMAP_NAMES                            # grouped catalogue: sequential/diverging/...
fpl.utils.get_cmap(name, alpha, gamma)              # a colormap as a numpy LUT
```

`make_colors_dict(labels, "tab10")` is the right way to get a stable label→color mapping; do not
build one with a `zip` over `itertools.cycle` unless you specifically want cycling.

## Other

- `subsample_array(arr, max_size, ignore_dims)` — subsamples while preserving dimensional
  proportions. Useful for a quick preview of something huge; not a substitute for `display_window`.
- `calculate_figure_shape(n)` → a roughly square `(n_rows, n_cols)` for `n` subplots.
- `normalize_min_max(a)` → 0-1.

## What fastplotlib accepts as "data"

`fpl.protocols.ArrayProtocol` is the whole contract: `dtype`, `ndim`, `shape`, `__getitem__`. Any
object with those four works — zarr, HDF5 datasets, a video reader, a spike-sorting recording, your
own lazy wrapper. You do **not** need a numpy array, and converting one to numpy just to plot it
defeats the point.

`FutureProtocol` covers objects whose reads return futures (async readers);
`CudaArrayProtocol` covers `__cuda_array_interface__`.

**Do not branch on the array library.** Call the array's own methods (`arr.max(axis=0)`) so torch,
jax and cupy arrays reduce on their own device. Never `import cupy`, never `isinstance`-check for a
backend, and never `np.asarray()` an array before reducing it — that copies a GPU array to host RAM.
