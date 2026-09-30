<!-- Generated from HoovR's documentation. Edit the source, not this copy. -->

# Code health

`hoovr metrics` scores R against budgets and ratchets them against a committed
baseline. This page defines each measure precisely, explains the ratchet, and then
turns the same discipline on HoovR's own Rust.

## The budgets

| Code | Budget | Default | Subject |
|---|---|---|---|
| `HR9001` | Cyclomatic complexity | < 22 | function |
| `HR9002` | Cognitive complexity | < 22 | function |
| `HR9003` | Nesting depth | < 5 | function |
| `HR9004` | Parameters without a default | < 6 | function |
| `HR9005` | Lines | < 100 | function |
| `HR9006` | Non-blank lines | < 500 | file |
| `HR9007` | Halstead difficulty | < 30 | function |
| `HR9008` | Maintainability index | ≥ 31 | function |
| `HR9009` | Duplicate bodies | < 1 | file |
| `HR9010` | Dynamic evaluation | < 4 | file |
| `HR9011` | Dead code | < 2 | file |
| `HR9012` | Line coverage | ≥ 80% | file |
| `HR9013` | CRAP | < 25 | function |
| `HR9014` | Surviving mutants | < 1 | file |

**A budget is an exclusive upper bound.** `cyclomatic = 22` means a function scoring
exactly 22 is over. Stated here and asserted in the tests because the other
convention is equally defensible and the two must never silently disagree. The
exceptions are the maintainability index and coverage, floors: a function, or for
coverage a file, scoring below it is over, and one scoring exactly the floor is within,
the same rule as dotnet-tools-fast.

`HR9009`, `HR9010`, `HR9011` and `HR9014` are **count rows**: the value is how many occurrences a file
holds, and the budget is an exclusive bound on that count like any other ceiling, so
`HR9009`'s default of 1 means none. When a file is at or over it, **each occurrence is reported at its own
line**, where `# nolint` suppresses it, rather than once for the file.

`HR9008`, `HR9009` and `HR9010` arrived in 1.1, so `hoovr lint` reports them only when `select`
names them; `hoovr metrics` always shows their rows. `HR9011`, also new in 1.1, is
measured only by `hoovr metrics --include-dead-code`, and `hoovr lint` refuses to
select it (see [Dead code](#dead-code)). `HR9012` and `HR9013`, also new, are read from
a coverage report: measured only by `hoovr metrics --coverage`, and `hoovr lint` refuses
to select them too (see [Coverage and CRAP](#coverage-and-crap)). `HR9014` likewise
comes from a mutation report, measured only by `hoovr metrics --mutation` (see
[Surviving mutants](#surviving-mutants)).

The numbers are not sacred. What matters is that they are fixed, visible, and
ratcheted — a threshold nobody can see is a threshold nobody meets.

## Where the numbers come from

Every threshold above was calibrated on 2026-09-21, during 0.1 development, with the
metrics of a HoovR development build from before v0.1.0, against the corpus as it was
then — 15,299 functions and 3,281 files from tidyverse, data.table, ggplot2, shiny and
nine more packages — to flag roughly the worst 1% of code written by people who are
good at R. The figures below are that calibration, kept as the record of why each
budget was chosen; they were not re-measured for 1.0, and the budgets have not moved
since. A budget that fires on one file in six of the best code in the language is not
a budget, and one that never fires is not either.

| Budget | % of corpus over it | Note |
|---|---|---|
| cyclomatic < 22 | 0.6% | |
| cognitive < 22 | 0.9% | |
| Halstead < 30 | 1.1% | 80 flagged **2 functions in 15,299** |
| nesting depth < 5 | 0.1% | |
| parameters without a default < 6 | 1.3% | counting *declared* formals flagged 7.3% at the same number |
| function lines < 100 | 0.9% | 50 flagged 3.7% |
| file lines < 500 | 4.2% | 250 flagged 16.2% |

Three of these were briefly set to `dotnet-tools-fast`'s numbers — `DF9002`'s 50 lines
per function, `DF9003`'s 250 per file, `DF9006`'s Halstead 80 — on the argument that two
tools should not hold users to different numbers for the same budget. Measuring said
otherwise, and the reasons are specific to R rather than general:

- **Length measures declarative bulk here more than logic.** Of the 559 corpus functions
  over 50 lines, 25% have a cyclomatic complexity of 1 and 57% are under 10. The longest
  are literal structures — a shiny UI tree at 320 lines, a cli theme at 182 — not
  tangled control flow. Function chaining is one instance of the same pattern; only 15%
  of corpus files contain a multi-line pipeline, so it is not the main driver.
- **R's parameter idiom is a long list of defaulted named arguments.**
  `data.table::fread()` declares 36 formals and defaults 35 of them; counting formals
  scored it 36, counting only the ones without a default scores it 1. At a budget of
  six that change rescues 796 of the 978 functions the old measure flagged. It is not
  the same as counting what a caller must pass — R only errors on a missing argument if
  the body evaluates it, so `ggplot2::theme()`'s 146 default-less formals are all
  optional and it still reports 146, which is the right answer for a 146-name
  signature.
- **Halstead difficulty does not reach C# levels in R.** A call-heavy language with a
  small operator vocabulary tops out near 31 at the 99th percentile.

File lines is still the loosest row at 4.2%, and deliberately: R files are genuinely
long (the 95th percentile is 459 non-blank lines), and pushing the budget to where it
flags 1% would put it near 1,000, which is not a constraint.

### The 1.1 budgets

Calibrated on 2026-09-28 with a development build of HoovR 1.1.0, all four rows in one
run over the same corpus, by the same rule: the integer budget whose share is closest
to 1%, the lower on a tie. A count row's share is of files at or over a per-file
budget. The corpus had changed since the 0.1 calibration and was not the same set: 12
packages (cli, data.table, devtools, dplyr, lintr, recipes, stringr, styler, testthat,
tibble, tidyr and withr), 2,595 files and 8,675 functions. ggplot2 and shiny were no
longer in it.

| Budget | % of corpus over it | Note |
|---|---|---|
| maintainability index ≥ 31 | 1.03% of functions | Visual Studio's 20 flags 0.13%, 11 functions in 8,675; 30 flags 0.82% |
| duplicate bodies < 1 | 0.62% of files | 23 copies in 16 files; a budget of 2 flags 0.23% and 3 flags 0.04%, so 1, none, is both the nearest to 1% and the smallest budget there is |
| dynamic evaluation < 4 | 0.89% of files | 302 occurrences in 111 files: 191 `<<-`, 43 `get()`, 31 `assign()`, 17 `exists()`, 11 `mget()`, 7 `eval()` of parsed text, 1 `get0()`, 1 `attach()` and no `->>`. A budget of 1, none, flags 4.28%, 3 flags 1.39% and 5 flags 0.62%. Without `<<-` and `->>`, 44 files have any (1.70%) and the nearest 1% would be 2, at 0.89% |
| dead code < 2 | 1.00% of files | With the CI image's R library, 171 installed packages: 255 findings in 60 files, 233 `unreachable-code` in 44 files and 22 `unused-import` in 18. A budget of 1, none, flags 2.31% and 3 flags 0.85%. `unreachable-code` alone flags 1.70% at 1; `unused-import` alone 0.69%. It says nothing about the 17 attached packages that were not installed |

Coverage and CRAP are not calibrated on the corpus. The corpus has no coverage reports,
and HoovR 1.1 shipped them uncalibrated for speed. CRAP's 25 is dotnet-tools-fast's, and
surviving mutants' 1, none per file, is dotnet-tools-fast's 0 in HoovR's exclusive terms.
Coverage's 80 rests on covr 3.6.5 run on the tests of three packages on 2026-09-30:
withr had 6 of 27 files under 80%, stringr 0 of 30, and tibble 7 of 33 (its report also
counts its C code). Every file was under 60% or over 70%, so any floor from 72 to 80
flags nearly the same files.

## The opt-in guardrails

Two rules that are not budgets and not ports of a lintr linter: they are the R
analogues of `dotnet-tools-fast`'s `DF9001` and `DF9004`, and they exist on the same
premise — an agent will ignore a style guide but cannot ignore a lint error, so the
conventions that actually shape generated code are the enforced ones.

| Code | Rule | dotnet-fast | Reports |
|---|---|---|---|
| `HR0300` | `no-comments` | `DF9001` | a `#` comment that is not documentation |
| `HR0301` | `magic-number` | `DF9004` | a numeric literal whose meaning is not named |

Both are **off unless named in `select`**, for the reason the `DF90xx` family is
opt-in: each encodes one team's answer to a question other teams answer differently,
and a linter that reported every comment in a project that never asked for it would be
uninstalled rather than configured.

`hoovr explain no-comments` and `hoovr explain magic-number` give the full exemption
lists. The short version: `no-comments` exempts roxygen, `# nolint` directives, the
roxygen2 generation marker, a shebang and a section marker; `magic-number` exempts the
allow-list (`0`, `1`, `-1`, `2`, as `DF9004`), a top-level assignment, a parameter
default and a subscript — but not a named argument at a call site, which names the
argument rather than the value.

`DF9008` (documentation only on a class or record) has no counterpart, deliberately:
roxygen attaches to whatever it documents and R has no class declaration to anchor the
equivalent rule to.

## Cyclomatic complexity

Starts at 1 and adds one for each point where control can diverge:

| Construct | Adds |
|---|---|
| `if` (each one, including each `else if`) | 1 |
| `for`, `while`, `repeat` | 1 |
| `&&`, `\|\|` (each operator) | 1 |
| each branch of a `switch()` after the first | 1 |
| `ifelse()` | 1 |
| each handler argument of `tryCatch()` / `withCallingHandlers()` | 1 |

A bare `else` adds nothing: the `if` it belongs to already counted the decision.
`&` and `|` add nothing either — they are arithmetic over vectors, not control flow.

The number is also the minimum count of test cases needed to cover the function. A
function at 34 needs 34 tests to be exercised fully, and almost certainly does not
have them. That is the argument for the budget, and it is why this measure is worth
having even though it is crude about *readability*.

## Cognitive complexity

The Sonar-style measure: it asks how hard the code is to **follow**, so nesting costs
more than breadth. Each control structure adds `1 + current nesting`, and entering one
increases the nesting inside it. An `else` or `else if` adds a flat 1 with no nesting
penalty, because a chain of alternatives is read linearly.

A *sequence* of the same logical operator adds 1 in total, not one per operator:
`a && b && c` is one condition to hold in your head, the same as `a && b`. Mixing
operators starts a new sequence and adds again.

The two measures disagree on purpose, and the disagreement is the useful part:

```r
# cyclomatic 4, cognitive 3           # cyclomatic 4, cognitive 6
if (a) 1                              if (a) {
if (b) 2                                if (b) {
if (c) 3                                  if (c) 3
                                        }
                                      }
```

Same number of paths, very different reading experience. A function can be within its
cyclomatic budget and over its cognitive one, which is the case where the budget is
telling you something a path count cannot.

## Nested functions

Measurement stops at a nested `function`: it is measured as its own function, and
folding its score into the parent would count the same code twice. A function whose
body is one large `lapply(x, function(y) ...)` therefore scores low while the closure
scores high. That is the intended reading — they are two units of code to understand,
and the report names the closure `<anonymous>` rather than guessing a name that would
send the reader to the wrong place.

## Maintainability index

One number per function, from 0 to 100, lower worse:

```text
MI = max(0, (171 - 5.2 ln V - 0.23 CC - 16.2 ln L) * 100 / 171)
```

- `V` is Halstead volume, `N log2 n`: the operators and operands the function uses in
  total, times the log of how many distinct ones it uses, with the partition Halstead
  difficulty uses.
- `CC` is its cyclomatic complexity.
- `L` is the lines the definition spans that hold anything, **comments included**,
  blanks not: the same non-blank count as `file-lines`, so spacing a function out cannot
  lower its score and packing it tight cannot raise it.

`V` and `L` are taken as at least 1, so an empty function scores exactly 100, and the
result is clamped at 0. It adds one number that falls when a function grows in volume,
branching and length together while each stays inside its own budget.

The score is **rounded to 4 decimal places**. Every other measure is exact arithmetic,
but a logarithm's last bit can differ between the platforms HoovR is built for, and
unrounded, a baseline written on Windows could "regress" by a rounding error on Linux.
Reports print one decimal place, so nothing visible is lost.

It is computed from the other measurements, not stored: the baseline records the
lowest score, and nothing else about a function changes.

## Duplicate bodies

A count, per file, of functions whose body is a copy of an earlier function's in the
same file. Two bodies are copies when they have the same parameter names, in the same
order, and the same tokens: spacing and comments do not tell them apart.

- **Parameter names are compared, defaults are not.** An R body reads its parameters by
  name, so the same code under different names reads different variables and is not a
  copy. The same code under different defaults is exactly the copy this row is for: one
  function with a parameter.
- **Only a braced body of two or more expressions counts.** A one-expression body,
  such as `NextMethod()`, `invisible(x)`, `x$name` or `UseMethod("f")`, is a delegation,
  not a copy. The same minimum as dotnet-tools-fast.
- **N copies count N - 1.** The first is the original. Each later copy is reported at
  its own line, naming the function and line it copies, with `<anonymous>` for a
  function that is not bound to a name.
- **A copy's nested functions are not counted again.** They are part of the copy.
- **Tokens are compared as written**, so `'a'` and `"a"` differ. Formatted code writes
  them the same way.
- **One file only.** Copies in different files are not compared.

## Dynamic evaluation

A count, per file, of the code whose meaning cannot be read from the source: code a
reader, or a static analyser, has to run in their head to know which variable it
touches or what it does. Each occurrence is reported at its own line.

| Counted | Why |
|---|---|
| `eval()` whose first argument is a call to `parse()`, `str2lang()` or `str2expression()`, qualified or not | It runs text as code. |
| `get()`, `get0()`, `mget()`, `assign()` or `exists()` whose first argument is not a string literal | The first argument is the variable's name; a computed one names a variable the source never writes. |
| `attach()` | It puts names no file declares on the search path. |
| every `<<-` and `->>` | They assign to whichever enclosing environment already has the name. |

| Not counted | Why |
|---|---|
| `:::` | It names exactly what it reads: a stability problem, not an analysability one. |
| `eval()` or `evalq()` of a quoted expression or a call object | The modelling functions and tidy evaluation work this way, and the code is still code. |
| `do.call()` and `match.fun()` | A computed string cannot be told apart from a function object. |
| `detach()` | It removes names rather than adding them. |
| lintr's other default undesirable functions (`library`, `source`, `setwd`, `options`, `sapply` and the rest) | They are about global state or type stability, not evaluation. |

`attach()`, `<<-` and `->>` are the evaluation-related entries in lintr 3.4.0's
`default_undesirable_functions` and `default_undesirable_operators`; `:::` is the
third default operator, left out as above.

- **Only the first argument as written is read.** `get(envir = e, x = "a")` counts,
  since its first argument is `e`; so does `mget(c("a", "b"))`, whose names are
  a call rather than a literal.
- **A parsed expression stored first is not followed.** `p <- parse(text = s)` then
  `eval(p)` is not counted: following it needs flow analysis.
- **The default is 4, not none.** `<<-` is common in R that manages state in a
  closure, and a budget of 1 would flag one file in 23.

## Dead code

A count, per file, of the findings of two lint rules, each reported at its own line:
`unreachable-code` (code after `return()`, `stop()`, `next` or `break`, or behind a
constant condition such as `if (FALSE)`) and `unused-import` (a `library()` or
`require()` of a package the file never uses). The project's `ignore`, `# nolint` and
per-package settings apply, so `hoovr lint --select unreachable-code,unused-import`
reproduces every one.

- **Opt-in: `hoovr metrics --include-dead-code`.** It lints every file, which is
  slower, and `unused-import` knows what a package exports only from its installed
  copy, so the count depends on the R libraries installed on the machine, as the
  rule's findings do. It says nothing about a package that is not installed.
- **Unmeasured without the flag.** The row renders `-` with the status
  `unmeasured: pass --include-dead-code`, never passes and never fails, and
  `--update-baseline` leaves it out. Pass `--include-dead-code` to both
  `--update-baseline` and `--check`, on machines with the same packages installed.
- **Not a `hoovr lint` budget.** The linter cannot measure it, so `select` or
  `--select` naming `dead-code` or `HR9011` is an error; select `unreachable-code` and
  `unused-import` instead. Naming it in `ignore` is harmless, and `hoovr explain
  dead-code` works.
- **The default is 2, not none.** A budget of 1 would flag one file in 43. In the
  corpus, 220 of the 233 `unreachable-code` findings are in test fixtures, nearly all
  the constant conditions of a formatter's `if (TRUE)` cases; the rest are mostly a
  `return()` or `stop()` left in to switch a function off.

### Unused functions

`unused-function` (`HR0022`) finds a second kind of dead code: a function a package
defines in `R/`, does not export, and never uses. It is an opt-in `hoovr lint` rule,
and the `dead-code` row does not count it yet. What counts as a use is in
[Rules that read the whole package](usage.md#rules-that-read-the-whole-package).

Measured on 2026-09-30 with the HoovR 1.1 rule over the twelve packages of the 1.1
calibration: 65 functions. Every finding in cli, data.table and dplyr, 10 in all, was
checked by hand against everything in its package, compiled code included, and none
is used. The first run found one that was: dplyr's C++ calls `dplyr_internal_signal`
by name, and the rule then did not read compiled code. It now reads it and counts a
name there as a use. A review then found two devtools findings live: `release_bullets`,
which usethis calls from the package's namespace, and `ignore_unused_imports`, whose
body only names `miniUI::miniPage` and the like so that R CMD check counts those
Imports as used. The rule now keeps both kinds, which also keeps testthat's
`silence_r_cmd_check`. The figures are from the corrected rule.

| Package | Functions reported |
|---|---:|
| cli | 2 |
| data.table | 1 |
| devtools | 1 |
| dplyr | 7 |
| lintr | 0 |
| recipes | 2 |
| stringr | 24 |
| styler | 2 |
| testthat | 4 |
| tibble | 19 |
| tidyr | 1 |
| withr | 2 |
| **Total** | **65** |

## Coverage and CRAP

Two rows read from a coverage report a test run already wrote. HoovR never runs tests.

```console
Rscript -e 'covr::to_cobertura(covr::package_coverage(), filename = "cobertura.xml")'
hoovr metrics --coverage cobertura.xml
```

- **Coverage (`HR9012`)** is each file's line coverage: of the lines the report counts as
  coverable in the file, the percentage the tests ran. It is a **floor**, default 80: a
  file below it is over, one exactly at it is within. It is per file rather than per
  function because most R functions have a handful of coverable lines, and CRAP already
  judges each function by its coverage.
- **CRAP (`HR9013`)** is per function: cc² × (1 − cov)³ + cc, where cc is its cyclomatic
  complexity and cov the share of the coverable lines from its first line to its last
  that the tests ran. A fully tested function scores its complexity, an untested one
  cc² + cc. A function with no coverable line counts as tested, and a nested function's
  lines count towards the function around it too. Default 25, and exactly 25 is over.
- **Rounded to 4 decimal places**, both, so a baseline does not move between platforms.
- **Repeatable.** `--coverage` can be given once per report, for shards of one package;
  a line counts as run if any report ran it.
- **Path matching.** covr names files relative to the package root (`R/add.R`). A
  measured file, made absolute, takes the report path it ends with at a `/` (or
  equals), and the longest such match wins. There is no guess by file name alone:
  `tests/testthat/utils.R` must not take `R/utils.R`'s numbers, so a report path never
  matches by ending with the measured path, which from `tests/testthat` is `utils.R`.
  A report path that two measured files end with goes to the outer one when the other
  is in a package inside it: cli's `R/test.R` beats its test fixture package's
  `tests/testthat/progresstest/R/test.R`. For two sibling packages it is an error,
  since it means two packages' `R/utils.R` in one run: measure one package at a time.
- **Hit counts** are read as numbers: covr writes them with R's `as.character()`, so a
  line run 100000 times reads `1e+05`. A count that is missing or not a number is an
  error, not a line never run.
- **A file the report does not name**, such as a test file, has no value: it neither
  lowers the coverage row nor has a CRAP score.
- **Errors.** A report that records no coverable line (covr writes one for a package
  whose tests run no R code) is an error rather than 100%; so is a report that names
  none of the measured files, and a file that is not a Cobertura report.
- **Unmeasured without the flag.** Both rows render `-` with the status `unmeasured:
  pass --coverage`, never pass or fail, and `--update-baseline` leaves them out. Pass
  `--coverage` to both `--update-baseline` and `--check`.
- **Cobertura only.** LCOV is not read: covr 3.6.5 has no LCOV writer.
- **Not `hoovr lint` budgets.** The linter cannot measure them, so `select` or
  `--select` naming `coverage`, `crap`, `HR9012` or `HR9013` is an error.
- **The defaults are not calibrated** on the corpus; see [The 1.1 budgets](#the-11-budgets).

## Surviving mutants

A row read from a mutation-testing report a test run already wrote. A mutation-testing
tool changes the code one small edit at a time (`>` to `>=`, `+` to `-`) and runs the
tests after each; a change that no test fails on **survived**, and marks behaviour
nothing checks. HoovR never runs tests.

```console
Rscript -e 'muttest::muttest(plan, reporter = muttest::JSONMutationReporter$new(path = "muttest.json"))'
hoovr metrics --mutation muttest.json
```

- **Surviving mutants (`HR9014`)** is a count row, per file. The report is read in the
  mutation-testing-elements JSON schema, which muttest's `JSONMutationReporter` writes.
- **What counts.** `Survived` and `NoCoverage` (no test ran the line) count. `Killed` and
  `Timeout` were caught. `RuntimeError`, `CompileError` and `Ignored` say nothing about
  the tests and are left out.
- **Each at its line.** Every survivor is reported at the line it changed, and listed
  under `surviving-mutants:` below the scoreboard.
- **Repeatable.** `--mutation` can be given once per report, and the reports add up.
- **Path matching** is as for `--coverage` (see [Coverage and CRAP](#coverage-and-crap)).
  A measured file the report does not name counts none.
- **Errors.** A report that scored no mutant is an error rather than a clean result; so
  is a report that names none of the measured files, and a file with no `files` object.
  muttest writes no report at all when its plan made no mutant.
- **Unmeasured without the flag.** The row renders `-` with the status `unmeasured: pass
  --mutation`, never passes or fails, and `--update-baseline` leaves it out. Pass
  `--mutation` to both `--update-baseline` and `--check`.
- **Not a `hoovr lint` budget.** `select` or `--select` naming `surviving-mutants` or
  `HR9014` is an error.
- **The default, 1, means none per file**, and is not calibrated; see
  [The 1.1 budgets](#the-11-budgets).

## Reported but never budgeted

- **Comment ratio.** There is no threshold at which it is good or bad, and a rule
  that pushed it up would be satisfied by noise.
- **Code / comment / blank line counts.** Context for the numbers above. A line with
  code *and* a trailing comment counts as code: it is a line the reader has to
  execute in their head.

## The ratchet

A budget says where a codebase should be. A baseline says where it **is**. Every real
codebase starts over budget somewhere, and a gate that fails until everything is
fixed gets switched off within a week.

```console
hoovr metrics --update-baseline    # write health-baseline.json
hoovr metrics --check              # exit 1 if anything got worse
```

The baseline records **two numbers per metric**, because either alone hides a
regression:

- the **worst value** anywhere in the measured set — catches a function getting more
  complex; for the maintainability index and coverage the worst value is the lowest;
- the **count over budget** — catches a second function joining it at the same score,
  which a worst-value number would miss entirely.

For a count row the worst value is the worst file's count, and the count over budget is
the number of occurrences in files at or over the budget. A new occurrence is a
regression when it lands in a file already at or over the budget, brings a file up to
it, or raises the worst file's count; one in a file that stays under the budget is not,
just as a function that grows but stays under its budget is not. At a budget of 1,
`duplicate-bodies`' default, every new occurrence is a regression.

An **unmeasured row never passes and never fails**. It renders `-` with the command
that would fill it in: `dead-code` without `--include-dead-code`, `coverage` and
`crap` without `--coverage`, and `surviving-mutants` without `--mutation`. A row that silently counts as passing is worse than no row.

A row that *improved* is reported as such, so a run can suggest refreshing the
baseline rather than leaving the slack available for someone to spend later.
