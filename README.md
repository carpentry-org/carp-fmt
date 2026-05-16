# carp-fmt

is a formatter for Carp. Right now, it is opinionated and not tunable. That’s
mostly because I do not know what other people would like their code to look
like. I would be grateful for input in the form of issues.

The output style is documented in [the style document](docs/STYLE.md).

## Install

```bash
git clone git@github.com:carpentry-org/carp-fmt
cd carp-fmt
carp -b main.carp
# to install
sudo install -m 755 out/carp-fmt /usr/local/bin/
```

## Usage

```
usage: carp-fmt [OPTIONS] FILE [FILE...]

options:
  -w, --write      rewrite the file in place
  -c, --check      exit non-zero if any file would change
  -h, --help       show this help

Default: format each FILE and print to stdout.
```

<hr/>

Have fun!
