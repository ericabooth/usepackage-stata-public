# usepackage

**Find, verify, and install the user-written packages — and now the data — a Stata do-file needs.**

A shared do-file shouldn't open with a stack of `ssc install` lines, or worse, a
paragraph telling the reader to go run `findit` and figure it out. `usepackage`
takes the list of packages the do-file needs and makes sure they are there:

```stata
usepackage estout coefplot fre statplot
```

For each name it checks whether it is already installed, then SSC, then the
`net search` catalogue (which covers the *Stata Journal* and the STB). It reports
what it found, what it skipped, and why, and ends with a one-line tally.

Named for, and inspired by, LaTeX's `\usepackage`.

## Install

```stata
net install usepackage, from("https://raw.githubusercontent.com/ericabooth/usepackage-stata-public/main/") replace force
discard
which usepackage
help usepackage
```

To pull the worked example alongside it, `net get` the ancillary file:

```stata
net get usepackage, from("https://raw.githubusercontent.com/ericabooth/usepackage-stata-public/main/")
do example_usepackage.do
```

Requires Stata 16 or newer.

## What version 2 changes

Version 1.0.0 has been on SSC since 2011. It worked, but it had sharp edges, and
it knew nothing about GitHub. Version 2 is a rewrite.

**It asks before guessing.** The name you type is often a *command*, not a
*package*: `dropmiss` ships inside `dm89_2`, `bacon` inside `st0197`. v1 would
install its best guess silently, and its `nearest` option would install a
same-ish name with no warning. v2 resolves the command-inside-package case
properly, then asks:

```stata
. usepackage dropmiss
[1] dropmiss
      not a package name, but dm89_2 ships a command called dropmiss
          from http://www.stata-journal.com/software/sj15-4
      install it? (y/n)
```

**It won't hang a batch run.** `usepackage` belongs at the top of a do-file, and
do-files get run unattended, where there is nobody to answer a prompt. In batch
mode it declines the inferred match and says how to proceed, rather than blocking
or installing something you didn't ask for. Add `noconfirm` to accept inferred
matches unattended.

**It handles ancillary files deliberately.** See below.

**It tells you when a personal copy is masking the package.** An `.ado` in your
PERSONAL directory beats anything `net install` writes to PLUS, so you can
install a package and still be running last year's code. `usepackage` says so
instead of reporting a clean success.

**Bugs fixed.** `update` referenced an undefined macro, so it never actually
uninstalled first. A `continue` inside `preserve` leaked, so the second
unmatched package in a list errored with *already preserved*. SSC was attempted
twice on the same path. `insheet` (long deprecated) is gone. And v1's help file
noted that the search output "cannot be suppressed" — v2 captures it silently.

## Ancillary files

Stata splits a package in two, and the split matters more than it looks.
*Installation files* (`.ado`, `.sthlp`, `.mata`, `.py`, …) get copied onto your
adopath by `net install`. *Ancillary files* (`.do`, `.dta`, `.csv`, `.xlsx`,
`.html`, `.js`) do **not** — they only arrive with `net get`, and they land in the
**current directory**. Which is which is decided purely by file extension.

That's why `findit spmap` offers two separate links, and why `flowbca`
(*Stata Journal* `st0535`) reports 2 installation files against 26 ancillary
example datasets and do-files.

`usepackage` fetches ancillary files **by default** — if an author shipped worked
examples, you probably want them — and tells you how many arrived:

```stata
. usepackage applyvarlabels, github("ericabooth/applyvarlabels-stata-public")
      installed from https://raw.githubusercontent.com/ericabooth/...
      + 2 ancillary file(s) fetched into the current directory
```

Use `noancillary` when you'd rather not fill a working directory with example
data. It tells you what you skipped and how to get it later:

```stata
. usepackage flowbca, noconfirm noancillary
      26 ancillary file(s) available; noancillary specified, so they were not fetched
      to get them later: net get st0535
```

## Packages on GitHub

A GitHub repository is installable when it carries a `stata.toc` and a
`<pkg>.pkg`. `usepackage` probes for them rather than making you get the raw URL
exactly right — branch `main` then `master`, and at each branch the repository
root then `ado/`, `src/`, `stata/`, `code/`:

```stata
usepackage applyvarlabels, github("ericabooth/applyvarlabels-stata-public")
usepackage sparkta2,       github("texas-2036/sparkta2-stata-public")
usepackage mypkg,          github("https://github.com/owner/repo")   // pasted URL
usepackage mypkg,          github("owner/repo#dev")                  // pin a branch
usepackage mypkg,          github("owner/repo:ado")                  // files in ado/
```

If there's no `stata.toc`, it says so and points you at `data()` — a repository of
datasets isn't a package.

## Data from GitHub

Plenty of the data a do-file needs now lives in a repository rather than a
package. Name the file and load it in one step:

```stata
usepackage, data("datasets/gdp") files("data/gdp.csv") useit
```

Name no files and `usepackage` lists every `.dta`/`.csv`/`.tsv`/`.xlsx`/`.txt` it
can see and asks before downloading in bulk.

Two things this handles that a hand-written `copy` does not.

**Branch guessing.** Older repositories are on `master`, newer ones on `main`.
`usepackage` finds whichever is live:

```stata
usepackage, data("fivethirtyeight/data") files("airline-safety/airline-safety.csv") useit
```

**Git LFS.** This one is a genuine trap. A repository that tracks large files
with Git LFS serves a *130-byte text pointer* from the raw endpoint instead of the
data, so a plain `copy` reports success and leaves you with a file Stata rejects
as `not Stata format`, `r(610)`. `usepackage` recognises the pointer and
re-fetches from GitHub's media endpoint:

```stata
. usepackage, data("scunning1975/mixtape") files("nsw_mixtape.dta") useit
      branch: main
      fetched nsw_mixtape.dta  (Git LFS -- pulled from the media endpoint)
      loaded nsw_mixtape.dta (445 obs, 11 vars)
```

## Auditing a do-file

`scan()` reads a do-file and reports the commands that don't resolve on this
machine — what a collaborator would be missing. Built-ins aren't flagged, and
`quietly`/`capture`/`noisily` prefixes are stripped before the command name is
read. It reports only; it never installs.

```stata
. usepackage, scan("analysis.do")
      these commands did not resolve (candidates to install):
         estout
         coefplot
         reghdfe
      install them with: usepackage estout coefplot reghdfe
```

## The pattern worth copying

At the top of a do-file you intend to hand to someone:

```stata
cap ssc install usepackage
usepackage estout coefplot reghdfe, noconfirm
if r(nfail) > 0 {
    di as error "missing packages: `r(unresolved)'"
    exit 601
}
```

`usepackage` stores `r(nreq)`, `r(nok)`, `r(nskip)`, `r(ndefer)`, `r(nfail)`,
`r(unresolved)` and `r(deferred)`, so a do-file can stop cleanly instead of
failing three hundred lines later on an unrecognised command.

## Options

| Option | What it does |
|---|---|
| `update` | reinstall the listed packages even if present |
| `nearest` | also consider similarly *named* packages (still asks) |
| `noconfirm` | accept inferred matches without asking |
| `noancillary` | don't fetch ancillary files (default is to fetch them) |
| `dryrun` | report the plan; install nothing |
| `from(url)` | install from this net source, skipping the search |
| `github(owner/repo)` | install a package from a GitHub repository |
| `data(owner/repo)` | fetch data files from a GitHub repository |
| `files(list)` | which repository files to fetch |
| `branch(name)` | branch to use (default: `main`, then `master`) |
| `into(dir)` | where to put fetched data (default: current directory) |
| `useit` | load the fetched dataset when exactly one was fetched |
| `scan(file)` | report commands in a do-file that don't resolve |

## Known limits

- Finding non-SSC packages means calling `net search`, which writes its catalogue
  to any open log even when run quietly. Nothing reaches the Results window, but
  expect the raw catalogue in a log file.
- `data()` without `files()` uses the GitHub API, which allows 60 unauthenticated
  requests an hour. If you hit that, name the files with `files()` instead.
- For a single known package, `ssc install` is shorter. `usepackage` earns its
  keep on lists, on non-SSC sources, and on data.

## Author and license

Eric A. Booth, Sr Researcher, Texas 2036 (eric.a.booth@gmail.com).

Version 1.0.0 (2011) was written at the Public Policy Research Institute, Texas
A&M University.

MIT. See [LICENSE](LICENSE).
