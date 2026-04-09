# ELSI Package Manager Design Notes

*Companion to ELSI-project-notes.md. Covers the `elsi` package manager command —
its interface, repository handling, index formats, and file locations.*

*Nothing here is implemented. It is a starting point.*

---

## Role of the Package Manager

The `elsi` command is the user-facing tool for managing packages on an installed
ELSI system. It handles searching, installing, removing, and listing packages.
It is not involved in the initial installation — that is the installer's job.
The installer creates `/var/elsipkg/` and populates the initial `repos` file
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
package mechb10x installed
mechb

$ elsi remove mechb
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
- `elsi install <name>` — fetch and install a package from a repository
- `elsi remove <name>` — remove an installed package (tombstone in index)
- `elsi list` — list installed packages
- `elsi update` — refresh all cached repo index files from their FTP sources
- `elsi gc` — compact tombstoned records from the installed index

---

## Filesystem Paths

```
/var/elsipkg/instpkgs.idx  ← installed package index
/var/elsipkg/repos         ← repository definitions (INI format, human-edited)
/var/elsipkg/<tag>.idx     ← per-repo cached package index (e.g. elsi.idx, home.idx)
```

The package manager must fail with a clear, actionable error if `/var/elsipkg/`
does not exist. It must not silently create the directory or produce cryptic
errors. Creating `/var/elsipkg/` is the installer's responsibility.

Tarball caching is explicitly out of scope for rev 0.1. Packages are fetched,
unpacked, and discarded. No tarballs are retained on the target disk.

---

## Repository Definition File: `repos`

`/var/elsipkg/repos` defines the package repositories available to the package
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

### Reserved Source Tag

The tag `LOCAL` (uppercase) is reserved in `instpkgs.idx` to indicate a package
installed directly from a local file rather than from any repository. It must
not be used as a repository name in `repos`.

---

## Per-Repo Index Files: `<tag>.idx`

Each repository has a cached copy of its server-side package index, stored at
`/var/elsipkg/<tag>.idx`. These files are fetched from the FTP server by
`elsi update` and are the source of truth for `elsi search` and `elsi info`
on packages not yet installed.

The format of the per-repo `.idx` files is not yet designed. It shares the
line-oriented flat-file philosophy of `instpkgs.idx` but serves a different
purpose — package discovery rather than installation tracking. See Deferred
Topics below.

---

## Installed Package Index: `instpkgs.idx`

The full format specification for `instpkgs.idx` is documented in
ELSI-project-notes.md under "Installed Package Index" and will be formally
specified in the `elsipkg.5` man page.

Key properties:
- Fixed-width records, 80 bytes including newline
- Status field supports `I` (installed), `R` (tombstone), `N` (needed), `U` (update)
- Removal flips status to `R` rather than deleting the record — single byte write
- Compaction via `elsi gc` is a separate explicit operation
- A file of only `N` records is a valid install queue — hand-editing is supported

---

## Deferred Topics

- **Per-repo `.idx` format** — the server-side index format needs its own spec.
  Discovery (what is available) vs. tracking (what is installed) are different
  use cases. The installed index's fixed-width record format may not be the right
  fit for a discovery index where description length matters more than seek speed.
- **`elsi repos` command** — listing configured repositories with their
  descriptions and connectivity status. Not yet designed.
- **`elsi gc` specification** — when is it safe to run, what does it report,
  does it require the system to be quiescent?
- **Ambiguity resolution policy** — short name resolution when a package exists
  in multiple repos or in both platform variants needs a fully specified
  algorithm, not just the outline above.
- **`elsipkg.5` man page** — the authoritative format reference for both
  `instpkgs.idx` and `repos`. Section 5 is correct for file format
  documentation.

---

*These notes were drafted April 2026 as a companion to ELSI-project-notes.md.*
*Nothing here is implemented. It is a starting point.*
