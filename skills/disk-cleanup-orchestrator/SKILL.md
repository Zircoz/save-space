---
name: disk-cleanup-orchestrator
description: Orchestrates subagents to analyze a directory recursively, gather file metadata, and provide disk cleanup recommendations for a human.
---

# Disk Cleanup Orchestrator

This skill allows you to act as an orchestrator to manage multiple subagents. Together, you will recursively analyze a specified folder, gather detailed file metadata, and present disk cleanup recommendations to a human-in-the-loop.

## When to Use
Use this skill when the user asks for help cleaning up their disk space, finding large or old files, or analyzing a directory to free up storage.

## Steps

### Phase 1: Planning & Subagent Dispatch
1. **Identify Target:** If not already provided, ask the human user which directory they want to analyze for disk cleanup.
2. **Initial Assessment:** Do a quick initial scan using standard shell tools (e.g., `find`, `du`) to understand the scope and scale of the directory.
3. **Task Partitioning:** Divide the work. You can assign different subdirectories or chunks of files to different subagents.
4. **Dispatch Subagents:** Launch the subagents in parallel using your agent dispatching capabilities. Provide them with their specific target paths and the instructions from "Phase 2".

### Phase 2: Subagent Execution (Instructions for Subagents)
When dispatching subagents, provide them with the following instructions:
1. Recursively iterate through your assigned folder or list of files.
2. For each file, collect the following metadata:
   - **File path**
   - **File size**
   - **Mimetype** (e.g., using `file --mime-type <file>` or `mimetype <file>`)
   - **Creation date**
   - **Last modified date**
3. Analyze the files to identify potential candidates for deletion. Look for:
   - Extremely large files (e.g., >100MB).
   - Old files untouched for a long time (e.g., >6 months).
   - Temporary files, logs, or cache directories.
   - Known build artifacts (e.g., `node_modules`, `dist/`, `.DS_Store`).
4. **Report Back:** Summarize your findings, highlighting the largest space consumers and best candidates for deletion. Pass this summarized information and metadata back to the orchestrator agent.

### Phase 3: Orchestrator Synthesis & Recommendation
1. **Aggregate:** As the orchestrator, wait for all subagents to complete their analysis and report back.
2. **Synthesize:** Aggregate the summaries and metadata provided by the subagents.
3. **Categorize:** Organize the findings logically (e.g., by file type, by age, by size, or by specific heavy folders).
4. **Draft Report:** Create a comprehensive and easy-to-read summary report of the directory's disk usage.
5. **Formulate Recommendations:** Generate a prioritized list of recommendations for the human user regarding which files or directories to delete to optimally reclaim space. Clearly show the potential space saved for each recommendation.
6. **Human-in-the-loop Approval:** Present the summary and recommendations to the human user. Ask for their explicit approval on which files to delete. **Do NOT delete any files without explicit confirmation from the human.**

### Phase 4: Cleanup (Optional)
1. If the human user approves specific deletion recommendations, execute the deletions safely.
2. Provide a final brief report detailing what was deleted and the total disk space successfully reclaimed.
