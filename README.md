# HoovR releases

Release binaries of **HoovR**, a linter and formatter for R written in Rust. It
applies lintr's default rules and styler's layout, and runs two to three orders of
magnitude faster than either.

This repository holds release assets only. Each
[release](https://github.com/RDalziel/hoovr-releases/releases) has one zip per
platform and a `SHA256SUMS` file to check them against.

## Install

On Windows:

```console
winget install RDalziel.HoovR
```

Anywhere else, download the zip for your platform from the latest release, check it
against `SHA256SUMS`, and put `hoovr` on your `PATH`.

| Platform | Asset |
|---|---|
| Windows x64 | `hoovr-<version>-x86_64-pc-windows-msvc.zip` |
| macOS Apple silicon | `hoovr-<version>-aarch64-apple-darwin.zip` |
| macOS Intel | `hoovr-<version>-x86_64-apple-darwin.zip` |
| Linux x64 (static) | `hoovr-<version>-x86_64-unknown-linux-musl.zip` |

## Use

```console
hoovr lint .        # lintr's rules; `--fix` applies the fixes
hoovr format .      # styler's layout, R Markdown and Quarto chunks included
hoovr --help
```
