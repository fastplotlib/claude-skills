# Neuroscience visualization with fastplotlib

Guidance for building **multi-modal neuroscience visualizations** for a scientist who wants to look
at their data and is not going to debug your code.

Parts 1-5 below cover the domain libraries, the pynapple → fastplotlib conversions, timebase
discipline, a recipe per modality, and out-of-core rules.

## Ground rules

- Go through the [user guide](https://www.fastplotlib.org/ver/dev/user_guide/index.html), the
  [examples gallery](https://www.fastplotlib.org/ver/dev/_gallery/index.html) and the
  [API reference](https://www.fastplotlib.org/ver/dev/api/index.html) before writing code. The
  reference files alongside this one carry the API detail; the ones you will need most are
  `references/ndwidget.md` and `references/graphics.md`.
- **Verify every call against the version the user actually has installed**, not from memory — the
  API moves. `python -c "import fastplotlib as fpl; print(fpl.__file__)"` prints the source
  location; read it for anything about the API you are not certain of — class and method names,
  arguments, defaults, return types, and what a call actually does.
- Use the coding and writing style of fastplotlib.
- Do not over-engineer. Do not write low quality code.
- Your ideas and code must be high quality, accurate, correct, concise, elegant and have no
  mistakes. Follow the coding styles, patterns and logic in the fastplotlib codebase.
- Do not make assumptions, ask if you are unsure. Do not make your own judgement calls. **In
  neuroscience the assumptions that matter are units and timebases** — if you do not know whether
  an array is in seconds, samples, or frames, or what its sampling rate is, ask. A wrong guess
  produces a figure that looks right and is wrong, and the user will not catch it.
- Always use domain-specific libraries for any analysis. Any re-implementation of a function,
  routine, or feature from scratch in numpy MUST be justified and proven to not already exist.
- **Always use [pynapple](https://pynapple.org) first.** It is very unlikely you will have to
  re-implement a neuroscience-specific function from scratch. Search its documentation and example
  gallery extensively before writing any new analysis function. Use
  [nemos](https://nemos-neuro.org) for neural models.

## The one idea behind all of this

Neuroscience data is many arrays, recorded at different rates, that only mean something **together
and aligned in time**. A 30 Hz behavior camera, a 30 kHz probe recording, 10 Hz two-photon frames,
125 kHz audio, and hand-scored behavior labels all describe the same few minutes.

So the shape of every one of these visualizations is:

**one reference space in seconds → N subplots, each mapping that reference index onto its own
sample indices.**

`fpl.NDWidget` is built for exactly this. You give it a reference range in seconds; each modality
declares its dimension names and hands over its own timestamps array as `slider_maps`. One slider,
or one drag of a linear selector, then moves everything consistently. Several `NDWidget`s can share
one `ReferenceIndices`, so subplots can live in separate windows.

Never write your own "current frame" variable, your own slider, or your own resampling to line two
modalities up.

## Modality → construct

| the data | shape you feed fastplotlib | construct |
|---|---|---|
| behavior video (mp4, codec-backed) | frame reader, `[time, m, n]` (+ rgb) | `add_video` |
| calcium/voltage imaging movie | `[time, m, n]` | `add_nd_image` |
| volumetric imaging | `[time, z, m, n]` | `add_nd_image`, `display_dims=(z, m, n)` |
| ΔF/F or dF traces per cell | `[n_cells, n_frames, 2]` | `add_nd_timeseries` (`LineStack`, or `ImageGraphic` for a heatmap) |
| spike raster / binned counts | `[n_units, n_bins, 2]` | `add_nd_timeseries(graphic_type=fpl.ImageGraphic)` |
| individual spikes (time × depth/amplitude) | `[1, n_spikes, 2]` | `add_nd_timeseries(graphic_type=fpl.ScatterCollection)` |
| raw/filtered extracellular traces | `[n_channels, n_samples, 2]` | custom `NDSlicer` over the recording → heatmap or `LineStack` |
| LFP or audio spectrogram | `[n_freqs, n_times, 2]` | `add_nd_timeseries(graphic_type=fpl.ImageGraphic)` |
| pose tracking / keypoints | `[n_keypoints, n_frames, 2]`, stacked from the `<kp>_x`/`<kp>_y` columns | `add_nd_scatter` |
| ROI footprints / contours | list of `[K, 2]` pixel coords | `fpl.ImageHighlightSelector(selection_options={"pixels": contours})` |
| ethogram / behavioral state | `[n_behaviors, n_times, 2]` integer codes | `add_nd_timeseries(graphic_type=fpl.ImageGraphic)` + a discrete colormap |
| trial-aligned responses | `[n_trials, n_timepoints, 2]` | `add_nd_timeseries` |
| tuning curves, waveforms, PSTHs | `[n_units, n_bins, 2]` | `add_nd_lines` or a plain `add_line_stack` |
| unit positions on a probe, embeddings | `[n_units, 2]` | `add_scatter` |

`fpl.utils.heatmap_to_positions(heatmap, xvals)` converts anything shaped `[n_rows, n_timepoints]`
into the `[n_rows, n_timepoints, 2]` that the timeseries constructs want. Use it rather than
`add_image(heatmap)` — it is what keeps a heatmap subplot on the shared reference index with a real time axis.

## The skeleton

```python
import numpy as np
import fastplotlib as fpl

# 1. load, per modality, keeping the readers lazy
video = AsyncVideoReader(video_path, buffer_size=512)
traces = ...            # [n_cells, n_frames, 2]
spikes = ...            # a pynapple TsGroup

# 2. one reference range: the intersection of what every modality covers, in seconds
start = max(video.time[0], trace_times[0])
stop = min(video.time[-1], trace_times[-1])
ranges = {"time": (start, stop, 1 / 30)}      # step = playback increment

# 3. subplots, named, as fractions of the canvas
extents = {
    "video":  (0,   0.4, 0,   1),
    "traces": (0.4, 1,   0,   0.5),
    "raster": (0.4, 1,   0.5, 1),
}
ndw = fpl.NDWidget(ranges=ranges, extents=extents, size=(1400, 800))

# 4. each modality maps the reference index onto its own array indices
ndw["video"].add_video(
    video, dims=("time", "m", "n"), display_dims=("m", "n"),
    slider_maps={"time": video.time}, name="video",
)
ndw["traces"].add_nd_timeseries(
    traces, ("cell", "time", "xy"), ("cell", "time", "xy"),
    slider_maps={"time": trace_times}, display_window=10.0,
    cmap="tab10", x_range_mode="auto", name="traces",
)

# 5. link the time axis of the time-series subplots
for name in ("traces", "raster"):
    subplot = ndw.figure[name]
    subplot.controller.add_camera(subplot.camera, include_state={"x", "width"})

ndw.show(maintain_aspect=False)

if __name__ == "__main__":
    fpl.loop.run()
```

Write scripts in that order — parameters, loading, reference range, layout, subplots, linking, interaction,
`fpl.loop.run()` last. That is the order every example here uses and it is the order a reader
follows.

## Anti-patterns

| Do not | Do instead | Why it matters |
|---|---|---|
| align modalities by index, or `int(t * fs)` | `slider_maps={"time": timestamps}` | clocks drift, frames drop; index alignment is silently wrong |
| resample everything onto a common rate to plot it | let each subplot keep its own rate and map through its timestamps | resampling destroys the raw data and hides dropped frames |
| bin spikes with a `for` loop or `np.histogram` | `tsgroup.count(bin_size=..., ep=...)` | pynapple already does it, correctly, with epochs |
| write your own PSTH / correlogram / tuning curve / bandpass | `pynapple` — see **Part 1 — Domain libraries** below | these are solved and validated |
| build your own slider with ipywidgets or imgui | `fpl.NDWidget` | you lose async fetch, windowing, playback and sync |
| one figure per modality, each with its own slider | one `ranges`, then `indices=ndw.indices` | otherwise the subplots drift apart |
| `np.load` / `read_video` a whole session | a lazy reader plus `display_window` | sessions are larger than RAM and much larger than VRAM |
| `display_window=None` on a 40-minute recording | a window in seconds | uploads the entire recording to the GPU |
| one graphic per cell / unit / keypoint | a collection, and index across it | N buffers and N draw calls |
| `add_image(heatmap)` for a time-varying raster | `heatmap_to_positions` + `add_nd_timeseries(graphic_type=fpl.ImageGraphic)` | keeps it on the shared reference index with a real time axis |
| write colors into the data to show a selection, then restore them | `fpl.ImageHighlightSelector` / `fpl.CollectionHighlightSelector` | highlights on the GPU, leaves your colors intact |
| rebuild ROI masks on every click | preload them as `selection_options` and select by index | the per-click work becomes an index write |
| recompute an analysis inside an animation or index handler | precompute, cache, or recompute only on an explicit UI change | it runs every frame |
| pass `compute_histogram` to `add_video` | it has no such argument | a video is never histogrammed; it needs random frame access and is very slow on codecs |
| a sequential colormap for categorical labels | a qualitative colormap (`tab10`, `Set1`) with integer `cmap_transform` | otherwise neighbouring labels look similar |
| `np.asarray(gpu_array)` before reducing | `arr.max(axis=0)` | copies a GPU/torch array to host RAM |
| `float64` data | `.astype(np.float32)` | cast and a warning on every upload |
| an unlabelled subplot in a 6-subplot figure | `names=` and a `subplot.title` | the user cannot tell them apart |

---

# Part 1 — Domain libraries

## The rule

**Any re-implementation of a function, routine, or feature from scratch in numpy MUST be justified
and proven to not already exist.** Before writing a loop over spike times, a binning routine, a
PSTH, a tuning curve, a correlogram, a bandpass filter, an epoch intersection, a decoder, or
anything neuroscience related: search the library. All of these are solved.

If after searching you are convinced nothing covers it, say so explicitly, name what you searched,
and then write the minimal thing.

**pynapple and nemos move.** Function names have been deprecated and generalized (the `compute_1d_*`
/ `compute_2d_*` family is deprecated in favour of n-dimensional `compute_tuning_curves`). Look up
the current name and signature at <https://pynapple.org> and <https://nemos-neuro.org>; do not write
a call from memory. If a library is not installed where you are working, fetch its docs and say
which calls you could not verify.

## pynapple — time series, timestamps, epochs

The default for anything with a time axis. Everything carries its own timestamps, so operations
between objects stay aligned and `restrict` is exact.

```python
import pynapple as nap

spikes = nap.TsGroup({i: nap.Ts(t=times_s) for i, times_s in enumerate(unit_times)})
lfp    = nap.Tsd(t=timestamps, d=signal)
traces = nap.TsdFrame(t=frame_times, d=dff, columns=cell_ids)   # [n_frames, n_cells]
movie  = nap.TsdTensor(t=frame_times, d=frames)                 # [n_frames, m, n]
trials = nap.IntervalSet(start=trial_starts, end=trial_ends)
```

Attributes: `.t` (timestamps), `.d` / `.values` (data), `.index`, `.columns`, `.time_support` (the
`IntervalSet` the object is defined over), `.metadata` on a `TsGroup`/`TsdFrame`.

Methods you will use constantly: `.restrict(intervalset)`, `.count(bin_size, ep=...)`,
`.bin_average(bin_size)`, `.value_from(other)`.

Analyses it already provides — **verify the current names in the docs**: n-dimensional tuning curves
(`compute_tuning_curves`), mutual information, auto/cross-correlograms, ISI distributions,
peri-event alignment (`compute_perievent`), event- and spike-triggered averages, decoding, power
spectral density, Morlet wavelet transforms, bandpass filtering, and shuffling/randomization for
statistics. NWB is read through PyNWB.

## nemos — encoding and decoding models

JAX-backed, GPU-accelerated, and designed to consume pynapple objects. `GLM`, `PopulationGLM`,
`ClassifierGLM`; observation models (`PoissonObservations`, `NegativeBinomialObservations`,
`GammaObservations`, `GaussianObservations`, `BernoulliObservations`); a composable `basis` module
for building design matrices; regularizers `Ridge`, `Lasso`, `GroupLasso`, `ElasticNet`. Look up the
basis class names and the `fit`/`predict`/`score` signatures before using them.

Plot the fit the same way as the data: predicted rate as a line over the observed rate, basis
functions as a `LineStack`, coefficients as an image.

## spikeinterface — extracellular recordings and sortings

- `BaseRecording`: `get_traces(start_frame, end_frame, channel_ids=None, return_in_uV=False)`,
  `get_num_channels()`, `get_num_samples()`, `get_times()`, `sampling_frequency`, `channel_ids`,
  `get_total_duration()`.
- `BaseSorting`: `get_unit_spike_train(unit_id, start_frame=None, end_frame=None,
  return_times=False)`, `unit_ids`, `get_num_units()`.
- `SortingAnalyzer` combines the two and carries waveforms, templates and quality metrics.

A recording is already lazy and already array-like enough for a custom `NDSlicer`; never call
`get_traces()` on a whole session.

## masknmf — calcium/voltage imaging demixing

`DemixingResults.from_hdf5(path)`, then `.to("cuda")` to keep the component arrays on the GPU. The
arrays (`pmd_array`, `ac_array`, `residual_array`, `fluctuating_background_array`,
`colorful_ac_array`) are lazy and torch-backed: **index them, never load the entire movie or huge
chunks into RAM, never `np.asarray` them.** Frame timings come from the acquisition metadata and are assigned by you
(`dmr.timings = np.load(...)`).

## Video readers

| reader | when |
|---|---|
| [`asyncvideo`](https://pypi.org/project/asyncvideo) `AsyncVideoReader(path, buffer_size=512)` | **the default.** Decodes ahead on a background thread, exposes `.shape` and `.time` |
| a `decord`-backed lazy reader of your own | fallback when asyncvideo cannot open the file |

`add_video` sends YUV planes straight to the GPU instead of converting each frame to RGB, which is
why it exists as a separate method.

**Pass the same reader to every graphic that shows that video.** One video in three subplots needs
one reader, not three: all three views ask for the same frame, so they share that one decode. A
reader decodes one request at a time and a new frame supersedes the one still in flight, which is
what keeps a slider drag responsive — frames you scrub past are never decoded.

## NWB, IBL/ONE

NWB: read with pynapple (PyNWB underneath) so you get `TsGroup`/`Tsd`/`IntervalSet` objects
directly, rather than raw h5py datasets.

IBL/ONE session layout:

```
<subject>/<session>/raw_video_data/_iblrig_<camera>Camera.raw.mp4
<subject>/<session>/001/alf/_ibl_<camera>Camera.times.npy          # frame timestamps
<subject>/<session>/001/alf/_ibl_<camera>Camera.lightningPose.pqt  # pose tracking
<subject>/<session>/001/alf/FOV_<nn>/mpci.times.npy                # imaging frame times
```

The `.times.npy` files are the `slider_maps` for that stream. That is the whole point of them.

---

# Part 2 — pynapple → fastplotlib

fastplotlib wants arrays. Convert at the boundary, and only at the boundary — do the analysis in
pynapple, then hand over `.t` and `.values`.

| you have | you want | conversion |
|---|---|---|
| `Tsd` | a line | `np.column_stack([tsd.t, tsd.d]).astype(np.float32)` |
| `TsdFrame` `[n_samples, n_signals]` | stacked traces or a heatmap | `fpl.utils.heatmap_to_positions(tsdframe.values.T, xvals=tsdframe.t)` → `[n_signals, n_samples, 2]` |
| `TsGroup` | a binned raster | `counts = tsgroup.count(bin_size=0.01, ep=ep)`, then `heatmap_to_positions(counts.values.T, xvals=counts.t)` |
| `TsGroup` | a spike scatter (unit index vs time) | see the raster recipe below |
| `TsdTensor` `[n_frames, m, n]` | a movie subplot | `add_nd_image(tsdtensor.values, ("time", "m", "n"), ("m", "n"), slider_maps={"time": tsdtensor.t})` |
| `IntervalSet` | epoch boundaries | `add_inf_line(np.concatenate([ep.start, ep.end]), axis="x")` |
| stimulus onsets, reward times, any event | 1D array of times in the reference space | `subplot.add_inf_line(times, axis="x")`, a plain graphic, not an `NDGraphic` |
| `IntervalSet` | shaded epochs | one `add_polygon` per interval, or a `PolygonGraphic` collection |
| tuning curves (DataFrame) | curves per unit | `heatmap_to_positions(tc.values.T, xvals=tc.index.values)` |
| `compute_perievent` output | trial-aligned traces | stack the aligned trials into `[n_trials, n_timepoints, 2]` |

Two things to keep in mind:

- `.count()` returns **bin centres** in `.t`. Use those as your `xvals` and as the `slider_maps`
  entry, so the raster and the sliders agree.
- Restrict with an `IntervalSet` *before* converting, not by slicing the numpy array afterwards.
  `tsgroup.restrict(trials)` is exact; index arithmetic is not.

---

# Part 3 — Timebase discipline

Every figure that is subtly wrong is wrong here.

1. **Decide the reference space once and write it in a comment.** Almost always seconds.
   `ranges={"time": (start, stop, step)}` is in those units, not samples or frames.
2. **Every modality gets its own `slider_maps` entry**, built from *its own* recorded timestamps:

   ```python
   slider_maps={"time": video.time}          # frame timestamps
   slider_maps={"time": counts.t}            # pynapple bin centres
   slider_maps={"time": recording.get_times()}
   slider_maps={"time": t_spec}              # spectrogram column times
   ```

   No entry means the reference value is used directly as an index — correct only when the reference
   units *are* indices.
3. **Never `int(t * fs)` when a timestamps array exists.** Hardware clocks drift and frames drop; a
   nominal rate diverges over a session.
4. **The reference range is the intersection**, not the union:
   `start = max(all_starts)`, `stop = min(all_stops)`.
5. **`step` is the playback and step-button increment.** Set it to the finest rate you actually want
   to step through — `1/30` for a 30 Hz camera, `0.001` for millisecond ephys.
6. Name variables for their units. An array of samples is not `t`.

---

# Part 4 — Recipes

Each of these is drawn from a working multi-modal viewer. Verify every call against the installed
version before using it.

## Spike raster from a TsGroup

```python
spikes = nap.TsGroup({i: nap.Ts(t / 25_000) for i, t in enumerate(spike_times)}, time_units="s")
ep = nap.IntervalSet(start=0, end=600)
counts = spikes.count(bin_size=0.01, time_units="s", ep=ep)      # pynapple bins it

raster = fpl.utils.heatmap_to_positions(counts.values.T, xvals=counts.t)

ndg = ndw["raster"].add_nd_timeseries(
    raster,
    ("unit", "time", "xy"),
    ("unit", "time", "xy"),
    graphic_type=fpl.ImageGraphic,          # one row per unit, color = spike count
    slider_maps={"time": counts.t},
    display_window=10.0,
    x_range_mode="auto",
    graphic_kwargs={"cmap": "gray_r"},
    name="raster",
)
```

`graphic_kwargs` is how you set `cmap`, `vmin` and `vmax` at construction. Setting them afterwards
on `ndg.graphic` also works, and a colorbar added by `compute_histogram=True` follows the change.

Bin size is a scientific choice. If you expose it in a UI, recompute through `spikes.count()` — do
not rebin an already-binned array:

```python
@ndw.figure["raster"].add_imgui_window(location="top", size=36, title=None)
def bin_size_ui(subplot):
    global bin_ms
    changed, new_ms = imgui.input_int("bin size (ms)", v=bin_ms, step=10)
    if changed:
        bin_ms = max(new_ms, 1)
        state = subplot.camera.get_state()
        counts = spikes.count(bin_size=bin_ms / 1000, time_units="s", ep=ep)
        ndg.data = fpl.utils.heatmap_to_positions(counts.values.T, xvals=counts.t)
        ndg.slider_maps = {"time": counts.t}
        subplot.camera.set_state(state)      # keep the view where the user left it
```

Label the rows with a real unit id rather than a row index:

```python
def raster_tooltip(pick_info):
    col, row = pick_info["index"]
    n = round(ndg.graphic.data[row, col])
    return f"unit: {spikes.index[row]}\nspikes: {n}"

ndg.graphic.tooltip_format = raster_tooltip
```

## Spikes as a scatter — time vs depth, or time vs amplitude

For sorted output where each spike is a point rather than a bin:

```python
# [1, n_spikes, 2] — one "graphic" holding every spike
td = np.column_stack([spike_times_s, spike_depths])[None].astype(np.float32)

ndw["depth"].add_nd_timeseries(
    td, ("l", "time", "xy"), ("l", "time", "xy"),
    graphic_type=fpl.ScatterCollection,
    slider_maps={"time": spike_times_s},     # searchsorted into the spike list
    display_window=1.0,
    max_display_datapoints=1_000_000,        # spikes are cheap points; allow many
    x_range_mode="auto",
    sizes=3,
    colors="y",
)
```

`max_display_datapoints` must be raised for spike scatters — the default of 1000 will decimate them
into meaninglessness. Set `edge_width=0` on the graphic; marker edges dominate at these sizes.

## Continuous extracellular traces

Wrap the recording in an `NDSlicer` subclass so the traces are read lazily for the current window
only:

```python
class RecordingSlicer(NDPositionsSlicer):
    @property
    def shape(self):
        return {
            self.display_dims[0]: self.data.get_num_channels(),
            self.display_dims[1]: self.data.get_num_samples(),
            self.display_dims[2]: 2,
        }

    async def get(self, indices):
        s = self._get_dw_slice(indices)                    # the display-window slice
        xs = self.data.get_times()[s]
        ys = self.data.get_traces(start_frame=s.start, end_frame=s.stop)[:: s.step]
        return {"data": np.stack([np.broadcast_to(xs[:, None], ys.shape), ys]).T}
```

then `add_nd_timeseries(recording, ..., slicer=RecordingSlicer, graphic_type=fpl.ImageGraphic)` for
a channel × time heatmap, or `LineStack` for stacked traces. `graphic_type` can be switched live, so
offer both. Set `vmin`/`vmax` explicitly (`-5, 5` for z-scored, or in µV) — an estimate from a
subsample of a probe recording is meaningless.

Pass `ranges` explicitly — a recording is not an `ArrayProtocol`, so no dim is auto-ranged from it.

## Calcium imaging: movie plus traces plus ROIs

Three subplots that share a reference index, plus a selection that links them:

```python
ndg_movie = ndw["movie"].add_nd_image(
    dmr.ac_array, ("time", "m", "n"), ("m", "n"),
    slider_maps={"time": dmr.timings},
    graphic_kwargs={"cmap": "gray"},
)
ndg_traces = ndw["traces"].add_nd_timeseries(
    fpl.utils.heatmap_to_positions(dff, xvals=dmr.timings),
    ("cell", "time", "xy"), ("cell", "time", "xy"),
    slider_maps={"time": dmr.timings},
    display_window=30.0, x_range_mode="auto",
    graphic_type=fpl.LineCollection,          # not LineStack: VisibilitySelector needs a collection
)

# ROI footprints preloaded once as selection options
roi_selector = fpl.ImageHighlightSelector(
    lut="tab10", lut_wrap="repeat",
    selection_options={"pixels": contours},   # list of [K, 2] row/col arrays, one per ROI
    options_color="w", options_alpha=0.1,     # unselected ROIs faint
    alpha=0.7,
)
roi_selector.add_graphic(ndg_movie.graphic)

trace_visibility = fpl.VisibilitySelector(ndg_traces.graphic, lut="tab10", lut_wrap="repeat")

# one master selection drives both; a bare selector means an identity mapping
sv = fpl.SelectionVector()
sv.add_selector(roi_selector)
sv.add_selector(trace_visibility)

@ndg_movie.graphic.add_event_handler("double_click")
def pick_roi(ev):
    col, row = ev.pick_info["index"]              # note: (col, row), not (row, col)
    i = int(np.argmin(np.linalg.norm(roi_centers - (row, col), axis=1)))
    if "Shift" in ev.modifiers:
        sv.append(i)
    else:
        sv.selection = [i]
```

Two things make this fast: the ROI masks are uploaded once as `selection_options`, and the highlight
is a GPU buffer write, not a recolor of the image data.

For multi-session alignment, register one selector per session with a mapping from the master index
to that session's local index — a 1-D array (`array[master] -> local`), a dict, or a
`(forward, inverse)` pair of callables. A `(selector, single_callable)` 2-tuple raises.

**Highlighting lines and scatters does not work right now.** `PositionsHighlightSelector` raises and
`CollectionHighlightSelector` silently does nothing, because no line or scatter graphic is built
with a highlightable material. Until that is fixed, use `VisibilitySelector` on a **collection**
(not a stack, which raises), or write the colors directly:
`ndg_traces.graphic.colors[selected] = "w"`. The image-side selectors are unaffected.

## Behavior video

```python
vid = AsyncVideoReader(path, buffer_size=512)
ndg = ndw["behavior"].add_video(
    vid,
    dims=("time", "m", "n"),
    display_dims=("m", "n"),
    slider_maps={"time": vid.time},
    name="video",
)
ndw["behavior"].subplot.tooltip.enabled = False    # pixel values are not interesting here
ndw.figure["behavior"].axes.visible = False
```

For an RGB mp4 read through `decord`/`LazyVideo`, use `add_nd_image` with the colour dim named and
declared:

```python
ndw["behavior"].add_nd_image(
    vid, dims=("time", "m", "n", "rgb"), display_dims=("m", "n", "rgb"),
    rgb_dim="rgb", slider_maps={"time": frame_times}, compute_histogram=False,
)
```

Omitting `rgb_dim` on 4D data makes fastplotlib treat it as a volume.

To let the user switch between camera angles in place, swap the reader on the existing graphic:

```python
@ndw["behavior"].subplot.add_imgui_window(location="top", size=36, title=None)
def camera_choice(subplot):
    global current
    for name in readers:
        if imgui.radio_button(name, current == name) and name != current:
            ndw["behavior"]["video"].data = readers[name]
            current = name
        imgui.same_line()
```

## Pose tracking / keypoints

Tracking output (DeepLabCut, SLEAP, Lightning Pose) is a DataFrame of `<keypoint>_x`,
`<keypoint>_y`, `<keypoint>_likelihood` columns. Reshape it into `[n_keypoints, n_frames, 2]`:

```python
keypoints = ["nose_tip", "paw_l", "paw_r", "tongue_end_l", "tongue_end_r"]

xy = np.dstack([
    np.stack([df[f"{k}_x"].values for k in keypoints]),
    np.stack([df[f"{k}_y"].values for k in keypoints]),
]).astype(np.float32)

nd_kp = ndw["behavior"].add_nd_scatter(
    xy, ("kp", "time", "xy"), ("kp", "time", "xy"),
    slider_maps={"time": df["times"].values},
    display_window=5.0,                                   # 5 seconds of positions at a time
    cmap="tab10",                                         # a color per keypoint
    name="keypoints",
)
```

Overlay this on the video subplot — same subplot, same reference index — and the keypoints track the animal.

`PandasSlicer` reads the columns straight off the DataFrame, so you can skip building the array. A
DataFrame has no dims of its own to name, so `dims` and `display_dims` are the same three names and
the next positional arg is `columns`, one `(x_col, y_col)` tuple per keypoint. `tooltip_columns`
puts that keypoint's likelihood in the tooltip:

```python
nd_kp = ndw["behavior"].add_nd_scatter(
    df, ("kp", "time", "xy"), ("kp", "time", "xy"),
    [(f"{k}_x", f"{k}_y") for k in keypoints],
    slicer=ndp_extras.Pandas,
    slider_maps={"time": df["times"].values},
    display_window=5.0, cmap="tab10", name="keypoints",
    slicer_kwargs={"tooltip_columns": [f"{k}_likelihood" for k in keypoints]},
)
```

Pass `ranges` explicitly here — a DataFrame is not an `ArrayProtocol`, so no dim is auto-ranged from
it, and a dim with no range raises `KeyError`.

**Use the tracking likelihood as alpha** so low-confidence points fade instead of lying to the
viewer. A windowed feature callable receives the data slice and the display-window slice and returns
one value per displayed datapoint:

```python
kp_colors = cmap.Colormap("tab10").lut(len(keypoints))

def alpha_from_likelihood(data, dw_slice):
    p = dw_slice.stop - dw_slice.start
    colors = kp_colors[:, None, :].repeat(p, axis=1)      # [n_keypoints, p, 4]
    for i, k in enumerate(keypoints):
        colors[i, :, -1] = likelihoods[k][dw_slice]
    return colors

ndw["behavior"].add_nd_scatter(..., colors=alpha_from_likelihood)
```

## Ethograms and behavioral state

A behavioral state raster is a heatmap of small integer codes, one row per behavior:

```python
# [n_behaviors, n_times, 2]: x is time, y is the state code (0 = none)
eth = np.dstack([np.broadcast_to(times[None], codes.shape), codes]).astype(np.float32)

nd_eth = ndw["ethogram"].add_nd_timeseries(
    eth, ("behavior", "time", "xy"), ("behavior", "time", "xy"),
    graphic_type=fpl.ImageGraphic,
    slider_maps={"time": times},
    display_window=10.0, x_range_mode="auto",
    graphic_kwargs={
        # a discrete colormap, one color per state, with a clim that pins code k to color k
        "cmap": cmap.Colormap(["white", "green", "orange", "red"]),
        "vmin": 0,
        "vmax": 3,
    },
)

def state_name(pick_info):
    col, row = pick_info["index"]
    code = round(nd_eth.graphic.data[row, col])
    return {0: "none", 1: "still", 2: "lick", 3: "groom"}[code]

nd_eth.graphic.tooltip_format = state_name
```

A **qualitative** colormap with an integer code and an explicit `vmin`/`vmax` (or
`cmap_range=(0, cmap.num_colors)` on a positions graphic) is what makes code *k* always the same
color. A sequential colormap here is a bug, not a style choice.

Label the rows with the behavior names via `subplot.axes.y.tick_format`, not a legend.

For annotation *entry*, the working shape is: a dataframe mirrored to a csv, a rasterized array for
display, an `ImguiWindow` form, and a double-click on the video or the ethogram to create or edit an
entry.

## Audio / vocalizations

```python
f, t_spec, spec = scipy.signal.spectrogram(audio, fs=fs, nfft=512, nperseg=512, noverlap=256)
f = f[1:][::-1]                       # drop DC, high frequencies at the top
spec = np.log(np.abs(np.flip(spec[1:], axis=0)) + 1e-12).astype(np.float32)

ndg = ndw["spectrogram"].add_nd_timeseries(
    np.dstack([np.broadcast_to(t_spec[None, :], spec.shape), spec]),
    ("freq", "time", "xy"), ("freq", "time", "xy"),
    graphic_type=fpl.ImageGraphic,
    slider_maps={"time": t_spec},
    display_window=5.0, x_range_mode="auto",
    graphic_kwargs={"cmap": "viridis", "metadata": {"f": f}},   # carry the frequency axis along
)

subplot = ndw.figure["spectrogram"]
subplot.axes.y.tick_format = lambda v, lo, hi: f"{round(f[min(max(round(v), 0), f.size - 1)] / 1e3)} kHz"
subplot.camera.maintain_aspect = False
ndg.graphic.tooltip_format = lambda pi: f"freq: {f[pi['index'][1]] / 1e3:.1f} kHz"
```

The row index is not a frequency, so **always** convert it in the tick formatter and the tooltip.
Stash the frequency axis in `metadata` so the graphic carries what its formatters need.

For LFP, use pynapple's spectral tools (power spectral density, Morlet wavelets) rather than
`scipy.signal` — they respect the `IntervalSet` and the timebase.

## Trial-aligned responses

```python
peri = nap.compute_perievent(spikes, tref=stim_times, minmax=(-0.5, 1.5))   # verify signature
psth = ...                                                                   # per-trial counts

figure[0, 0].add_line_stack(
    fpl.utils.heatmap_to_positions(psth, xvals=lags),
    separation=(0, 1, 0), cmap="viridis",
)
figure[0, 0].add_inf_line(np.array([0.0]), axis="x", colors="r", dash_pattern="--")   # stimulus onset
```

An `add_inf_line` at t=0 is the cheapest way to make an aligned plot readable.

## Volumetric and multi-plane imaging

```python
ndw[0, 0].add_nd_image(
    volume_movie, ("time", "z", "m", "n"), ("m", "n"),        # one plane; z becomes a slider
    slider_maps={"time": frame_times},
    graphic_kwargs={"cmap": "gray"},
)
```

Leaving `z` out of `display_dims` turns it into a second slider, which is what you usually want for
multi-plane imaging — the user scrolls depth and time independently.

Putting `z` **in** `display_dims` renders an `ImageVolumeGraphic` instead, so the whole stack is
drawn at each timepoint and only `time` stays on a slider:

```python
ndw[0, 0].add_nd_image(
    volume_movie, ("time", "z", "m", "n"), ("z", "m", "n"),   # a volume per timepoint
    slider_maps={"time": frame_times},
    graphic_kwargs={"cmap": "gray"},                          # `mode` defaults to "mip"
)
```

---

# Part 5 — Out-of-core

Sessions are larger than RAM and much larger than VRAM. The rules:

1. **Pass the lazy object, not an array.** `asyncvideo`, `decord`, zarr, HDF5, a
   `spikeinterface.BaseRecording`, a masknmf GPU array — anything with `dtype`, `ndim`, `shape` and
   `__getitem__` works.
2. **Always set `display_window`** on positions/timeseries subplots, in reference units. This is the
   single setting that makes a 40-minute recording viewable.
3. **`max_display_datapoints`** caps points per graphic by decimating the window. Default 1000; raise
   it for spike scatters, leave it low for dense traces.
4. **`window_funcs`** reduces over a slider dim on the fly — `{"time": (np.mean, 2.5)}` averages ±1.25
   s around the current position. The function must take `axis` and `keepdims` and must not drop the
   dimension.
5. **Do not subsample by hand** to make something fit. That throws away the data the user wanted to
   look at and defeats the windowing.
6. On a multi-GPU machine, select the adapter before creating any figure:

   ```python
   adapter = fpl.enumerate_adapters()[0]
   print(adapter.info)
   fpl.select_adapter(adapter)
   ```
