---
name: clean-home
description: Master orchestrator that runs organize-downloads, organize-pictures, and organize-documents across the three common user folders (Downloads, Pictures, Documents) and produces a unified space-reclaim summary. Use when asked to clean up the home folder, free up disk space across personal files, or get an overview of what is taking up space in user directories.
---

# Clean Home

Run all three personal-folder skills in sequence, produce a unified summary,
and let the user approve cleanup per folder. This skill orchestrates
`organize-downloads`, `organize-pictures`, and `organize-documents` — it does
not duplicate their logic.

## Core Rules
- Each sub-skill enforces its own safety rules. This orchestrator adds no
  extra permissions beyond what those skills allow.
- Never delete anything without per-folder explicit human approval.
- Run folders in sequence (not parallel) to keep approval conversations clear
  and avoid concurrent writes to overlapping destinations (e.g. both Downloads
  and Documents skills moving PDFs to `~/Documents/`).

## Step 1 — Confirm scope

Ask the user which folders to include. Default is all three:

> "I'll scan Downloads, Pictures, and Documents and give you a combined summary
> before asking what to clean up. Want to skip any of these folders?"

Resolve the correct base paths for the current OS before proceeding:
- **Linux:** `$HOME/Downloads`, `$HOME/Pictures`, `$HOME/Documents`
  (honour `$XDG_DOWNLOAD_DIR` / `$XDG_PICTURES_DIR` / `$XDG_DOCUMENTS_DIR` if set)
- **macOS:** `~/Downloads`, `~/Pictures`, `~/Documents`
- **Windows (WSL):** `/mnt/c/Users/$USERNAME/` prefixed equivalents

## Step 2 — Fast size overview (before invoking sub-skills)

Get a quick top-level picture without a full scan:

```bash
for DIR in "$HOME/Downloads" "$HOME/Pictures" "$HOME/Documents"; do
  [ -d "$DIR" ] && du -sh "$DIR" 2>/dev/null || echo "$DIR not found"
done
```

Present to the user:
```
Folder        |    Size
───────────────────────
~/Downloads   |  8.3 GB
~/Pictures    | 22.1 GB
~/Documents   |  4.7 GB
───────────────────────
Total         | 35.1 GB
```

## Step 3 — Run sub-skills in sequence

Invoke each included sub-skill one at a time. After each skill completes and
the user has approved (or skipped) that folder's cleanup, move to the next.

Order: **Downloads → Pictures → Documents**

Downloads is first because it has the highest-confidence reclaimable space
(old installers and archives) and the fastest scan. Pictures is second because
it requires the Haiku cost confirmation before proceeding. Documents is last
because it involves the most nuanced user judgment (version clusters).

When invoking each sub-skill, pass the resolved absolute path confirmed in Step 1.

## Step 4 — Unified summary

After all three sub-skills complete, present a consolidated report:

```
Folder        | Scanned | Deleted | Moved  | In _review | Space freed
──────────────────────────────────────────────────────────────────────
~/Downloads   |  8.3 GB |  4.6 GB | 600 MB |        —   |   4.6 GB
~/Pictures    | 22.1 GB |  3.8 GB |   —    |   1.2 GB   |   3.8 GB
~/Documents   |  4.7 GB |  1.1 GB |   —    |        —   |   1.1 GB
──────────────────────────────────────────────────────────────────────
Total         | 35.1 GB |  9.5 GB | 600 MB |   1.2 GB   |   9.5 GB
```

Note any items moved to `~/Pictures/_review/` that still need human attention.
