# Release notes

What changed in each release, in plain English, newest first. Downloads are on the
[releases page](https://github.com/RDalziel/hoovr-releases/releases).

## Unreleased

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
