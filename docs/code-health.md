<!-- Generated from HoovR's documentation. Edit the source, not this copy. -->

# Code health

`hoovr metrics` scores R against seven budgets and ratchets them against a committed
baseline. This page defines each measure precisely, explains the ratchet, and then
turns the same discipline on HoovR's own Rust.

## The seven budgets

| Code | Budget | Default | Subject |
|---|---|---|---|
| `HR9001` | Cyclomatic complexity | < 22 | function |
| `HR9002` | Cognitive complexity | < 22 | function |
| `HR9003` | Nesting depth | < 5 | function |
| `HR9004` | Parameters without a default | < 6 | function |
| `HR9005` | Lines | < 100 | function |
| `HR9006` | Non-blank lines | < 500 | file |
| `HR9007` | Halstead difficulty | < 30 | function |

**A budget is an exclusive upper bound.** `cyclomatic = 22` means a function scoring
exactly 22 is over. Stated here and asserted in the tests because the other
convention is equally defensible and the two must never silently disagree.

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
  complex;
- the **count over budget** — catches a second function joining it at the same score,
  which a worst-value number would miss entirely.

An **unmeasured row never passes and never fails**. It renders `-` with the command
that would fill it in. A row that silently counts as passing is worse than no row.

A row that *improved* is reported as such, so a run can suggest refreshing the
baseline rather than leaving the slack available for someone to spend later.
