# What weft needs from twill

weft is written in twill and does not run yet. This file is the reason: the
language and runtime features the source uses that `mode systems` does not
provide today, with the file and function that needs each one and what weft does
in the meantime.

It is a work queue for the language, not a complaint. Every entry was reached by
writing real code and hitting the wall, which is the only way a list like this is
worth anything. Where there is a workaround it is described, because how ugly a
workaround is says how badly the feature is wanted.

The baseline is milestone 1 of `docs/self-hosting.md` in the twill repository:
`mode systems`, `I64`, `Str` with length, byte indexing and slicing, `Arr[T]`,
`Dict[Str, V]`, `struct`, and `read_file`. Everything below is beyond that.

## Blocking: weft cannot draw anything without these

### 1. F64 as a first-class type in the systems subset

**Needs:** `F64` values, literals, arithmetic, comparison, `%`, and the
conversions `f64(I64)` and `i64(F64)`
**Used by:** every source file. `src/scale.tw` alone is arithmetic on F64 from
top to bottom.
**Status:** `docs/self-hosting.md` names `F64` once, as an enum payload in the
token example, and specifies nothing else about it. Section 1.2 is about `I64`.

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
**Status:** these exist in numeric mode as differentiable tensor operations. Whether
they exist on a systems-mode `F64` is unspecified.

`src/fmtnum.tw` detects NaN with `v != v` and infinity by comparing against
`1.0e308`. The first is correct and idiomatic; the second is a guess about the
representation and should be a builtin. A NaN loss is the most important event in
a training run and weft goes to some trouble to keep it visible, so detecting one
cannot rest on a magic constant.

### 3. `chr(I64) -> Str`

**Needs:** a byte as a one-byte string
**Used by:** `src/canvas.tw` (`braille`), `tests/chart_test.tw`
**Status:** twill's own `src/term/ansi.tw` calls `chr(27)`, so the twill sources
already assume it. It is not in the self-hosting builtin list.

Braille glyphs are U+2800 plus an eight-bit dot mask, encoded to UTF-8 by hand as
three bytes. Without `chr` there is no way to produce a character from a computed
code point and the entire braille canvas has to become a 256-entry lookup table
of literals.

### 4. A clock

**Needs:** `now_ms() -> I64`, monotonic
**Used by:** `src/live.tw` (`push`), `examples/loss.tw`
**Status:** not in the systems subset at any milestone.

Every rate limit in the live plot is expressed in milliseconds, and twill's own
`src/term/frame.tw` and `src/cli/progress.tw` take a `now_ms` argument for the
same reason. Passing the time in from the caller is the right shape for testing
and the wrong shape for a training loop, which has no reason to know what time it
is. It must be monotonic: a wall clock that steps backwards over an NTP
correction makes the repaint limiter refuse to paint for as long as the step.

### 5. Writing to standard output and to a file

**Needs:** `write_out(Str)`, `write_file(path, Str)`
**Used by:** `examples/loss.tw`
**Status:** in section 1.2 of the self-hosting design, not in milestone 1.

Nothing in `src/` writes anything: every function returns a string, following
the rule twill states for `src/term/`, which is what lets a whole chart be
rendered into a buffer and compared in a test. So this blocks only the examples
and the SVG export, not the library. It is listed because a plotting library
whose plots cannot leave the process is not finished.

## Painful: written around, badly

### 6. Bitwise operators that do not collide with the logical ones

**Needs:** a spelling for bitwise or, and, shift that is not `or` and `and`
**Used by:** `src/canvas.tw` (`set_bit`, `bit_set`)
**Status:** section 1.2 lists "Bitwise `and or xor shl shr not` on I64", which
are the same words as the short-circuiting logical operators.

Setting one bit in a dot mask is written as `mask + pow2(bit)` guarded by a test,
and reading one as `(mask / pow2(bit)) % 2 == 1`, rather than depending on the
operand types to disambiguate an overloaded keyword. That is a loop and a divide
where the hardware has an instruction, on the hottest path in the library. A
distinct spelling, `bitor` and `bitand` or the C operators, would remove all of
it.

### 7. String building is quadratic

**Needs:** a growable string buffer, or `Bytes` with `push` and a conversion
**Used by:** `src/chart.tw` (`cells`, `render`), `src/svg.tw` (`render`)
**Status:** `Bytes` is in section 1.2 with `push`. Whether the `Str` returned by
concatenation is copied each time is unspecified, and on the obvious
implementation it is.

A frame of a live plot is built by repeated `out = out + piece` across a few
hundred pieces. At 30 repaints a second for six hours that is the difference
between a plot and a plot that is also a benchmark. `Bytes` landing with a cheap
conversion to `Str` fixes it without a new type.

### 8. No sort

**Needs:** a sort on `Arr[F64]`, or a comparator-taking sort on `Arr[T]`
**Used by:** `src/bars.tw` (`sorted_copy`, for the quartiles that set the
histogram bin width)
**Status:** numeric mode has `sort`, `argsort` and `topk` on tensors. The systems
subset has no ordering operation on `Arr`.

weft ships an insertion sort. It is correct and it is quadratic, and it is called
on the caller's entire sample. It is acceptable only because a histogram is drawn
once rather than per frame, and that is not a property to rely on.

### 9. Immutable top-level bindings

**Needs:** a `const`, or `let` at the top level being read-only
**Used by:** `src/canvas.tw` (`QUADRANTS`), `src/theme.tw` (`DENSITY`),
`src/sparkline.tw` (`LEVELS`), `src/svg.tw` (`HEX`)
**Status:** `Arr` has reference semantics and `let` binds a handle, so every one
of these glyph tables is writable by any importer.

These are lookup tables. A library whose palette can be reassigned by a caller,
accidentally or otherwise, has no way to keep the promise the theme file makes
about which colour means what.

### 10. Optional and named arguments, or record update

**Needs:** either default parameter values, or an update expression such as
`Chart { ..c, log_y: true }`
**Used by:** `src/chart.tw` (`chart`, `fix_y`), `src/svg.tw` (`box`)
**Status:** not in the design.

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
**Status:** `std/` names modules compiled into the binary; `src/term/` is not one
of them, and every other import is a file path.

weft imports twill's terminal code as
`../twill_modules/twill/src/term/caps.tw`, out of the copy spool vendors. That
works and it hard-codes twill's internal directory layout, which twill has never
promised. The alternative was to copy the capability ladder into weft, and two
implementations of what `NO_COLOR` means is exactly the failure the shared file
prevents.

### 12. A test runner

**Needs:** a `twill test` that collects `tests/*_test.tw`
**Used by:** everything in `tests/`
**Status:** none. Same gap spool records.

Every test file is a program that calls its cases at the bottom and ends with
`report`, which exits non-zero on a failure. That is enough for CI on the day
`mode systems` runs, and it means a new test file is invisible to CI until
someone adds it to the workflow by hand.

### 13. Multiple return values

**Needs:** tuples, or destructuring a returned struct
**Used by:** `src/scale.tw` (`Span`), `src/heatmap.tw` (`Range`),
`src/canvas.tw` (`Cp` in twill's own `width.tw`, for the same reason)
**Status:** not in the design, and a struct is the stated answer.

Every function that computes a low and a high declares a two-field struct to hand
them back. It works, and it puts four single-use type names in a library that has
eleven real ones. Low priority: this is a readability complaint, not a wall.
