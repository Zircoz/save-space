# Save space skills

This repository contains reusable skills for AI agents compatible with the [skills.sh](https://skills.sh) framework.

## Available Skills

### `disk-cleanup-orchestrator`

Orchestrates subagents to analyze a directory recursively, gather file metadata, and provide disk cleanup recommendations for a human.

## Installation

You can install these skills for your coding agent using the `skills` CLI. No installation is required; simply use `npx`:

```bash
# Install all skills in this repository
npx skills add Zircoz/save-space

# Install a specific skill
npx skills add Zircoz/save-space --skill disk-cleanup-orchestrator
```

## Using with Agents.sh

The skills in this repository are particularly well-suited for [Agents.sh](https://agents.sh), a framework designed to connect specialized agents to coordinate and get work done together.

### `disk-cleanup-orchestrator` on Agents.sh

The `disk-cleanup-orchestrator` is a perfect fit for Agents.sh's multi-agent workflow capabilities.

1.  **Define the Orchestrator:** Connect your Agents.sh instance to this skill. The orchestrator agent will handle the planning and synthesis phases.
2.  **Coordinate Subagents:** The orchestrator will leverage Agents.sh's coordination features to spin up subagents. Each subagent will be assigned a chunk of the filesystem to analyze.
3.  **Human-in-the-Loop:** Agents.sh's conversational interface is ideal for the final step. The orchestrator will present the synthesized findings to you (the user) and ask for explicit approval before executing any deletions, keeping you fully in control.
