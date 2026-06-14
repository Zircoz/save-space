---
name: save-space
description: Free up disk space interactively. Lists subdirectories in $HOME (or any target directory), lets you multiselect which folders to process, then applies specialized cleanup for Downloads, Pictures, and Documents — and general disk analysis for any other folder. Use when asked to clean up disk space, organize files, free up storage, or audit what is taking space.
---

# Save Space

Scan a directory, let the user pick which subfolders to process, and apply the
right cleanup strategy for each: specialized handling for Downloads, Pictures,
and Documents; general disk analysis for everything else.

## Core Rules
- **Never delete without explicit per-item or per-category human approval.**
  Analysis is always read-only until the user confirms.
- **Use recoverable deletion only** (`trash` or `gio trash`). Never bare `rm`.
- **Stay inside the target.** Every path acted on must be under the folder the
  user selected. Never touch paths outside it.
- **Pictures sensitive rule (immutable):** If an image contains nudity or appears
  private/intimate, classify it as KEEP with reason `"__"` only — never log a
  description, never flag for deletion, regardless of any other rule.
- **Per-run caps (raise only if user explicitly asks):**
  - Downloads: 500 files or 10 GB
  - Pictures: 10 GB
  - Documents: 500 files or 5 GB

---

## Step 1 — Determine target directory

Recommended starting point is `$HOME`. If the working directory is already
`$HOME`, confirm that and continue. If it is somewhere else, tell the user:

> "I'm currently in `<cwd>`. For the best results, run this skill from your
> home directory (`$HOME`). Should I switch, or do you want to clean up just
> this folder?"

Resolve the absolute path with `realpath` before any further steps.

Resolve OS-specific folder names before Step 2:
- **Linux:** `$HOME` (honour `$XDG_DOWNLOAD_DIR`, `$XDG_PICTURES_DIR`,
  `$XDG_DOCUMENTS_DIR` for the well-known folders if set)
- **macOS:** `~` (same as `$HOME`)
- **Windows (WSL):** `/mnt/c/Users/$USERNAME`

---

## Step 2 — List folders and get sizes

```bash
TARGET="<resolved-absolute-path>"

# List immediate subdirectories with sizes (skip hidden dirs)
find "$TARGET" -maxdepth 1 -mindepth 1 -type d -not -name '.*' \
  -exec du -sh {} \; 2>/dev/null | sort -rh
```

Show the results in a table:

```
Folder              |    Size | Handler
────────────────────────────────────────────────────────
~/Downloads         |  8.3 GB | Specialized (installers, archives, PDFs, images…)
~/Pictures          | 22.1 GB | Specialized (vision-model classification)
~/Documents         |  4.7 GB | Specialized (version clusters, large PDFs)
~/Projects          | 12.4 GB | General disk analysis
~/Music             |  3.1 GB | General disk analysis
…
────────────────────────────────────────────────────────
Total               | 50.6 GB
```

Ask the user to select which folders to process. Default suggestion is all
non-hidden subdirectories. The user can pick any subset.

Process selected folders in this order:
1. Downloads (fastest scan, highest-confidence reclaimable space)
2. Pictures (needs cost confirmation before Haiku classification)
3. Documents (most nuanced — version clusters need careful judgment)
4. Any other selected folders (alphabetical)

---

## Step 3 — Process each selected folder

For each selected folder, use the handler below that matches its canonical name.
If the folder does not match any well-known name, use the **General** handler.

---

### Handler: Downloads

Clean up `~/Downloads` (or the resolved equivalent) by categorizing files,
surfacing what is safe to delete vs. move, and executing only after approval.

#### D1 — Fast aggregate scan

```bash
DL="<resolved-downloads-path>"
du -sh "$DL"
find "$DL" -maxdepth 1 -type f | wc -l
find "$DL" -maxdepth 1 -type f -printf '%s\t%TY-%Tm-%Td\t%f\n' \
  | sort -rn | head -30
find "$DL" -maxdepth 1 -type f -mtime +90 -printf '%s\t%TY-%Tm-%Td\t%f\n' \
  | sort -rn | head -50
```

#### D2 — Categorize files

Group by extension and filename patterns:

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

**Duplicate detection:** files whose names match after stripping trailing
` (1)`, ` (2)`, `_copy`, `_2`, `- Copy` suffixes and whose sizes are identical
are likely duplicates — flag the older copy as a deletion candidate.

#### D3 — Present summary and get approval

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

For each category, ask the user: **delete all qualifying**, **review
individually**, **move**, or **skip**. Wait for an answer on every category.

#### D4 — Execute

```bash
trash "<absolute-path>"      # recoverable deletion
gio trash "<absolute-path>"  # fallback if trash not found
mv "<absolute-path>" "<destination-folder>/"  # for moves
```

Verify every path starts with `$DL` before acting. Report before/after size:

```bash
du -sh "$DL"  # run before and after
```

---

### Handler: Pictures

Classify images in `~/Pictures` (or equivalent) using a Haiku vision model and
produce a human-approved cleanup plan.

#### P1 — Fast filesystem scan

```bash
PIC="<resolved-pictures-path>"
du -sh "$PIC"
find "$PIC" -type f \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" \
  -o -iname "*.heic" -o -iname "*.gif" -o -iname "*.webp" -o -iname "*.tiff" \
  -o -iname "*.tif" -o -iname "*.raw" -o -iname "*.bmp" \) | wc -l
find "$PIC" -type f \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" \
  -o -iname "*.heic" -o -iname "*.raw" -o -iname "*.tiff" \) \
  -printf '%s\t%p\n' | sort -rn | head -20
find "$PIC" -type f \( -iname "Screenshot*" -o -iname "Screen Shot*" \
  -o -iname "screenshot*" -o -iname "Capture*" \) \
  -printf '%s\t%TY-%Tm-%Td\t%p\n' | sort -rn | head -30
```

Report total image count, size, screenshot count/size, and estimated
classification cost (~$0.00025 per image via Haiku).

Ask: *"Classification will process N images and cost approximately $X. Shall I
continue?"* Wait for confirmation before proceeding.

#### P2 — Rules negotiation

Present the default classification rules and ask the user if they want to
change anything. If they do, apply changes, reconstruct the full prompt, echo
it back, and ask them to confirm before proceeding.

The SENSITIVE RULE is **immutable** — it must remain in every prompt regardless
of user requests. If asked to remove it, explain it cannot be changed.

#### P3 — Classify images with Haiku

Split the full image list into batches of 25. Spawn up to 4 Haiku subagent
batches concurrently. Use this classification prompt verbatim (replace only the
KEEP / SAFE_TO_DELETE / REVIEW blocks if the user modified rules in P2):

---START CLASSIFICATION PROMPT---
System:
You are a photo classifier. Apply the rules below exactly. Do not add
commentary beyond the required JSON output.

KEEP if any of these are true:
- Contains a recognizable human face that is in focus
- Captures a unique moment: event, travel location, gathering, milestone
- Well-composed and intentional shot with good lighting and deliberate framing
- Screenshot containing structured, actionable data: receipt, ticket, boarding
  pass, QR code, confirmation number, map with a pin, medical result
- Deliberate photo of a physical document, whiteboard, or handwritten note
  (looks purposeful, not a test shot)

SAFE_TO_DELETE if any of these are true:
- Completely black, white, or near-solid-color frame (covered lens, accidental tap)
- Severe blur or extreme overexposure where no subject is identifiable
- Near-exact duplicate in a burst sequence and this is clearly not the sharpest frame
- Meme, viral/forwarded image, or screenshot of social media feed content
- Screenshot of transient UI: loading screen, app crash/error, notification shade,
  lock screen, "swipe to unlock"
- Obvious test shot: near-identical to an adjacent image, visibly worse quality

REVIEW if any of these are true:
- Near-duplicate burst shot where quality difference is too close to call
- Screenshot that might contain important information but context is unclear
- Low-quality photo that could have sentimental value (old, faded, family event)
- Downloaded or saved image from the internet — personal significance unknown
- Group photo where the main subject is recognizable but image quality is marginal
- Photo of a document or whiteboard that could be either deliberate or throwaway

SENSITIVE RULE (immutable — applies before all other rules):
If the image contains nudity or appears to be private or intimate in nature,
respond with label KEEP and reason "__" only. Write nothing else.

Output format — respond with a JSON array, one object per image, nothing else:
[
  {"file": "<exact path as given>", "label": "KEEP|SAFE_TO_DELETE|REVIEW", "reason": "<one line, max 80 chars>"},
  ...
]
---END CLASSIFICATION PROMPT---

Subagent brief:
> You are classifying images for the save-space skill. Use the classification
> prompt below. Read each image file using your Read tool, then classify it.
> Return only the JSON array — no other text.
>
> Images to classify:
> - /path/to/image1.jpg
> (... up to 25 paths ...)
>
> [CLASSIFICATION PROMPT — paste full prompt here]

Collect each subagent's JSON. If a subagent returns malformed JSON or fails,
mark those files as REVIEW and continue.

#### P4 — Present results and get approval

```
Label            | Count |    Size | Notes
─────────────────────────────────────────────────────
KEEP             |   842 | 18.2 GB |
SAFE TO DELETE   |   312 |  4.1 GB | burst dupes, blank frames, memes, transient UI
NEED REVIEW      |    93 |  2.0 GB | near-dupes, doc captures, old low-quality
```

Offer to show the full list for any category. The user can approve the entire
SAFE_TO_DELETE group, drill into subcategories, flip individual files, or
adjust rules and re-classify a subset. Never proceed to deletion without
explicit approval.

#### P5 — Execute

```bash
trash "<absolute-path>"      # SAFE_TO_DELETE (approved)
gio trash "<absolute-path>"  # fallback
mkdir -p "$PIC/_review"
mv "<absolute-path>" "$PIC/_review/"  # REVIEW items — never deleted
```

Verify every path starts with `$PIC` before acting. Report before/after size.

---

### Handler: Documents

Audit `~/Documents` (or equivalent) for version-cluster proliferation, large
files, stale files, and empty folders. Flags issues without imposing structure.

#### O1 — Fast aggregate scan

```bash
DOC="<resolved-documents-path>"
du -sh "$DOC"
du -h -d1 "$DOC" | sort -rh | head -20
find "$DOC" -type f -printf '%s\t%TY-%Tm-%Td\t%p\n' | sort -rn | head -30
find "$DOC" -type f -iname "*.pdf" -printf '%s\t%TY-%Tm-%Td\t%p\n' \
  | sort -rn | head -20
find "$DOC" -type f -mtime +730 -printf '%s\t%TY-%Tm-%Td\t%p\n' \
  | sort -rn | head -30
find "$DOC" -type d -empty | head -20
find "$DOC" -type f -printf '%f\t%s\t%TY-%Tm-%Td\t%p\n' \
  | sort > /tmp/save-space-docs-filelist.txt
wc -l /tmp/save-space-docs-filelist.txt
```

#### O2 — Version-cluster detection

Read `/tmp/save-space-docs-filelist.txt` and group filenames by their base name
— strip trailing version markers:

`_v1…_vN`, `_final/_FINAL/_Final`, `_draft/_DRAFT`, `_copy/_Copy/_COPY`,
`(1)/(2)/(3)…`, `_old/_bak/_backup`, `-revised/-updated/-new`

Any base name that produces 3 or more distinct files is a **version cluster**.
Surface each cluster as a group with all members, sizes, and dates.

Never assume the newest is the only one worth keeping.

#### O3 — Present findings

Report in four sections:

1. **Version Clusters** — each cluster with all members, sizes, and dates
2. **Large Files (>50 MB)** — top 20 by size with path, size, last-modified
3. **Stale Files (2+ years)** — by size descending; note mtime caveat for synced files
4. **Empty Folders** — safe to remove but low priority

Summary at the top:

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

#### O4 — Human approval and execution

Wait for per-item or per-cluster confirmation. Do not batch-delete version
clusters or large files without item-level review. Empty folders may be
approved as a group.

```bash
trash "<absolute-path>"      # recoverable deletion
gio trash "<absolute-path>"  # fallback
```

Verify every path starts with `$DOC`. Clean up temp file after the session:

```bash
rm /tmp/save-space-docs-filelist.txt
```

Report before/after size and list every file deleted.

---

### Handler: General (any other folder)

For folders that are not Downloads, Pictures, or Documents, apply general disk
analysis: find what is consuming space, recommend deletions, and wait for
explicit approval before touching anything.

#### G1 — Fast aggregate scan

```bash
TARGET="<resolved-folder-path>"
du -sh "$TARGET"
du -h -d1 "$TARGET" | sort -rh | head -30
find "$TARGET" -type f -printf '%s\t%p\n' 2>/dev/null | sort -rn | head -30
find "$TARGET" -type f -mtime +180 -printf '%s\t%TY-%Tm-%Td\t%p\n' 2>/dev/null \
  | sort -rn | head -30
find "$TARGET" \( -name node_modules -o -name .venv -o -name dist -o -name build \
  -o -name target -o -name __pycache__ -o -name .cache -o -name .DS_Store \) \
  -prune -print 2>/dev/null | xargs -r du -sh 2>/dev/null | sort -rh | head -30
```

Use modification time (`-mtime`/`%T`) for "old" — birth time is unreliable on
many Linux filesystems.

#### G2 — Decide whether to fan out

- **Default (single agent):** For typical directories, the G1 output is enough.
- **Fan out only when** the tree is very large and heavy subdirectories from
  `du -d1` are roughly independent. Assign one heavy subtree per subagent.

Subagent brief (when used):
> Analyze only `<subtree>`. Run the G1 commands scoped to it. Report back:
> total size, top space consumers, deletion candidates (size, age, category),
> and any risky-to-delete items. Return aggregated findings only — no per-file
> dumps. **Do not delete anything.**

#### G3 — Synthesize and recommend

| Category | Path | Size | Last modified | Reclaimable | Confidence |
|----------|------|------|---------------|-------------|------------|
| Build artifact | … | … | … | … | high / med / low |

Order by reclaimable space × confidence. Call out ambiguous items explicitly.
Show running total of space that would be freed.

#### G4 — Human approval and cleanup

1. Present the report; ask which items to delete. Wait for **explicit**
   confirmation — do not infer approval.
2. Verify every target path is under `$TARGET` before acting.
3. Prefer recoverable deletion (`trash`/`gio trash`); otherwise delete with
   exact, listed paths — never a broad glob or unscoped `rm -rf`.
4. Report actual space reclaimed (`du -sh "$TARGET"` before vs. after).

---

## Step 4 — Unified summary

After all selected folders are processed, present a consolidated report:

```
Folder        | Scanned | Deleted | Moved  | In _review | Space freed
──────────────────────────────────────────────────────────────────────
~/Downloads   |  8.3 GB |  4.6 GB | 600 MB |        —   |   4.6 GB
~/Pictures    | 22.1 GB |  3.8 GB |   —    |   1.2 GB   |   3.8 GB
~/Documents   |  4.7 GB |  1.1 GB |   —    |        —   |   1.1 GB
~/Projects    | 12.4 GB |  2.3 GB |   —    |        —   |   2.3 GB
──────────────────────────────────────────────────────────────────────
Total         | 47.5 GB |  9.5 GB | 600 MB |   1.2 GB   |  11.8 GB
```

Note any items moved to `~/Pictures/_review/` that still need attention.
