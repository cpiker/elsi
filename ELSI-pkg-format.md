# ELSI Package Format

*Companion to ELSI-pkg-contributors.md. Specifies the internal structure of
an ELSI package tarball and the format of the MANIFEST file it contains.*

*Nothing here is implemented. It is a starting point.*

---

## Overview

An ELSI package is a tar archive (`.TGZ` or `.TAR` — compression decision
pending, see ELSI-project-notes.md) containing:

- A MANIFEST file describing the package and its contents
- A `files/` subtree of files to be installed to the target filesystem
- Optional install and removal scripts

The package manager never pipes a tarball directly to the filesystem root.
It extracts to a temporary directory on a Minix filesystem, validates the
contents against the MANIFEST, then copies files to their destinations under
control of the package manager. This means tarballs are never extracted to
FAT media — which is correct, since the paths inside (`usr/share/man/man1/`,
long filenames) are not FAT-16 compatible. FAT media is transport only.

---

## Tarball Structure

Everything in the tarball lives under a single top-level directory named
after the 8-character package filename stem. Extracting the tarball never
scatters files.

```
nslk_X10/
    MANIFEST
    files/
        usr/bin/nslookup
        usr/share/man/man1/nslookup.1
    preinst             (optional)
    postinst            (optional)
    prerm               (optional)
```

Script filenames (`preinst`, `postinst`, `prerm`) follow 8.3 convention with
no extension. There is no `postrm` script — by the time removal is complete,
the package manager has already deleted the package's files, so there is
nothing left to execute reliably. Anything that needs to happen at removal
time while files are still present belongs in `prerm`.

Config files live in `files/` at their target paths like everything else.
Their special status — do not overwrite if the user has modified this file —
is declared in MANIFEST, not encoded in their location in the tarball.

---

## MANIFEST Format

*Not yet designed. The MANIFEST is the metadata record inside the tarball:
package name, version, platform, description, dependency declarations, MD5
checksums of delivered files, and declaration of config files that should
not be overwritten on reinstall. Format will be specified here when settled.
It should be consistent with ELSI's general line-oriented flat file philosophy.*

---

## Deferred Topics

- **MANIFEST format** — the internal layout and field definitions. Leading
  candidate is INI-style, consistent with `/var/elsi/repos`, with repeated
  keys for multi-value fields (dependencies, checksums, config file declarations,
  long description lines). Not yet settled.
- **Install sequence** — the step-by-step operations the package manager
  performs from tarball fetch through index update. Depends on MANIFEST format
  being settled first.
- **Config file handling** — mechanism for detecting whether a declared config
  file has been modified since install. MD5 comparison against the MANIFEST
  baseline is the leading candidate; no separate state file needed.
- **Temp directory management** — where the package manager extracts tarballs,
  how much space it needs to reserve, and cleanup on abort or crash.

---

*Drafted April 2026 as a companion to ELSI-pkg-contributors.md.*
*Nothing here is implemented. It is a starting point.*
