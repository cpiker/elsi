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
mechb  mech-recorder  3025 battletech mech record sheet generator

$ elsi info mech-recorder
mech-recorder - 3025 battletech mech record sheet generator
Use this program to design legal battletech record sheets and print them out on
a dot-matrix printer. All mech-mountable equipment for the 3025 era is supported.

$ elsi install mech-recorder
package mech-recorder installed

$ elsi remove mech-recorder
```

### Name Resolution

The preferred user-facing identifier is the Name field — the human-readable
package name stored in `instpkgs.idx` and the cached repo indexes (e.g.
`mech-recorder`, `nslookup`, `elks-viewer`). This is what users type and what
`elsi search` and `elsi list` display.

Commands also accept the 5-character filename stem prefix (e.g. `mechb`, `nslk_`),
which the package manager resolves to the full 8-character Filename stem. This
form is useful in scripts. If a stem prefix is ambiguous — matching both
`mechbX10` and `mechbP10` — the package manager prefers the entry matching the
current platform and warns if a cross-platform match was used.

If a Name or stem prefix matches entries in multiple repositories, the repository
listed first in `repos` wins. A warning is emitted if the same package name
appears in more than one repo.

### Planned Commands (rev 0.1)

- `elsi search <terms>` — search package descriptions across all cached repo indexes
- `elsi info <name>` — show full description for a package (installed or available)
- `elsi install <name>` — fetch and install a package from a repository
- `elsi remove <name>` — remove an installed package (tombstone in index)
- `elsi list` — list installed packages
- `elsi update` — refresh all cached repo index files from their FTP sources
- `elsi tidy` — compact tombstoned records from the installed index

---

## Filesystem Paths

```
/var/hwreport              ← hardware detection report (installer-generated)
/var/elsi/instpkgs.idx     ← installed package index
/var/elsi/repos            ← repository definitions (INI format, human-edited)
/var/elsi/<reponame>.idx   ← per-repo cached package index (e.g. elsi.idx, home.idx)
/var/elsi/lock             ← package manager lock file (prevents concurrent elsi runs)
/var/elsi/instpkgs/<stem>      ← per-package info file: description + file list
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
served from the cached repo index until install time.

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
# Repo names must be 8 characters or fewer.
# They appear in the Src field of the package index and must fit within it.
# Examples: elsi  home  local  nasa  cern  myrepo
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

- Section headers (`[reponame]`) name the repository. The name must be 1–8
  characters. It appears as the Src field value in `instpkgs.idx` and as the
  stem of the cached index file (`<reponame>.idx`). The 8-character limit
  matches the width of the Src field — a repo name that doesn't fit in the
  index is not a valid repo name. Repo index filenames (`<reponame>.idx`) are
  legal 8.3 filenames for any repo name up to 8 characters.
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
  intentional. It documents the 8-character name limit and keyword syntax
  inline, where a user will see it when they open the file to add a repo.
  Not everyone reads man pages before editing a config file.

### Reserved Source Tag

The tag `LOCAL` (uppercase) is reserved in `instpkgs.idx` to indicate a package
installed directly from a local file rather than from any repository. It must
not be used as a repository name in `repos`.

---

## Per-Repo Index Files: `<reponame>.idx`

Each repository has a cached copy of its server-side package index, stored at
`/var/elsi/<reponame>.idx`. These files are fetched from the FTP server by
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
- Fixed-width records, 128 bytes including newline
- Name field (24 chars) is the human-facing identifier; Filename stem (8 chars) is the internal primary key
- Status field supports `I` (installed), `R` (tombstone), `N` (needed), `U` (update)
- Removal flips status to `R` rather than deleting the record — single byte write
- Compaction via `elsi tidy` is a separate explicit operation
- A file of only `N` records is a valid install queue — hand-editing is supported
- Each installed package has a corresponding `instpkgs/<stem>` file (description +
  file list); retained through `R` status, removed by `elsi tidy`

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
  file format. Section 5 is correct for file format documentation.

---

*These notes were drafted April 2026 as a companion to ELSI-project-notes.md.*
*Nothing here is implemented. It is a starting point.*
