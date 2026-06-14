# save-space

Reusable agent skills for AI coding assistants — compatible with the [Agent Skills](https://agentskills.io) open standard and installable via the [`npx skills`](https://github.com/vercel-labs/skills) CLI.

Works with Claude Code, Cursor, Codex, OpenCode, and [70+ other agents](https://github.com/vercel-labs/skills#supported-agents).

---

## Skills

### `save-space`

One skill that covers everything: scans `$HOME` (or any directory), lets you
**multiselect** which subfolders to process, then applies the right cleanup
strategy for each one.

| Folder | Strategy |
|--------|----------|
| `~/Downloads` | Categorizes by type (installers, archives, PDFs, images…), flags stale and duplicate files, produces a per-category approve-or-skip plan |
| `~/Pictures` | Uses a Haiku vision model to classify every image as Keep / Safe to Delete / Needs Review based on rules you confirm upfront |
| `~/Documents` | Detects version-cluster proliferation (`report_v1`, `report_FINAL`…), surfaces large PDFs and stale files — without imposing any folder structure |
| Any other folder | General disk analysis: top space consumers, old files, build artifacts/caches — fan-out subagents for very large trees |

**Use when:** asked to free up disk space, clean up downloads/photos/documents,
organize files, or audit what is taking up storage.

**How it works:**
1. Confirms your target directory — recommends `$HOME`, but works anywhere
2. Lists subdirectories with sizes and lets you pick which ones to process
3. Applies the right handler for each selected folder in sequence
4. Waits for explicit per-category or per-item approval before touching anything
5. Presents a unified before/after summary with totals across all folders

**Privacy:** An immutable rule in the Pictures handler ensures that private or
intimate images are silently labelled Keep with no description logged, regardless
of any other rule changes.

---

## Installation

Install using [`npx skills`](https://github.com/vercel-labs/skills) — no global install required:

```bash
# Install for the current project
npx skills add Zircoz/save-space

# Install globally (available across all your projects)
npx skills add Zircoz/save-space -g

# Target a specific agent
npx skills add Zircoz/save-space -a claude-code
```

The CLI symlinks the skill into your agent's config directory (e.g. `.claude/skills/` for Claude Code). Use `--copy` if you prefer an independent copy over a symlink.

---

## Usage

Once installed, invoke the skill directly:

```
/save-space
```

Claude Code discovers skills in `.claude/skills/` (project-scoped) and
`~/.claude/skills/` (global). Other agents follow their own paths — the
`npx skills` CLI handles placement automatically.

---

## Skill format

Skills follow the [Agent Skills open standard](https://agentskills.io): a directory containing a `SKILL.md` file with YAML frontmatter.

```
save-space/
└── SKILL.md
```

```yaml
---
name: save-space
description: Free up disk space interactively. Lists subdirectories in $HOME (or any target directory)…
---

# Save Space

[instructions...]
```

**Required frontmatter fields:**

| Field | Rules |
|---|---|
| `name` | Lowercase, hyphens only, max 64 chars |
| `description` | What it does and when to use it, max 1024 chars |

The description is loaded into Claude's context at startup (~100 tokens). The full `SKILL.md` body is only loaded when the skill is triggered — so adding skills has minimal context cost until they're actually used.

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
