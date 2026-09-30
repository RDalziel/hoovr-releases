<!-- Generated from HoovR's documentation. Edit the source, not this copy. -->

# Stability

From 1.0.0, HoovR follows [semantic versioning](https://semver.org). For a
command-line tool, semver does not say what "the public API" is, so this page does:
what every 1.x release keeps working, what a 1.x release may still change and how it
tells you, and what the promise does not cover.

In short: a CI job written against 1.0.0 keeps running through 1.x, with the same
commands, flags, `hoovr.toml` keys, exit codes, JSON, GitHub annotations and rule
names, and R code that calls the `hoovr` R package keeps working. Formatter layout
changes only in a minor release, whose release notes list every change, and a new
rule never switches itself on.

## What each kind of release may do

| Release | May contain |
|---|---|
| Patch (1.0.x) | Bug fixes. No new flags, keys or rules, and no change to formatter layout except to fix broken output (see [Formatter layout](#formatter-layout)). |
| Minor (1.x.0) | Everything a patch may, plus new commands, flags, flag values, `hoovr.toml` keys, JSON fields, rules and budgets; formatter layout changes; a rule switched on by default; changes to metric definitions; a newer lintr behind `lintr-compatible`; deprecations. Each is listed in the [release notes](https://github.com/RDalziel/hoovr-releases/releases). |
| Major (2.0.0) | Anything on this page marked stable may change or go, after a deprecation (see [Deprecation](#deprecation)). |

## Stable across 1.x

### Commands and flags

The commands `format`, `lint`, `metrics`, `rules` and `explain`, and bare
`hoovr [PATH...]` meaning `hoovr lint`. For each command, every flag that 1.0.0 accepts
(`hoovr <command> --help` lists them), the values each flag accepts, and each flag's
default. A flag may gain a value in a minor release; no flag loses one in 1.x.

### Exit codes

| Code | Meaning |
|---|---|
| 0 | nothing to report |
| 1 | findings, unformatted files, or a code-health regression |
| 2 | the command could not run |

These three codes, with these meanings, are the whole set. A new reason for a run to
fail to complete exits 2, never a new code. [Usage](usage.md#exit-codes) lists what
exits 2 today.

### `hoovr.toml` keys

Every key 1.0.0 accepts at the top level (`required-version`, `exclude`), in
`[format]`, `[lint]` (with `[lint.severity]` and `[lint.undesirable-functions]`) and
`[metrics]`: its section, its name and the values it accepts. The example
[`hoovr.toml`](hoovr.toml) lists them and
[Usage](usage.md#every-key) the reference; a test holds both to the keys HoovR accepts.
No key is renamed or removed in 1.x; should one be renamed, its old name stays accepted
for the rest of 1.x. A key's default changes only in a minor release, listed in the
release notes.

New keys arrive in minor releases. Unknown keys are an error, which catches typos, so a
`hoovr.toml` that uses a key added in 1.y needs 1.y or later on every machine that
reads it: the developers' and CI's. `required-version = ">=1.y"` makes an older HoovR
say so, rather than report an unknown key.

### JSON output, version 1

Four documents carry `"version": 1`:

- `hoovr lint --format json`: the findings report.
- `hoovr format --format json`: the format report, each file's status.
- `hoovr rules --format json`: the rule and budget catalog.
- `hoovr metrics --format json`, which is also the `health-baseline.json` that
  `hoovr metrics --update-baseline` writes and `--check` reads.

The findings report's fields, their types and their meaning (a `column` counts
characters, a `path` under the working directory is relative to it, with `/` separators) are
listed in [usage.md](usage.md#json-version-1), and the format report's in
[usage.md](usage.md#the-format-report-format---format-json-version-1).

Within version 1, every field keeps its name, its type and its meaning, and an
enumerated field keeps its values (a finding's `severity` is `style`, `warning` or
`error`; a file's `status` in the format report is `reformatted`, `would-reformat`,
`unchanged` or `error`). A minor release may add fields, so a consumer should ignore
fields it does not know. Removing or renaming a field, or changing its type, changes the version number,
and happens only in a major release. A `health-baseline.json` written by any 1.x release
is read by every later 1.x release.

### GitHub annotations

`hoovr lint --format github` writes one workflow command per finding,
`::<level> file=<path>,line=<line>,col=<column>::<message>`, and that form is stable:
the level for each severity (`notice` for `style`, `warning`, `error`), `file` spelt as
the JSON report spells a path, and `line` and `col` counted as the JSON report counts
them ([usage.md](usage.md#github)). The message after the second `::` is text for a
person, and is not covered.

### Rule codes and names

Every rule and budget code (`HR0001`, `HR9001`) and name (`assignment`, `cyclomatic`)
that `hoovr rules` lists in 1.0.0, and `HR0000`, the code for a file that does not parse.
In 1.x no code or name is removed, and none is reused for a different rule.

If a rule is ever renamed, its old name keeps working everywhere a name is accepted:
`--select`, `--ignore`, `select`, `ignore` and `[lint.severity]` in `hoovr.toml`, and
`# nolint` comments. Retiring a rule follows the [deprecation](#deprecation) policy.

### The R package

The `hoovr` R package's exported functions: `install_hoovr()`, `hoovr_path()`,
`lint()`, `lint_dir()`, `lint_package()`, `style_text()`, `style_file()` and
`style_dir()`. In every 1.x release of the package each keeps its name, its arguments
(their names, their order and their defaults) and the shape of what it returns: the
class, and the columns of a data frame with their types, and for `style_file()` and
`style_dir()` the values of `status`. A function may gain an argument, at the end and
with a default that keeps the old behaviour, and a data frame may gain a column, in a
minor release. Removing or renaming a function or an argument follows the
[deprecation](#deprecation) policy, with the warning from R's `.Deprecated()`. What
the functions print, and the text of their messages, warnings and errors, may change in
any release.

`lint()`, `style_file()` and `style_dir()` read HoovR's version 1 JSON reports, not its
text, so no HoovR release in the range the package supports changes what they read.

## May change in a minor release, always listed

### Formatter layout

The layout `hoovr format` produces, for a given profile and configuration, changes only
in a minor release, and that release's release notes list every change. A patch
release changes layout only to fix output that is broken: output that does not parse,
that parses to different code, that loses a comment, or that changes again when
formatted a second time.

This covers the `default`, `opinionated` and `styler-compatible` profiles. For a
`hoovr format --check` gate in CI, pin the HoovR version and take minor upgrades on
purpose, running `hoovr format` once after each.

The `air-compatible` profile is **experimental** and outside this promise: its layout
may change in any release, and no air parity figure is published for it. So are the
layouts of the `[format]` values that are air's model, which that profile sets:
`bracket-breaks = "expanded"`, `brace-bodies = "always"` and
`brace-comment-on-own-line = true`. The keys themselves, `bracket-breaks` and
`brace-bodies` among them, and every value they accept are stable, as every key is (see
[`hoovr.toml` keys](#hoovrtoml-keys)); only the layout those values produce is not.

### Rules

- **New rules arrive default-off.** A rule added in 1.x runs only when a project names
  it in `select` or `--select`; `hoovr rules` marks it `opt-in` in its text and
  Markdown output (the JSON catalog has no field for it). An empty `select` means every
  default-on rule, so a project that pins nothing sees no findings from a rule added
  since. It can still see findings change, from a rule fix or a rule switched on by
  default, below, and from the budgets (see [Metrics](#metrics)).
- **A rule becomes default-on only in a minor release**, listed in the release notes. So
  does a change to a rule's default severity, since that can change what `--fail-on`
  lets through.
- **A rule fix is a bug fix.** A false positive removed or a false negative caught can
  come in any release, and is listed in the release notes.

### The `lintr-compatible` profile

`lintr-compatible` tracks one lintr release: the release pinned for that HoovR release,
which its release notes name (lintr 3.4.0 for 1.0). HoovR is measured against it, and
the differences that remain are listed in the [parity record](measurements.md). A patch release
keeps the pinned lintr. Moving to a newer lintr that changes what the profile reports
comes only in a minor release, listed in the release notes. `styler-compatible` likewise
follows the styler pinned for the release (styler 1.11.0 for 1.0), and a move to a newer
styler is a layout change (see [Formatter layout](#formatter-layout)).

### Metrics

How a metric is counted (cyclomatic and cognitive complexity, nesting depth, parameter
count, function and file lines, Halstead difficulty, maintainability index, duplicate
bodies, dynamic evaluation, dead code, line coverage, CRAP, surviving mutants) and the default budgets change only in a minor release, listed in the release notes. A project
commits its baseline, so a patch release that recounted a metric would fail
`hoovr metrics --check` for code that did not change.

A new code-health budget (`HR9xxx`) likewise arrives only in a minor release, listed in
the release notes, and default-off, as a new rule does: `hoovr lint` reports it only when a
project names it in `select` or `--select`, until a later minor release switches it on
by default. The seven budgets in 1.0 are all on by default. `dead-code` (`HR9011`),
`coverage` (`HR9012`), `crap` (`HR9013`) and `surviving-mutants` (`HR9014`) are the
exceptions: `hoovr metrics` rows measured only with `--include-dead-code`, `--coverage`
or `--mutation`, which `hoovr lint` cannot
measure and refuses to select; `dead-code`'s count comes from the `unreachable-code` and
`unused-import` rules.

## Deprecation

Something on this page marked stable is removed only in a major release, and only after
at least one minor release in which using it prints a warning on stderr naming what to
use instead. Until it is removed, a deprecated flag, key, value or rule name works
exactly as before, and the warning does not change the exit code. Each deprecation is
listed under **Deprecated** in the release notes of the release that makes it.

1.0.0 deprecates nothing.

## What counts as a breaking change

A breaking change needs a major release. In 1.x, each of these is one:

- removing or renaming a command, a flag, a flag value, a `hoovr.toml` key or a value
  it accepts;
- changing what an exit code means, or adding a new exit code;
- removing or renaming a field in a version 1 JSON document, changing its type, or
  removing a value of an enumerated field;
- removing a rule or budget code or name, reusing one for a different rule, or dropping
  an old name after a rename;
- changing a GitHub annotation's level for a severity, or how its `file`, `line` or
  `col` is given;
- removing or renaming an exported function of the R package or one of its arguments,
  changing an argument's default, or changing the shape of what a function returns;
- making an invocation or a `hoovr.toml` that 1.0.0 accepted an error.

These are not breaking, and can come in a minor release:

- new commands, flags, flag values, keys, JSON fields and rules (default-off), and new
  budgets;
- new R package functions, new arguments with defaults at the end of an argument list,
  and new columns in a data frame the package returns;
- formatter layout changes, listed in the release notes;
- a rule switched on by default, a default severity or a default budget changed, a
  metric recounted, or `lintr-compatible` moved to a newer lintr, each listed in the
  release notes.

Bug fixes can come in any release.

## Not covered

- **The Rust library crates.** The crates the binary is built from are internal to it.
  None is published to crates.io, and their APIs may change in any release. The
  promise is the `hoovr` binary's.
- **Human-readable output.** The text of `--format text`, of messages (in any format,
  JSON and the `--format github` annotations included), of `hoovr explain`, of
  `hoovr rules` in text and Markdown, of `--help`, and what the R package prints may
  change in any release. A tool should read `--format json`.
- **The `air-compatible` profile** (see [Formatter layout](#formatter-layout)).
- **Parity and performance figures.** They are measurements of a release, published in
  the [parity](measurements.md) and [performance](measurements.md) records, not promises.

## Platforms

Each 1.x release publishes binaries for:

| Platform | Target | Notes |
|---|---|---|
| Windows x64 | `x86_64-pc-windows-msvc` | C runtime linked statically: no Visual C++ redistributable needed |
| Linux x86-64 | `x86_64-unknown-linux-musl` | statically linked: runs on any glibc or musl distribution |

macOS binaries are not published for 1.0. The macOS builds were cross-compiled, and
nothing in the release could run them before they shipped. HoovR 1.0 does not support
macOS, and `hoovr::install_hoovr()` stops on a Mac and says so.
