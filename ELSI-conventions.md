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

## Target Hardware Terminology

ELSI targets **IA-16 hardware** — the 8086, 8088, NEC V20/V30, 80186, 80188,
and 80286 class machines that ELKS supports. Use "IA-16 hardware" in
documentation and code comments. Do not use "8088" as a stand-in for the whole
class, and do not use "sub Linux 1.0 capable" or similar informal phrases.
ELSI is a 16-bit distribution. The 386 and above are out of scope as test
targets.

---

## Data Formats

- **Line-oriented flat files, not JSON.** JSON was considered and explicitly
  rejected for all ELSI on-target data files. Parsing JSON in a small C binary
  on IA-16 hardware is painful and the format offers nothing that a well-designed
  flat file does not. This applies to the installed package index, the repository
  list, the server-side package index, and the hardware database.
- **Human-readable and human-editable by design.** Hand-editing is not the
  primary interaction style but is explicitly not prevented.
- **Fixed-width records where seeking matters.** The installed package index
  uses 80-byte fixed-width records to allow direct seeks and to be friendly to
  80-column editors. Apply the same principle elsewhere if random access is
  needed.
- **INI-style format for human-edited configuration files.** The `repos` file
  uses INI-style named sections with keyword lines. This format was chosen for
  its simplicity, human-editability, and natural support for `#` comments.
  Repeated keywords within a section are valid and ordered (e.g. multiple `ftp`
  lines for mirrors). ELSI's INI parser is its own; no external library is
  assumed.

---

## File Extensions

ELSI uses file extensions selectively, following the same reasoning as Linux's
`dpkg` and similar tools: **extensions describe function, not format encoding.**

- `.idx` is used for index files — it says "this is an index." It does not
  describe the internal format. This is analogous to `.list` or `.md5sums` in
  dpkg's per-package info files.
- Human-edited configuration files carry **no extension**, consistent with Linux
  convention (cf. `fstab`, `passwd`, `hosts`, `repos`). The format is documented
  in the man page, not advertised in the filename.
- Do not use `.ini`, `.cfg`, `.dat`, `.tab`, `.rec`, or similar format-describing
  extensions for ELSI data files. These either carry misleading connotations
  (`.tab` implies tab-delimited; `.dat` implies binary) or describe format rather
  than function.
- The 8.3 filename constraint applies to all files that may reside on FAT-12/16
  filesystems. Package filenames follow the `NNNNNPVV.TGZ` scheme. Index files
  use the repo tag (1–5 chars) plus `.idx`. All names must fit within 8.3.

---

## Filesystem Philosophy

ELKS does not follow the Linux Filesystem Hierarchy Standard (FHS) and does not
pretend to. Its standard directory tree is minimal: `/bin`, `/dev`, `/usr`,
`/usr/lib`, `/mnt`, `/etc`, `/root`, and `/home`. There is no `/var`, `/tmp`,
`/run`, or `/sbin` in a stock ELKS install. ELSI should not feel bound by FHS
conventions either.

The guiding principle for ELSI-created paths is: **do not create a subdirectory
until the number of files in the parent directory justifies it.** Directories
consume directory entries and add path overhead; on IA-16 hardware with Minix
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
- Repo tag names are 1–5 characters. They appear as the stem of per-repo index
  files (`<tag>.idx`) and in the Source field of the installed package index.
  Choose tags that are meaningful short identifiers: `elsi`, `home`, `local`.

---

*Established April 2026.*
