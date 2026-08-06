> Sanitized mirror of Forgejo `serhii/bluray-burner`. Source code is not published here.
>
> Commit texts: `commits/`. Need the code? Email: sergeyalpatov1@gmail.com
> Source: Forgejo `serhii/bluray-burner` | Synced: 2026-10-05T01:37:30Z

---

# bluray-burner

Batch-burn ROM archives onto Blu-ray (BD-R 25GB) with minimal user interaction.

The script splits the source into **units** and takes them **strictly in alphabetical
order**, packing them onto a disc until it is full. As soon as the next unit does not fit,
the disc is closed (free space is left unused). The process is iterative: after each disc
the burned units are recorded in a manifest, and a re-run continues where it left off.

## Unit layouts

On the first run (before the project file is created) the script asks how you want to
split the source into burn units. The chosen layout is saved and a re-run never re-asks.

| Mode | A unit is... | Fits |
|---|---|---|
| `files` | each **file** in the source root | flat `.zip` sets (No-Intro dumps) |
| `folders` | each **subfolder** | the original per-folder mode |
| `recursive` | **every file** anywhere in the tree | nested collections of any depth |

The default is picked automatically (flat sets -> `files`, folder collections -> `folders`,
mixed/deep trees -> `recursive`) and can be overridden with `--mode`. Regardless of the
layout, the units are always selected **in alphabetical order**.

## Features

- Automatic disc capacity detection via `dvd+rw-mediainfo` (manual input if the drive is empty)
- Flat ISO layout: every file lands in the disc root (Joliet name pre-check + automatic
  shortening of names that are too long for Joliet - see below)
- Disc label: first 3 letters of the alphabetically first file - first 3 letters of the
  alphabetically last file, e.g. `TKA-TOB`
- Live burn progress in the terminal (`growisofs`, up to 6x on BD-R)
- Live staging progress in the terminal (overall percentage of the disc's bytes
  copied to `~/.burn_staging`, one line per second + a line per unit)
- Disc pre-check before burning (blank / not blank), auto-eject after burn
- `.burned_manifest` (project + progress) + `.burn_sizes.tsv` cache next to the sources
- `--dry-run`: show the full plan for all discs without burning
- Oversized-unit guard: a unit bigger than one disc is reported and skipped, never
  silently wedges the run

## Dependencies

| Dependency | Package | Purpose |
|---|---|---|
| `growisofs` | dvd+rw-tools | burns the ISO directly to the drive, no temp file |
| `dvd+rw-mediainfo` | dvd+rw-tools | disc capacity and state detection |
| `genisoimage`/`mkisofs` | cdrtools (Arch) / genisoimage (Debian) | builds the ISO stream |
| `eject` | eject | disc ejection after burn |
| `python3` | python | manifest, sizes, math |
| `rsync` | rsync | only for `--safety staging` |

### Install with one command

**Arch Linux / Manjaro:**
```bash
sudo pacman -S --needed dvd+rw-tools eject python rsync
```
(`dvd+rw-tools` pulls in `cdrtools` for `genisoimage`/`mkisofs`.)

**Debian / Ubuntu / Mint:**
```bash
sudo apt-get update && sudo apt-get install -y dvd+rw-tools genisoimage eject python3 rsync
```

**Fedora:**
```bash
sudo dnf install -y dvd+rw-tools genisoimage eject python3 rsync
```

**openSUSE:**
```bash
sudo zypper install -y dvd+rw-tools genisoimage eject python3 rsync
```

**Alpine:**
```bash
sudo apk add dvd+rw-tools genisoimage eject python3 rsync
```

### Optical drive permissions

The user needs access to `/dev/sr*`. On Arch add yourself to the `optical` group
(other distros use `cdrom` or `optical` - check `ls -l /dev/sr0`):

```bash
sudo usermod -aG optical "$USER"   # or cdrom
# then log out and back in
```

On systems with `uaccess`/ACL the logged-in user may already have access - verify with
`[ -w /dev/sr0 ] && echo ok`.

## Usage

```bash
# Full plan without burning (first run: pick a unit layout)
./burn_bluray.sh --source /path/to/roms --dry-run

# Burn (insert disc -> Enter -> progress -> eject -> next disc)
./burn_bluray.sh --source /path/to/roms
```

## Environment file (`.env`)

A `.env` file next to `burn_bluray.sh` is loaded automatically, so you do not
have to pass the same flags every time. Values act as defaults and can always
be overridden by command-line flags (CLI wins).

```bash
# .env  (see .env.example) - a copy is NOT committed to git
SOURCE_DIR="/run/user/$(id -u)/gvfs/smb-share:server=...,share=.../path"
SAFETY=staging        # cache | staging | none
#CAPACITY_GB=25
#DEVICE=/dev/sr0
#SPEED=6
#MODE=files
```

With that in place the common runs reduce to:

```bash
./burn_bluray.sh --dry-run     # plan
./burn_bluray.sh               # burn     
```

The network path can use `$(id -u)` / `$(whoami)` since it is sourced by bash.

Burn flow:

1. First run asks for the unit layout (saved into `.burned_manifest`)
2. Capacity is detected automatically, the disc plan is shown (units, sizes, label)
3. Press Enter - the burn starts with live `N% done` progress
4. After the burn the disc is ejected, units are marked, the script waits for the next disc
5. `q` at any time - progress is saved, a re-run continues

### Options

```
--source DIR    Source directory (required)
--mode MODE     Unit layout: files | folders | recursive
                (default: interactive on first run, then remembered)
--device DEV    Burner device (default: /dev/sr0, auto-finds /dev/sr*)
--speed N       Write speed (default: maximum, 6x on Pioneer BDR-UD04)
--capacity GB   Disc capacity in GB, skip auto-detection
--rescan        Recompute the .burn_sizes.tsv cache
--safety MODE   cache | staging | none (default: cache)
                cache: pre-read all files into RAM (with retries) before
                burning to catch NAS read failures before the disc is touched
                staging: copy disc contents to ~/.burn_staging before burning,
                so the burn never touches the network mount (needs free space
                for one disc; the staging area is cleaned after a burn).
                Shows live overall progress in percent while copying.
                Automatically downgraded to cache when the source is on a
                local disk - staging against a local disk is pointless
                none: burn directly, no safety checks (risky on a network
                source - a mid-burn read failure wastes the disc)
--dry-run       Show the plan for all discs and exit
-h, --help      Show help
```

> **Network sources (SMB/NFS/gvfs):** `--safety staging` is the most reliable mode -
> the disc contents are copied locally first, so a mid-burn NAS hiccup can never waste
> a disc. `cache` pre-reads files into the page cache, but with less RAM than the disc
> size the later reads still hit the network.

## How it works

1. **Layout**: on first run the unit layout is chosen (or taken from `--mode`) and saved
   into `.burned_manifest`. The project file is a JSON object:
   `{"mode": "files|folders|recursive", "burned": [...]}`. Old-format manifests
   (a bare JSON array) are migrated to `folders` mode automatically.
2. **Selection**: unburned units are sorted in alphabetical order (`LC_ALL=C` /
   codepoint order for determinism).
3. **Packing**: units are added to the disc in order while the total size fits the disc
   capacity; the first unit that does not fit starts the next disc. A unit larger than
   a whole disc is reported and skipped.
4. **Label**: the alphabetically first and last file names across all disc units are
   found, the first 3 letters of each are joined with `-` (e.g. `TKA-TOB`).
5. **Protection**: `--safety` controls how the source is protected before the burn:
   `cache` (pre-read into RAM, default), `staging` (local copy in `~/.burn_staging`)
   or `none`. The source filesystem type is detected automatically
   (`findmnt`/`stat -f`); `staging` is downgraded to `cache` when the source is on a
   local disk. A burn is only started when every source file was read successfully -
   a NAS failure can never waste a disc mid-burn.
6. **Burn**: `growisofs -Z /dev/sr0 -R -J -joliet-long -V LABEL <files>` - the ISO is
   built and written on the fly, no temp file (a 25GB ISO does not fit in /tmp).
   genisoimage places every file argument in the disc root, giving the flat layout.
   Files whose name has to be shortened are passed as `-graft-points` pairs
   (`new-name=/path/to/file`) instead, so the source keeps its original name.
7. **Progress**: successful burn -> units into the manifest -> eject -> next disc.
   Success is verified from the burn log: growisofs exit code 0, no error markers, and
   genisoimage's `extents written` summary. growisofs ejects the tray after the burn
   (over USB the reload can fail, and on BD-R SRM+POW the track stays "partial
   incremental" - neither is a failure), so the drive state after the burn is ignored.

## Fault tolerance (NAS outages)

The manifest lives on the network share, but a NAS that powers off right after a
successful burn would otherwise lose that disc's progress and make a re-run re-burn
the same disc (wasting media). To prevent this the project state is kept in **two**
places:

- **local shadow** `~/.burn_state/<hash>.json` - authoritative, always writable
- **NAS manifest** `.burned_manifest` - portable replica next to the sources

Every state change is written to the shadow **first** (always succeeds), then synced
to the NAS best-effort. Planning reads the shadow (healed to the shadow+NAS union at
startup). So if the share is unreachable when a disc is marked burned, the progress
still lands in the shadow, and the next run (when the share is back) heals the NAS
manifest and continues with the **next** disc - the burned one is never re-burned.
If staging/caching fails because the source became unreachable, the script exits
cleanly (progress is safe) instead of looping. To move the whole project to another
machine, copy the NAS manifest (and the shadow, if any) - the union is applied.

## Long names and Joliet collisions

`-joliet-long` lets Joliet carry 103 characters, and genisoimage **silently cuts every
longer name at exactly 103 characters** (extension included in the cut). Two files that
share their first 103 characters therefore get the same Joliet name and genisoimage
aborts the whole build (`have the same Joliet name`, rc=255) - without the pre-check
that would happen *after* 20+ GB have been staged.

Instead of cancelling the disc, the script plans the on-disc names:

| Situation | What happens |
|---|---|
| Name fits, no twin | kept as it is |
| Colliding names, **byte-identical** files | the copy is dropped and marked `skipped` in the project (the first one is burned) |
| Colliding names, **different** files | both are burned, names are shortened to fit (a `~N` tag keeps them unique) |
| Name longer than 103 chars, no twin | shortened so the extension and the informative tail survive |

The shortened name keeps the head and the tail of the original and its extension, e.g.

```
Tohoku ... Kanshuu~1no Oni Training (Japan).zip
Tohoku ... Kyouju~2(Taiwan) (Tw, Cn Region Lock).zip
```

This happens **on the disc only** (`-graft-points`): the source files, the staging copy
and the project state keep the original names. On Linux the full original name is still
visible through the Rock Ridge records; the 103-char Joliet name is what Windows and
legacy readers see. `--dry-run` prints every planned rename.

## Notes

- Source files are **not deleted** after burning - they stay on the network share,
  so a failed disc can simply be re-burned
- The `.burn_sizes.tsv` cache is created on first run (scanning ~400GB over the network
  takes time); run with `--rescan` after adding new files
- Auto capacity detection uses the drive's `READ CAPACITY` (works on blank discs);
  a 2% safety margin is applied, and after staging the actual staged size is checked
  against the usable capacity before the burn starts - an oversized set can never
  waste a disc (a 24.87GB set on a 24.76GB BD-R failed at the last block)
- With an empty drive the script asks to insert a disc or enter the capacity manually
- The local shadow `~/.burn_state/` is per-source (keyed by a hash of the source
  path) and can be removed safely: the next run rebuilds it from the NAS manifest
