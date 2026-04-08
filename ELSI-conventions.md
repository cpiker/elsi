# ELSI Project Conventions

*Project-wide conventions for documentation, code, and data formats.
Companion to ELSI-project-notes.md and its associated design documents.*

*This document is a living reference. Add conventions here as they are
established, not after the fact.*

---

## Documentation

- **Format**: All official ELSI documentation is written in Markdown.
- **One topic per file**: Keep design documents focused. The installer has
  its own file; the package manager design will have its own file; and so on.
  Cross-reference by filename rather than duplicating content.
- **Status note**: Every design document should carry a note near the top
  indicating whether it reflects implemented behaviour or is still a design
  sketch. The standard phrasing is: *"Nothing here is implemented. It is a
  starting point."* Remove or update this note when the relevant code exists.

---

## Data Formats

- **Line-oriented flat files, not JSON.** JSON was considered and explicitly
  rejected for all ELSI on-target data files. Parsing JSON in a small C binary
  on an 8088 is painful and the format offers nothing that a well-designed flat
  file does not. This applies to the installed package index, the repository
  list, the server-side package index, and the hardware database.
- **Human-readable and human-editable by design.** Hand-editing is not the
  primary interaction style but is explicitly not prevented.
- **Fixed-width records where seeking matters.** The installed package index
  uses 80-byte fixed-width records to allow direct seeks and to be friendly to
  80-column editors. Apply the same principle elsewhere if random access is
  needed.

---

## Filesystem Philosophy

ELKS does not follow the Linux Filesystem Hierarchy Standard (FHS) and does not
pretend to. Its standard directory tree is minimal: `/bin`, `/dev`, `/usr`,
`/usr/lib`, `/mnt`, `/etc`, `/root`, and `/home`. There is no `/var`, `/tmp`,
`/run`, or `/sbin` in a stock ELKS install. ELSI should not feel bound by FHS
conventions either.

The guiding principle for ELSI-created paths is: **do not create a subdirectory
until the number of files in the parent directory justifies it.** Directories
consume directory entries and add path overhead; on XT-class hardware with Minix
filesystems and limited RAM, this is not free. A subdirectory must earn its place.

Applied in practice:
- `/var/elsipkg/` is justified — the package manager owns multiple related files
  with more planned, and grouping them is correct.
- `/var/hardware` lives directly in `/var` — it is one file with no siblings yet.
  If ELSI-specific files in `/var` multiply, a subdirectory can be created then.

---

## Code

*To be filled in as implementation begins. Expected entries: language choices
per component, tab/indent convention, comment style by domain.*

---

## File Naming

- Package filenames follow the 8.3 convention (`NNNNNPVV.TGZ`) for FAT-12/16
  compatibility. See ELSI-project-notes.md for the full encoding scheme.
- Design documents are named `ELSI-<topic>.md`.

---

*Established April 2026.*
