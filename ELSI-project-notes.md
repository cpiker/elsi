# ELSI Project Notes

**ELSI** — an easily installable distribution of ELKS Linux targeting vintage hardware.

```
}     {
 }( ){
 (^ ^)
  \_/
~ elsi ~
```

The name sounds like a pet name, which is fitting — these old machines are kept alive
out of affection, not utility. Pronounce it "el-see" (English) or "el-shee" (Japanese,
エルシ). The name is intentionally simple and works well in both languages.

---

## What ELSI Is

ELSI is a proposed downstream distribution built on top of the
[ELKS](https://github.com/ghaerr/elks) (Embeddable Linux Kernel Subset) project.
ELKS is a Linux-like kernel targeting 16-bit IA-16 hardware (8086, 8088, 80286, NEC
V20/V30 and compatibles). ELSI's goal is to make ELKS accessible to people who want
to run it on vintage hardware without having to build from source and navigate
make menuconfig themselves.

ELSI does not intend to fork or structurally change ELKS. It rides on top of it.

---

## The Problem ELSI Solves

ELKS has no package manager and no installer. Getting it running on a specific
piece of hardware requires:

- Cloning the repo and building a cross-compiler toolchain
- Navigating make menuconfig with hardware-specific knowledge
- Building a kernel image with the right options compiled in
- Writing that image to the correct media

This is fine for developers. It is a barrier for people who just want to run Linux
on their old PC/XT or PC-98.

### Why a Conventional Package Manager Is Hard

ELKS has no kernel modules and no shared library support. All hardware support is
compiled into the kernel at build time. This means:

- There is no runtime hardware abstraction layer to hide configuration from packages
- The "kernel" is itself a hardware-specific artifact
- A package manager for userspace apps is tractable; a package manager that
  handles kernel selection is a harder problem

### The Fat Kernel Assumption

Rather than trying to solve the full combinatorics problem, ELSI assumes a
**fat kernel** strategy: ship a small number of pre-built kernel images that
include everything a user on that platform is likely to need, with runtime
parameters (NIC IRQ/base address, etc.) handled via `/bootopts` where ELKS
already supports it.

All three supported NICs (NE1K/NE2K, WD8003/WD8013, 3Com 3C509) can be compiled
into a single kernel simultaneously. NIC parameters are runtime-configurable via
`/bootopts`. This significantly reduces the kernel matrix.

---

## Target Hardware Platforms

ELSI targets IA-16 hardware — 8086, 8088, NEC V20/V30, 80186, 80188, and 80286
class machines. The 386 and above are out of scope as test targets, though ELKS
itself may run on them. When describing the target in documentation and code
comments, use "IA-16 hardware" rather than "8088" or other specific CPU names.
ELSI is a 16-bit distribution.

ELSI targets two parallel hardware platforms, sharing as much userspace
infrastructure as possible:

### Platform 1: IBM PC Compatible (ibmpc)

- IBM PC, PC/XT, PC/AT and compatible clones
- Covers 8088, 8086, NEC V20/V30 drop-in upgrades, 80286
- All use `CONFIG_ARCH_IBMPC` in ELKS
- One fat kernel covers this entire class
- Target era: original IBM PC (1981) through approximately 1995

### Platform 2: NEC PC-98 (pc98)

- The dominant Japanese personal computer platform 1982–late 1990s
- NEC PC-9801 and PC-9821 series; Epson PC-98 clones
- Uses `CONFIG_ARCH_PC98` in ELKS
- Requires a completely separate kernel — different BIOS, different I/O port
  addresses, different video system, different expansion bus (C-bus vs ISA)
- 18+ million units sold; enormous retro community in Japan
- Including this platform is intended to attract Japanese contributor talent

### Out of Scope (for now)

The following ELKS targets are explicitly out of scope for ELSI:

- `CONFIG_ARCH_8018X` — ROM-based embedded systems, no disk
- NEC V25 port — embedded microcontroller target, not PC-compatible
- Solo/86 — homebrew single-board computer
- WonderSwan — handheld game console
- ROM images of any platform

---

## What Makes PC-98 Distinct

PC-98 is not an IBM PC clone. Despite using x86-compatible processors, it is
a completely different machine at the hardware level:

- **CPU**: 8086 (16-bit data bus) vs IBM PC's 8088 (8-bit data bus); later models
  used V30, 80286, 80386, up to Pentium
- **Bus**: Proprietary C-bus instead of ISA
- **Video**: Custom NEC µPD7220 graphics controller; 640×400 resolution from 1982,
  designed for kanji rendering. Does not use INT 10h.
- **I/O port mapping**: All standard chips (8259 PIC, 8253 PIT, 8237 DMA) present
  but at completely different port addresses than IBM PC
- **BIOS**: Incompatible with IBM PC BIOS
- **Floppy**: 1.2MB 5.25" format standard (different from IBM's 1.2MB AT format)

PC-98 dominated Japan with 60%+ market share through the early 1990s. Windows 95
and DOS/V (which added Japanese character support to standard IBM PCs) killed it.
The platform was discontinued in 2003. The retro community around it in Japan is
large, active, and technically deep.

---

## Kernel Matrix

Under the fat kernel assumption, the minimum kernel set for ELSI is:

| Kernel | Platform | Notes |
|--------|----------|-------|
| `elsi-ibmpc.img` | IBM PC/XT/AT and clones | All three NICs compiled in |
| `elsi-pc98.img` | NEC PC-98 / Epson clones | Separate build required |

The installer floppy targets are separate from the runtime kernel images:

| Installer Image | Platform | Floppy Target |
|----------------|----------|---------------|
| `elsi-ibmpc-install.img` | IBM PC installer | 360KB DD (5.25") |
| `elsi-pc98-install.img` | PC-98 installer | 720KB DD (3.5") |

The pc98 installer targets 720KB rather than 360KB due to the expected size cost
of Japanese text/kanji support. Both targets are working assumptions pending
measurement (see Open Questions).

This is a dramatically smaller matrix than the full ELKS build space. Most of the
ELKS configuration complexity (filesystem type, memory model, graphics driver
variants) can be resolved by defaulting to the most inclusive options in a fat build,
accepting some size overhead in exchange for not requiring users to recompile.

---

## Userspace Package Manager

Userspace packages (static binaries, config files, man pages) are tractable with
a simple package manager. ELKS binaries are statically linked IA-16 ELF or a.out
executables. There are no shared library dependencies to resolve.

A minimal package manager for ELSI would:

- Define a package as a tarball (see Package Format discussion below) with a
  sidecar manifest
- Provide install / remove / list / search / info operations
- Track installed packages in the installed package index (see below)
- Share the same package format across ibmpc and pc98 platforms where binaries
  are compatible

Think early Slackware, not apt. The goal is "works reliably on 640K" not
"elegant dependency resolution." Debian `.deb` format was considered and rejected —
the format itself is lightweight but `dpkg` is not, and ELSI's dependency model
is too simple to justify the tooling weight.

Full package manager design — command line interface, index formats, repository
handling — is documented in ELSI-pkgmgr-notes.md.

### Package Format

Packages are tarballs (`NNNNNPVV.TGZ`) containing static binaries, config files,
and a manifest. The 8.3 filename encodes package metadata to remain compatible
with FAT-12/FAT-16 filesystems:

```
NNNNNPVV.TGZ
│    ││└─ minor version (base-36: 0-9, A-Z)
│    │└── major version (base-36: 0-9, A-Z)
│    └─── platform code (see below)
└──────── package name, up to 5 characters, underscore-padded if shorter
```

Package names shorter than 5 characters are padded with underscores to maintain
fixed-width 8.3 stems. For example, a package named `sh` becomes `sh___`.

Platform codes:

| Code | Meaning |
|------|---------|
| `X` | ibmpc — IBM PC/XT compatible binary |
| `P` | pc98 — NEC PC-98 binary |
| `I` | ia16 — compiled binary verified compatible on both X and P platforms |
| `N` | noarch — scripts, configs, man pages; no binary code |

The `ia16` designation is an assertion that someone has actually tested the binary
on both platforms, not a default assumption. Promotion to `ia16` should be a
deliberate act. Some packages like BASIC and Lua that ship with ELKS are strong
`ia16` candidates, subject to libc verification.

The human-readable package name and full metadata live in the manifest file, not
the filename. The 8.3 name is a unique key; the manifest is where humans read.
The package manager should warn at manifest-parse time if two entries map to the
same 8.3 filename (collision detection).

Version encoding covers `00` (0.0) through `ZZ` (35.35) — sufficient headroom
for software targeting a 16-bit kernel.

### Compression: An Open Question

It is not yet determined whether packages should be gzip-compressed tarballs
(.TGZ) or plain uncompressed tarballs (.TAR). The tradeoff is:

- **Gzip**: smaller on disk and smaller to transfer over NIC, but decompression
  consumes CPU cycles on IA-16 hardware that has none to spare
- **Plain tar**: larger transfer size but trivial to unpack; may be faster
  end-to-end on slow NIC or SLIP connections where CPU is the bottleneck

**This should be measured before committing to a format.** Build a representative
package both ways, transfer over a simulated SLIP link and a NE2K NIC, and compare
wall-clock install time. The answer may differ between NIC and SLIP paths, which
would complicate the format decision.

Tarball caching on the target machine is explicitly out of scope for rev 0.1.
Tarballs are fetched, unpacked, and discarded. Only the per-repo index files are
cached locally (see Filesystem Paths).

### Package Integrity

Packages should be verified against a checksum manifest (MD5 is sufficient for
this use case — computationally cheap, fits the era, not being used for security).

---

## Installed Package Index

The installed package index is ELSI's runtime record of what is on the system.
It lives at `/var/elsi/instpkgs.idx` (see Filesystem Paths below).

### Design Principles

The index is a fixed-width, line-oriented flat text file. JSON was considered and
explicitly rejected — parsing JSON in a small C binary on IA-16 hardware is painful
and the format offers nothing that a well-designed flat file does not. The index is
human-readable and human-editable by design. Hand-editing is not the primary
interaction style but is explicitly not prevented.

The file also serves as a valid worklist format: a file containing only `N`
(needed) records is an install queue. A user can hand-edit status fields and run
a single command to bring the system into sync with the edited list. This
interaction style is supported, not just tolerated.

Fixed-width records (128 bytes including newline) allow direct seeks by record
number. The wider record accommodates a human-readable Name field alongside the
machine-facing Filename stem, with room for a useful Description.

Removed packages are not deleted from the index immediately. Instead the status
byte is flipped to `R` (tombstone) and the record is retained. This makes removal
a single-byte write rather than a file rewrite, which is important on slow
hardware. Compaction is a separate explicit operation (`elsi tidy`).

### File Layout

```
ELSI Installed Package Index v1
Fixed-width 128-byte records including the newline. Package list begins on row 6.
See man elsipkg.5 for more information
S|Name                    |Filename|Src     |LastAct |Description                                                              
-|------------------------|--------|--------|--------|-------------------------------------------------------------------------
I|ash                     |ash__X10|home    |20260407|Bourne-compatible shell (installed as /bin/sh)                           
N|basic                   |basicP11|elsi    |********|PC-98 BASIC interpreter                                                  
R|elsipkg                 |man5_N0K|LOCAL   |20260401|File formats man pages, 0.20 test version only!                          
```

### Field Definitions

**S — Status (column 1, 1 character)**

| Code | Meaning |
|------|---------|
| `I` | Installed |
| `R` | Removed (tombstone — record retained, package absent) |
| `N` | Needed — queued for install, no action taken yet |
| `U` | Update available |

A record with status `N` must always have `********` in the LastAct field. If a
`N` record has a date, the index is inconsistent and the package manager should
warn.

**Name — Human-readable package name (columns 3–26, 24 characters, space-padded)**

The human-facing package name used on the command line and in `elsi search`,
`elsi info`, and `elsi list` output. This is the name users type. Not constrained
to 8.3 — may contain hyphens and may be longer than the 5-character filename stem
prefix. Space-padded on the right. Examples: `nslookup`, `elks-viewer`, `mbrpatch`.

The package manager also accepts the 5-character stem prefix for scripting
convenience, but Name is the preferred user-facing argument for all `elsi` commands.

**Filename — Package filename stem (columns 28–35, 8 characters, space-padded)**

The 8.3 package filename stem without the `.TGZ` extension. Encodes the 5-character
package name prefix, platform code, and two-digit version per the package filename
convention above. This is the primary key for the index — all internal lookups,
file fetches, and `instpkgs/` file references use the stem. See `elsipkg.5` for
decoding.

**Src — Repository name (columns 37–44, 8 characters, left-aligned, space-padded)**

The name of the repository this package was installed from, referencing an entry
in `/var/elsi/repos`. The field is 8 characters wide for the same reason repo names are constrained to 8 characters: both must fit
here without truncation. Two keywords are reserved and may not be used as repo names:

| Keyword | Meaning |
|---------|---------|
| `LOCAL` | Installed directly from a file on local disk, not from any repository |
| `CHEAT` | Reserved |

Repository tags are user-defined strings up to 8 characters. The installer may
create a default repo entry named `elsi` pointing at a canonical mirror, but `elsi`
is not a reserved word — it is just a conventional choice.

**LastAct — Date of last action (columns 46–53, 8 characters)**

Format `YYYYMMDD`. Written on install, removal, or any status change. `********`
when the system clock was unavailable or untrusted at the time of action, or when
no action has yet been taken (status `N`).

Not guaranteed reliable on all hardware. RTC availability varies widely on IA-16
class machines (see Clock Reliability below). LastAct is informational; do not
use it for dependency or ordering decisions.

**Description — Free text (columns 55–127, 73 characters, space-padded)**

Human-readable label. May include version notes or warnings at the author's
discretion. Not parsed by the package manager. The description in the index is
a short label; full package descriptions live in the package manifest.

### Repository File

`/var/elsi/repos` defines the package repositories available to the package
manager. It uses INI-style format: one named section per repository, with keyword
lines underneath. See ELSI-pkgmgr-notes.md for the full format specification and
a sample file.

### Man Page

A `elsipkg.5` man page is needed. Section 5 is the correct section for file
format documentation (per Unix convention: "File formats and conventions",
canonical example `/etc/passwd`). The man page should document field layout,
column positions, reserved keywords, status codes, date format, the repos file
format, and the package filename encoding/decoding scheme.

---

## Clock Reliability on IA-16 Hardware

Wall-clock time is unreliable on a significant fraction of ELSI target machines.
This is not just a Y2K concern — it is a hardware reality.

Example: the Leading Edge Model D has its real-time clock on I/O port `0x300`.
This conflicts with the default base address of XTIDE SD card readers, which are
a common modern storage upgrade for XT machines. Users must disable the RTC to
use the storage device. There is no workaround — it is a hardware address conflict.

Tick timing (elapsed time, relative intervals) remains reliable even when the RTC
is absent or wrong. Absolute wall-clock time is not.

**Proposed heuristic for the package manager:** treat the RTC as absent if the
reported year is before 1990 or after 2040. Write `********` to LastAct and emit
a warning to the user. Do not fail the operation — the package install or removal
should proceed regardless.

This condition is common enough that it should not be alarming. The warning should
be calm and informational, not an error.

---

## Filesystem Choice

ELKS supports both Minix and FAT-16 filesystems on the installed system, and
ELSI supports both. FAT-16 works correctly under ELKS and is a reasonable choice,
particularly for users coming from a FreeDOS or DOS background who want the target
disk to remain accessible from DOS tools.

Two conventions follow from this:

- **Do not assume Minix in ELSI code or documentation.** Any logic or
  documentation that requires a specific filesystem must state that requirement
  explicitly and justify it. Implicit Minix assumptions are bugs.
- **Encourage Minix to end users.** The installer should present Minix as the
  recommended default, with a brief note that it preserves Unix file permissions
  and ownership. Users who prefer FAT-16 can select it without friction. The
  recommendation exists to nudge new users toward a more Unix-like experience;
  it is not a restriction.

The installer floppy itself is FAT-12. This is not a choice — it is a requirement
for compatibility with the BIOS boot process and standard floppy tooling.

---

## Filesystem Paths

ELKS has a compact directory tree compared to a full Linux system. The standard
directories are `/bin`, `/dev`, `/etc`, `/usr`, `/usr/lib`, `/mnt`, `/root`, and
`/home`. There is no `/var` in a stock ELKS install.

ELSI creates `/var` as part of first-time installation. This is a deliberate
design decision: ELSI is a downstream distribution, not just a configuration
overlay on ELKS, and owning `/var` reflects that. It also leaves room for future
ELSI-specific runtime state (cache, logs, etc.) without touching `/etc`.

ELSI filesystem paths:

```
/var/hwreport              ← hardware detection report (installer-generated)
/var/elsi/instpkgs.idx     ← installed package index
/var/elsi/repos            ← repository definitions (INI format)
/var/elsi/<reponame>.idx   ← per-repo cached package index (e.g. elsi.idx, home.idx)
/var/elsi/lock             ← package manager lock file (prevents concurrent elsi runs)
/var/elsi/instpkgs/<stem>      ← per-package info file: description + file list
```
Notes on the `.idx` extension: it is used for index files because it describes
function (this is an index) rather than format encoding. It follows the same
reasoning as `.list` and `.md5sums` in Debian's dpkg — extensions that describe
what the file *is*, not how it is internally structured. The `repos` file carries
no extension, consistent with Linux convention for human-edited configuration
files (cf. `fstab`, `passwd`, `hosts`). Info files under `instpkgs/` carry no
extension — the directory context makes their purpose clear.

Tarball caching is out of scope for rev 0.1. The `/var/elsi/` directory
contains no package tarballs — only index files, the repository definition,
the lock file, and the `instpkgs/` tree.

Future paths (not yet designed):

```
/var/elsi/log              ← install/remove history log
```

The package manager must fail with a clear, actionable error message if
`/var/elsi/` does not exist. It must not silently create files elsewhere or
produce cryptic errors. The installer is responsible for creating `/var/elsi/`
and `/var/elsi/instpkgs/` during first-time setup; the package manager is not
responsible for creating them.

---

## Installation Concept

The installer floppy bootstraps the target machine to a minimal running state
with network connectivity. Packages are then fetched over the network from a
package server — typically a modern Linux box on the same LAN. The floppy does
not carry packages. This mirrors how early 1990s Linux distributions worked
before internet distribution was ubiquitous.

Network connectivity is established via ISA NIC (NE1K/NE2K, WD8003/WD8013, or
3Com 3C509) or SLIP over a serial port. The package server requires no special
software — a standard FTP daemon serving a directory of package tarballs and a
plaintext index is sufficient.

The installer is a single-purpose program that runs as init on the installer
floppy. It owns the console from first boot, maintains a small state file on the
floppy to survive multiple reboots gracefully, and creates `/var/elsi/` and `/var/elsi/instpkgs/` on
the target before invoking the package manager.

Full installer design — phase state machine, UI assumptions, hardware detection,
multi-boot handling — is documented in ELSI-installer-notes.md.

---

## Development Environment

ELKS can be cross-compiled on Linux. The ELKS project provides a build script
(`build.sh`) that handles toolchain setup. QEMU provides reasonable PC-98 and
IBM PC emulation for kernel development without requiring physical hardware.

Recommended emulators for testing:

- **IBM PC**: QEMU (well supported by ELKS team)
- **PC-98**: QEMU with PC-98 mode, or DOSBox-X (has dedicated PC-98 mode)
- **PC-98 physical**: NEC PC-9801 VM/VX/RX era machines are well-documented
  and findable; watch for working FDD (1.2MB 5.25" format)

---

## Relationship to ELKS

ELSI is downstream of ELKS. The intent is to:

- Track ELKS releases rather than maintain a fork
- Contribute fixes and improvements back upstream where possible
- Not change ELKS project structure (non-starter; ELSI is not the upstream maintainer)
- Provide the distribution layer (pre-built images, installer, package manager)
  that ELKS deliberately does not provide

The ELKS project description explicitly lists "vintage PCs" as a target use case
alongside embedded systems. ELSI focuses that use case without contradicting
the upstream mission.

---

## Name and Branding

**ELSI** — no required expansion, but "Embedded Linux System Image" works if one
is needed. The EL preserves lineage to ELKS. Simple, pronounceable in English
and Japanese. Sounds like a pet name, which fits the ethos.

The mascot is a text-mode elk/deer face suitable for printing in an installer:

```
}     {
 }( ){
 (^ ^)
  \_/
~ elsi ~
```

The `^ ^` eyes are intentionally anime-style, a nod to the Japanese retro
computing community that ELSI hopes to attract.

---

## Open Questions

- Can ibmpc and pc98 userspace packages actually share binaries, or do
  platform differences in system call handling or libc require separate builds?
  (Needs investigation against current ELKS libc.)
- Installer design is covered in ELSI-installer-notes.md. Remaining open
  questions (C vs shell, programmatic reboot reliability, NIC autodetection)
  are tracked there.
- ~~CHS geometry requires a per-machine build step~~ — resolved. CHS is only a
  compile-time parameter for ELKS pre-built hard disk images, which ELSI does not
  use. The `sys` command queries CHS from the BIOS at install time. Not a problem
  for ELSI. See ELSI-installer-notes.md.
- Who maintains the PC-98 build? This is a community/logistics question as much
  as a technical one.
- Will there be one canonical ELSI package mirror, or a loose community
  federation? Has implications for how repos are named and distributed.
- If a future ELKS version adds `/var` to the standard directory tree, should
  ELSI migrate its state files there and how would that migration work?
- **RESEARCH NEEDED**: What is the actual size of a minimal ELKS kernel image for
  a purpose-built `installer` defconfig target — stripped to networking (NIC + SLIP),
  a shell, and minimal userspace with no editors or man pages? Build ELKS with this
  config and measure the image. This determines whether the ibmpc installer fits on
  a 360KB DD floppy or requires 720KB. Working assumption is 360KB for ibmpc and
  720KB for pc98, but this is unverified.
- **RESEARCH NEEDED**: Does Japanese text/kanji support in the pc98 ELKS build have
  a measurable impact on kernel or userspace image size? Quantify before committing
  to the 720KB floppy target for pc98.
- **RESEARCH NEEDED**: Is plain `.tar` faster than `.tgz` for package installation
  on IA-16 hardware? Measure wall-clock install time over both NIC and SLIP
  paths for a representative package in both formats. CPU cost of decompression
  may exceed transfer-time savings on slow connections.

---

## Deferred Topics

Topics raised but not yet designed:

- **`elsipkg.5` man page** — documents the installed package index format, field
  layout, column positions, status codes, date convention, reserved keywords,
  the repos file format, the per-repo `.idx` stanza format, the `instpkgs/` combined
  file format, and package filename encoding. Section 5 is correct for file
  format documentation.
- **Server-side package index format** — the stanza format (stem / short desc /
  long desc / md5 / blank line) is outlined in ELSI-pkgmgr-notes.md. Full
  specification belongs in the `elsipkg.5` man page.
- **Lock file protocol** — `/var/elsi/lock` prevents concurrent `elsi` runs.
  ELKS is multi-user; the lock design (create, detect stale lock, release on
  crash) needs a formal spec before implementation.
- **`elsi tidy` operation** — compaction of tombstoned (`R`) records and their
  `instpkgs/` files from the installed index. Needs a spec for when it is safe to
  run, what it reports, and crash recovery during the index rewrite.
- **Clock reliability heuristic** — the proposed rule (year < 1990 or year > 2040
  → treat as absent) should be reviewed and codified as a named constant in the
  package manager source, not a magic number.

---

## References

- ELKS upstream: https://github.com/ghaerr/elks
- ELKS wiki: https://github.com/ghaerr/elks/wiki
- PC-98 hardware architecture: https://people.freebsd.org/~kato/pc98-arch.html
- PC-98 introduction: https://people.freebsd.org/~kato/pc98-intro.html
- ELKS v0.9.0 release (most recent at time of writing):
  https://github.com/ghaerr/elks/releases/tag/v0.9.0

---

*These notes were drafted April 2026 from exploratory conversations about
ELKS architecture, package management feasibility, and distribution design.
Nothing here is implemented. It is a starting point.*
