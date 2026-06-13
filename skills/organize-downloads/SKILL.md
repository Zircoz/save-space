---
name: organize-downloads
description: Analyze the ~/Downloads folder (or OS equivalent) to categorize files by type, flag stale installers/archives/documents, detect duplicates, and produce a human-approved cleanup and move plan. Use when asked to clean up downloads, free space from the downloads folder, or tidy up downloaded files.
---

# Organize Downloads

Clean up the Downloads folder by categorizing files, surfacing what is safe to
delete vs. move, and executing only after explicit human approval.

## Core Rules
- Never delete without explicit per-item or per-category human approval.
- Prefer moving to an appropriate folder over deleting when the file has lasting value.
- Use recoverable deletion (`trash`/`gio trash`) — never bare `rm`.
- Cap a single run at 500 files or 10 GB of deletions unless the user explicitly raises the limit.

## Step 1 — Locate the Downloads folder

Resolve the correct path for the current OS:
- **Linux:** `$XDG_DOWNLOAD_DIR` if set, otherwise `~/Downloads`
- **macOS:** `~/Downloads`
- **Windows (WSL):** `/mnt/c/Users/$USERNAME/Downloads`

Confirm the resolved absolute path with the user before scanning.

## Step 2 — Fast aggregate scan

```bash
DL="<resolved-absolute-path>"

# Total size
du -sh "$DL"

# File count (top level only — Downloads folders are usually flat)
find "$DL" -maxdepth 1 -type f | wc -l

# Largest individual files with dates
find "$DL" -maxdepth 1 -type f -printf '%s\t%TY-%Tm-%Td\t%f\n' \
  | sort -rn | head -30

# Files not modified in 90+ days
find "$DL" -maxdepth 1 -type f -mtime +90 -printf '%s\t%TY-%Tm-%Td\t%f\n' \
  | sort -rn | head -50
```

## Step 3 — Categorize

Group files into these categories using extension and filename patterns:

| Category | Extensions / Patterns | Default posture |
|---|---|---|
| Installers | `.dmg` `.exe` `.deb` `.rpm` `.pkg` `.AppImage` `.msi` | Delete if >30 days old |
| Archives | `.zip` `.tar.gz` `.tar.bz2` `.rar` `.7z` `.gz` `.tgz` | Delete if >90 days old |
| PDFs | `.pdf` | Move to `~/Documents/` |
| Images | `.jpg` `.jpeg` `.png` `.heic` `.gif` `.webp` | Move to `~/Pictures/` |
| Videos | `.mp4` `.mov` `.mkv` `.avi` `.webm` | Flag size; ask user |
| Audio | `.mp3` `.flac` `.wav` `.m4a` `.ogg` | Flag; ask user |
| Documents | `.docx` `.xlsx` `.pptx` `.odt` `.csv` `.txt` `.md` | Move to `~/Documents/` |
| Scripts / Executables | `.sh` `.py` `.js` `.bin` | Flag only — never auto-delete |
| Other | anything else | Flag; ask user |

**Duplicate detection:** within each category, files whose names match after
stripping trailing ` (1)`, ` (2)`, `_copy`, `_2`, `- Copy` suffixes and whose
sizes are identical are likely duplicates — flag the older copy as a deletion
candidate, keep the newest.

## Step 4 — Present summary and get approval

Present a table before touching anything:

```
Category         | Count |    Size | Recommendation
─────────────────────────────────────────────────────
Installers       |    42 |  3.1 GB | Delete >30 days old (38 qualify)
Archives         |    17 |  1.4 GB | Delete >90 days old (11 qualify)
PDFs             |    23 |  180 MB | Move to ~/Documents/
Images           |    61 |  420 MB | Move to ~/Pictures/
Videos           |     4 |  2.2 GB | Review — large files
Duplicates       |     8 |   90 MB | Delete older copies
Scripts / Other  |    31 |  310 MB | Needs review
─────────────────────────────────────────────────────
Reclaimable      |       |  4.6 GB
Moveable         |       |  600 MB
```

For each category, ask the user to choose: **delete all qualifying**, **review
individually**, **move**, or **skip**. Wait for an answer on every category
before proceeding to Step 5.

## Step 5 — Execute

Operate strictly in the order the user approved. For each approved item:

```bash
# Recoverable deletion
trash "<absolute-path>"
# fallback if trash not found:
gio trash "<absolute-path>"

# Move to destination
mv "<absolute-path>" "<destination-folder>/"
```

Before each operation, verify the path starts with `$DL` (for deletions) or is
an approved destination (for moves). Report a before/after size:

```bash
du -sh "$DL"   # run before and after
```

List every file deleted or moved in the final report.
