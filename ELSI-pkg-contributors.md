# ELSI Package Contributors Guide

*This document is for people who want to create or maintain packages for the
ELSI package repository. It covers package format, versioning conventions,
dependency declarations, and submission guidelines.*

*Nothing here is implemented. It is a starting point.*

---

## What Is an ELSI Package

An ELSI package is a tarball containing static binaries, configuration files,
man pages, and a manifest, plus optional install and uninstall scripts. Packages
are built for IA-16 hardware and installed by the `elsi` package manager onto
running ELSI systems. See ELSI-project-notes.md for a full description of the
target platform.

All ELSI binaries are statically linked. There are no shared library
dependencies to manage at present — if your package delivers a binary, it must
be a self-contained IA-16 ELF or a.out executable. This simplification holds
for as long as ELKS lacks shared text segment infrastructure. See *Dependency
Philosophy* below for the longer-term picture.

---

## Package Filename

Package filenames follow the 8.3 convention for FAT-12/16 compatibility:

```
NNNNNPVV.TGZ
│    ││└─ minor version (base-36: 0–9, A–Z)
│    │└── major version (base-36: 0–9, A–Z)
│    └─── platform code (see below)
└──────── package name, up to 5 characters, underscore-padded if shorter
```

Package names shorter than 5 characters are padded with underscores.
For example, a package named `sh` becomes `sh___`.

### Platform Codes

| Code | Meaning |
|------|---------|
| `X` | ibmpc — IBM PC/XT compatible binary |
| `P` | pc98 — NEC PC-98 binary |
| `I` | ia16 — tested and verified compatible on both X and P platforms |
| `N` | noarch — scripts, config files, man pages; no binary code |

The `I` (ia16) designation is an assertion that the package has actually been
tested on both platforms. It is not a default assumption. Promotion to `ia16`
should be a deliberate act by the maintainer.

---

## Versioning

ELSI uses a two-component version scheme: **major** and **minor**, encoded as
two base-36 digits in the package filename. This is semver without the patch
level.

- **Minor version** increments for bug fixes and backwards-compatible changes
  that do not alter the external interface. Patch-level distinctions are
  deliberately collapsed into minor: if you fixed a bug, ship a new minor
  version.
- **Major version** increments for breaking interface changes, and resets minor
  to zero.

Version `00` (major=0) means the package makes no interface stability guarantee.
This follows the same convention as the broader Unix/Linux ecosystem. See
*Packages Tracking Upstream Software* below for how to handle projects that
stay at 0.x indefinitely.

### The Semver Social Contract

ELSI does not have a compiler enforcing version discipline. Package maintainers
are trusted to get this right, and other maintainers are depending on them to
do so.

> A major version increment is a promise to dependent packages that they will
> break. Do not increment the major version for additive or compatible changes.
> Do not increment only the minor version for breaking changes.

A breaking change is any change that could cause a package depending on yours
to malfunction without modification. This includes: changed command-line
interfaces that scripts rely on, changed config file formats, removed
functionality, changed binary interfaces if your package is a library.

### Packages Tracking Upstream Software

If your package tracks an upstream project that uses its own version numbers,
use the upstream version number in your package filename. Do not invent a
parallel version scheme — it is confusing and a maintenance burden.

Many mature upstream projects remain at major version 0 indefinitely. ELKS
itself is a 28-year-old project currently at version 0.9. This is common
practice in the Linux ecosystem and the version number should not be taken
as a signal of instability.

For packages permanently at major=0, document the stability and interface
guarantees in the package description rather than inferring them from the
version number.

---

## Dependencies

Packages may declare dependencies on other packages. The `elsi` package manager
resolves the full transitive dependency set at install time and prompts the
user to install any missing packages automatically.

### Declaring Dependencies

Dependencies are declared in the package manifest (see *Package Contents*
below). The dependency format is:

```
requires: <name>
requires: <name> <major>
requires: <name> <major>.<minor>
```

The package name is the only required field. Version constraints are optional
and should be added only when the maintainer has verified they matter. The
guiding principle is: **declare the minimum you actually know.**

Examples:

```
requires: lua
requires: lua 5
requires: lua 5.1
```

`requires: lua` means any installed version of lua satisfies this dependency.
`requires: lua 5` means any lua with major version 5. `requires: lua 5.1` means
lua 5.1 specifically — use this only when you have tested against that version
and know earlier minor versions are incompatible.

The package manager reads as many fields as are present. Parsers must silently
ignore any fields beyond what they understand, so the format remains extensible
without breaking older tools.

When a bare `requires: lua` is found and lua is not installed, the package
manager will offer to install the first lua package found by repository priority
order.

### The 0.x Exception

Dependencies on packages permanently at major=0 should specify both major and
minor version, since major=0 carries no interface stability promise. Under the
relaxed dependency model this exception becomes more important, not less:
a bare `requires: elks` would be nearly meaningless, so packages depending on
ELKS-version-specific behaviour should be explicit:

```
requires: elks 0.9
```

### Dependency Philosophy

ELSI's dependency model is intentionally simple, with an eye toward what may
be needed later.

**What is true today:** All ELSI binaries are statically linked. There are no
shared library dependencies at link time. If your package depends on another
package, it is because it invokes that package's *commands* at runtime, not
its libraries at link time. Dependency lists are therefore short — typically
zero or one entry.

**What may change:** ELKS has latent shared text segment infrastructure dating
to its early history. If this is ever developed into a first-class facility
with heterogeneous binaries sharing code segments, the dependency model will
need revisiting — ABI stability guarantees and more precise version constraints
would become necessary. ELSI's dependency declaration format is designed to
accommodate this without a format break: additional constraint fields can be
added after the version without invalidating existing declarations.

This is not a near-term concern. It is noted so that the format is not
inadvertently designed into a corner.

**Other properties of the current model:**

- One version of any package is installed at a time. There is no version
  coexistence.
- The `elsi` package manager resolves transitive dependencies automatically
  and prompts the user. It does not silently install without confirmation.
- If a dependency cannot be found in any configured repository, the install
  fails with a complete list of all unsatisfiable dependencies before any
  packages are fetched.

A typical ELSI dependency list is short. A Lua application depends on `lua`.
A shell script depends on `ash` (or whichever shell provides `sh`). Most
packages have zero or one dependency.

---

## Package Contents

*(Format not yet fully specified. This section is a placeholder.)*

A package tarball contains:

- **Files** — the binaries, configs, and man pages to install, with their
  target filesystem paths
- **Manifest** — metadata: package name, version, platform, short description,
  long description, dependency declarations, MD5 checksums of delivered files,
  list of config files that should not be overwritten on reinstall
- **Install script** (optional) — shell script run after files are copied
- **Pre-install script** (optional) — shell script run before files are copied
- **Uninstall script** (optional) — shell script run before files are removed

The manifest format and script conventions are not yet designed. This section
will be expanded when they are.

---

## Install Script Environment

*(This section is a placeholder. The bootstrap package set and guaranteed
environment need to be formally defined before implementation.)*

Some dependencies are so fundamental that every package implicitly relies on
them and need not declare them: the kernel, and libc (which is linked
statically at build time and has no runtime presence). These are never declared
as package dependencies.

A small set of **bootstrap packages** — currently expected to be just `sh` and
the package manager itself — are guaranteed present whenever install and uninstall
scripts run. The package manager will not allow bootstrap packages to be removed.
Scripts may rely on bootstrap package facilities without declaring a dependency.

Beyond the bootstrap set, scripts must not assume anything is installed that is
not declared as a package dependency.

Notable things that are explicitly **not** guaranteed in the script environment:

- Network access
- Reliable wall-clock time — the RTC may be absent or wrong on some target
  hardware (see ELSI-project-notes.md, *Clock Reliability*). This is not an
  edge case: "why does my machine always think it's 1980?" is a common
  experience on vintage PC hardware. Install and uninstall scripts must not
  call `date` and act on the result, log timestamps as meaningful data, or
  fail based on the current year. If your script needs to record when something
  happened, let the package manager handle it — it already writes `********`
  to the Last Act field when the clock is untrustworthy.
- Any package outside the bootstrap set that is not a declared dependency

*(The full list of guaranteed utilities, the bootstrap package set definition,
and the rule against recursive `elsi` invocations from scripts will be specified
here when the manifest format and installer environment are designed.)*

---

## Conflict Detection

The `elsi` package manager detects file conflicts before installing. If your
package delivers a file already owned by an installed package, installation
stops with a clear error. Files are not silently clobbered.

Only one package may own any given filesystem path. If two packages both want
to deliver `/bin/sh`, that is a conflict and must be resolved by the maintainers
before both packages can coexist in the repository.

---

## Config Files

Config files that users are expected to edit should be declared in the manifest
as config files. The package manager will not overwrite a declared config file
on reinstall or upgrade if the installed copy has been modified.

*(The exact mechanism for this — `.new` files, a hash comparison, or a simple
flag — is not yet decided.)*

---

## Submission

*(Not yet designed. Expected to cover: where to send packages, review process,
how packages get into the official repository, how to report a hardware report
back to the project.)*

---

## Open Questions

- **Manifest format** — the internal layout of the manifest file is not yet
  specified. It should be line-oriented and human-readable, consistent with
  ELSI's general format philosophy.
- **Script interpreter** — install and uninstall scripts are assumed to be
  shell scripts, but the shell available at install time depends on what is
  already installed. The installer floppy environment and the running system
  environment may differ. This needs a formal convention.
- **Config file handling** — the policy for not clobbering user-edited config
  files needs a concrete mechanism before implementation.
- **`elsi repos` command** — listing and verifying configured repositories.
  Not yet designed.
- **Shared text and ABI stability** — if ELKS develops shared text segment
  support for heterogeneous binaries, the dependency model will need to address
  ABI versioning. The current format can accommodate additional constraint
  fields without breaking existing declarations, but the policy would need
  design work. This is a future concern, not a rev 0.1 concern.

---

*Drafted April 2026 as a companion to ELSI-project-notes.md and
ELSI-pkgmgr-notes.md. Nothing here is implemented. It is a starting point.*
