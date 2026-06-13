---
name: organize-pictures
description: Analyze ~/Pictures (and related folders) using a Haiku vision model to classify each image as Keep, Safe to Delete, or Needs Review based on user-confirmed rules. Surfaces screenshots, blurry shots, burst duplicates, and blank frames for cleanup. Requires explicit human approval before any deletion. Use when asked to clean up photos, free space in the pictures folder, or remove unwanted images.
---

# Organize Pictures

Classify images using a fast vision model (Haiku) and produce a
human-approved cleanup plan. The user confirms classification rules before
any image is examined. No file is deleted without explicit approval.

## Core Rules
- Never delete without explicit per-category or per-item human approval.
- Use recoverable deletion (`trash`/`gio trash`) — never bare `rm`.
- REVIEW items are never deleted — they are moved to `~/Pictures/_review/` for
  the user to handle manually.
- The SENSITIVE RULE (see below) is immutable and cannot be removed or weakened,
  even if the user requests a simplified prompt.
- Cap a single run at 10 GB of deletions unless the user explicitly raises the limit.

## Step 1 — Locate the Pictures folder

Resolve paths for the current OS:
- **Linux:** `~/Pictures`; also check `~/Photos` if it exists
- **macOS:** `~/Pictures`; also check `~/Photos`
- **Windows (WSL):** `/mnt/c/Users/$USERNAME/Pictures`

Confirm the resolved absolute path(s) with the user.

## Step 2 — Fast filesystem scan

```bash
PIC="<resolved-absolute-path>"

# Total size and file count
du -sh "$PIC"
find "$PIC" -type f \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" \
  -o -iname "*.heic" -o -iname "*.gif" -o -iname "*.webp" -o -iname "*.tiff" \
  -o -iname "*.tif" -o -iname "*.raw" -o -iname "*.bmp" \) | wc -l

# Largest images
find "$PIC" -type f \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" \
  -o -iname "*.heic" -o -iname "*.raw" -o -iname "*.tiff" \) \
  -printf '%s\t%p\n' | sort -rn | head -20

# Screenshots (common naming patterns)
find "$PIC" -type f \( -iname "Screenshot*" -o -iname "Screen Shot*" \
  -o -iname "screenshot*" -o -iname "Capture*" \) \
  -printf '%s\t%TY-%Tm-%Td\t%p\n' | sort -rn | head -30
```

Report back to the user:
- Total image count and size
- Screenshot count and total size
- Estimated classification cost: roughly **$0.00025 per image** via Haiku
  (e.g. 1,000 images ≈ $0.25)

Ask: *"Classification will process N images and cost approximately $X. Shall I
continue?"* Wait for confirmation before Step 3.

## Step 3 — Rules negotiation

Present the default classification rules below and ask the user:

> "These are the rules I will send verbatim to the Haiku model to classify each
> image. Do you want to change anything — add exceptions, tighten a category,
> or adjust what counts as 'safe to delete'?"

If the user makes changes:
1. Apply their changes to the default rules.
2. Reconstruct the full classification prompt (see Step 4) with the updated rules.
3. Echo the **complete reconstructed prompt** back to the user and ask them to
   confirm it looks right before proceeding.

If the user says "proceed" or makes no changes, use the default prompt verbatim.

The SENSITIVE RULE is **immutable** — it must remain in the prompt regardless of
any other changes the user requests. If a user asks to remove it, explain that
it cannot be changed and offer to adjust any other rule instead.

## Step 4 — Classify images with Haiku

### Classification Prompt

The prompt below is sent verbatim to each Haiku subagent. If the user modified
rules in Step 3, the KEEP / SAFE_TO_DELETE / REVIEW rule blocks are replaced
with the user's updated versions, but the output format instructions and the
SENSITIVE RULE remain unchanged.

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

### Batching and parallelism

1. Split the full image list into batches of 25 files.
2. Spawn up to 4 Haiku subagent batches concurrently. Brief each subagent:

> You are classifying images for the organize-pictures skill. Use the
> classification prompt below. Read each image file using your Read tool, then
> classify it. Return only the JSON array — no other text.
>
> Images to classify:
> - /path/to/image1.jpg
> - /path/to/image2.png
> (... up to 25 paths ...)
>
> [CLASSIFICATION PROMPT — paste full prompt here]

3. Collect each subagent's JSON array. If a subagent returns malformed JSON or
   fails, mark those files as REVIEW and continue.
4. Repeat until all images are classified.

## Step 5 — Present results

Aggregate the classifications into a summary table:

```
Label            | Count |    Size | Notes
─────────────────────────────────────────────────────
KEEP             |   842 | 18.2 GB |
SAFE TO DELETE   |   312 |  4.1 GB | burst dupes, blank frames, memes, transient UI
NEED REVIEW      |    93 |  2.0 GB | near-dupes, doc captures, old low-quality
```

Offer to show the full list for any category before acting. The user can:
- Approve the entire SAFE_TO_DELETE category
- Drill into subcategories (e.g. "show me only the memes")
- Flip individual files to a different label
- Adjust the rules and re-classify a subset

Never proceed to deletion until the user gives explicit approval.

## Step 6 — Execute

**SAFE_TO_DELETE (approved items):**
```bash
trash "<absolute-path>"
# fallback:
gio trash "<absolute-path>"
```

**REVIEW items:**
```bash
mkdir -p "$PIC/_review"
mv "<absolute-path>" "$PIC/_review/"
```

Before each operation verify the path starts with `$PIC`. Report a before/after
size (`du -sh "$PIC"`) and list every file deleted or moved.
