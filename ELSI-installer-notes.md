# ELSI Installer Design Notes

*Companion to ELSI-project-notes.md. Covers the installer program specifically —
its role, structure, UI assumptions, and boot-phase state machine.*

*Nothing here is implemented. It is a starting point.*

---

## Role of the Installer

The installer is the program that runs on the installer floppy to bring a target
machine from bare hardware to a running ELSI system with network connectivity and
a base package set installed.

The installer floppy carries no packages. It carries only what is needed to reach
a package server on the local network. Packages are fetched over the network and
installed to the target disk during the installer session.

This mirrors how early 1990s Linux distributions worked before internet
distribution was ubiquitous.

---

## Concept of Operations

The installer proceeds through several broad stages. Not all stages require
reboots; the multi-boot problem is confined to the network configuration stage.

### Stage 1: Hardware Detection

Before asking the user anything about network configuration or package selection,
the installer identifies the hardware it is running on and reports what it found.
This serves several purposes:

- Gives the user immediate confirmation that the machine booted correctly
- Catches obvious wrong-platform situations early (wrong floppy, wrong arch)
- Produces a permanent hardware record written to the installed system under
  `/var/hwreport` that can be used for debugging and bug reports
- Provides input to later stages — detected hardware may inform `/bootopts`
  defaults or warn about known conflicts

Hardware detection must be done carefully. Some probes are safe (reading the
BIOS data area, reading known fixed memory addresses); others risk hanging the
machine if they poke at port addresses where unexpected hardware may live. The
detection sequence is staged: the installer prints what it is about to check
before making the check. If the machine hangs mid-probe and the user reboots,
the state file records which probe was in progress. On the next boot the
installer can warn: *"Last time we tried to check [thing] and did not come back.
Skip it this time? [Y/n]"*

The hardware report file at `/var/hwreport` is a permanent plain text file
written by the installer and left on the installed system. It is not a package
and is not tracked by the package manager. It is system-generated metadata.

#### Hardware Database (Future)

Over time, as users run ELSI on diverse hardware and submit their hardware
reports, a community hardware database can be built. This would be a flat file
(line-oriented, consistent with ELSI's general format philosophy) mapping known
machine fingerprints — BIOS date strings, equipment word contents, known port
assignments — to profiles that inform the installer about quirks specific to
that machine.

The Leading Edge Model D is a concrete example of why this matters: its
real-time clock lives at I/O port `0x300`, which conflicts with the default
base address of XTIDE storage controllers. Without a hardware profile, the
installer has no way to know about this conflict. With one, it can warn the
user before they spend an afternoon debugging.

The hardware database ships with ELSI and is updated with each release as the
community contributes profiles. The format for machine profiles is not yet
designed. A mechanism for users to submit hardware reports back to the project
(even informally, such as emailing the hardware file to a mailing list) is the
feedback loop that grows the database over time.

The DOS-era `MSD.EXE` (Microsoft Diagnostics, shipped with MS-DOS 6) is a
useful reference for what "report what the BIOS told us" looks like as a user
experience, even if its implementation is not reusable here.

### Stage 2: Network Configuration

The installer configures network access — either via ISA NIC or SLIP — so that
packages can be fetched from a server on the local network. This is the stage
that may require multiple reboots. See *The Multi-Boot Problem* below.

### Stage 3: Package Fetch and Install

With network connectivity established, the installer fetches the server-side
package index, presents package or profile selection to the user, then fetches,
verifies, and installs packages to the target disk one at a time.

### Stage 4: Target Setup

The installer configures the target disk for standalone booting: bootloader,
runtime kernel image, and a starter `/bootopts` carried over from the network
configuration stage. On successful completion the installer cleans its state
from the floppy and instructs the user to reboot into their new system.

---

## The Installer Is Init

The installer floppy's `/bootopts` sets `init=` to point directly at the installer
binary. When ELKS boots the installer floppy, it comes up straight into the
installer — no getty, no login prompt, no shell the user can accidentally escape
to.

Consequences of this design:

- The installer owns the console completely from first boot
- There is no intermediate shell or init process to clean up after
- The installer is responsible for any cleanup before reboot — it cannot
  `exit()` and expect a parent process to handle shutdown
- On a successful or failed exit, the installer must explicitly halt or reboot
  the machine

This is the correct design for a single-purpose boot floppy. It is simpler,
more controlled, and eliminates an entire class of user errors.

---

## UI Assumptions

The installer runs on hardware that has just booted a minimal ELKS kernel. The
display environment is fully constrained and fully known:

- 80×25 text console (IBM PC CGA or MDA; PC-98 640×400 text mode)
- No mouse
- No X, no curses library of meaningful weight at this stage
- The user has a keyboard and can read lines of text

This is a constraint, but also a gift: the installer does not need to be a
TUI application. A linear prompt-and-response flow — think early Slackware
`setup`, or a guided shell script — is entirely appropriate and much lighter
than a full-screen menu system.

Full-screen ncurses-style UI would be cosmetically nicer but adds code weight
to a floppy that is already tight on space. The mascot can be printed at
startup. Everything else is lines of text and prompts.

---

## The Multi-Boot Problem

NIC IRQ and IO base address are configured via `/bootopts` and parsed at kernel
boot time. There is no runtime interface to reconfigure NIC parameters after the
kernel has started. This means:

- Any change to NIC settings requires writing a new `/bootopts` and rebooting
- A user whose NIC is not on the default settings will need to reboot at least
  once to apply their configuration and test it
- If settings are wrong, each correction cycle is another reboot

The installer treats multiple boots as a first-class design assumption, not an
edge case. The goal is to make each reboot feel like resuming a process, not
starting over.

### SLIP Is Different

SLIP has the same `/bootopts` constraint for COM port IRQ, but in practice it
is less of a problem:

- COM1 and COM2 are almost always at their standard IRQs (4 and 3). These are
  ELKS defaults. Most users will not need to change them.
- Baud rate for SLIP is set when `slattach` is invoked in userspace — not at
  boot time — so it is adjustable without rebooting.
- The only SLIP scenario that forces a reboot is a non-default COM port IRQ,
  which is unusual.

The hard part of SLIP is not rebooting — it is the two-machine synchronization
dance. The user must run `slattach` on the host machine and on the ELKS side in
the right order with matching baud rates before anything works. The installer
must print explicit, step-by-step host-side instructions and pause for the user
to complete them.

NIC paths may require several reboots. SLIP paths will require at most one
(possibly zero).

---

## Installer State File

To make multiple boots feel like resuming rather than restarting, the installer
maintains a small state file at `/state` on the floppy's own root filesystem.

The file is plain key=value, one entry per line:

```
phase=nic_verify
nic=ne0
irq=3
base=0x300
```

At every boot, the installer reads `/state` first. If the file does not exist,
it is a fresh install. If it exists, the installer jumps to the appropriate
phase and uses the saved values — no re-asking questions already answered.

The floppy must be writable for this to work. The installer must check this
explicitly at startup and fail with a clear message if the floppy is
write-protected. A write-protected floppy is a hard stop — the installer cannot
proceed.

---

## Boot Phase State Machine

### Phase: `fresh` (no state file)

1. Display the mascot and a welcome message
2. Confirm platform (ibmpc or pc98) — catches wrong-floppy errors early
3. Ask: NIC or SLIP?
4. Branch to `nic_configure` or `slip_configure`

### Phase: `nic_configure`

1. Ask NIC type (NE1K/NE2K, WD8003/WD8013, 3Com 3C509)
2. Show defaults for that card type; explain where to find jumper/DIP settings
3. Ask for IRQ and base address (accept defaults with Enter)
4. Write NIC settings to `/bootopts` on the floppy
5. Write `phase=nic_verify` plus settings to `/state`
6. Tell the user exactly what is about to happen and why a reboot is needed
7. Reboot (or prompt user to reboot if programmatic reboot is unreliable)

### Phase: `nic_verify`

1. Print the settings being tested (from `/state`)
2. Attempt to bring up the network interface
3. **Success**: proceed immediately to `server_configure` — no reboot needed
4. **Failure**: print what failed; offer to correct settings; if correcting,
   overwrite `/bootopts` and `/state` and reboot again (back to `nic_verify`
   on next boot)

### Phase: `slip_configure`

1. Ask COM port (default: COM1) and baud rate (default: 9600)
2. If non-default COM IRQ is needed, write to `/bootopts` and reboot
   (`phase=slip_verify`); otherwise proceed immediately
3. Print explicit host-side instructions:
   - The exact `slattach` command to run on the Linux host
   - The exact `ifconfig` command to bring up the host SLIP interface
   - Pause and wait for user to confirm host side is ready
4. Run `slattach` on the ELKS side and bring up the interface
5. **Success**: proceed to `server_configure`
6. **Failure**: diagnose and offer retry

### Phase: `slip_verify`

Same as `nic_verify` but for a SLIP COM port IRQ change. Expected to be rare.

### Phase: `server_configure`

(Reached in the same boot session as a successful `nic_verify` or
`slip_configure` — no reboot between these phases.)

1. Ask for FTP server IP address (raw IP — no DNS assumed on an 8088)
2. Test connectivity (FTP `LIST` of the package directory)
3. **Success**: write server address to `/var/elsi/repos` on the target,
   write `phase=fetch` to `/state`, proceed immediately
4. **Failure**: print diagnosis; offer to correct IP and retry

### Phase: `fetch`

1. Fetch the server-side package index
2. Display available packages; offer profile selection (minimal / standard / full)
   or individual package selection via paginated list
3. Mark selected packages as `N` (needed) in the installed package index
4. Fetch, verify checksum, and install packages one at a time
5. Print package name and progress for each; report failures clearly with
   skip-or-retry option
6. On completion, proceed to `target_setup`

### Phase: `target_setup`

1. Set up the bootloader on the target disk
2. Copy the runtime kernel image to the target
3. Write a starter `/bootopts` for the installed system (with NIC or SLIP
   settings carried over from `/state`)
4. Remove `/state` from the floppy (installation complete; clean slate for
   any future re-use of the floppy)
5. Print completion summary
6. Tell the user to remove the floppy and reboot into their new system

---

## Open Questions (Installer-Specific)

- **C binary or shell script?** A shell script is more hackable and readable;
  a compiled C binary is smaller and can do more precise things (hardware
  checks, direct reboot). The likely answer is a hybrid: shell script for
  flow and prompts, C binary for anything requiring direct hardware access or
  precise binary parsing. This is undecided.
- **Programmatic reboot**: Can the installer reliably reboot the machine from
  software on all target hardware? Some XT-class machines are finicky. If not,
  the installer should print "Please reboot now" and halt rather than attempting
  a reboot that might not work cleanly.
- **NIC autodetection**: Should the installer attempt to probe for a NIC before
  asking the user for settings? Probing ISA cards without knowing their address
  is risky (wrong port reads can hang hardware), but probing the known ELKS
  defaults first is low-risk and could save a reboot for users with standard
  configurations. Undecided.
- **Target disk setup**: CHS geometry is not a problem for ELSI. The CHS
  compile-time parameter in ELKS applies only to pre-built hard disk images,
  which ELSI does not use. The ELKS `sys` command queries the target drive's
  CHS values from the BIOS at install time and stamps them into the boot sector
  directly. `target_setup` can rely on `sys` for this without any per-machine
  build step.
- **Floppy re-use**: The installer removes `/state` on successful completion.
  Should it also restore `/bootopts` to a clean default, or leave it as-is
  (reflecting the last successfully tested NIC config)? Leaving it as-is seems
  more useful for re-runs on the same hardware.

---

*These notes were drafted April 2026 as a companion to ELSI-project-notes.md.*
*Nothing here is implemented. It is a starting point.*
