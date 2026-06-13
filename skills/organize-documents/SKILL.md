---
name: organize-documents
description: Analyze ~/Documents to surface large files, version-cluster proliferation (report_v1, report_v2, report_FINAL), old unused files, and empty folders. Flags issues without imposing a folder structure — the user decides what to do. Requires explicit human approval before any action. Use when asked to clean up documents, find duplicate or versioned files, or audit the documents folder.
---

# Organize Documents

Audit the Documents folder for space hogs, version-cluster proliferation, stale
files, and empty directories. This skill flags and recommends — it does not
impose a folder structure. The user decides what to reorganize, archive, or delete.

## Core Rules
- Never delete without explicit per-item or per-cluster human approval.
- Use recoverable deletion (`trash`/`gio trash`) — never bare `rm`.
- Do not suggest reorganizing folder structure unless the user asks for it.
- Cap a single run at 500 files or 5 GB of deletions unless the user explicitly raises the limit.

## Step 1 — Locate the Documents folder

Resolve the correct path for the current OS:
- **Linux:** `~/Documents`
- **macOS:** `~/Documents`
- **Windows (WSL):** `/mnt/c/Users/$USERNAME/Documents`

Confirm the resolved absolute path with the user before scanning.

## Step 2 — Fast aggregate scan

```bash
DOC="<resolved-absolute-path>"

# Total size and depth-1 breakdown
du -sh "$DOC"
du -h -d1 "$DOC" | sort -rh | head -20

# Largest individual files
find "$DOC" -type f -printf '%s\t%TY-%Tm-%Td\t%p\n' \
  | sort -rn | head -30

# Large PDFs specifically (common space hog)
find "$DOC" -type f -iname "*.pdf" -printf '%s\t%TY-%Tm-%Td\t%p\n' \
  | sort -rn | head -20

# Files not modified in 2+ years
find "$DOC" -type f -mtime +730 -printf '%s\t%TY-%Tm-%Td\t%p\n' \
  | sort -rn | head -30

# Empty directories
find "$DOC" -type d -empty | head -20

# All filenames (for version-cluster detection in Step 3)
find "$DOC" -type f -printf '%f\t%s\t%TY-%Tm-%Td\t%p\n' \
  | sort > /tmp/save-space-docs-filelist.txt
wc -l /tmp/save-space-docs-filelist.txt
```

## Step 3 — Version-cluster detection

Read `/tmp/save-space-docs-filelist.txt` and group filenames by their **base
name** — the filename with trailing version markers stripped. Strip patterns:
- `_v1`, `_v2`, `_v3` ... `_vN`
- `_final`, `_FINAL`, `_Final`
- `_draft`, `_DRAFT`
- `_copy`, `_Copy`, `_COPY`
- `(1)`, `(2)`, `(3)` ...
- `_old`, `_bak`, `_backup`
- `-revised`, `-updated`, `-new`

Any base name that produces 3 or more distinct files is a **version cluster**.
Surface each cluster as a group:

```
Cluster: "project_proposal" (5 files, 48 MB)
  project_proposal_v1.docx       2022-03-01   8.2 MB
  project_proposal_v2.docx       2022-03-14   9.1 MB
  project_proposal_draft.docx    2022-03-20   9.4 MB
  project_proposal_FINAL.docx    2022-04-01   9.8 MB
  project_proposal_FINAL2.docx   2022-04-03  11.5 MB   ← newest
```

The user will know which ones are safe to collapse. Never assume the newest is
the only one worth keeping — some clusters are deliberate revision histories.

## Step 4 — Present findings

Report findings in four sections, in this order:

### 1. Version Clusters
List each cluster with all members, sizes, and dates. Ask the user to review
each cluster and decide which versions to keep.

### 2. Large Files (>50 MB)
List the top 20 largest files with path, size, and last-modified date.
Highlight large PDFs specifically — these are often downloadable references
that can be retrieved again if needed.

### 3. Stale Files (not modified in 2+ years)
List by size descending. Note that mtime on documents can be misleading
(synced files may show today's date). Flag this caveat.

### 4. Empty Folders
List all empty directories. These are safe to remove but low priority.

Present a summary at the top:

```
Finding                 | Count |    Size
────────────────────────────────────────
Version clusters        |    12 | 230 MB  (47 redundant files)
Files >50 MB            |     8 | 1.4 GB
Stale files (2+ years)  |   143 | 890 MB
Empty folders           |    17 |    —
────────────────────────────────────────
Reclaimable (estimated) |       | ~1.8 GB
```

## Step 5 — Human approval and execution

For each finding category, wait for the user to specify exactly what to act on.
Do not batch-delete an entire category without item-level confirmation for
version clusters and large files. Empty folders may be approved as a group.

```bash
# Recoverable deletion
trash "<absolute-path>"
# fallback:
gio trash "<absolute-path>"
```

Before each operation verify the path starts with `$DOC`. Clean up the
temporary file list after the session:

```bash
rm /tmp/save-space-docs-filelist.txt
```

Report a before/after size (`du -sh "$DOC"`) and list every file deleted.
