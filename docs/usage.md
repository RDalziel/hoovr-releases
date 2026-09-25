<!-- Generated from HoovR's documentation. Edit the source, not this copy. -->

# Usage

## Commands

```console
hoovr [PATH...]                  # lint (read-only); the default because writing should be asked for
hoovr lint [PATH...]
hoovr format [PATH...]
hoovr metrics [PATH...]
hoovr rules
hoovr explain <RULE>
```

Paths default to the current directory. A path naming a file is used whatever its
extension — an explicit argument is an instruction. A directory is walked with
`.gitignore` and `.ignore` respected (even outside a git repository), so nothing
reformats `renv/library`, and without the generated R that `exclude` names (see
[Leaving files out](#leaving-files-out)). Symbolic links met in a walk are
not followed: a link to a file is skipped, and a link loop cannot trap the walk. A
link named on the command line is used, and writing to it writes the file it points
to. A file named twice, under two spellings (`hoovr format . R/a.R`), is worked on once.

Files are read as UTF-8, and a UTF-8 byte order mark at the start is allowed: it is
not part of the R, and `format` and `lint --fix` keep it. A file that is not UTF-8,
and a file over 64 MiB, is reported on stderr (`R/old.R: not UTF-8 (line 12); re-save
as UTF-8`) and skipped. The run goes on with the other files and exits 2.

Recognised extensions: `.R` and `.r`, files named `.Rprofile`, `Rprofile` and
`.Rprofile.site`, and the R Markdown and Quarto documents `.Rmd`, `.rmd`, `.Rmarkdown`
and `.qmd`. `.Rout` files, which `R CMD BATCH` writes, are transcripts rather than R and
are not walked. In a document only the knitr chunks are R. `format` changes nothing
outside them, and `lint` reports each finding at its line in the document. `lint --fix`
reports a document's findings but does not fix them. When formatting a document from
stdin, pass `--stdin-file-path report.Rmd` so HoovR knows what it is reading.

Which chunks count as R follows the tool each command matches. `format` takes
`{r}`, `{webr}` and `{webr-r}` chunks, in any case, as styler does. `lint` takes every
chunk whose engine knitr has not registered, as lintr does: `{asciicast}` is linted,
and `{python}`, `{R}` and a chunk that sets `engine =` are not.

### Exit codes

| Code | Meaning |
|---|---|
| 0 | nothing to report |
| 1 | findings, unformatted files, or a code-health regression |
| 2 | the command could not run |

Three rather than two, so a CI job can tell "the code has a problem" from "the tool
has a problem". Exit 2 means one of:

- a path given on the command line does not exist (`hoovr: srcc: No such file or
  directory`). A gate pointed at a misspelt directory fails rather than checking
  nothing and passing.
- part of a directory could not be read. Each failure is reported on stderr as it
  happens and the rest of the run goes ahead, but the exit code is 2, because the
  files that could not be read were not checked. `metrics --update-baseline` writes
  nothing in that case.
- a file cannot be read, is not UTF-8, is over 64 MiB, or cannot be written. Each
  such file is reported on stderr and the rest of the run goes ahead: every other
  file is still formatted, linted or measured, and reported.
- a rule name in `--select`, `--ignore` or `hoovr.toml` is unknown (see
  [Configuration](#configuration)).
- `hoovr.toml` cannot be read or parsed, has an unknown key or a value out of range,
  or has a `required-version` this HoovR does not meet (see
  [Errors in the file](#errors-in-the-file)).
- `metrics --check` finds no baseline, or one written in a format version this HoovR
  does not read.
- flags that cannot go together, such as `format --stdin --check` or
  `rules --format markdown --fixable`, or a flag value out of range, such as
  `--line-length 0`.
- HoovR itself hit a bug. It prints `hoovr: internal error while processing <file>`
  and where to report it.

### `hoovr lint`

```console
hoovr lint --fix R/                        # apply every safe fix, then report the rest
hoovr lint --select assignment,commas R/   # only these rules (replaces the config's list)
hoovr lint --ignore line-length R/         # also skip these (adds to the config's list)
hoovr lint --no-health R/                  # rules only, no code-health budgets
hoovr lint --fail-on warning R/            # exit 0 unless something is at least a warning
hoovr lint --no-r-libs R/                  # read no installed R library: the same findings on any machine
hoovr lint --line-length 120 R/            # line-length's limit for this run (20 to 500)
hoovr lint --profile lintr-compatible R/   # lintr's findings where HoovR differs on purpose
hoovr lint --format json R/                # for a tool
hoovr lint --format github R/              # inline annotations on a pull request
```

`--select` and `--ignore` take rule names or codes, comma-separated
(`--select assignment,HR0104`). `--fail-on` takes `style` (any finding, the default
behaviour), `warning` or `error`. `--profile` takes `default` or `lintr-compatible`;
[Profiles](#profiles) says what the second changes.

`--fix` applies fixes in repeated passes, because one fix can uncover another
(removing a trailing `;` can leave trailing whitespace). Overlapping fixes are
applied in source order and the loser is left for the next pass, which is what makes
it converge. Only a file a fix changed is written.

`format` and `lint --fix` write each file by writing a temporary file beside it and
renaming it over the original, so an interrupted run leaves every file either as it
was or finished, never half written. A read-only file is not written.

`lint` also reports the code-health budgets (`HR9001`-`HR9007`, listed by
`hoovr rules`) as findings. They answer to the same controls as rules: `--select` and
`select` (a list that names no budget reports none), `--ignore` and `ignore`,
`[lint.severity]` (a budget is a `warning` unless it sets one), and `# nolint` on the
line the finding is reported at, which for a function is the line its definition
starts on, and for `file-lines` is line 1. `--no-health` switches every budget off,
and `[metrics]` sets the thresholds. `hoovr metrics` measures the same budgets and
ignores these lint settings.

The output formats are described in [Output formats](#output-formats).

### Installed packages

Some findings depend on what R packages the machine running `lint` has installed,
because they depend on what R would load there:

- `undefined-function` and `undefined-variable` read the exports of each package a file
  attaches with `library()` or `require()` (and of testthat, for a file in
  `tests/testthat/`), and of each package its `NAMESPACE` imports whole with `import()`.
  A package HoovR cannot find could provide any name, so the rules say nothing about
  the names read in a file that attaches one, or anywhere in a package that imports one.
- `package-not-installed` reports a `library()` of a package installed nowhere HoovR
  looks.
- `object-name` and `object-length` accept `tidy.my_class` as a method, as lintr does,
  when the package's `NAMESPACE` imports the generic: `importFrom(generics, tidy)`.
  An import from a package known not to be installed provides no generic. For R's
  own packages HoovR knows which exports are generics (`importFrom(utils, head)` is
  one, `importFrom(utils, zip)` is not); from any other package, every name imported
  with `importFrom()` counts as one. A whole `import(pkg)` provides the generics of R's
  own packages only; name another package's with `extra-s3-generics`.

HoovR does not run R, so it cannot ask R for `.libPaths()`. It looks in these places,
in order, and the first library holding a package is the one read, as in R:

1. The renv project library, for a file inside an renv project (the file's directory,
   or a directory above it, holds `renv.lock`): each `renv/library/.../R-<version>/<platform>`
   directory, newest R first. renv switches to it from the project's `.Rprofile`, which
   HoovR does not run. Unlike R under renv, HoovR still reads the libraries below after it.
   The project is found for each file, so one run over several renv projects, or from a
   directory above them, reads each project's library for its own files only.
2. The directories listed in `R_LIBS`, `R_LIBS_USER` and `R_LIBS_SITE`, in that order.
3. `library` under `R_HOME`.
4. The platform's usual places. On Windows: `%LOCALAPPDATA%\R\win-library\<version>`,
   each R installation's `library` under `%LOCALAPPDATA%\Programs\R`,
   `%USERPROFILE%\Documents\R\win-library\<version>`, and each R installation's
   `library` under `%ProgramFiles%\R`. On Linux: `~/R/<platform>/<version>`,
   `/usr/local/lib/R/site-library`, `/usr/lib/R/site-library`,
   `/usr/local/lib/R/library`, `/usr/lib/R/library` and `/opt/R/<version>/lib/R/library`.

So one commit can lint differently on two machines. A developer who lacks one package
gets `package-not-installed` and exit 1, while a CI runner with no R at all gets
neither, because where no library is found nothing is known to be missing. For findings
that are the same on every machine, pass `--no-r-libs` or set `r-libraries = false`
under `[lint]`. Then no library is read: `package-not-installed` never fires, and the
name rules stay quiet wherever they would need an attached or imported package other
than R's own. R's own packages are built into HoovR and do not depend on the machine.

Libraries are looked for only when a rule that reads them runs: `hoovr lint --select
assignment` reads none, and does not read the package's other files either.

### `hoovr format`

```console
hoovr format R/                            # rewrite in place
hoovr format --check R/                    # CI gate: report and exit 1, write nothing
hoovr format --diff R/                     # unified diff, write nothing
hoovr format --stdin < file.R              # for an editor
hoovr format --line-length 100 R/          # the width for this run (20 to 500)
hoovr format --profile opinionated R/      # layout from the code alone
hoovr format --profile styler-compatible   # styler's decisions, for a project moving from styler
hoovr format --profile air-compatible      # air's decisions; experimental
hoovr format --check --format json R/      # the report as JSON, for a tool
```

`--format json` reports every file and what happened to it as one JSON document (see
[the format report](#the-format-report-format---format-json-version-1)), in
place of the text report; `--format text` is the default. It cannot be combined with
`--diff` or `--stdin`, which write a diff or the formatted source to stdout instead of
a report.

`--profile` picks a set of formatting decisions in place of the `[format]` section;
`line-width` (or `--line-length`), `indent-width`, `indent-style` and `line-ending`
carry over, and a warning names any other key the profile replaces (see
[Profiles and `[format]`](#profiles-and-format)). `default` is the configured
layout, `opinionated` lays out from the code alone, ignoring the author's line
breaks, and `styler-compatible` makes styler's decisions.

`air-compatible` is **experimental**. It is outside the 1.x layout promise in
[stability.md](stability.md), so its output may change in any release, and its parity
with air is not yet measured for 1.0. It does not read `air.toml`, and it does not
honour air's `# fmt: skip` comments. It is not yet idempotent on every input: where it
braces a multi-line `function` passed as an argument before the last, a second run can
change the output, so `hoovr format --check` can fail straight after `hoovr format`.

A file that does not parse is reported on stderr and left alone. So is one that nests
expressions more than 1000 levels deep, which HoovR reports as a syntax error rather
than parse. The deepest of the 2,422 UTF-8 files in HoovR's benchmark corpus nests 24
levels by that count (data.table's `R/data.table.R`, measured on the 1.0.0 release
candidate, which reported itself as 0.1.1).
R refuses 50 nested brackets itself, but it accepts some deeper input that HoovR
refuses: a few thousand nested prefix operators, `<-`, `if` bodies or `else if`
branches, and a left-associative or postfix chain of any length, such as
`a + a + ...`, `x |> f() |> ...` or `x$a$a...`, past 1000 terms. `lint` reports both as `HR0000`. `--stdin`
echoes the input back unchanged on a syntax error, because an editor piping a buffer
through a formatter must get its buffer back. It does the same with input that is not
UTF-8, and exits 2. `--stdin` and `--stdin-file-path` take no PATH and cannot be combined
with `--check` or `--diff`. With `--stdin-file-path`, `hoovr.toml` is looked for from
that path, not from the working directory, so an editor gets the project's settings
wherever it runs HoovR from.

### `hoovr metrics`

```console
hoovr metrics                              # the scoreboard, plus the worst offenders
hoovr metrics --top 0                      # scoreboard only
hoovr metrics --update-baseline            # write health-baseline.json
hoovr metrics --check                      # exit 1 on a regression against the baseline
hoovr metrics --check --baseline health/baseline.json   # a baseline kept somewhere else
hoovr metrics --format json                # the baseline document itself
```

`--baseline` names the baseline file that `--check` reads and `--update-baseline`
writes; it defaults to `health-baseline.json` in the working directory.

`--check` exits 2 when there is no baseline to check against: a gate with nothing to
compare would always pass. `--check` and `--update-baseline` cannot be combined, and
`--format` takes `text` or `json`. A baseline records its format `version`, and one
written in a version this HoovR does not read is refused; `--update-baseline`
rewrites it.

### `hoovr rules` and `hoovr explain`

```console
hoovr rules                                # every rule and budget: code, name, severity, fix
hoovr rules --fixable                      # only the rules `lint --fix` can repair
hoovr rules --format json                  # the catalog for a tool (version 1)
hoovr rules --format markdown              # the catalog as Markdown tables
hoovr explain seq-along                    # why a rule exists, by name or code
hoovr explain HR9001                       # a budget too
```

`--format` takes `text` (the default), `json` or `markdown`. The Markdown is the whole
catalog, so `--fixable` cannot be combined with it. [rules.md](rules.md) is that
Markdown. `hoovr explain` takes a name or code in any case, as `--select` does.

## Profiles

A profile is a named set of decisions, for a project that wants to match a reference
tool rather than HoovR's own judgement.

### Lint profiles

`hoovr lint --profile <name>`, or `lintr-compatible = true` under `[lint]`:

| Profile | What it reports |
|---|---|
| `default` | HoovR's own judgement where it and lintr differ on purpose. |
| `lintr-compatible` | lintr's findings in those places instead, including where lintr is wrong about R. |

Use `lintr-compatible` to switch a project from lintr without its findings changing,
then move to `default` when the team is ready to look at the difference. The R
package's `lint()` uses `lintr-compatible` unless told otherwise; the `hoovr` command
uses `default`. The lintr it matches is the one pinned for the HoovR release, named in
the release notes; a newer lintr that changes findings comes only in a minor release
([stability.md](stability.md#the-lintr-compatible-profile)).

What `lintr-compatible` changes, each measured against lintr 3.4.0:

- **`assignment`** skips a wrong operator wherever lintr's exemption for implicit
  assignments does: anywhere under a call's arguments or an `if`, `while` or `for`
  header without parentheses around it, bodies included. The default skips only an
  assignment that is itself the argument or the condition.
- **`paren-body` and `brace-spacing`** do not report `for (x in xs){` with a body over
  several lines: in R's parse data a `for` body follows its condition, not a `)`, so
  lintr never sees it.
- **`function-body-braces`** does not report a multi-line unbraced body when a
  parameter's default is braced, `function(y =\n  {}) NULL`, as lintr does not.
- **`undefined-function` and `undefined-variable`** see a script's top-level names as
  lintr does: only assignments that are whole top-level statements, so a variable set
  inside a top-level `if` or `{` and read in a function is reported. And `x$a <- 1` in a
  function with no `x` anywhere is not reported, because codetools takes `x` for a local
  of the function; R fails there.
- **A package that is not installed** is checked as lintr checks it, against nothing
  but the file itself, rather than against the package's own `R/` files.
- **Files in `tests/testthat/`** get no testthat functions attached, so each call to
  `expect_equal()` is reported as lintr reports it; and a method of a generic the
  package imports is reported in a helper there, as lintr's `find_package()` gives up
  before reaching the package.

Both profiles run the same rules. Two of HoovR's default rules correspond to lintr
linters that lintr has but does not run by default, `any_is_na_linter` and
`undesirable_function_linter`; for lintr's default set, add
`ignore = ["any-is-na", "undesirable-function"]` under `[lint]`. A few differences remain
in both profiles, where HoovR reports something lintr misses or declines to report
prose that happens to parse; [the parity record](measurements.md) lists them.

### Format profiles

`hoovr format --profile <name>`:

| Profile | Layout |
|---|---|
| `default` | The `[format]` section: HoovR's layout, keeping the line breaks the author wrote and wrapping long lines. Changes layout only, never tokens. |
| `opinionated` | Layout from the code alone: the author's line breaks are not kept, so two layouts of the same code format the same. |
| `styler-compatible` | styler's decisions, including its two token rewrites (`=` to `<-` and `'` to `"`), for a project moving from styler. |
| `air-compatible` | **Experimental.** air's decisions. Outside the 1.x layout promise, with no published air figure. |

A profile replaces the layout keys of `[format]` and keeps the project's
`line-width`, `indent-width`, `indent-style` and `line-ending`; see
[Profiles and `[format]`](#profiles-and-format).

## Configuration

One `hoovr.toml`, found by walking up from the first path given (the working directory
when there is none), so a project has one configuration regardless of which
subdirectory a command runs in. There is no config file in the defaults' way: without
one, the defaults apply and no error is raised. With `format --stdin-file-path`, the
walk starts from that path.

The example [`hoovr.toml`](hoovr.toml) lists every key, each at its default, with a
comment on each. [Every key](#every-key) below is the reference.

### Which file applies

One configuration applies to every path in a run: the one found from the **first**
path. `hoovr lint pkgA pkgB` applies `pkgA/hoovr.toml` to both packages. When another
path has a `hoovr.toml` of its own, HoovR warns on stderr and names the file it is not
reading:

```console
$ hoovr format --check pkgA pkgB
hoovr: warning: /work/pkgB/hoovr.toml is not read: one configuration applies to every path, found from the first (/work/pkgA/hoovr.toml applies); run hoovr once per project to use each
```

Run HoovR once per project to use each project's settings. A `hoovr.toml` further down
a directory being walked is not read either. Choosing the nearest file for each file
may come in a later release; it would be a new behaviour, not a change to this one.

### Errors in the file

A mistake in `hoovr.toml` stops the run with exit 2 and a one-line message that starts
with the file and names the key:

```console
$ hoovr lint
hoovr: /work/pkg/hoovr.toml:3:1: in [format] indent-with: unknown field `indent-with`, expected one of `line-width`, `indent-width`, ...
hoovr: /work/pkg/hoovr.toml: [format] line-width = 8 is out of range: it takes 20 to 500
```

- **An unknown key is an error**, in every section. A typo should not silently leave
  the default in place.
- **A value of the wrong type or out of range is an error.** Widths (`line-width`,
  `line-length`, and `--line-length`) take 20 to 500, `indent-width` 1 to 16, and each
  `[metrics]` budget 1 or more.
- **An unknown rule name is an error.** Every entry in `select`, `ignore` and
  `[lint.severity]`, and in `--select` and `--ignore`, has to be a rule or budget code
  or name from `hoovr rules`. Case does not matter (`hr0001`, `Assignment`). An unknown
  one exits 2 and suggests the nearest match:

  ```console
  $ hoovr lint --select assigment
  hoovr: unknown rule `assigment` in --select (did you mean `assignment`?); `hoovr rules` lists them all
  ```

### Newer keys and older HoovR

Keys are part of the 1.x promise ([stability.md](stability.md#hoovrtoml-keys)):

- **New keys arrive in minor releases.** Because an unknown key is an error, a
  `hoovr.toml` that uses a key added in 1.y needs HoovR 1.y or later on every machine
  that reads it, developers' and CI's alike.
- **`required-version` says so up front.** It is checked before anything else in the
  file, so an older HoovR stops with `this configuration needs HoovR >=1.1, and this is
  HoovR 1.0.0` rather than with an unknown key. It takes one or more comparisons
  joined by commas (`">=1.1"`, `">=1.1, <2"`), with `>=`, `>`, `<=`, `<` or `==`; a
  bare version means exactly that version.
- **No key is renamed or removed in 1.x**, and a key's values only grow. Should a key
  ever be renamed, its old name stays accepted for the rest of 1.x, as a rule's old
  name does.

### Leaving files out

A directory walk leaves out what git would: `.gitignore` files (inside a git repository
or not), `.git/info/exclude` and git's global excludes file. It also honours `.ignore`
files, which use the same syntax and are for what HoovR should skip but git should
not.

On top of those, `exclude` lists gitignore-style patterns relative to the directory
holding `hoovr.toml`: `data-raw/` leaves out every `data-raw` directory, and
`/R/zzz.R` only the one next to the file. Its default leaves out R that another tool
writes, and that the next regeneration would overwrite:

```toml
exclude = ["**/R/RcppExports.R", "**/R/cpp11.R", "**/R/import-standalone-*.R"]
```

Setting `exclude` replaces that list, so copy it in to keep it. `exclude = []` excludes
nothing. A file named on the command line is used even when excluded: `hoovr format
R/RcppExports.R` formats it. `lint` still reads an excluded file of a package for the
functions it defines, because the package's other files call them.

In a run over several paths, the `hoovr.toml` found from the first applies to all of
them (see the warning under [Which file applies](#which-file-applies)). For a path
outside that file's directory, only the patterns that match at any depth apply: as in
a `.gitignore`, those with no `/` except a trailing one (`data-raw/`,
`*_generated.R`) and those starting `**/`, the defaults among them. So a second
package's `R/RcppExports.R` and `data-raw/` are left out as in the first. A pattern
with a leading or inner `/`, such as `/R/zzz.R` or `inst/extdata/`, is anchored to the
first project's directory and does not reach the second.

A walk that `exclude` leaves with no R file at all, having left out at least one,
fails: `hoovr lint`, `format` and `metrics` print
``hoovr: R: `exclude` in hoovr.toml leaves no R file to check`` and exit 2, because a
run that checked nothing is not a pass. When other paths in the run still have files,
the emptied one gets a warning instead. A directory that held no R file to begin with
is not `exclude`'s doing, and is not reported, whatever else `exclude` left out of it.

### Profiles and `[format]`

`hoovr format --profile` swaps the layout decisions in `[format]` for the profile's
fixed set. The project's conventions carry over: `line-width` (or `--line-length`),
`indent-width`, `indent-style` and `line-ending`. When `hoovr.toml` sets any other
`[format]` key to something the profile replaces, HoovR says which on stderr:

```console
hoovr: warning: --profile opinionated replaces [format] hanging-call-arguments from hoovr.toml
```

Under `styler-compatible` the carried-over `indent-width` is styler's `indent_by`, and
HoovR indents as styler does with it: every level steps by `indent-width`, except a
`function` declaration's parameters, which styler keeps two columns in from the
signature at any `indent_by`. styler cannot indent with tabs, so `indent-style = "tab"`
with `styler-compatible` gives tab-indented output that no styler run produces, and
HoovR warns that it leaves styler parity.

`--line-length` on the command line overrides both the formatter's width and the
`line-length` rule's, so the two cannot disagree.

### Every key

A key in a table is written `[section] key`; `[lint.severity]` and
`[lint.undesirable-functions]` are tables of their own. A test holds this table, the
keys HoovR accepts, and the worked example to the same set.

| Key | Default | What it does |
|---|---|---|
| `required-version` | none | The HoovR versions this file needs, such as `">=1.1"`. See [Newer keys and older HoovR](#newer-keys-and-older-hoovr). |
| `exclude` | generated R, above | Gitignore-style patterns, relative to this file, for files a walk leaves out. See [Leaving files out](#leaving-files-out). |
| `[format] line-width` | `80` | Columns the formatter aims for, 20 to 500. A soft target: a long string cannot be broken. |
| `[format] indent-width` | `2` | Columns per indent level, 1 to 16. |
| `[format] indent-style` | `"space"` | `"space"` or `"tab"`. |
| `[format] line-ending` | `"auto"` | `"auto"` keeps the file's own; `"lf"` or `"crlf"` forces one. |
| `[format] blank-lines-max` | `2` | Runs of blank lines longer than this collapse. |
| `[format] persistent-line-breaks` | `true` | Keep a bracketed construct broken where the author broke it, as styler does. |
| `[format] wrap-long-lines` | `true` | Break a construct that exceeds `line-width`. styler does not. |
| `[format] assignment-operator` | `"preserve"` | `"arrow"` rewrites a statement-level `=` to `<-`, as styler does. |
| `[format] quote-style` | `"preserve"` | `"double"` rewrites `'a'` to `"a"` where that changes nothing else, as styler does. |
| `[format] brace-bodies` | `"preserve"` | Brace a braceless `if`/`for`/`while` body: `"multiline"` is styler's rule, `"always"` air's (experimental). |
| `[format] space-after-comment-hash` | `false` | Put a space after a lone `#`, as styler does. |
| `[format] hanging-call-arguments` | `false` | styler's hanging layout for `fn(data, arg = value)`. |
| `[format] brace-comment-on-own-line` | `false` | Move a comment after `{` onto its own line, as air does (experimental). |
| `[format] bracket-breaks` | `"positions"` | `"positions"` keeps the author's break positions, as styler does; `"expanded"` is air's model (experimental). |
| `[format] break-pipe-chains` | `false` | Break a chain of two or more pipes after every pipe, as styler does. |
| `[format] keep-chunk-indent` | `true` | Keep an indented R Markdown or Quarto chunk's code under its fence. |
| `[format] keep-alignment` | `true` | Keep spaces that line arguments up in columns, as styler does. |
| `[format] pipe-call-parens` | `false` | Rewrite `x %>% f` to `x %>% f()`, as styler does. |
| `[lint] select` | `[]` | Rules to run, by name or code. Empty means every default-on rule; an opt-in rule runs only when named. |
| `[lint] ignore` | `[]` | Rules to skip, by name or code. Applied after `select`. |
| `[lint.severity]` | empty | A rule's name or code mapped to `"style"`, `"warning"` or `"error"`. |
| `[lint] line-length` | `80` | Where `line-length` reports, 20 to 500. |
| `[lint] object-name-max` | `30` | The longest name `object-length` accepts. |
| `[lint.undesirable-functions]` | built-in list | Function names mapped to the reason `undesirable-function` gives. Setting it replaces the built-in list; an empty table flags nothing. |
| `[lint] respect-nolint` | `true` | Honour `# nolint` comments. |
| `[lint] allowed-magic-numbers` | `["-1", "0", "1", "2"]` | Numbers `magic-number` accepts as written, `L` suffix dropped. |
| `[lint] extra-s3-generics` | `[]` | S3 generics the project defines, so `object-name` accepts their methods' dotted names. |
| `[lint] interpolating-functions` | glue and cli | Functions that read `{name}` out of their strings, so `unused-local` counts those names as used. Setting it replaces the list. |
| `[lint] lintr-compatible` | `false` | Reproduce lintr's findings where HoovR differs on purpose, as `lint --profile lintr-compatible` does. |
| `[lint] r-libraries` | `true` | Read the R libraries installed on the machine; `false` reads none, as `lint --no-r-libs` does. See [Installed packages](#installed-packages). |
| `[metrics] cyclomatic` | `22` | Cyclomatic complexity per function. |
| `[metrics] cognitive` | `22` | Cognitive complexity per function. |
| `[metrics] nesting-depth` | `5` | Nesting depth within a function. |
| `[metrics] parameters` | `6` | Parameters without a default, `...` not counted. |
| `[metrics] function-lines` | `100` | Lines per function. |
| `[metrics] file-lines` | `500` | Non-blank lines per file. |
| `[metrics] halstead-difficulty` | `30` | Halstead difficulty per function. |

The seven `[metrics]` budgets are exclusive upper bounds: a function scoring exactly
the budget is over it. [Code health](code-health.md) defines each measure.

The layouts marked experimental are those of the `air-compatible` profile, and share
its status: outside the 1.x layout promise, so they may change in any release. The keys
and their values are stable.

## Leaving code unformatted

The formatter leaves alone anything between `# styler: off` and `# styler: on`, and
any line ending in `# styler: off`. These are styler's own markers and work the same
way: the lines come back exactly as written. An `off` with no `on` after it runs to the
end of the file. air's `# fmt: skip` and `# fmt: skip file` are not honoured, in any
profile.

## Suppressing a finding

All five lintr spellings work, by rule name or id:

```r
x = 1        # nolint
x = 1        # nolint: assignment.
x = 1        # nolint: assignment, line-length.
# nolint next: assignment.
x = 1
# nolint start: assignment.
x = 1
y = 2
# nolint end
```

`nolint next` covers the line straight after the comment, blank or not, and not the
comment's own line: as in lintr 3.4.0, a `# nolint next: ...` comment that is too long
is still reported by `line-length`. A line reached by both a `nolint next` and its own
`# nolint` has both lists suppressed. As in lintr, exactly one space separates `nolint`
from `next`: `# nolint  next` is a bare `# nolint` on its own line. An unterminated
`nolint start` runs to end of file. `respect-nolint = false` turns the whole mechanism
off, for a CI job that wants the unfiltered picture.

A code-health budget is suppressed the same way, by its name or code, on the first
line of the function:

```r
parse_everything <- function(x) { # nolint: cyclomatic.
```

## CI

```yaml
- run: hoovr format --check .
- run: hoovr lint --format github .
- run: hoovr metrics --check
```

`--format github` emits workflow commands, which GitHub renders as inline annotations
on the diff (see [Output formats](#output-formats)). Run it from the checkout's root,
the default working directory of a step, so the paths it reports are the repository's.

For code health, commit `health-baseline.json` and let `--check` fail only on a
regression. A build that fails until every existing offender is fixed gets switched
off; a build that fails when the number gets worse gets fixed.

## Editors

The formatter reads stdin and writes stdout, which is all most editors need:

```console
hoovr format --stdin
```

For diagnostics, read `hoovr lint --format json` (see [Output formats](#output-formats)).

## Output formats

`hoovr lint --format` takes `text` (the default), `json` or `github`. Each reports the
same findings: rule findings and code-health budget findings alike, filtered by the same
`select`, `ignore`, `[lint.severity]` and `# nolint`.

In every format:

- `line` and `column` are 1-based. A column counts characters (Unicode scalar values),
  not bytes, as lintr's `column_number` and editors do: in `x <- "é"; y = 1` the `=`
  is at column 13. A tab is one character. A budget finding is at column 1 of its
  line.
- A finding in an R Markdown or Quarto document is at its line in the document.
- Files come in order of the path the format reports, compared component by component,
  and a file's findings by line, column and code.

### `text`

One line per finding, `path:line:column: CODE message`, then a summary line
(``3 findings (2 fixable with `--fix`)``, or `All checks passed`). The path is spelt as it
was given or found, with the platform's separator. Colour is used only when stdout is
a terminal and `NO_COLOR` is unset. The text is for people and may change in any
release ([stability.md](stability.md#not-covered)).

### `json`, version 1

One JSON object, written to stdout after the run:

```json
{
  "version": 1,
  "summary": { "files": 1, "filesChecked": 12, "findings": 2, "fixable": 1 },
  "findings": [
    {
      "path": "R/a.R",
      "code": "HR0001",
      "rule": "assignment",
      "message": "Use `<-` for assignment, not `=`",
      "line": 3,
      "column": 5,
      "severity": "style",
      "fixable": true
    }
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `version` | integer | The schema version: `1` in every 1.x release. |
| `summary.files` | integer | Files with at least one finding. |
| `summary.filesChecked` | integer | Files read and checked, with findings or without. A file that could not be read is not counted; it is reported on stderr and the run exits 2. |
| `summary.findings` | integer | The length of `findings`. |
| `summary.fixable` | integer | Findings with `fixable: true`. |
| `findings[].path` | string | The file, with `/` between components on every platform (`R/a.R`, never `.\R\a.R`) and without `.` components. See [Paths in `json` and `github`](#paths-in-json-and-github). |
| `findings[].code` | string | The rule or budget code: `HR0001`, `HR9001`. `HR0000` is a file that does not parse. |
| `findings[].rule` | string | The rule or budget name: `assignment`, `cyclomatic`. |
| `findings[].message` | string | What is wrong, for a person. The wording may change in any release; match on `code`. |
| `findings[].line` | integer | 1-based line. |
| `findings[].column` | integer | 1-based column, in characters. |
| `findings[].severity` | string | `style`, `warning` or `error`, after `[lint.severity]`. |
| `findings[].fixable` | boolean | Whether `hoovr lint --fix` can repair it. |

The contract, from [stability.md](stability.md#json-output-version-1): within version 1
no field is removed or renamed, none changes its type or its meaning as given here, and
`severity` keeps its three values. A 1.x minor release may add fields, so a consumer
should ignore fields it does not know. Anything else is a new `version`, and happens
only in a major release. The document is written even when some files could not be
read (exit 2): its findings then cover the files that could.

`hoovr rules --format json` (the rule and budget catalog), `hoovr metrics --format
json` (the `health-baseline.json` document) and `hoovr format --format json` (below)
carry their own `"version": 1` under the same promise.

### The format report (`format --format json`), version 1

One JSON object, written to stdout after every file has been formatted (or, with
`--check`, checked), with one entry per file, in the order of the lint report's files:

```json
{
  "version": 1,
  "summary": { "files": 3, "reformatted": 1, "wouldReformat": 0, "unchanged": 1, "errors": 1 },
  "files": [
    { "path": "R/a.R", "status": "reformatted", "message": null },
    { "path": "R/b.R", "status": "unchanged", "message": null },
    { "path": "R/c.R", "status": "error", "message": "2:1: expected `)` to close the call" }
  ]
}
```

| Field | Type | Meaning |
|---|---|---|
| `version` | integer | The schema version: `1` in every 1.x release. |
| `summary.files` | integer | The length of `files`: every file found, whatever happened to it. |
| `summary.reformatted` | integer | Files with status `reformatted`. |
| `summary.wouldReformat` | integer | Files with status `would-reformat`. |
| `summary.unchanged` | integer | Files with status `unchanged`. |
| `summary.errors` | integer | Files with status `error`. |
| `files[].path` | string | The file, as the lint report spells it (see [Paths in `json` and `github`](#paths-in-json-and-github)). |
| `files[].status` | string | `reformatted` (it was rewritten), `would-reformat` (under `--check`, it would be), `unchanged` (already formatted) or `error` (it was not formatted). |
| `files[].message` | string or null | Why a file with status `error` was not formatted: it does not parse, or could not be read or written. `null` for every other status. The wording may change in any release. |

The exit codes are those of the text report: 1 when a file would be reformatted under
`--check` or does not parse, and 2 when a file could not be read or written. The
document is written either way, with those files' entries as `error`. The same
per-file problems also go to stderr, as in the text report, and a file that could not
be read is counted in `summary.files`, where `lint`'s `filesChecked` leaves it out.

The contract is the lint report's: within version 1 no field is removed or renamed,
none changes its type or its meaning as given here, and `status` keeps its four values.
A 1.x minor release may add fields.

### `github`

One [workflow command](https://docs.github.com/en/actions/reference/workflow-commands-for-github-actions)
per finding, which GitHub Actions renders as an annotation on the pull request's diff,
and no summary:

```text
::notice file=R/a.R,line=3,col=5::HR0001 Use `<-` for assignment, not `=`
```

- The level follows the severity: `style` is `::notice`, the right weight for a pull
  request, `warning` is `::warning` and `error` is `::error`.
- `file` is the path as JSON reports it (see [below](#paths-in-json-and-github)).
- The message is the code, a space and the finding's message.
- Values are escaped as the Actions runner reads them: `%`, carriage return and line
  feed as `%25`, `%0D` and `%0A` everywhere, and also `:` and `,` as `%3A` and `%2C`
  in `file`. A message quoting a call that spans lines stays one annotation.

The form is stable across 1.x ([stability.md](stability.md#github-annotations)): one
`::<level> file=<path>,line=<line>,col=<column>::<message>` line per finding, with the
level, the path, the line and the column as given here. The message text may change in
any release, as the text format may; a tool that needs more than the annotation should
read `json`.

### Paths in `json` and `github`

Both formats report a file's path the same way, with `/` between components on every
platform and without `.` components. Otherwise the path is what was given on the
command line, or found under a directory given there:

- A relative path stays relative, as given: `./R/a.R` and `R\a.R` are reported as
  `R/a.R`, and `../other/a.R` as `../other/a.R`. A `..` is kept, not resolved.
- An absolute path is made relative to the working directory when it starts with it:
  run in `C:\work`, `C:\work\R\a.R` is reported as `R/a.R`. The comparison is of the
  path as written against the working directory as the operating system reports it,
  which on Linux has symbolic links resolved, so an absolute path that reaches the
  working directory through a symbolic link stays absolute.
- Any other absolute path stays absolute, with `/`: `C:/other/a.R`, `/srv/other/a.R`.
  A drive letter is kept, and a `\\?\` prefix is dropped.

Pass paths relative to the repository root, and run HoovR from there, to get the paths
GitHub needs to place an annotation on a pull request.

## Migrating from lintr

`hoovr lint --profile lintr-compatible` gives lintr's findings where HoovR would
otherwise differ on purpose (see [Lint profiles](#lint-profiles)); set
`lintr-compatible = true` under `[lint]` to make it the project's default. What else
carries over, and what needs translating:

- **`# nolint` comments carry over unchanged.** All five of lintr's spellings work (see
  [Suppressing a finding](#suppressing-a-finding)), and a `nolint` list may name
  lintr's linters as well as HoovR's rules: `# nolint: object_name_linter.` and
  `# nolint: object_name.` both suppress `object-name`.
- **`.lintr` is not read.** Translate it into `hoovr.toml` once. `--select`, `--ignore`
  and the keys in `hoovr.toml` take HoovR's rule names or codes, not lintr's linter
  names: `hoovr rules` and the [rule catalog](rules.md) give the lintr linter each rule
  corresponds to.

| `.lintr` | `hoovr.toml` |
|---|---|
| `linters: linters_with_defaults(line_length_linter(120))` | `line-length = 120` under `[lint]` |
| `linters: linters_with_defaults(object_name_linter = NULL)` | `ignore = ["object-name"]` under `[lint]` |
| `linters: linters_with_defaults(object_length_linter(40))` | `object-name-max = 40` under `[lint]` |
| `linters: linters_with_defaults(undesirable_function_linter(fun = ...))` | a `[lint.undesirable-functions]` table, name to reason |
| `linters: list(assignment_linter(), commas_linter())`, only those | `select = ["assignment", "commas"]` under `[lint]` |
| `exclusions: list("R/generated.R")` | `exclude` at the top level: `exclude = ["/R/generated.R"]`, plus the default patterns to keep them (see [Leaving files out](#leaving-files-out)) |
| `exclusions: list("R/a.R" = 10:20)` | a `# nolint start` / `# nolint end` pair around those lines; `hoovr.toml` has no per-line exclusion |
| `encoding: "latin1"` | none: HoovR reads UTF-8 only, so re-save the files as UTF-8 |

Some linter options have no equivalent. `indentation` checks lintr's default
two-space indent, and `object-name` lintr's default styles; a project that configures
either differently in `.lintr` can `ignore` the rule. `hoovr lint` warns about none of
this, so check a project's `.lintr` by hand when switching.

Two more differences from `lintr::lint_package()`: HoovR lints every R file under the
path it is given that `.gitignore` and `exclude` do not leave out, not only `R/`,
`tests/` and lintr's other directories; and it reports the code-health budgets
(`HR9xxx`) too, which `--no-health` switches off. The R package keeps lintr's function
names, `hoovr::lint()`, `lint_dir()` and `lint_package()`, and uses the
`lintr-compatible` profile by default.

## Migrating from styler

`hoovr format --profile styler-compatible` makes styler's decisions, including its two
token rewrites, `=` to `<-` and `'` to `"`. The profile is chosen on the command line,
so pass it everywhere the formatter runs: in CI, in the editor, and in any hook. The
default profile changes layout only and keeps the author's tokens.

| styler | HoovR |
|---|---|
| `styler::style_pkg()`, `style_dir("R")` | `hoovr format --profile styler-compatible .` |
| `style_pkg(dry = "fail")`, a CI check | `hoovr format --check --profile styler-compatible .` |
| `style_file("R/a.R")` | `hoovr format --profile styler-compatible R/a.R` |
| `tidyverse_style(indent_by = 4)` | `indent-width = 4` under `[format]` |
| `# styler: off` / `# styler: on` | the same comments (see [Leaving code unformatted](#leaving-code-unformatted)) |
| `exclude_files`, `exclude_dirs` | `exclude` at the top level, and `.gitignore` |

HoovR does not format the code in roxygen `@examples` blocks, which styler does by
default (`include_roxygen_examples = TRUE`), so those lines stay as written. It has no
cache and needs none. Other `tidyverse_style()` options, `scope` and `strict` among
them, and custom transformers have no equivalent. The R package keeps styler's
function names, `hoovr::style_text()`, `style_file()` and `style_dir()`, and uses the
`styler-compatible` profile by default.
