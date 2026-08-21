<p align="center">
  <img alt="twill" src="https://raw.githubusercontent.com/twill-lang/twill/main/assets/twill-mark.png" width="96">
</p>

<h1 align="center">weft</h1>

<p align="center">
  <b>Plots for <a href="https://github.com/twill-lang/twill">twill</a>: in the terminal, and out of it.</b><br>
  Written in twill.
</p>

<p align="center">
  <img alt="weft" src="https://img.shields.io/badge/weft-v0.1.0-7FE3C4?style=flat-square&labelColor=12332C">
  <img alt="written in twill" src="https://img.shields.io/badge/written%20in-twill-A8DCCB?style=flat-square&labelColor=12332C">
  <img alt="status: tests passing" src="https://img.shields.io/badge/tests-passing-D2F0E4?style=flat-square&labelColor=12332C">
  <img alt="MIT" src="https://img.shields.io/badge/license-MIT-4FB79B?style=flat-square&labelColor=12332C">
</p>

---

## It runs

`weft` is written in twill, in `.tw` files, using `mode systems`. That subset
did not exist when this library was written, so for a long time none of the code
here executed and this section said so. It runs now: the 6 test suites under
`tests/` pass, and CI runs them against a released twill on every push rather
than gating on the prose in this file.

**twill 1.7.0 is the minimum, and it is a real floor rather than a cautious
one.** `src/theme.tw:33` and `src/svg.tw:35` dispatch on a palette index using
integer literals as `match` patterns:

```rust
match i % 4 {
  0 => accent(),
  1 => pale(),
  2 => teal(),
  _ => mint(),
}
```

A literal in a pattern position arrived in twill 1.7.0; before it, a pattern was
a case name and at most one binder. Under twill 1.6.7 this same suite gives 3
passed and 3 failed, and all three failures are that one construct:

```
line 22: in import "theme.tw": line 33:5: expected identifier but found "0"
line 4: in import "../src/svg.tw": line 35:5: expected identifier but found "0"
```

`docs/needs.md` is the list of what this library asked the language for. It now
records which of those 1.7 delivered and which are still open.

## Getting started

```bash
# Assets: linux-amd64, linux-arm64, darwin-amd64, darwin-arm64,
# windows-amd64.exe.
curl -fsSL -o twill \
  https://github.com/twill-lang/twill/releases/download/v1.7.1/twill-v1.7.1-linux-amd64
chmod +x twill
./twill --version        # Twill 1.7.1

git clone https://github.com/twill-lang/weft && cd weft
../twill test tests
../twill run examples/loss.tw
```

`twill test tests` prints:

```
ok    tests/canvas_test.tw
ok    tests/chart_test.tw
ok    tests/fmtnum_test.tw
ok    tests/live_test.tw
ok    tests/scale_test.tw
ok    tests/svg_test.tw

6 file(s): 6 passed, 0 failed
```

`twill run examples/loss.tw` trains a stand-in model for 4000 steps. Attached to
a terminal it repaints the braille plot in place and prints nothing else. With
stdout redirected, as it is in CI and as it was when the block below was
captured, it takes the no-tty branch and logs one line every fifty steps rather
than emitting escape bytes:

```
step 50  loss 0.892  best 0.892
step 100  loss 0.514  best 0.514
step 150  loss 0.372  best 0.372
...
step 3950  loss 0.073  best 0.073
step 4000  loss 0.072  best 0.072
step 4000  loss 0.072  best 0.072
```

81 lines, of which steps 200 through 3900 are elided above. The last line is
step 4000 a second time: `close` flushes a final log line whether or not the
step it reports has already been logged, which on a run whose length is a
multiple of `LOG_EVERY` means the last step is printed twice.

Either way it also writes `examples/loss.svg`: the same code as the chart in the
next section, drawn from this 4000-step run rather than the 400-step one
committed under `assets/`.

## The plot it exists for

<p align="center">
  <img alt="a training loss curve, exported from weft as SVG" src="assets/loss-curve.svg" width="720">
</p>

That is the SVG export of a training run: 400 steps, the axis chosen by the same
code that draws the terminal version, the palette twill's. In the terminal the
same series is braille, repainted in place, and the axis holds still while the
numbers arrive.

## Status

Nothing is claimed to work here unless a file under `tests/` or `examples/`
runs it.

| Piece | State | Exercised by |
| --- | --- | --- |
| Tick selection on the 1/2/5 ladder, and the settling live axis | works | `tests/scale_test.tw` |
| Line series | works | `tests/chart_test.tw` |
| Scatter, `Mark.Dot` | runs, untested. Nothing constructs a `Dot` series | nothing |
| Bar charts and histograms, with bin selection | works | `tests/chart_test.tw` |
| Sparklines | works | `tests/chart_test.tw` |
| Heatmaps | runs, untested. `src/heatmap.tw` is reached only through `src/svg.tw` | nothing |
| Braille and block canvas | works | `tests/canvas_test.tw` |
| ASCII fallback, and no escape byte on a plain terminal | works | `tests/chart_test.tw` |
| `render_framed`, the rounded panel and its ASCII degradation | works | `tests/chart_test.tw` |
| The live loss curve: streaming, downsampling, in-place repaint | works | `tests/live_test.tw`, `examples/loss.tw` |
| SVG export sharing the terminal's axis | works | `tests/svg_test.tw`, `examples/loss.tw` |
| Tests | 6 suites, 6 passing, collected by `twill test tests` in CI | -- |
| Reading twill's terminal code for capability negotiation | works, and no longer needs vendoring: it is `std/term/` now | every suite |
| Interactivity, panning, zooming | **not planned.** weft draws, it does not browse | -- |
| A PNG or PDF backend | **not planned.** SVG is the export | -- |
| 3-D, contour, violin, box plots | **not planned for v0.1** | -- |

## The worked example

A training loop with a live plot. This is the whole of the API for the case
almost everyone hits first.

```rust
mode systems

import "std/term/caps" as cp
import "../src/live.tw" as live

fn main() {
  let plot = live.new(cp.detect(), "training loss", 72, 12)

  let step: I64 = 0
  while step < 4000 {
    let loss = train_one_step(step)
    write_out(live.push_now(plot, loss))   # usually returns ""
    step = step + 1
  }
  write_out(live.close(plot))
}
```

That is `examples/loss.tw` minus the SVG it also writes, so it is a program that
runs rather than a sketch. The import paths are the ones a file inside this
repository uses; a project that vendors weft with spool reaches the same module
as `twill_modules/weft/src/live.tw`. `std/term/caps` is twill's own and is
compiled into the binary.

`push_now` is called every step and returns the empty string most of the time,
because the repaint rate limit decides when a frame is due. The cost per step is
an add and a compare. It reads the monotonic clock itself; the `push` underneath
it still takes the time as a parameter, which is what lets `tests/live_test.tw`
assert anything about the limiter.

A chart built up and rendered once:

```rust
import "../src/chart.tw" as ch
import "../src/svg.tw" as svg

let c = ch.chart("val loss by learning rate", 72, 14)
c.x_label = "epoch"
c.series.push(ch.series_y("1e-3", losses_a, ch.Line))
c.series.push(ch.series_y("3e-4", losses_b, ch.Line))

write_out(ch.render(c, cp.detect()))                    # to the terminal
write_file("compare.svg", svg.render(c, svg.box(720, 320)))   # and to a file
```

Both calls take the same `Chart`. The SVG and the terminal derive their scale
from one `Axis`, so a plot in a pull request and a plot on a screen never show
different ranges of the same data.

`ch.render_framed(c, cp.detect())` is the same terminal plot inside a rounded
panel, the title set into the top border rather than printed above the plot. It
degrades to an ASCII frame where the terminal cannot draw box glyphs, and reads
as one object when a report stacks several plots down a page.

## What is actually hard here

**Ticks.** Divide a range by six and the labels come out 0.037, 0.074, 0.111,
and the reader has to do arithmetic to answer "roughly what is this". weft
picks the step from the 1/2/5 ladder and then moves the axis bounds outward onto
it. `src/scale.tw`.

**An axis that does not jitter.** Rescaling from the data on every frame makes
the whole plot slide under the reader, and every individual frame of it is
correct. weft grows the range immediately when a point falls outside it, and
shrinks only after the data has stayed inside the middle two thirds of the range
for thirty consecutive frames. `scale.settle`.

**A run longer than the plot.** A 90,000-step run cannot keep every point.
Points are folded into buckets on arrival and the buckets halve when the history
fills, so the beginning of the curve, which is the part people look at, stays on
screen. A ring buffer would show the last N steps and hide the drop.
`live.compact`.

**Degrading.** The ladder is twill's, from `docs/cli-design.md`, not a second
one invented here.

| Signal | weft draws |
| --- | --- |
| truecolour | full palette, the heatmap ramp, per-series colour |
| 256 colour | flat per-series colour; the heatmap becomes glyph density, because a quantised ramp reads as a defect |
| plain, `NO_COLOR`, not a tty | no escape bytes at all; the plot is exactly the characters it would have printed |
| unicode | braille, 2 by 4 dots per cell; eighth-blocks for bars |
| no unicode | ASCII marks, `#` bars, `.` and `:` density |
| tty | in-place repaint, identical frames never resent |
| no tty | one log line every fifty steps, never one per step |

## The dependency on twill's terminal code

twill already solved capability negotiation, colour quantising and width-aware
truncation, and `docs/cli-design.md` records the rules, including the
degradation ladder in the table above. weft uses that code rather than writing a
second copy with its own opinion about what `NO_COLOR` means.

This section used to say the code was not importable, and that weft reached it
by file path into spool's vendored copy of the twill repository, which
hard-coded twill's internal directory layout. That is over. The modules are
compiled into the binary and have a name:

```rust
import "std/term/caps" as cp
```

`src/` imports `std/term/caps`, `ansi`, `width`, `box` and `frame`, and a
working checkout has no `twill_modules/` directory in it at all. That was entry
11 in [`docs/needs.md`](docs/needs.md), and it is the one weft asked for that
changed the shape of the repository.

## Palette

twill's, unchanged.

| Role | Hex | Where |
| --- | --- | --- |
| pale | `#D2F0E4` | the second series |
| mint | `#A8DCCB` | tick labels in the SVG |
| accent | `#7FE3C4` | the first series, titles |
| teal | `#4FB79B` | axes, rules, gridlines |
| deep | `#12332C` | the SVG plate, the low end of the heatmap ramp |

Series colours are assigned accent, pale, teal, mint, in that order. Past four
series, colour has stopped being the distinguisher and the legend is doing the
work.

## Layout

```
src/
  scale.tw      ticks, bounds, projection, the settling live axis
  canvas.tw     braille, blocks, ASCII: a subcell dot grid
  chart.tw      axes, gutter, legend, the render loop
  bars.tw       bar charts and histograms, with bin selection
  heatmap.tw    a matrix as colour, as density where colour is missing
  sparkline.tw  one line of shape, no axes
  live.tw       the streaming loss curve
  svg.tw        the same charts, off the terminal
  theme.tw      the palette and what each colour means
  fmtnum.tw     axis labels, which is the only place a number is written out
tests/          six suites, collected by `twill test`. harness.tw holds the
                assertions and the counter, not the runner: the toolchain has
                one now. bars, heatmap, sparkline and theme have no file of
                their own; the first and third are covered from chart_test.tw
                and the other two are not covered at all
docs/needs.md   what weft asked the language for, and what 1.7 delivered
examples/       loss.tw, the training loop above
```

## The sibling repositories

- [twill](https://github.com/twill-lang/twill), the language.
- [spool](https://github.com/twill-lang/spool), the package manager.
- [warp](https://github.com/twill-lang/warp), data pipelines and dataset
  loaders. warp loads it, weft draws it.

## Licence

MIT.
