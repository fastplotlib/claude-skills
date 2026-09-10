# fastplotlib-skills

A [Claude Code](https://code.claude.com) plugin that helps Claude write
[fastplotlib](https://www.fastplotlib.org) code — interactive, GPU-accelerated scientific
visualization — including multi-modal neuroscience visualization with the `NDWidget`.

Without it, LLMs tend to write fastplotlib code with matplotlib habits or other garbage: a whole
array reassigned on every frame, hand-rolled sliders, and APIs that don't exist. This plugin
supplies the current API, the performance rules, and a long list of anti-patterns.

# SCOPE, RESPONSIBILITY & SCIENTIFIC INTEGRITY

This skill helps write fastplotlib code. It cannot tell anyone whether the result is scientifically
correct. Scientific integrity is **your** responsibility as the user of this tool.

- **Verifying the scientific integrity of anything you build with LLM/AI tools is your
  responsibility.** It is not the tool's, not this skill's, and not the library's.
- **It is very easy to create visualizations that look right at first glance but are subtly
  wrong.** Examples: setting data axes in the wrong units, incorrect alignment across datasets,
  misleading colormaps that imply non-existent structure in the data, `vmin`/`vmax` that clips your data, etc.
  This is not an exhaustive list. None of these raise an error, and _none of them are bugs_.
- **LLMs sound confident.** When real humans communicate they indicate their confidence
  in various ways. LLMs give no measure of that and they are confidently incorrect, which can
  even mislead experts if they're not paying attention. LLMs also make judgement calls instead of asking
  you, unless you tell it not to repeatedly. And it will quietly do something subtly different
  from what you asked. It will produce code that runs and _looks_ reasonable,
  which you will not notice unless you already know what the right answer is supposed to look like.
- **LLMs are only tools.** They cannot validate the scientific integrity of your visualization or
  your analysis, and they cannot grasp the full scope of the scientific questions, the experiments,
  or the data you are working with.
- **You must know how to perform sanity checks for your specific datasets and experiments.** Check
  the shapes, units, sampling rates and timebases yourself. Verify any data that has to be
  cross-checked. **If you do not know how you would notice an error, you are not in a position
  to trust or verify the output**.
- **Garbage in, garbage out.** An LLM is not going to magically produce better analysis or
  visualization from bad data. It will render bad data confidently and attractively.

**Rule of thumb:** if you already know what the code should look like and *typing* or boilerplate is
your real limit, an LLM can be helpful. If you do not know what the answer is even supposed to
look like, proceed with caution and please consult an expert.

## Install

```
/plugin marketplace add fastplotlib/fastplotlib-skills
/plugin install fastplotlib@fastplotlib-skills
```

That is a one-time setup. Afterwards Claude loads the skill by itself whenever you ask for a
fastplotlib visualization — you do not need to mention it. To load it explicitly:

```
/fastplotlib:fastplotlib
```

## What is in it

`SKILL.md` holds the core material: how to choose between a `Figure` and an `NDWidget`, which
graphic to use for which data shape, the four rules that decide whether the code is fast, and the
anti-pattern table. Claude reads it whenever the skill activates.

Detailed material lives in `references/` and is read only when it is relevant:

| file | covers |
|---|---|
| `graphics.md` | every graphic type with its real defaults; collections and their accessors |
| `properties-and-events.md` | updating data efficiently, uniform vs per-datapoint buffers, events |
| `figures-and-subplots.md` | layouts, cameras, linking views, animations |
| `ndwidget.md` | n-dimensional and multi-modal viewers, custom `NDSlicer` subclasses |
| `selectors.md` | linear/region/rectangle/polygon selectors, highlighting, visibility |
| `imgui-guis.md` | sliders, buttons, colorbars, right-click menus |
| `cursors-and-tooltips.md` | `Cursor`, `Tooltip`, `TextBox` |
| `namespace-and-backends.md` | the `fpl.` namespace, notebooks vs scripts, GPU selection, transparency, coordinate spaces |
| `utils.md` | colormap and array helpers |
| `neuroscience.md` | modality-by-modality recipes, pynapple/nemos/spikeinterface integration, timebase discipline |

## License

Apache-2.0, matching fastplotlib. See [LICENSE](LICENSE).
