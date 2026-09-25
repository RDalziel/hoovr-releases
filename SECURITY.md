# Security

## Reporting a vulnerability

Please report a security problem privately, not in a public issue: use **Report a
vulnerability** under this repository's **Security** tab
([github.com/RDalziel/hoovr-releases/security](https://github.com/RDalziel/hoovr-releases/security)).
Say which version `hoovr --version` prints, the platform, and how to reproduce the
problem. You will hear what is being done about it, and when a fix is released.

Examples of what counts: HoovR writing to a file it was not asked to write, reading
outside the paths it was given, running code from the files it reads (it never runs
R), a crash or unbounded memory use caused by a crafted input file, or the R package's
`install_hoovr()` installing a binary that does not match the release's `SHA256SUMS`.

## Supported versions

Security fixes go into a new release of the latest minor version, 1.x. Releases before
1.0.0 get none: update to the latest release.

## What HoovR does and does not do

- It reads the R files, R Markdown and Quarto documents it is given, the project's
  `hoovr.toml`, and, for some lint rules, the rest of the package a file belongs to and
  what the R packages installed on the machine export. It never runs R or any code it
  reads.
- It writes only when asked to: `hoovr format` and `hoovr lint --fix` rewrite the files
  they are given, and `hoovr metrics --update-baseline` writes the baseline file.
- It makes no network connections. The R package's `install_hoovr()` downloads the
  binary from this repository's releases over HTTPS, and checks it against the
  release's `SHA256SUMS` before installing it.
