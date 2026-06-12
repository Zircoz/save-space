---
name: disk-cleanup-orchestrator
description: Analyze a directory to find what is consuming disk space and recommend safe deletions for a human to approve. Use when asked to free up disk space, find large/old files, clean build artifacts/caches, or audit a folder's storage usage. Fans out to subagents only for very large trees.
---

# Disk Cleanup Orchestrator

Find what is eating disk space in a target directory and produce a prioritized,
human-approved cleanup plan. Lean on shell tools for measurement (they aggregate
far faster than per-file inspection) and use agents only for judgment.

## When to Use
The user wants to free up disk space, find large or old files, clear caches /
build artifacts, or understand where storage is going in a directory.

## Core Rules
- **Measure with the shell, not per-file loops.** `du`, `find -size`, and
  `find -mtime` aggregate in one pass. Never `stat`/`file` every file.
- **Never delete without explicit per-item human approval.** Analysis is
  read-only until then.
- **Stay inside the target.** Every path acted on must be under the directory
  the user named. Never touch paths outside it.
- **Don't trust extensions alone for "safe to delete."** A `node_modules` or
  `dist/` may belong to an active project; flag, don't assume.

## Step 1 — Scope
If the user didn't name a directory, ask which one. Confirm the absolute path
(`realpath`) so all later commands and the final deletions are unambiguous.

## Step 2 — Fast aggregate scan (orchestrator, always)
Run these directly — they answer most cleanup questions without any subagents:

```bash
TARGET="/absolute/path"

# Total size and the heaviest top-level subdirectories
du -sh "$TARGET"
du -h -d1 "$TARGET" | sort -rh | head -30

# Largest individual files (size-sorted, no per-file calls)
find "$TARGET" -type f -printf '%s\t%p\n' 2>/dev/null | sort -rn | head -30

# Files not modified in 6+ months (mtime is reliable; birth time often is not)
find "$TARGET" -type f -mtime +180 -printf '%s\t%TY-%Tm-%Td\t%p\n' 2>/dev/null \
  | sort -rn | head -30

# Common reclaimable patterns and their sizes
find "$TARGET" \( -name node_modules -o -name .venv -o -name dist -o -name build \
  -o -name target -o -name __pycache__ -o -name .cache -o -name .DS_Store \) \
  -prune -print 2>/dev/null | xargs -r du -sh 2>/dev/null | sort -rh | head -30
```

Note: use modification time (`-mtime`/`%T`) for "old". Creation/birth time is
unavailable on many Linux filesystems and should not be relied on.

## Step 3 — Decide whether to fan out
- **Default (single agent):** For typical directories, the Step 2 output is
  enough. Skip subagents and go to Step 4.
- **Fan out only when** the tree is very large (e.g. tens of GB / many heavy
  subtrees) and the heavy subdirectories from `du -d1` are roughly independent.
  Assign **one heavy subtree per subagent** so they don't rescan shared paths.

Subagent brief (when used):
> Analyze only `<subtree>`. Run the Step 2 commands scoped to it. Report back a
> short summary: total size, top space consumers, deletion candidates (with
> size, age, and category), and any items that look risky to delete (active
> project deps, recently used, ambiguous). Return aggregated findings only — no
> per-file dumps. **Do not delete anything.**

## Step 4 — Synthesize & recommend
Aggregate findings (from the scan and any subagents) into a report the user can
act on. Categorize candidates and quantify the win:

| Category | Path | Size | Last modified | Reclaimable | Confidence |
|----------|------|------|---------------|-------------|------------|
| Build artifact | … | … | … | … | high / med / low |

Order recommendations by reclaimable space × confidence. Call out anything
ambiguous explicitly rather than burying it. Show the running total of space
that would be freed.

## Step 5 — Human approval & cleanup
1. Present the report and ask which items to delete. Wait for **explicit**
   confirmation — do not infer approval.
2. Before deleting, re-confirm every target path is under `$TARGET`.
3. Prefer recoverable deletion when available (`trash`/`gio trash`); otherwise
   delete with exact, listed paths — never a broad glob or unscoped `rm -rf`.
4. Report what was deleted and the actual space reclaimed (`du -sh "$TARGET"`
   before vs. after).
