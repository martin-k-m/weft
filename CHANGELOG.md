# Changelog

Every entry below is a commit in this repository. Nothing is listed that I
cannot point at.

## Unreleased

Seventeen commits since `v0.1.0`, all of them either following twill's releases
or fixing what following them exposed.

### The minimum twill is now 1.7.0, and it is load-bearing

`src/theme.tw:33` and `src/svg.tw:35` dispatch the palette index on integer
literals in `match` arms. A literal in a pattern position arrived in 1.7.0;
before it a pattern was a case name and at most one binder. Under 1.6.7 the
suite gives 3 passed and 3 failed, all three of them that one construct:

```
line 22: in import "theme.tw": line 33:5: expected identifier but found "0"
line 4: in import "../src/svg.tw": line 35:5: expected identifier but found "0"
```

Moving onto 1.7 did not by itself raise the floor: the suite passed unchanged on
1.7 the day CI was pinned to it. The dispatch rewrite that came after is what
made 1.7 required.

### It runs

- The 6 suites under `tests/` pass. `twill test tests` under 1.7.1 reports
  `6 file(s): 6 passed, 0 failed`, and CI runs that rather than a hand-kept list
  of files.
- `twill run examples/loss.tw` exits 0. CI runs the examples as well as checking
  them, because `twill check` follows imports for enum declarations only and
  cannot see a call into another module whose signature changed.
- `examples/loss.tw` no longer calls `main()` itself. `twill run` executes a
  systems-mode file's top level and then calls `main()`, which it has done since
  1.6.1, so the example was training the model and drawing the curve twice.
- CI checks every `.tw` file in the repository rather than the three directories
  `src tests examples`.

### twill's terminal code stopped being a vendored path

`src/term/caps`, `ansi`, `width`, `box` and `frame` are `std/term/...` now and
are compiled into the binary. Every import that used to reach into
`../twill_modules/twill/src/term/` is a `std/` import, and a working checkout
has no `twill_modules/` directory. That was entry 11 in `docs/needs.md`.

### Types, from following 1.5 and 1.6

- `RES_BRAILLE/RES_BLOCK/RES_ASCII` became `enum Resolution { Braille, Block,
  Ascii }` and `MARK_LINE/MARK_DOT` became `enum Mark { Line, Dot }`. Six
  dispatch sites are exhaustive matches; each had ended in an `else` that
  silently meant ASCII, or a dot.
- `push_now` was added beside `live.push`, on `mono_ns` rather than a wall
  clock. `push` keeps taking the time as a parameter, because a plot whose test
  cannot control its clock has no test.
- `live_test` stopped taking twenty minutes.
- The 1.5 sweep found arithmetic that was quietly wrong: integer division
  written as `/`, which returned a float and made counts, indices and loop
  bounds fractional, and `not(x)` where the bitwise `bnot(x)` was meant.

### Prose

- Repository URLs and the module path point at the `twill-lang` org.
- `docs/needs.md` records which entries twill delivered and which are open,
  against 1.7.1 rather than against a guess.
- The README's status table names the test or example that exercises each piece,
  and says so where nothing does: heatmaps and `Mark.Dot` scatter run but are
  untested.

## v0.1.0 (2026-08-15)

Ported to twill 1.5. The library was written before `mode systems` could run it,
so this tag is the source and the design, not a working release.
