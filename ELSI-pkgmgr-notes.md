# ELSI Package Manager Design Notes

*Companion to ELSI-project-notes.md. Covers the `elsi` package manager command —
its interface, repository handling, index formats, and file locations.*

*Nothing here is implemented. It is a starting point.*

---

## Role of the Package Manager

The `elsi` command is the user-facing tool for managing packages on an installed
ELSI system. It handles searching, installing, removing, and listing packages.
It is not involved in the initial installation — that is the installer's job.
The installer creates `/var/elsi/` and populates the initial `repos` file
before handing off to the package manager for the first package fetch.

---

## Command Line Interface

The following session illustrates the intended command line behaviour:

```sh
$ elsi search battletech sheet
mechb10x - 3025 battletech mech record sheet generator

$ elsi info mechb
mechb - 3025 battletech mech record sheet generator
Use this program to design legal battletech record sheets and print them out on
a dot-matrix printer. All mech-mountable equipment for the 3025 era is supported.

$ elsi install mechb
btech requires the following packages:
   luacu [luacuX01] - LUA Curses Interface extra small for ELKS
   lua   [lua__X51] - LUA Interpreter
Install these as well? (y/n)

package luacu installed
package lua__ installed
package mechb10x installed
mechb

$ elsi remove mechb

$ elsi assume luacuX01
luacu marked as assumed present.
```

### Short Name Resolution

The full package identifier is the 8-character filename stem (`mechb10x`).
Commands also accept the 5-character package name prefix (`mechb`), which the
package manager resolves to the full stem. If a short name is ambiguous — for
example, `mechb` matching both `mechbX10` and `mechbP10` — the package manager
prefers the entry matching the current platform and warns if a cross-platform
match was used.

If a short name matches entries in multiple repositories, the repository listed
first in `repos` wins. A warning is emitted if the same package name appears in
more than one repo.

### Planned Commands (rev 0.1)

- `elsi search <terms>` — search package descriptions across all cached repo indexes
- `elsi info <name>` — show full description for a package (installed or available)
- `elsi install <name>` — fetch and install a package, resolving dependencies
- `elsi remove <name>` — remove an installed package (tombstone in index)
- `elsi assume <stem>` — assert that a package is present without installing it
- `elsi list` — list installed packages
- `elsi update` — refresh all cached repo index files from their FTP sources
- `elsi tidy` — compact tombstoned records from the installed index

---

## Dependency Resolution

### Transitive Resolution

`elsi install` resolves the full transitive dependency set before fetching
anything. If package A requires B, and B requires C, all three are resolved
before the user is prompted or any network activity begins.

Resolution uses an iterative worklist algorithm, not recursion. Recursion is
the wrong tool for graph walking on any platform, and is particularly
inadvisable on IA-16 hardware where the default process stack is 2KB. The
worklist approach:

1. Seed the worklist with the direct dependencies of the requested package.
2. Pull one entry from the worklist.
3. If already installed or already in the resolved set, skip it.
4. Otherwise add it to the resolved set and push its dependencies onto the
   worklist.
5. Repeat until the worklist is empty.

This naturally handles diamond dependencies (A requires B and C; B also
requires C — C appears once in the resolved set) without special-casing.

The worklist is a fixed-size array. For a package set of realistic ELSI size,
this is never a constraint in practice, but if the worklist overflows the
implementation must fail with a clear message rather than silently producing
an incomplete dependency set.

### Missing Package Handling

The full resolution pass completes before any error is reported. If multiple
dependencies are unsatisfiable, all are collected and reported together in a
single error message. The install does not proceed.

```
Cannot install btech: the following required packages are not available
in any configured repository:
   luacu
   lua
```

This is friendlier than failing on the first missing package and forcing the
user to iterate.

### Dependency Prompt

When dependencies beyond the requested package must be installed, the user is
shown the full set and asked to confirm:

```
btech requires the following packages:
   luacu [luacuX01] - LUA Curses Interface extra small for ELKS
   lua   [lua__X51] - LUA Interpreter
Install these as well? (y/n)
```

The display shows the short name, the resolved package stem in brackets, and
the short description from the repo index. Repository source is not shown
unless the same package name appears in multiple repos, in which case the
winning repo is noted.

The `?` option (per-package confirmation loop) is reserved for a future
revision. It is not implemented in rev 0.1 but the prompt should use `(y/n)`
phrasing that does not preclude adding `?` later.

### CHEAT Packages and Dependency Satisfaction

A package with source field `CHEAT` (see *elsi assume* below) is treated as
satisfying any dependency on that package name. The package manager does not
warn at dependency-check time about assumed packages — the sysadmin has already
made their assertion.

---

## elsi assume

`elsi assume <stem>` records a package as present in `instpkgs.idx` without
fetching or installing anything. It is the sysadmin's assertion that the
package exists — useful when a binary has been installed by hand, copied
directly, or is known to be present from a source outside ELSI's knowledge.

The resulting index record has:

- Status `I` (installed — it is being asserted as present)
- Source field `CHEAT`
- Last Act `********` (no reliable install time)

The info file for an assumed package contains:

```
This package was not installed by elsi.
The sysadmin asserted it exists. elsi took their word for it.
---
```

No file list follows the delimiter. `elsi remove` on a CHEAT package therefore
has no files to delete from the filesystem. It prompts:

```
luacu was asserted, not installed. No files will be removed.
Remove the record? (y/n)
```

The source tag `CHEAT` is an internal identifier. It appears in `instpkgs.idx`
and is documented in `elsipkg.5`. It does not appear in user-facing prompts or
error messages, which use the word "asserted" or "assumed" instead. Finding
`CHEAT` in the index or the man page is left as a small reward for the curious.

### Reserved Source Tags

The following source tags are reserved in `instpkgs.idx` and must not be used
as repository names in `repos`:

| Tag   | Meaning |
|-------|---------|
| `LOCAL` | Installed directly from a file on local disk, not from a repository |
| `CHEAT` | Asserted present by sysadmin via `elsi assume`. Nothing was installed. |

---

## Filesystem Paths

```
/var/hwreport              ← hardware detection report (installer-generated)
/var/elsi/instpkgs.idx     ← installed package index
/var/elsi/repos            ← repository definitions (INI format, human-edited)
/var/elsi/<tag>.idx        ← per-repo cached package index (e.g. elsi.idx, home.idx)
/var/elsi/lock             ← package manager lock file (prevents concurrent elsi runs)
/var/elsi/instpkgs/<stem>  ← per-package info file: description + file list
```

The `instpkgs/` subdirectory holds one file per installed package, extracted from
the package tarball at install time. Each file combines the full package
description and the list of files written to the filesystem, separated by a
`---` delimiter:

```
3025 battletech mech record sheet generator
Use this program to design legal battletech record sheets and print
them on a dot-matrix printer. All 3025-era equipment is supported.
---
/usr/bin/mechb
/usr/share/man/man1/mechb.1
/etc/mechb.conf
```

`elsi remove` reads the file list section to know what to delete from the
filesystem. `elsi info` reads the description section. Info files carry no
extension — the directory context makes their purpose clear.

Info files are retained through tombstone (`R`) status and deleted only when
`elsi tidy` compacts the record out of the installed index. A package with
status `N` (needed, not yet fetched) has no info file — its description is
served from the cached repo index until install time. A CHEAT package has an
info file but no file list.

The `lock` file is created at the start of any operation that modifies
`instpkgs.idx` or the `instpkgs/` tree, and removed on clean exit. Its presence
prevents concurrent `elsi` invocations from corrupting the index. ELKS is
multi-user; concurrent package operations are a real risk.

The package manager must fail with a clear, actionable error if `/var/elsi/`
does not exist. It must not silently create the directory or produce cryptic
errors. Creating `/var/elsi/` and `/var/elsi/instpkgs/` is the installer's
responsibility.

Tarball caching is explicitly out of scope for rev 0.1. Packages are fetched,
unpacked, and discarded. No tarballs are retained on the target disk.

---

## Repository Definition File: `repos`

`/var/elsi/repos` defines the package repositories available to the package
manager. It uses INI-style format.

### Format

```
# ELSI Repository Definitions
#
# Repo names must be 5 characters or fewer.
# They appear in the package index and must fit the Source field.
# Examples: elsi  home  local  nasa  cern
#
# Repositories are searched in the order they appear in this file.
# To change search priority, reorder the repository blocks.
#
# For repositories with mirrors, list your preferred address first.
# Addresses are tried in order — the next is only contacted if the
# previous one is unreachable. There is no load balancing.
#
# Keywords per repository:
#   desc  <one-line description, displayed by elsi repos>
#   ftp   <address>  (repeat for multiple mirrors, tried in order)
#   file  <path>     (local directory, e.g. mounted CD-ROM — future)

[elsi]
desc  Official ELSI package repository
ftp   elsi.example.org
ftp   192.168.1.5

[home]
desc  My local packages on the LAN
ftp   192.168.1.10
```

### Design Notes

- Section headers (`[reponame]`) name the repository. The name must be 1–5
  characters. It becomes the Source field tag in `instpkgs.idx` and the stem
  of the cached index file (`<tag>.idx`).
- `ftp` is a transport keyword, not a URL scheme. It tells the package manager
  which protocol to use. The address that follows is a bare hostname or IP.
  Raw IP addresses are always valid; hostnames require DNS resolution to be
  available on the running system (present in ELKS since v0.6.0).
- `file` is a reserved future keyword for locally-mounted media such as a
  CD-ROM. Not implemented in rev 0.1 but noted in the comment block to set
  expectations.
- Multiple `ftp` lines within a repository define mirrors, tried in order.
  There is no load balancing or round-robin — the first reachable address is
  used exclusively for that session.
- `desc` is a one-line human-readable description of the repository, displayed
  by `elsi repos` (or equivalent listing command). Optional but recommended.
- `#` introduces a comment. Comments are allowed anywhere in the file.
- Repository search order is determined by the order of sections in the file.
  To change priority, reorder the sections. There is no `priority` keyword —
  position in the file is the priority.
- The comment block at the top of the installer-generated `repos` file is
  intentional. It documents the 5-character name limit and keyword syntax
  inline, where a user will see it when they open the file to add a repo.
  Not everyone reads man pages before editing a config file.

---

## Per-Repo Index Files: `<tag>.idx`

Each repository has a cached copy of its server-side package index, stored at
`/var/elsi/<tag>.idx`. These files are fetched from the FTP server by
`elsi update` and are the source of truth for `elsi search` and `elsi info`
on packages not yet installed.

Unlike `instpkgs.idx`, the per-repo index is read-only on the client and
replaced wholesale by `elsi update`. It requires no random access — `elsi search`
scans it linearly. Fixed-width records would waste wire bytes; the format is
variable-length, one stanza per package, blank-line terminated:

```
mechb10x
3025 battletech mech record sheet generator
Use this program to design legal battletech record sheets and print
them on a dot-matrix printer. All 3025-era equipment is supported.
md5 d41d8cd98f00b204e9800998ecf8427e

sh___X10
Bourne-compatible shell
The standard ELKS shell. Supports job control, here-docs, and most
POSIX sh syntax. Statically linked, no dependencies.
md5 7215ee9c7d9dc229d2921a40e899ec5f

```

Field order within a stanza: filename stem (primary key and first line),
short description (one line, used by `elsi search`), long description (one or
more lines), `md5` checksum line, blank line terminator.

The `md5` line gives `elsi install` the checksum it needs to verify a fetched
tarball without a second round-trip to the server. MD5 is chosen for
computational cheapness on IA-16 hardware, not security.

`elsi search` reads the stem and short description line of each stanza; it
skips to the next blank line without reading the long description unless a
match is found. `elsi info` on an uninstalled package reads the full stanza.

The full format specification belongs in the `elsipkg.5` man page. See
Deferred Topics below.

---

## Installed Package Index: `instpkgs.idx`

The full format specification for `instpkgs.idx` is documented in
ELSI-project-notes.md under "Installed Package Index" and will be formally
specified in the `elsipkg.5` man page.

Key properties:
- Fixed-width records, 80 bytes including newline
- Status field supports `I` (installed), `R` (tombstone), `N` (needed), `U` (update)
- Removal flips status to `R` rather than deleting the record — single byte write
- Compaction via `elsi tidy` is a separate explicit operation
- A file of only `N` records is a valid install queue — hand-editing is supported
- Each installed package has a corresponding `instpkgs/<stem>` file (description +
  file list); retained through `R` status, removed by `elsi tidy`
- CHEAT packages have an info file with no file list section

---

## Deferred Topics

- **`elsi repos` command** — listing configured repositories with their
  descriptions and connectivity status. Not yet designed.
- **`elsi tidy` specification** — when is it safe to run, what does it report,
  does it require the system to be quiescent? Must also remove the corresponding
  `instpkgs/<stem>` file when compacting a tombstoned record.
- **`elsi tidy` and the lock file** — `elsi tidy` rewrites `instpkgs.idx` in full
  (read, filter, rewrite). It must hold the lock for the entire rewrite. Specify
  crash recovery: if the process dies mid-rewrite, is the index recoverable?
- **Ambiguity resolution policy** — short name resolution when a package exists
  in multiple repos or in both platform variants needs a fully specified
  algorithm, not just the outline above.
- **`elsipkg.5` man page** — the authoritative format reference for
  `instpkgs.idx`, `repos`, the per-repo `.idx` stanza format, and the `instpkgs/`
  file format. Section 5 is correct for file format documentation. Must document
  the `CHEAT` source tag and the `elsi assume` command.
- **Initial package count** — the total number of packages in an ELSI 0.1
  release needs to be counted against the current ELKS userspace to verify
  that worklist sizing assumptions hold and that dependency graphs are as
  shallow as expected. This should be done before implementation begins.

---

*These notes were drafted April 2026 as a companion to ELSI-project-notes.md.*
*Nothing here is implemented. It is a starting point.*
