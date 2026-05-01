# syslock / sysunlock

Perl scripts for managing filesystem immutability and append-only flags on
OpenBSD, Linux, and macOS. Files and directories are organized into named
groups in a config file; syslock and sysunlock operate on one or more groups
at a time, making it practical to lock and unlock only the minimum set of
files needed for a given task.

Primary platform is OpenBSD, where system-level immutability flags (schg)
enforced by the kernel securelevel mechanism provide genuine security. On
Linux and macOS the flags serve as a useful safeguard against accidental
administrator damage — a meaningful "speedbump" — but not a strong security
boundary in the same sense.

Uses pledge/unveil on OpenBSD. No privilege separation (all operations
require root, except uchg/uappnd operations with a user config file).

## Immutability Flags by Platform

**OpenBSD/BSD**:
- `schg` — system immutable: cannot be removed unless securelevel=0
  (single-user mode). Genuine security enforcement.
- `uchg` — user immutable: can be removed by root at any time.
  Effective accident prevention; speedbump for an attacker.
- `sappnd` — system append-only (securelevel=0 to remove)
- `uappnd` — user append-only (removable by root at any time)

**macOS**:
- `schg` and `uchg` both supported. kern.securelevel is always 0,
  so neither requires single-user mode to remove. The difference is
  that only root can set/unset schg, while non-root users can set/unset
  uchg on files they own. Do not raise kern.securelevel on macOS — there
  is no single-user mode to recover from it.
- `uappnd` and `sappnd` supported; same caveats apply.

**Linux**:
- `+i` — immutable (root only, removable at any time)
- `+a` — append-only (root only, removable at any time)
- Managed via chattr/lsattr. Equivalent to uchg/uappnd in security terms.

**Honest assessment**: Only OpenBSD's schg/sappnd flags, combined with a
raised kern.securelevel, provide genuine kernel-enforced security. The
uchg/uappnd flags (and all Linux/macOS flags) are best understood as
protection against accidents and a meaningful obstacle for an attacker,
not an absolute barrier. That said, locking /etc configs, startup scripts,
binaries, and libraries with uchg has real operational value — an
`rm -rf` gone wrong or a typo won't wipe critical files.

## Recommended Approach

Start with uchg groups, which are trivial to unlock and cause no damage if
you need to make changes. Once comfortable with the group structure, consider
adding schg for files that rarely or never need to change outside of
single-user mode — binaries, libraries, the kernel, and critical startup
scripts are good candidates. Most production use involves a mix: schg for
the true crown jewels, uchg for regularly modified configs.

Use overlapping groups so you can unlock the minimum set needed for a given
task, or unlock a broader set (or everything) for a major system upgrade.
The sample config files illustrate this pattern.

## Usage

```
syslock [options]           Lock all files in config
syslock -g group[:flag]     Lock files in named group(s)
sysunlock [options]         Unlock all files in config
sysunlock -g group[:flag]   Unlock files in named group(s)
```

**Options**:
```
-g group[:flag]   Operate on named group(s), comma-separated.
                  Optional :flag suffix (e.g., :uchg, :uappnd, :+a)
                  restricts to the intersection of the group and flag type.
-a                Audit mode: report files that should be locked (syslock)
                  or unlocked (sysunlock) but aren't. Exit 0 if consistent,
                  1 if not.
-q                Quiet (with -a only): suppress output, return exit code only.
                  Used by install.pl to verify a group is unlocked before
                  proceeding with installation.
-f                Force: lock schg/sappnd flags even if securelevel > 0.
                  (syslock only)
-s                Operate only on files marked with ! in the config (KARL
                  relink/reorder files). (OpenBSD only)
-w                After locking, wait for kernel reorder and library relink
                  to complete before locking ! files. Used in rc.securelevel
                  so the system locks automatically after OpenBSD's KARL
                  process finishes. (syslock only, OpenBSD only)
-c file           Use specified config file instead of default.
-v                Verbose output.
-d                Debug output.
-V                Display version.
```

**Group syntax with flag type**:

When the default immutable-flag is schg, files that should use uchg instead
are tagged with the "uchg" group name. To lock/unlock only those files:
```
syslock -g newsyslog-logs:uchg
sysunlock -g newsyslog-logs:uchg
```

Similarly, append-only files use the "uappnd" group (or "+a" on Linux):
```
syslock -g acct:uappnd
sysunlock -g acct:uappnd
```

## Config File

Default location: `/etc/syslock.conf` (root); `~/.syslock.conf` (non-root,
for uchg/uappnd operations only).

**Global setting** (optional; defaults shown above by platform):
```
immutable-flag: schg
```

**Group and file entries**:
```
group: groupname [groupname ...]
/path/to/file-or-directory
/path/with/glob.*
```

A file or directory inherits all groups from the most recent `group:` line.
Multiple group names on one line means the entry belongs to all of them —
this is how overlapping groups work. Files can appear under multiple
`group:` lines to belong to many groups.

**Path flags** (prefix on path):
```
+   Do not recurse into subdirectories
-   Lock/unlock directory contents but not the directory itself
!   Only operated on by syslock/sysunlock -s (KARL relink files, OpenBSD)
```

See the sample config files in `etc/` for complete annotated examples for
OpenBSD, Linux, and macOS.

## Sample Group Structure

The sample configs define overlapping groups following this pattern:

- **etc** — frequently modified configs (rc.conf.local, pf.conf, doas.conf, etc.)
- **fstab** — /etc/fstab alone, given its own group because it is
  particularly painful to accidentally damage
- **etcrare** — /etc files rarely needing changes (rc, rc.conf, login.conf,
  sysctl.conf, newsyslog.conf, etc.)
- **configs** — overlaps with etc; unlock for config editing tasks
- **system** — system-level files; use with schg group for strongest protection
- **binaries** — /usr/bin, /usr/sbin, /usr/local/bin, /usr/local/sbin
- **libraries** — /usr/lib, /usr/libexec, /usr/local/lib, etc.
- **kernel** — /bsd and kernel relink files
- **ssl** — /etc/ssl certificate directory
- **ssh** — /etc/ssh
- **signify** — /etc/signify key directory
- **mail** — mail server config files
- **acct** — process accounting log files (live log append-only, rotated immutable)
- **newsyslog-logs** — syslog files (live append-only, rotated immutable)
- **acct-logs**, etc. — broader log group names for operating on all log files

For a major system upgrade, unlock everything. For routine config edits,
unlock just `etc` or `configs`. For log rotation, unlock only the relevant
log group. For package installation, [install.pl](https://github.com/lippard661/distribute)
uses `sysunlock -a -q -g group` to verify the needed group is unlocked
before proceeding.

## OpenBSD KARL Integration

OpenBSD's KARL feature rebuilds the kernel and relinks key libraries on each
reboot to reorder ROP gadgets. This requires those files to be unlocked
briefly. Files involved are marked with `!` in the config.

Library relinking occurs early in the boot process, *before* rc.securelevel
runs — so files must be unlocked before exiting single-user mode, not just
before rc.securelevel. The workflow is:

**In single-user mode**, before issuing `exit` to return to multi-user:
```sh
sysunlock -s    # unlock ! files so library relinking can proceed
exit            # return to multi-user; KARL runs, then rc.securelevel raises securelevel
```

**In `/etc/rc.securelevel`**, after the securelevel is raised:
```sh
/usr/local/bin/syslock -swf
# -s: lock only !-marked files, leaving any other unlocked groups alone
# -w: wait for kernel reorder/relink to complete before locking
# -f: force lock after securelevel has already been raised
```

The `-s` flag is important: it locks only the KARL-related files, not
everything. Any other groups you deliberately unlocked (for a port build
or other work in single-user mode) are left in their current state.

If you need to exit to multi-user without locking anything — for example,
for a ports build that fails under single-user mode — unlock rc.securelevel
and comment out the syslock line before issuing `exit`, otherwise it will
re-lock on the securelevel transition.

The tradeoff: keeping KARL files locked most of the time disables the
reordering mitigation but maintains immutability of those files. Whether
to prioritize ROP reordering or immutability depends on your threat model.

## Operational Examples

**Log rotation** (add to crontab around newsyslog):
```sh
/usr/local/bin/sysunlock -g newsyslog-logs
/usr/sbin/newsyslog
/usr/local/bin/syslock -g newsyslog-logs
```

Or to run the relock asynchronously:
```sh
/usr/local/bin/sysunlock -g newsyslog-logs
echo "/usr/local/bin/syslock -g newsyslog-logs" | /usr/bin/at now + 5 minutes
```

**DNSSEC zone signing** (dns group covers both DNS config files and zone
files; use :uchg to unlock only zone files, leaving schg config files locked):
```sh
/usr/local/bin/sysunlock -g dns:uchg
/usr/local/bin/sign-zones.sh
/usr/local/bin/syslock -g dns:uchg
```

**System upgrade** (unlock everything to avoid partial install failures):
```sh
sysunlock
syspatch   # or pkg_add -u, or whatever upgrade mechanism
syslock
```

**Verify a group is unlocked** (used by install.pl):
```sh
sysunlock -a -q -g configs || { echo "configs group is locked"; exit 1; }
```

## Installation

### Recommended: OpenBSD signed package

```
pkg_add ./syslock-<version>.tgz
```

Or using [install.pl](https://github.com/lippard661/distribute) on OpenBSD,
Linux, or macOS.

The OpenBSD package is signed with signify. To verify:
```
signify -C -p discord.org-2026-pkg.pub -x syslock-<version>.tgz
```
Public key: https://www.discord.org/lippard/software/discord.org-2026-pkg.pub

### Manual installation

```sh
cp src/syslock.pl /usr/local/bin/syslock
ln -s /usr/local/bin/syslock /usr/local/bin/sysunlock
chmod 755 /usr/local/bin/syslock
cp etc/syslock.conf /etc/syslock.conf      # OpenBSD
# or: cp etc/linux_syslock.conf /etc/syslock.conf
# or: cp etc/macos_syslock.conf /etc/syslock.conf
chmod 600 /etc/syslock.conf
```

### Dependencies

- Perl 5 (standard modules only: strict, warnings, Getopt::Std)
- chflags/sysctl (OpenBSD/macOS)
- chattr/lsattr (Linux)

## Security Notes

- `/etc/syslock.conf` should be mode 0600 (root only); it reveals your
  locking strategy. syslock will warn if it finds the config world-readable.
- On OpenBSD, include `/etc/syslock.conf` itself in a locked group (it
  appears in the sample config) so the locking configuration is protected.
- Non-root users may use `~/.syslock.conf` for uchg/uappnd operations on
  files they own.
- The audit flag (-a) is used by [install.pl](https://github.com/lippard661/distribute)
  to verify groups are in the expected state before and after installation.

## Related Tools

- [distribute](https://github.com/lippard661/distribute) — uses syslock audit mode during package installation
- [reportnew](https://github.com/lippard661/reportnew) — log monitoring; syslock manages append-only/immutable log files
- [sigtree](https://github.com/lippard661/sigtree) — file integrity monitoring; manages its own spec file flags internally
- [rsync-tools](https://github.com/lippard661/rsync-tools) — rsync automation; syslock used in pre/post hooks

## Author

Jim Lippard
https://www.discord.org/lippard/
https://github.com/lippard661

## License

See individual files for license information.

## Changelog

See docs/ChangeLog for detailed modification history.
