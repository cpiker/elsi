# ELSI Project Chat Handoff
*Summary of chat session ending April 10, 2026. Use this to start the next session.*

---

## What We Covered This Session

### GitHub Workflows for Building Packages
- CI can build IA-16 packages using the ia16-elf toolchain on standard Ubuntu runners
- Testing requires QEMU in PC/XT mode — workable but nontrivial to maintain in CI
- The `ia16` platform designation still requires human verification on both platforms
- FTP transport assumption means GitHub Release artifacts need a mirror that serves FTP
- Practical recommendation: CI builds and checksums packages, deposits to GitHub Releases,
  FTP mirror pulls from there when the project matures

### First Distribution Scope
Agreed content for ELSI v0.1 distribution:
1. Installer as a bootable `.img` file
2. ELKS userspace divided into packages (see CSV)
3. mbrpatch utility (from MBR88 project — see below)
4. At least one original graphical game (see below)
- nano-x, 8086 toolchain are in-scope packages but NOT default installs
- elksdoom explicitly last priority — unplayable on most XT/AT hardware

### ELKS Package Structure Analysis
Cloned and read the ELKS repo directly. Key files examined:
- `elkscmd/config.in` — userland menuconfig
- `elkscmd/Make.install` — tag-based install system
- `elkscmd/Applications` — master app list with tags
- `elkscmd/ExtApplications` — external apps (nano-x, toolchain, lua, elksmoria, etc.)

**The tag system is the real abstraction.** Size presets (360K, 720K, 1440K, HD)
are just pre-baked tag bundle aliases. Tags accumulate as size increases:
- `:boot` + `:360k` = minimum functional system
- `:720k` adds elvis, networking, sort/sed/find/diff etc.
- `:1200k` adds tar, compress, kilo, screen, cron, basic, etc.
- `:1440k` adds man, mouse, paint(GUI), more shell utils
- `:2880k` adds busyelks, remsync, synctree, mail
- `:*` (HD) = everything including extapps

### Package Groups for Installer
Installer will present keyboard-navigable group selection (arrow keys + spacebar,
80x25 text, no mouse). Groups settled on:

| Group | Notes |
|-------|-------|
| base | Always installed; no user choice |
| disk | fdisk, mbrpatch, mkfs, fsck, mkfat, fsck-dos, df, ramdisk |
| editors | mined(on), kilo(on), elvis(off) |
| shellutils | tar, grep, find, diff, sort, sed, head, tail, cut, wc, tr, etc. |
| sysutils | screen, cron, passwd, stty, kill, lp, lpd, server daemons etc. |
| languages | basic(off), bc(off), lua(off) |
| games | tetris, invaders, advent, the ELSI game, moria, sl, matrix, ttyclock |
| tui | fm(filemanager), cons, ecalc |
| gui | nano-x suite + paint (all off by default) |
| devel | 8086 toolchain, headers, libc86, syscall man pages, examples |
| comms | elkirc, ngircd, ekermit, tinyirc, remsync, synctree, mail |
| optional | busyelks, disasm, elks-viewer, dflat, bobcat, kilomacs, doom |

Groups will evolve based on user feedback.

### Man Pages Decision
- Man pages ride with their binary package — no separate man pages group
- `man` command moves into base (required since pages ride with packages)
- ELKS man2 (syscall reference) goes in devel group as its own package (no binary)
- ELKS has ~150 man pages total across sections 1,2,3,4,5,7,8
- Notable gaps: no man pages for ash, basic, kilo, screen, tui programs, extapps

### partype Dropped
mbrpatch can print partition info, extract MBR to file, make all kinds of changes,
write it back. partype is fully redundant. Removed from disk group.

### mbrpatch / MBR88 Status
- Your project: https://github.com/cpiker/mbr88
- 8088-safe MBR bootloader + inspection/manipulation tool
- Tested on 2 of 5 planned platforms: modern Dell Inspiron (Linux) and Leading Edge Model D
- Remaining 3 platforms are machines you own
- Release strategy: post links in forum threads where people are actively dealing with
  dual-boot problems, get motivated testers, broader announcement later
- Will be a first-class tool in the ELSI disk group

### The ELSI Game
- Goal: original graphical game to be the "cool new thing" in v0.1 distribution
- Haerr's challenge: "go make something that works with mono-graphics" and he'll
  add Hercules support back to nano-x
- NOT using nano-x — direct framebuffer access, takes over the screen (DOS-style)
- Genre: illustrated adventure or similar — static scene images + text/menu interaction
- Graphics target: Hercules-capability monochrome (720x348, 1bpp)
- Three display drivers behind a thin abstraction layer:
  - Hercules (B000:0000, interleaved scanlines, 720x348)
  - LEMD (same capabilities as Hercules, different port/memory addresses)
  - CGA mode (640x200 mono or 320x200 4-color, TBD)
- LEMD has an off-screen second 32K buffer — useful for page-flip scene transitions
- Art style: high-contrast monochrome designed intentionally (woodcut aesthetic)
- Two-virtual-console idea noted: text login on one, game owns the other (kernel feature
  request, implement game first to motivate it)
- Implementation deferred — not started in this chat

---

## Open Item From End of This Chat: Index Record Redesign

**This is where the next chat should start.**

### The Problem
The installed package index (`/var/elsi/instpkgs.idx`) currently uses the 8.3
filename stem as both the storage key AND the human-facing package name. This
conflates two separate concerns:

1. **Filename** (`nslk_X10.tar`) — FAT-16 transport constraint, machine-facing
2. **Natural name** (`nslookup`) — human identifier, used on command line and in index
3. **Short description** (`DNS lookup utility`) — one line of context
4. **Long description** — unconstrained, lives in `/var/elsi/instpkgs/<stem>`

The 80-byte record limit cannot accommodate a natural name field alongside all
the other required fields without sacrificing something.

### Proposed Resolution
- **Go to 128-byte fixed-width records** — preserves seek-by-record-number,
  friendly to all editors and filesystem geometries, 200 packages = 25.6KB
- **Add a natural name field** — proposed ~12 characters, covers nearly all
  ELKS command names without being extravagant
- **Filename stem remains the primary key** — machine-facing, 8 chars
- **Natural name is the human-facing identifier** — used in `elsi search`,
  `elsi install`, `elsi info` output

### Proposed 128-byte record sketch (not yet finalized)
```
S|NaturalName |Filename|Source|Description                         |Last Act
-|------------|--------|------|------------------------------------|--------
I|nslookup    |nslk_X10|elsi  |DNS lookup utility                  |20260407
```

Fields to nail down in next chat:
- Exact width of natural name field (12 proposed)
- Whether description gets the freed-up space or stays ~52 chars
- Updated column position table
- Impact on `elsipkg.5` man page spec
- Impact on `elsi search` and `elsi install` command behavior
  (natural name is now the user-facing argument, stem is internal)

---

## CSV File
A draft package CSV (`elsi-packages-draft.csv`) was generated with 141 packages.
Paste it into the next chat. It needs:
- A `natural_name` column added once field width is settled
- Package name stems trimmed to 5 chars (44 currently exceed this)
- `comms` group possibly merged into `sysutils` or `optional` (thin group)
- `fm` (file manager) possibly moved from `tui` to `sysutils`

---

## Conventions Reminders (carry these into every ELSI chat)
- Target hardware: **IA-16** (not "8088", not "sub Linux 1.0")
- Data formats: **line-oriented flat files, not JSON** (explicitly rejected)
- File extensions: `.idx` for indexes, no extension for human-edited config files
- Fixed-width records where seeking matters
- INI-style format for `repos` and similar human-edited config
- `/var/elsi/` is ELSI's home; installer creates it, package manager does not
- Package filename: `NNNNNPVV.TGZ` (5-char name + platform code + 2-char version)
- Platform codes: `X`=ibmpc, `P`=pc98, `I`=ia16 (tested both), `N`=noarch
- Source tags in index: `LOCAL` and `CHEAT` are reserved words

---

*One-time summary. Do not regenerate.*
