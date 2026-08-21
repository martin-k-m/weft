# What weft needs from twill

weft is written in twill. This file was the reason it did not run: the language
and runtime features the source uses that `mode systems` did not provide, with
the file and function that needs each one and what weft does in the meantime.

Most of it is now history. twill 1.7 closed every blocking entry, and the whole
test suite passes under 1.7.1. The entries are kept, in their original order and
numbering, with what actually shipped recorded against each one, because a work
queue that deletes its finished items stops being evidence that writing real
code against a young language is how the queue got written.

It is a work queue for the language, not a complaint. Every entry was reached by
writing real code and hitting the wall, which is the only way a list like this is
worth anything. Where there is a workaround it is described, because how ugly a
workaround is says how badly the feature is wanted.

A fourteenth entry belongs at the top and is not numbered with the rest, because
weft never wrote it down until 1.7 made it moot: **a literal as a `match`
pattern.** `src/theme.tw:33` and `src/svg.tw:35` dispatch a palette index on
`0`, `1`, `2`, `_`. Before 1.7.0 a pattern was a case name and at most one
binder, and both files were a parse error: `expected identifier but found "0"`.
This is the one place in the ecosystem where twill 1.7.0 is a hard floor rather
than a cautious pin.

The baseline is milestone 1 of `docs/self-hosting.md` in the twill repository:
`mode systems`, `I64`, `Str` with length, byte indexing and slicing, `Arr[T]`,
`Dict[Str, V]`, `struct`, and `read_file`. Everything below is beyond that.

## Delivered: what used to block weft from drawing anything

### 1. F64 as a first-class type in the systems subset

**Needs:** `F64` values, literals, arithmetic, comparison, `%`, and the
conversions `f64(I64)` and `i64(F64)`
**Used by:** every source file. `src/scale.tw` alone is arithmetic on F64 from
top to bottom.
**Status:** **delivered.** `F64` literals, the four operations, comparison, `%`,
`f64(I64)` and `i64(F64)` all work in `mode systems` under 1.7.1. `i64(-2.7)` is
`-2`, so truncation is toward zero as `src/scale.tw` assumes.

Originally: `docs/self-hosting.md` named `F64` once, as an enum payload in the
token example, and specified nothing else about it. Section 1.2 was about `I64`.

A plotting library is float arithmetic with some string formatting attached. The
systems subset was designed around a compiler, where integers are the whole job,
and the result is that the one numeric type a plot needs is the one the subset
does not describe. Nothing here needs anything exotic: the four operations,
comparison, `%`, and explicit conversion in both directions.

`src/scale.tw` also depends on `i64()` truncating toward zero, which is stated,
and on that being the only rounding the language does implicitly, which is not.
`floor_div` and `ceil_div` exist because truncation floors positives and ceilings
negatives, and an axis that behaves differently either side of zero clips the
bottom of any plot that crosses it.

### 2. The float math builtins, in systems mode

**Needs:** `log`, `exp`, and NaN and infinity as values that can be produced,
compared and detected
**Used by:** `src/chart.tw` (`project_y`, the log axis), `src/bars.tw`
(`cbrt`, `sturges`), `src/fmtnum.tw` (`is_nan`, `is_inf`)
**Status:** **mostly delivered.** `log`, `exp`, `sqrt`, `pow`, `floor` and `abs`
all take a systems-mode `F64` under 1.7.1, and infinity is producible (`1.0/0.0`
prints `+Inf`) and comparable. Two gaps remain, both small:

- No `is_nan` or `is_inf` builtin. `src/fmtnum.tw:135` still detects infinity by
  comparing against `1.0e308`, which is a guess about the representation. A NaN
  loss is the most important event in a training run and weft goes to some
  trouble to keep it visible, so detecting one should not rest on a magic
  constant. NaN itself is fine: `v != v` is correct and idiomatic.
- No `cbrt`. `src/bars.tw:205` defines it as `exp(log(v) / 3.0)`, which is
  adequate for choosing a histogram bin width and would not be for anything
  else.

`src/fmtnum.tw` detects NaN with `v != v` and infinity by comparing against
`1.0e308`. The first is correct and idiomatic; the second is a guess about the
representation and should be a builtin. A NaN loss is the most important event in
a training run and weft goes to some trouble to keep it visible, so detecting one
cannot rest on a magic constant.

### 3. `chr(I64) -> Str`

**Needs:** a byte as a one-byte string
**Used by:** `src/canvas.tw` (`braille`), `tests/chart_test.tw`
**Status:** **delivered.** `chr` is a builtin under 1.7.1. `src/canvas.tw:187`
builds each braille glyph from three `chr` calls, and `tests/chart_test.tw`
asserts a plain-terminal render contains no `chr(27)`.

Braille glyphs are U+2800 plus an eight-bit dot mask, encoded to UTF-8 by hand as
three bytes. Without `chr` there is no way to produce a character from a computed
code point and the entire braille canvas has to become a 256-entry lookup table
of literals.

### 4. A clock

**Needs:** `now_ms() -> I64`, monotonic
**Used by:** `src/live.tw` (`push`), `examples/loss.tw`
**Status:** **delivered, under another name.** `mono_ns()` is monotonic and in
nanoseconds. `src/live.tw:122` is `fn push_now(l, value) = push(l, value,
mono_ns() / 1000000)`, and `push` keeps its `now_ms` parameter so
`tests/live_test.tw` can choose the time and assert on the frame limiter.

Every rate limit in the live plot is expressed in milliseconds, and twill's own
`src/term/frame.tw` and `src/cli/progress.tw` take a `now_ms` argument for the
same reason. Passing the time in from the caller is the right shape for testing
and the wrong shape for a training loop, which has no reason to know what time it
is. It must be monotonic: a wall clock that steps backwards over an NTP
correction makes the repaint limiter refuse to paint for as long as the step.

### 5. Writing to standard output and to a file

**Needs:** `write_out(Str)`, `write_file(path, Str)`
**Used by:** `examples/loss.tw`
**Status:** **delivered.** `write_out` and `write_file` both exist.
`twill run examples/loss.tw` exits 0, logs 80 lines to stdout and writes
`examples/loss.svg`.

Nothing in `src/` writes anything: every function returns a string, following
the rule twill states for `src/term/`, which is what lets a whole chart be
rendered into a buffer and compared in a test. So this blocks only the examples
and the SVG export, not the library. It is listed because a plotting library
whose plots cannot leave the process is not finished.

## Painful: written around, badly

### 6. The bitwise operators, written down

**Needs:** the language guide to state that `and`, `or`, `xor`, `shl` and `shr`
on two `I64` values are bitwise, and what `shr` does to a negative one
**Used by:** `src/canvas.tw` (`set_bit`, `bit_set`)
**Status:** **open, and now measured rather than guessed at.** Under 1.7.1,
`12 xor 10` is `6`, `12 shl 2` is `48` and `-8 shr 1` is `-4`, so those three are
bitwise and `shr` is arithmetic. But `12 and 10` is `10` and `12 or 10` is `12`:
`and` and `or` are the short-circuiting logical operators, returning an operand
rather than a bitwise result. There is no other spelling -- `&` and `|` are
syntax errors. So section 1.2's "Bitwise `and or xor shl shr not` on I64" is
wrong about two of the six, and the guide still does not cover any of them.

Setting one bit in a dot mask is therefore written as `mask + pow2(bit)` guarded
by a test, and reading one as `(mask / pow2(bit)) % 2 == 1`, which is a loop and
a divide where the hardware has an instruction, on the hottest path in the
library. That is a lot of caution to pay for an unwritten rule. Either the guide
says it, or the operators get distinct spellings; weft does not mind which.

### 7. String building is quadratic

**Needs:** a growable string buffer, or `Bytes` with `push` and a conversion
**Used by:** `src/chart.tw` (`cells`, `render`), `src/svg.tw` (`render`)
**Status:** **open.** `Bytes` is in section 1.2 with `push`, and it does not
exist in 1.7.1: `Bytes()` is `shape error: unknown name "Bytes"`. Whether the
`Str` returned by concatenation is copied each time is still unspecified, and on
the obvious implementation it is.

A frame of a live plot is built by repeated `out = out + piece` across a few
hundred pieces. At 30 repaints a second for six hours that is the difference
between a plot and a plot that is also a benchmark. `Bytes` landing with a cheap
conversion to `Str` fixes it without a new type.

### 8. No sort

**Needs:** a sort on `Arr[F64]`, or a comparator-taking sort on `Arr[T]`
**Used by:** `src/bars.tw` (`sorted_copy`, for the quartiles that set the
histogram bin width)
**Status:** **open.** 1.7.1 has a systems-mode `sort`, but only on strings:
`sort` on an `Arr[F64]` is `runtime error: sort on a list expects every element
to be a string`. numeric mode has `sort`, `argsort` and `topk` on tensors. There
is still no ordering operation for `Arr[F64]` and no comparator-taking sort.

weft ships an insertion sort. It is correct and it is quadratic, and it is called
on the caller's entire sample. It is acceptable only because a histogram is drawn
once rather than per frame, and that is not a property to rely on.

### 9. Immutable top-level bindings

**Needs:** a `const`, or `let` at the top level being read-only
**Used by:** `src/canvas.tw` (`QUADRANTS`), `src/theme.tw` (`DENSITY`),
`src/sparkline.tw` (`LEVELS`), `src/svg.tw` (`HEX`)
**Status:** **open.** There is no `const` in 1.7.1: `const K: I64 = 1` is a
syntax error at the top level and inside a function. `Arr` has reference
semantics and `let` binds a handle, so every one of these glyph tables is
writable by any importer.

These are lookup tables. A library whose palette can be reassigned by a caller,
accidentally or otherwise, has no way to keep the promise the theme file makes
about which colour means what.

### 10. Optional and named arguments, or record update

**Needs:** either default parameter values, or an update expression such as
`Chart { ..c, log_y: true }`
**Used by:** `src/chart.tw` (`chart`, `fix_y`), `src/svg.tw` (`box`)
**Status:** **open.** In 1.7.1 a default parameter value is a syntax error at
the `=`, and `P { ..p, b: 3 }` is a syntax error at the `.`. Neither is in the
design.

A chart has a dozen settings and almost every caller changes two of them. The
constructor therefore takes the three that are always given and the rest are
mutated onto the struct afterwards, which means every optional setting is a
statement rather than an argument, and no configuration can be built by a pure
expression. `fix_y` exists purely to give two of those mutations a name.

## Would improve it

### 11. `src/term/` reachable as a standard-library module

**Needs:** `import "std/term/caps"`, or some other stable name for twill's
terminal primitives
**Used by:** `src/canvas.tw`, `src/chart.tw`, `src/theme.tw`, `src/live.tw`,
and every test
**Status:** **delivered, and it is the entry that changed the repository.**
`std/term/caps`, `ansi`, `width`, `box` and `frame` are all importable under
1.7.1. Every file that used to reach into `../twill_modules/twill/src/term/`
imports `std/term/...` instead, and a working checkout has no `twill_modules/`
directory at all.

Originally weft imported twill's terminal code as
`../twill_modules/twill/src/term/caps.tw`, out of the copy spool vendors. That
worked and it hard-coded twill's internal directory layout, which twill had
never promised. The alternative was to copy the capability ladder into weft, and
two implementations of what `NO_COLOR` means is exactly the failure the shared
file prevents.

### 12. A test runner

**Needs:** a `twill test` that collects `tests/*_test.tw`
**Used by:** everything in `tests/`
**Status:** **delivered.** `twill test tests` collects all six suites and prints
`6 file(s): 6 passed, 0 failed`. CI runs that one line instead of a hand-kept
list, so a new test file is picked up by adding it.

Every test file is still a program that calls its cases at the bottom and ends
with `report`, which exits non-zero on a failure. That shape is what the runner
collects, so it stayed.

### 13. Multiple return values

**Needs:** tuples, or destructuring a returned struct
**Used by:** `src/scale.tw` (`Span`), `src/heatmap.tw` (`Range`),
`src/canvas.tw` (`Cp` in twill's own `width.tw`, for the same reason)
**Status:** **open.** `(1, 2)` is a syntax error in 1.7.1. Not in the design,
and a struct is the stated answer.

Every function that computes a low and a high declares a two-field struct to hand
them back. It works, and it puts four single-use type names in a library that has
eleven real ones. Low priority: this is a readability complaint, not a wall.
