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
/disk-cleanup-orchestrator
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
