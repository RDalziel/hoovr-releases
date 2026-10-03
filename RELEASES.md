# Release notes

What changed in each release, in plain English, newest first. Downloads are on the
[releases page](https://github.com/RDalziel/hoovr-releases/releases).

## Unreleased

## 1.1.1

`lintr-compatible` and `styler-compatible` are still matched against lintr 3.4.0 and
styler 1.11.0, as in 1.1.0.

Linting a very large single file is much faster. A 67,000-line file took 12 seconds in
1.1.0 and now takes a third of a second, because some rules did work that grew with the
square of the file's length. Ordinary files were not noticeably affected, and every
finding is exactly the same as before.

## 1.1.0

`lintr-compatible` and `styler-compatible` are still matched against lintr 3.4.0 and
styler 1.11.0, as in 1.0.0.

Three new rules for dead code, all switched off until you ask for them:

- `unreachable-code` finds code that can never run: anything after a `return()`,
  `stop()`, `next` or `break` in the same block, and the branch that `if (FALSE)` or
  `if (TRUE) ... else` rules out.
- `unused-import` finds a `library()` or `require()` whose package the file never uses.
- `unused-function` finds a function in a package's `R/` folder that the package neither
  exports nor uses anywhere.

The first two are ports of lintr's `unreachable_code_linter` and `unused_import_linter`, and
report nothing lintr does not on real packages. They differ on purpose in a few places:
`unreachable-code` does not flag the punctuation after a nested `return()` or `stop()`,
and `unused-import` stays silent about a package it cannot read, one that ships
datasets, and data.table, bit64 and tidyverse. Turn them on with
`select = ["unreachable-code", "unused-import"]` in `hoovr.toml`, or
`hoovr lint --select unreachable-code,unused-import`.

`unused-function` is HoovR's own. R code often reaches a function through its name as
text, so the rule counts a function as used whenever its name could be meant: in
`NAMESPACE`, anywhere in the package's R code, in any string or roxygen comment there,
anywhere else in the package (tests, vignettes, documentation and the rest), or as the
start (ending in `_` or `.`) or end (beginning with one) of a name built with
`paste0()`. It also leaves alone the functions other tools call by name, such as the
`release_bullets()` that usethis reads, and a function that only names other packages'
functions so that R CMD check counts them as used. It only looks inside a package, reads the
whole package even to lint one file, and says nothing when `NAMESPACE` exports by
pattern. It reads the package's compiled code too, since C or C++ can call an R
function by name. Every function it reported in three well-known R packages was checked
by hand and is genuinely unused. Turn it on with `select = ["unused-function"]`.

`hoovr metrics` has a new row, the maintainability index: one score from 0 to 100 for
each function, from its size, branching and length. Lower is worse, and a function
scoring under 31 is over budget; 31 comes from measuring real R packages. `hoovr lint`
reports it only if you name it: `select = ["maintainability"]`. A
`health-baseline.json` from 1.0 keeps working; run `hoovr metrics --update-baseline` to
start tracking the new row.
`hoovr rules --format json` gains a `floor` field on each budget, `true` for the
maintainability index and coverage, and `hoovr rules` shows their budgets as `>=31` and
`>=80`.

A new `duplicate-bodies` row counts functions in the same file whose bodies are the same
code, ignoring spacing and comments. Each copy is reported at its own line, naming the
function it copies. The budget is none per file; `hoovr lint` reports it only if you name
it: `select = ["duplicate-bodies"]`.

A new `dynamic-evaluation` row counts code whose meaning can't be read from the source:
`eval(parse(text = ...))`, `get()`/`assign()`/`exists()` with a computed name,
`attach()`, and `<<-`. Each is reported at its own line. The budget is fewer than 4 per
file, from measuring real R packages, where `<<-` is common; `hoovr lint` reports it
only if you name it: `select = ["dynamic-evaluation"]`.

`hoovr metrics --include-dead-code` adds a `dead-code` row: the findings of
`unreachable-code` and `unused-import`, counted per file and reported at their lines.
It is off by default because it is slower and depends on which R packages are
installed; without it the row shows `-` and is never checked, so pass the flag to both
`--update-baseline` and `--check`. The budget is fewer than 2 per file, from measuring
real R packages. `hoovr lint` can't measure it and refuses to select it: select
`unreachable-code` and `unused-import` instead.

`hoovr metrics --coverage cobertura.xml` adds two rows from a coverage report you
already have, such as the one `covr::to_cobertura()` writes; HoovR never runs your
tests. `coverage` is the share of each file's lines the tests ran, and a file under 80%
is over budget. `crap` scores each function from its complexity and how much of it the
tests ran: untested, branchy code scores high, and 25 or more is over budget. Both
numbers are starting points, not yet measured against real R packages. Without
`--coverage` the rows show `-` and are never checked, so pass it to both
`--update-baseline` and `--check`. `hoovr lint` can't measure them and refuses to select
them.

`hoovr metrics --mutation muttest.json` adds a `surviving-mutants` row from a
mutation-testing report, such as the one the muttest package's `JSONMutationReporter`
writes: each small change to your code that no test noticed, listed at its line. HoovR
never runs the tests itself. The budget is none per file, a starting point not yet
measured against real R packages. Without `--mutation` the row shows `-` and is never
checked; `hoovr lint` refuses to select it.

Halstead difficulty no longer counts the `)` or `,` just after an inline function such
as `sapply(x, function(i) i + 1)`, so some functions score slightly lower; a 1.0
baseline still passes.

The `hoovr metrics` scoreboard's name column is two characters wider, so check any
script that cuts its text output by column. Its JSON output is unchanged apart from the
additions above.

## 1.0.0

The first release with a stability promise. From 1.0.0 on, a CI job keeps working
across every 1.x release, with the same commands, flags, exit codes, `hoovr.toml` keys,
JSON output and rule names. Formatter layout changes only in a minor release, listed
here, and a new rule never switches itself on ([stability.md](docs/stability.md)).

`lintr-compatible` and `styler-compatible` are matched against lintr 3.4.0 and styler
1.11.0. Measured on the release candidate over real packages with those two profiles,
HoovR agrees with lintr 3.4.0 on 99.55% of its findings and formats 95.4% of files
byte-identically with styler 1.11.0; [measurements](docs/measurements.md) has the
method and the rest.

### Things to check when you upgrade

- **An unknown rule name is now an error.** A name in `--select`, `--ignore`, or
  `select`, `ignore` or `[lint.severity]` in `hoovr.toml` that matches no rule stops
  the run with exit 2 and suggests the nearest name. Before, `--select assigment`
  switched every rule off and reported that all was well. Every command reads
  `hoovr.toml`, so a typo there now stops `hoovr format` too.
- **Out-of-range settings are errors.** Widths take 20 to 500, `indent-width` 1 to 16,
  and each budget 1 or more.
- **Code-health budgets follow the lint controls.** The `HR9xxx` findings of
  `hoovr lint` now answer to `select`, `ignore`, `[lint.severity]` and `# nolint`, as
  rule findings do. A project whose `select` names rules but no budget sees no budget
  findings any more: add the budgets' names to keep them.
- **`hoovr metrics --check` fails without a baseline.** With nothing to compare
  against, it used to pass having checked nothing.
- **Formatter layout under `styler-compatible` changed** to match styler in more
  places, and every profile but the default now keeps the project's `indent-width`,
  `indent-style` and `line-ending`. Run `hoovr format` once after upgrading.
- **The `air-compatible` profile is experimental.** Its output may change in any
  release, and no figure against air is published until one is measured. It is not
  yet idempotent on every input: where it braces a multi-line `function` passed as an
  argument before the last, a second run can change the output.
- **The R package's `style_file()` and `style_dir()` return every file.** Each row now
  has `path`, `status` and `message`, and `status` is `"reformatted"`,
  `"would-reformat"`, `"unchanged"` or `"error"`. Code that looked for
  `"would reformat"` needs the new spelling.

### New

- **`hoovr format --format json`**: what happened to each file, as one JSON document
  for tools, versioned and stable across 1.x like the lint report.
- **GitHub annotations and the R package are part of the stability promise**: the
  annotation's level, file, line and column, and the R package's functions, their
  arguments and what they return.
- **`exclude` in `hoovr.toml`**, gitignore-style patterns for what a directory walk
  leaves out. By default it leaves out R that another tool generates, such as
  `R/RcppExports.R`, so formatting an Rcpp package no longer rewrites it.
- **`hoovr lint --no-r-libs`**, and `r-libraries = false`, for findings that are the
  same on every machine: no installed R library is read.
- **renv projects**: a file inside one is checked against the project's own library
  first, as R under renv would load it.
- **`required-version` in `hoovr.toml`**, so an older HoovR says which version a
  project needs instead of reporting a key it does not know.
- **Every `hoovr.toml` key is documented**, with its default, and the example file
  lists them all.
- **A published stability promise**, and documentation you can read without access to
  the source: install, usage, rules, code health, stability and measurements, in this
  repository.
- **The R package is installable**: each release carries its source,
  `hoovr_<version>.tar.gz`. The package checks that the binary it runs is a version it
  supports, and `style_file()` and `style_dir()` take styler's `dry` argument.
- **Licence notices in every release zip**: `NOTICE` credits what HoovR ports from
  lintr, styler and rex, and `THIRD-PARTY-LICENSES` covers everything the binary is
  built with.

### Fixed

- **Columns count characters, not bytes**, as lintr and editors do, so a finding after
  an `é` on the same line is at the right column.
- **JSON and GitHub output report portable paths**: relative to the working directory,
  with `/` on every platform. JSON's summary gains `filesChecked`. GitHub annotations
  escape what the Actions runner decodes, so a message spanning lines or a Windows path
  no longer breaks one.
- **One unreadable file no longer stops a run.** A file that is not UTF-8 or cannot be
  read is reported and skipped, the other files are still checked, and the run exits 2.
- **Writes are atomic**: an interrupted `format` or `lint --fix` leaves each file either
  as it was or finished. A UTF-8 byte order mark is kept.
- **Very deeply nested R is a syntax error, not a crash.** More than 1000 levels of
  nesting used to overflow the stack.
- **`# nolint next`**, lintr's fifth spelling, works.
- **`hoovr lint --fix` no longer changes what an `=` assignment means** in the few
  places where `<-` would parse differently.
- Many lint fixes against lintr 3.4.0, among them `line-length` on CRLF files,
  `undefined-function` for replacement calls and for packages that ship with R, and
  `unused-local` inside purrr's `~ { }` lambdas.

### Removed

- **macOS binaries.** Releases are Windows x64 and Linux x86-64 only. The macOS builds
  were cross-compiled and nothing could run them before they shipped, so HoovR 1.0 does
  not support macOS.

## 0.1.1

A fix for Windows.

- **`hoovr.exe` no longer needs the Visual C++ runtime.** 0.1.0 could fail to start on
  a Windows machine without the Visual C++ redistributable installed. The C runtime is
  now built in.
- Releases now refuse to build when the tag and the version the binary reports
  disagree.

## 0.1.0

The first release: a formatter, a linter and a code-health scorer for R, in one
executable.

- **`hoovr format`** lays R out in the tidyverse style. By default it changes layout
  only, never tokens, and never loses a comment. R Markdown and Quarto documents are
  formatted inside their R chunks and left alone everywhere else.
- **`hoovr lint`** runs 39 rules, most of them lintr's default linters, two of them
  opt-in, with fixes where the edit cannot change what the code does. It honours four
  of lintr's five `# nolint` spellings.
- **`hoovr metrics`** scores cyclomatic and cognitive complexity, Halstead difficulty,
  nesting depth, parameter count and function and file length against budgets, with a
  baseline that fails a build only when code health gets worse.
- Text, JSON and GitHub-annotation output.
- Binaries for Windows x64, Linux x86-64 (static) and macOS. macOS is no longer
  published after this release.
