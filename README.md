# hfsplus-readonly-fix

An AI-agent skill that fixes Linux machines refusing to write to **journaled HFS+** (macOS) drives.

## The Problem

When you plug in a Mac-formatted drive on Linux, writes fail with `Read-only file system` — even though the mount looks perfectly normal. The kernel log shows the real reason:

```
hfsplus: write access to a journaled filesystem is not supported,
use the force option at your own risk, mounting read-only.
```

Linux refuses to write to journaled HFS+ volumes **by design**: the kernel's `hfsplus` driver cannot replay the Mac journal, so it falls back to read-only. A plain `mount -o rw` will never help — the flag is silently ignored. The volume must be mounted with the kernel `force` option.

## The Fix

The skill walks an agent through the whole sequence:

1. **Detect** — confirm the mount is `ro`/`hfsplus` and map the label to a device node
2. **Unmount** — free the volume, handling `DeviceBusy` (e.g. a file-manager window holding it)
3. **Remount with force** — no sudo required:

   ```bash
   udisksctl mount -b /dev/sdXY --options force,rw
   ```

4. **Verify** — confirm `findmnt` actually shows `rw` before trusting it

> ⚠️ **Risk:** the kernel itself warns the `force` option is "at your own risk." Writing to a journaled HFS+ volume can corrupt it. The safest long-term fix is to copy data off to an exFAT or Linux-native drive.

This skill was extracted from a real troubleshooting session where the exact steps above were executed successfully.

## Requirements

- Linux with the `hfsplus` kernel module (standard)
- `udisksctl` (part of `udisks2`)
- No sudo needed for the primary path

## Installation

### Claude Code / OpenCode / Codex

Copy the skill into your agent's skill directory:

```bash
git clone https://github.com/Raymondycp/write-to-HFS.git
```

```bash
# Claude Code
mkdir -p ~/.claude/skills
cp -r hfsplus-readonly-fix/skills/hfsplus-readonly-fix ~/.claude/skills/

# OpenCode
mkdir -p ~/.config/opencode/skills
cp -r hfsplus-readonly-fix/skills/hfsplus-readonly-fix ~/.config/opencode/skills/

# Codex
mkdir -p ~/.codex/skills
cp -r hfsplus-readonly-fix/skills/hfsplus-readonly-fix ~/.codex/skills/
```

Project-scoped (recommended for shared repos) — drop it in `.opencode/skills/` (OpenCode), `.claude/skills/` (Claude Code), or `.codex/skills/` (Codex) inside your project.

## Usage

The skill triggers automatically when the agent sees any of:

- Write fails with `Read-only file system` on an HFS+ mount
- `findmnt` shows `type hfsplus` with `OPTIONS ... ro`
- Kernel log contains `write access to a journaled filesystem is not supported`

You can also ask directly:

> My Mac drive won't let me write to it on Linux, fix it.

## Layout

```
skills/
  hfsplus-readonly-fix/
    SKILL.md    # The skill (frontmatter + detection + fix + common mistakes)
```

## License

[MIT](LICENSE)
