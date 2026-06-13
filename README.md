# save-space

Reusable agent skills for AI coding assistants — compatible with the [Agent Skills](https://agentskills.io) open standard and installable via the [`npx skills`](https://github.com/vercel-labs/skills) CLI.

Works with Claude Code, Cursor, Codex, OpenCode, and [70+ other agents](https://github.com/vercel-labs/skills#supported-agents).

---

## Skills

### `disk-cleanup-orchestrator`

Orchestrates subagents to recursively analyze a directory, collect file metadata, and surface disk cleanup recommendations — with human-in-the-loop approval before any deletion.

**Use when:** the user asks to free up disk space, find large or old files, or audit a directory for storage cleanup.

**How it works:**
1. Scans the target directory to understand scope
2. Partitions the work and dispatches parallel subagents across subdirectories
3. Each subagent collects path, size, mimetype, creation date, and last-modified date per file — and flags candidates (large files, stale files, temp/cache/build artifacts) without deleting anything
4. The orchestrator aggregates all findings into a prioritized report
5. Presents recommendations to you and waits for explicit approval before touching anything

---

### `organize-downloads`

Categorizes files in `~/Downloads` by type (installers, archives, PDFs, images, documents, etc.), flags stale and duplicate files, and produces a per-category cleanup and move plan.

**Use when:** the user asks to clean up or organize their Downloads folder.

**How it works:**
1. Resolves the Downloads path for the current OS (honours `$XDG_DOWNLOAD_DIR` on Linux)
2. Scans for largest files, oldest files, and duplicates (name + size match)
3. Groups files into categories with a recommended action for each
4. Presents a summary table and waits for per-category approval
5. Deletes via `trash`/`gio trash`; moves files to appropriate destination folders

---

### `organize-pictures`

Classifies images in `~/Pictures` using a Haiku vision model — each image is labeled **Keep**, **Safe to Delete**, or **Needs Review** based on rules the user confirms before any image is examined.

**Use when:** the user asks to clean up photos, remove unwanted images, or free space in the pictures folder.

**How it works:**
1. Fast filesystem scan reports total image count, size, and estimated classification cost (~$0.00025/image via Haiku)
2. Presents default classification rules (what counts as keep vs. safe to delete vs. review); user can modify any rule before proceeding
3. Classification prompt is rebuilt verbatim from agreed rules and echoed back to the user for confirmation
4. Images are classified in parallel batches of 25 by Haiku subagents
5. Results are aggregated into a summary table; user approves before any deletion
6. Approved deletions go to trash; REVIEW items move to `~/Pictures/_review/` — never auto-deleted

**Privacy:** a hard-coded, immutable rule ensures that private or intimate images are silently labeled Keep with no description logged, regardless of any other rule changes.

---

### `organize-documents`

Audits `~/Documents` for version-cluster proliferation (`report_v1`, `report_FINAL`, `report_FINAL2`…), large PDFs, stale files, and empty folders — without imposing any folder structure.

**Use when:** the user asks to clean up documents, find versioned or duplicate files, or audit the documents folder.

**How it works:**
1. Scans for largest files, oldest files, and empty directories
2. Detects version clusters by stripping common suffixes (`_v2`, `_FINAL`, `_copy`, etc.) and grouping files that share a base name — any base with 3+ versions is surfaced as a cluster
3. Presents findings in four sections: version clusters, large files, stale files, empty folders
4. User decides per-cluster and per-file what to keep; nothing is batch-deleted without review
5. Cleans up via `trash`/`gio trash`

---

### `clean-home`

Master orchestrator that runs `organize-downloads`, `organize-pictures`, and `organize-documents` in sequence and produces a unified space-reclaim summary across all three folders.

**Use when:** the user wants a full personal-folder cleanup in one go, or wants an overview of what is taking space across Downloads, Pictures, and Documents.

**How it works:**
1. Confirms which folders to include (all three by default) and resolves OS-specific paths
2. Shows a quick size overview before any scanning begins
3. Runs each sub-skill in order: Downloads → Pictures → Documents; waits for the user to approve or skip each folder before moving on
4. Presents a unified before/after summary with totals across all folders

---

## Installation

Install using [`npx skills`](https://github.com/vercel-labs/skills) — no global install required:

```bash
# All skills in this repo
npx skills add Zircoz/save-space

# A specific skill
npx skills add Zircoz/save-space --skill disk-cleanup-orchestrator

# Install globally (available across all your projects)
npx skills add Zircoz/save-space -g

# Target a specific agent
npx skills add Zircoz/save-space -a claude-code
```

The CLI symlinks skills into your agent's config directory (e.g. `.claude/skills/` for Claude Code). Use `--copy` if you prefer independent copies over symlinks.

---

## Using the skills

Once installed, skills are picked up automatically when relevant. You can also invoke any skill directly by name:

```
/disk-cleanup-orchestrator   # audit any directory for disk usage
/organize-downloads          # clean up ~/Downloads
/organize-pictures           # classify and clean up ~/Pictures
/organize-documents          # audit ~/Documents for version clutter
/clean-home                  # run all three personal-folder skills in sequence
```

Claude Code discovers skills in `.claude/skills/` (project-scoped) and `~/.claude/skills/` (global). Other agents follow their own paths — the `npx skills` CLI handles placement automatically.

---

## Skill format

Skills follow the [Agent Skills open standard](https://agentskills.io): a directory containing a `SKILL.md` file with YAML frontmatter.

```
disk-cleanup-orchestrator/
└── SKILL.md
```

```yaml
---
name: disk-cleanup-orchestrator
description: Orchestrates subagents to analyze a directory recursively, gather file metadata, and provide disk cleanup recommendations for a human.
---

# Disk Cleanup Orchestrator

[instructions...]
```

**Required frontmatter fields:**

| Field | Rules |
|---|---|
| `name` | Lowercase, hyphens only, max 64 chars |
| `description` | What it does and when to use it, max 1024 chars |

The description is loaded into Claude's context at startup (~100 tokens). The full SKILL.md body is only loaded when the skill is triggered — so adding many skills has minimal context cost until they're actually used.

---

## Contributing a skill

1. Create a directory under `skills/` named after your skill
2. Add a `SKILL.md` with frontmatter and clear step-by-step instructions
3. Optionally add a `scripts/` or `resources/` directory for supporting files
4. Open a PR — the `npx skills add Zircoz/save-space` command will pick it up automatically

See the [Agent Skills best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) guide for authoring tips.

---

## References

- [Agent Skills open standard](https://agentskills.io)
- [`npx skills` CLI](https://github.com/vercel-labs/skills) — package manager for agent skills
- [Claude Code skills docs](https://code.claude.com/docs/en/skills)
- [Anthropic skills repository](https://github.com/anthropics/skills) — official example skills
- [Agent Skills Directory](https://www.skills.sh) — community skill discovery
