# HoovR

A fast formatter, linter and code-health scorer for **R**, written in Rust. One
executable with nothing else to install: it applies lintr's default rules and styler's
layout, and runs two to three orders of magnitude faster than either.

```console
$ hoovr lint R/
R/model.R:2:13: HR0004 Use `seq_along(x)` instead of `1:length(x)`, which is wrong when the input is empty
R/model.R:3:18: HR0005 Use `&&` in a condition, not `&`
R/model.R:4:12: HR0001 Use `<-` for assignment, not `=`
3 findings (3 fixable with `--fix`)
```

| Command | What it does | Reference tool |
|---|---|---|
| `hoovr format` | Rewrites files to tidyverse layout, R Markdown and Quarto chunks included | [styler](https://styler.r-lib.org) |
| `hoovr lint` | Every lintr default linter, and more rules, with safe fixes | [lintr](https://lintr.r-lib.org) |
| `hoovr metrics` | Complexity, size and nesting against budgets, with a ratchet that fails a build when code health gets worse | — |

This repository holds HoovR's releases and its documentation. The source is not
public.

## Install

| Platform | How |
|---|---|
| Windows x64 | `winget install HoovR.HoovR` (**pending review** by the Windows Package Manager; until it is listed, download the zip) |
| Windows x64, Linux x86-64 | The zip for your platform from the [latest release](https://github.com/RDalziel/hoovr-releases/releases/latest), checked against `SHA256SUMS` |
| R, on Windows or Linux | The R package from the release's `hoovr_<version>.tar.gz`, then `hoovr::install_hoovr()` |

macOS is not supported in 1.0. Step-by-step instructions, checking a download,
updating and uninstalling: [docs/install.md](https://github.com/RDalziel/hoovr-releases/blob/main/docs/install.md).

## Use

```console
hoovr                         # lint the current directory (read-only)
hoovr lint --fix R/           # apply every safe fix
hoovr format R/               # rewrite to canonical layout
hoovr format --check R/       # CI gate: exit 1 if anything would change
hoovr metrics --check         # fail if code health regressed against the baseline
hoovr explain seq-along       # why a rule exists
hoovr rules                   # the whole catalog
```

Exit codes: `0` nothing to report, `1` findings, `2` the command could not run.
Configuration is one `hoovr.toml`; [docs/hoovr.toml](https://github.com/RDalziel/hoovr-releases/blob/main/docs/hoovr.toml) lists every key
at its default.

Moving from lintr or styler? `hoovr lint --profile lintr-compatible` gives lintr's
findings where HoovR would differ on purpose, and `hoovr format --profile
styler-compatible` makes styler's decisions. `# nolint` comments carry over as they
are. [Migrating from lintr](https://github.com/RDalziel/hoovr-releases/blob/main/docs/usage.md#migrating-from-lintr) and
[from styler](https://github.com/RDalziel/hoovr-releases/blob/main/docs/usage.md#migrating-from-styler) cover the rest.

## Documentation

- [Install](https://github.com/RDalziel/hoovr-releases/blob/main/docs/install.md): every platform, checking a download, updating
- [Usage](https://github.com/RDalziel/hoovr-releases/blob/main/docs/usage.md): commands, flags, profiles, configuration, CI, editors, output formats
- [Rules](https://github.com/RDalziel/hoovr-releases/blob/main/docs/rules.md): the catalog, with each rule's lintr equivalent
- [Code health](https://github.com/RDalziel/hoovr-releases/blob/main/docs/code-health.md): the budgets, how each is measured, and the ratchet
- [Stability](https://github.com/RDalziel/hoovr-releases/blob/main/docs/stability.md): what every 1.x release keeps working
- [Measurements](https://github.com/RDalziel/hoovr-releases/blob/main/docs/measurements.md): agreement with lintr and styler, and speed
- [Release notes](https://github.com/RDalziel/hoovr-releases/blob/main/RELEASES.md): what changed in each release

## Stability

From 1.0.0 the commands, flags, exit codes, `hoovr.toml` keys, JSON output and rule
codes and names stay stable across 1.x. Formatter layout changes only in a minor
release, listed in the release notes, and new rules arrive switched off. The details
are in [docs/stability.md](https://github.com/RDalziel/hoovr-releases/blob/main/docs/stability.md).

## Support

Report a bug or ask a question in this repository's
[issues](https://github.com/RDalziel/hoovr-releases/issues). Say which version
`hoovr --version` prints, the platform, and the smallest R file that shows the
problem. For a security problem, see [SECURITY.md](https://github.com/RDalziel/hoovr-releases/blob/main/SECURITY.md) instead.

Supported: Windows x64 and Linux x86-64; the R package on R 4.1.0 and later. Fixes go
into the latest release.

## License

[MIT](https://github.com/RDalziel/hoovr-releases/blob/main/LICENSE). HoovR ports some logic and data from lintr, styler and rex, credited in
[NOTICE](https://github.com/RDalziel/hoovr-releases/blob/main/NOTICE). Each release zip also carries `THIRD-PARTY-LICENSES`, for the Rust
crates, the Rust standard library and musl the binary is built with.
