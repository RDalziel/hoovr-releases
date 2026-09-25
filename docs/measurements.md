# Measurements

How closely HoovR agrees with lintr and styler, and how fast it is. Every figure here
was produced by running the reference tools themselves on the same files, never by
reading their documentation, and each is a measurement of a particular build rather
than a promise ([stability.md](stability.md#not-covered)). The figures are refreshed
when a release is re-measured.

**What was measured.** The HoovR 1.0.0 release candidate, on 25 September 2026, before
its version number was set: it reported itself as `hoovr 0.1.1`. The references were
**lintr 3.4.0** (with codetools 0.2-20) and **styler 1.11.0**, with styler's cache off,
on R 4.6.1. The files were twelve open-source R packages at fixed revisions: withr,
stringr, cli, tibble, data.table, tidyr, dplyr, testthat, lintr, styler, recipes and
devtools, 2,425 R files and 292,772 lines, plus their R Markdown and Quarto documents.

## Agreement with styler

The formatter is compared byte for byte with styler 1.11.0, with HoovR in its
`styler-compatible` profile so that a difference is a layout disagreement rather than
a known policy difference (the default profile never rewrites tokens; styler does).
styler runs with `include_roxygen_examples = FALSE`, matching HoovR, which leaves
roxygen examples alone.

| Measure | Result |
|---|---|
| Constructs, one case each | **94.3%** byte-identical (215 of 228) |
| Indentation cases at widths 3, 4 and 8 | **100%** (63 of 63) |
| Whole R files, twelve packages | **95.4%** byte-identical (2,303 of 2,414) |
| Whole R files, leaving out styler's own test inputs | **99.73%** (1,818 of 1,823) |
| R Markdown and Quarto documents | **97.8%** (178 of 182); **91.28%** counting the 13 HoovR declines to format |

A file has to agree on every construct in it, so a whole-file figure can be lower than
the construct one. By package: withr, stringr, cli, tibble, tidyr, recipes, testthat and
lintr 100%, dplyr 99.5% (193 of 194), devtools 98.9% (93 of 94), data.table 96.6% (84 of
87). data.table is in the set precisely because it is not written in the tidyverse
style. styler's own test inputs, 591 files written to exercise its rules one layout at a
time, match on 485 (82.1%). styler could not format 11 of the files, which are left out
of the counts.

## Agreement with lintr

The linter is compared with lintr 3.4.0's default linters. Findings are matched by line
and rule, on the rules HoovR implements with a declared lintr equivalent
([rules.md](rules.md) lists each), and every lintr default linter is among them.

| Measure | Result |
|---|---|
| Rule cases, one construct each | **100%** detection agreement (106 of 106 findings) |
| Real findings, `--profile lintr-compatible`, eleven packages | **99.55%** agree (13,862 of 13,925) |
| Lines lintr flags that HoovR also flags | **99.54%** (10,654 of 10,703) |
| Lines HoovR flags that lintr also flags | **99.11%** (10,654 of 10,750) |
| Real findings, the default profile | **98.64%** agree (13,735 of 13,925) |
| R Markdown and Quarto documents, `lintr-compatible` | **99.43%** agree (1,758 of 1,768) |

lintr's own package is measured but left out of these counts: only one copy of lintr
can be loaded at a time, so lintr 3.4.0 cannot resolve the names its own development
source defines.

Where the two differ on purpose, HoovR's default profile reports what R does;
`--profile lintr-compatible` reports what lintr does instead
([usage.md](usage.md#lint-profiles) lists each difference). Most of the default
profile's extra findings are `=` assignments in places lintr exempts as implicit
assignments, such as inside a call's arguments, which HoovR reports.

A few differences remain in both profiles, each a judgement rather than a gap:

| Case | lintr | HoovR |
|---|---|---|
| `x[1,]` | no space required before `]` | reported, and the fix agrees with styler's `x[1, ]` |
| `# repo: r-lib/withr`, a comment that parses as R | reported as commented-out code | not reported: it holds no call, assignment or control structure |
| `#' #' # a note` in a roxygen example | reported as commented-out code | not reported |
| A space before a comma after an argument that spans lines, `call(\n  2 ) ,` | missed | reported |
| `# nolint` inside a string, or in the middle of a comment | the line is skipped | not a directive: only a comment that starts with `# nolint` is one |
| A closure passed to `assign()` that reads the enclosing function's variable | reported | not reported, since R finds the variable |
| A method of a generic the package registers, in a helper under `tests/testthat/` | reported | not reported: the package is the nearest one |
| S3 methods of a generic imported with a whole `import(pkg)` | exempt when the running R session says the generic is one | exempt when `pkg` is one of R's own packages, or the generic is named in `extra-s3-generics` or registered with `S3method()`: a static tool cannot ask a session |
| A local variable used only inside a cli string, as `{x}` | reported as unused: lintr reads glue and rlang interpolation, not cli's | not reported, under `lintr-compatible` too: the variable is used |

Known differences not yet resolved: five `seq_linter` findings HoovR reports and lintr
does not, in the R files; ten findings lintr reports and HoovR does not, and three the
other way, in the documents; and one part of lintr's `object_usage_linter`, codetools'
check for `...` used where there is none, which HoovR does not implement.

## Speed against lintr and styler

Each tool doing its own default work over the same files: lintr its default linters,
HoovR every one of those plus its own rules and the code-health budgets; styler its
default transformers against `hoovr format --check`. On x86-64 Linux in a container
with 16 CPUs, with the files on the container's own disk and no other job running.
HoovR's time is the fastest of five runs; lintr and styler, at about a second per file,
ran once.

| Package | Files | Lines | `hoovr lint` | lintr | Ratio | `hoovr format --check` | styler | Ratio |
|---|---|---|---|---|---|---|---|---|
| withr | 58 | 4,529 | **0.022 s** | 4.39 s | 195x | **0.015 s** | 14.13 s | 939x |
| stringr | 67 | 5,616 | **0.025 s** | 7.43 s | 294x | **0.016 s** | 17.32 s | 1,083x |
| tibble | 114 | 23,371 | **0.051 s** | 13.77 s | 269x | **0.027 s** | 82.81 s | 3,034x |
| dplyr | 195 | 39,340 | **0.056 s** | 38.97 s | 697x | **0.030 s** | 148.03 s | 4,909x |

styler never finishes one of dplyr's files, `data-raw/starwars.R` (it has run for
hours), so that file was left out of styler's run. Neither side caches: styler's cache
was off, and HoovR has none. The HoovR timed here is a Linux build of the same source
as the release's static binary; timed back to back over all twelve packages, the two
were within noise of each other.

Over all twelve packages, the 2,422 R files that are UTF-8 and 292,760 lines, HoovR took
251 ms to lint with the code-health budgets off (`hoovr lint --no-health`) and 101 ms to
check formatting (fastest of five). The other three files are in a Windows code page;
HoovR reports such a file and skips it, and they were left out of the timing.

## Not measured

- **air.** The `air-compatible` format profile is experimental, and no figure against
  air is published until one is measured against a pinned air version.
- **arity and jarl.** No comparison with the other Rust tools for R is published for
  1.0: the earlier one was taken on an older build and has not been repeated.
- **macOS.** No macOS binary is published for 1.0, and none has been measured.
