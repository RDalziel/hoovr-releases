# Installing HoovR

HoovR is one executable, `hoovr`, with nothing else to install: no R, no Rust, no
runtime. R is needed only for the R package, and for rules that read installed R
packages when there are some to read.

| Platform | Release asset | Notes |
|---|---|---|
| Windows x64 | `hoovr-<version>-x86_64-pc-windows-msvc.zip` | C runtime linked in: no Visual C++ redistributable needed |
| Linux x86-64 | `hoovr-<version>-x86_64-unknown-linux-musl.zip` | statically linked: runs on any glibc or musl distribution |

Every release is at
[github.com/RDalziel/hoovr-releases/releases](https://github.com/RDalziel/hoovr-releases/releases),
with a `SHA256SUMS` file listing the hash of each asset. From 1.0.0 each release also
carries the R package's source, `hoovr_<version>.tar.gz`.

**macOS is not supported in 1.0**: no macOS binary is published, and the R package's
`install_hoovr()` stops on a Mac and says so. Releases 0.1.0 and 0.1.1 carried macOS
zips that were cross-compiled and never run before they shipped.

## Windows: winget

```console
winget install HoovR.HoovR
```

**Pending review.** The package has been submitted to the Windows Package Manager
repository and is waiting for Microsoft's review; until it is accepted, `winget
install HoovR.HoovR` finds nothing. Use a direct download below in the meantime. Once
it is listed, `winget upgrade HoovR.HoovR` updates it and `winget uninstall
HoovR.HoovR` removes it.

## Windows: direct download

In PowerShell, from the folder you downloaded the release's zip and `SHA256SUMS` into
(replace `1.0.0` with the version you downloaded):

```powershell
$zip = 'hoovr-1.0.0-x86_64-pc-windows-msvc.zip'
$want = (Select-String -Path SHA256SUMS -Pattern ([regex]::Escape($zip))).Line.Split(' ')[0]
$have = (Get-FileHash $zip -Algorithm SHA256).Hash.ToLower()
if ($have -ne $want) { throw "$zip does not match SHA256SUMS" }
Expand-Archive $zip -DestinationPath .
```

The zip holds a folder named after it, with `hoovr.exe`, `LICENSE`, `NOTICE`,
`THIRD-PARTY-LICENSES` and a `README.md`. Move `hoovr.exe` to a folder on your `PATH`,
or add its folder to `PATH`, then check it runs:

```console
hoovr --version
```

## Linux

```console
version=1.0.0
base=https://github.com/RDalziel/hoovr-releases/releases/download/v$version
curl -LO "$base/hoovr-$version-x86_64-unknown-linux-musl.zip"
curl -LO "$base/SHA256SUMS"
sha256sum -c SHA256SUMS --ignore-missing
unzip "hoovr-$version-x86_64-unknown-linux-musl.zip"
install -m 755 "hoovr-$version-x86_64-unknown-linux-musl/hoovr" ~/.local/bin/hoovr
hoovr --version
```

`sha256sum -c` must print `OK` for the zip. `~/.local/bin` is on the `PATH` in most
distributions; any directory on it will do, `/usr/local/bin` for every user of the
machine. The binary is statically linked, so the same file runs on Debian, Ubuntu,
Fedora, Alpine and the rest, and in a container with nothing else in it.

## R users: the R package

The `hoovr` R package runs HoovR from R, under lintr's and styler's function names:
`hoovr::lint()`, `lint_dir()`, `lint_package()`, `style_text()`, `style_file()` and
`style_dir()`. It is not on CRAN or r-universe. Each release carries its source, and
it is plain R, so installing from source needs no compiler on any platform:

```r
install.packages("jsonlite")   # its one dependency outside base R
install.packages(
  "https://github.com/RDalziel/hoovr-releases/releases/download/v1.0.0/hoovr_1.0.0.tar.gz",
  repos = NULL,
  type = "source"
)
hoovr::install_hoovr()   # downloads the matching binary for this platform
```

The tarball is attached to releases from 1.0.0 on; change both `1.0.0`s for a later
release. Its hash is in the release's `SHA256SUMS`.

`install_hoovr()` downloads the binary of the release whose version is the package's,
checks it against that release's `SHA256SUMS`, and puts it in
`tools::R_user_dir("hoovr", "data")`, installing nothing anywhere else. R 4.5 and later
compute SHA-256 themselves; on older R it uses the digest package, and stops rather
than install a binary it cannot check if digest is not installed. The package supports
R 4.1.0 and later.

The package looks for a binary in this order: the `HOOVR_PATH` environment variable,
the one `install_hoovr()` installed, then `hoovr` on the `PATH`. So a binary you
already installed with winget or a download is used without a second download, as
long as its version is one the package supports: each version of the package drives
the binaries of its own major version, from its own minor version on.

## Updating

- **winget**: `winget upgrade HoovR.HoovR`, once the package is listed.
- **A download**: replace the `hoovr` binary with the new release's.
- **The R package**: install the new release's tarball as above, then run
  `hoovr::install_hoovr()` again.

HoovR follows semantic versioning from 1.0.0; [stability.md](stability.md) says what
every 1.x release keeps working. Formatter layout changes only in minor releases, so
a CI job that runs `hoovr format --check` should pin a version and take minor upgrades
on purpose, running `hoovr format` once after each.

## Uninstalling

- **winget**: `winget uninstall HoovR.HoovR`.
- **A download**: delete the `hoovr` binary. HoovR writes only what a command asks it
  to, the files it formats or fixes and a code-health baseline: no cache, no settings,
  no registry entries.
- **The R package**: `remove.packages("hoovr")`, and delete the directory
  `tools::R_user_dir("hoovr", "data")` holding the binary `install_hoovr()` downloaded.

## From source

The source repository is not public, so HoovR cannot be built from source or installed
with `cargo install`. The release binaries are the supported way to run it.
