---
name: unwrap
description: Resume from a /wrap checkpoint. Finds the most recent checkpoint for this project, presents the briefing with tiered priorities, checks for drift, and gets you back to work.
---

# /unwrap

Cold-start resume. Finds the latest /wrap checkpoint for this project and loads it.

## Process

### Step 1: Find checkpoints

Use Glob to find checkpoint files (no bash needed):

```
Glob: ~/.wrap/checkpoints/*.md
```

Also check if SESH.md exists in the current directory:

```
Glob: ./SESH.md
```

If no checkpoint files found and no SESH.md: tell the user "No checkpoints or SESH.md found. Run /wrap first to save session state." Stop here.

If checkpoints exist: Read the frontmatter of each file (just the first few lines between `---` markers) and filter for checkpoints where the `project:` field matches the current working directory. Sort by filename (they're timestamped, so alphabetical = chronological). Take the most recent match.

If no checkpoints match this project but SESH.md exists: fall back to reading SESH.md. Tell the user "No local checkpoints for this project. Loading SESH.md instead." Present the SESH.md content and skip to Step 4.

### Step 2: Load and present the checkpoint

Read the most recent matching checkpoint file. Present it with tiered priorities:

```
RESUMING SESSION
════════════════════════════════════════
Title:    [from ## Working on: heading]
Branch:   [from frontmatter]
Saved:    [timestamp, formatted as relative time e.g. "2 hours ago"]
Project:  [from frontmatter]
════════════════════════════════════════

[Summary section]

[Direction section]

RESUME HERE
  1. [top priority items — these are what the user confirmed as next up]

ALSO IN FLIGHT
  - [remaining work items — real but not first priority]

PARKING LOT
  - [parked items — acknowledged, low priority, do NOT act on these]
```

**Priority rules for the new session:**
- **Resume Here** items are the work. Present the first one as "First up: [item]."
- **Remaining Work** items are context. Mention them but don't start them unless the user asks.
- **Parking Lot** items are explicitly deprioritized. Do NOT suggest working on them. Do NOT create tasks for them. Do NOT act on them unless the user specifically promotes one. They exist only so the user remembers they were discussed.

If the checkpoint uses the old format (single "Next Steps" section without tiers), treat all items as Resume Here. Old checkpoints are fully compatible.

### Step 2.5: SESH.md staleness check

If SESH.md exists in the current directory, compare its timestamp against the checkpoint being loaded.

Read the first few lines of SESH.md to extract its `**Timestamp:**` field. Compare against the checkpoint's `timestamp:` frontmatter field.

- If SESH.md timestamp is OLDER than the checkpoint: the file is stale. Flag it:
  "SESH.md is stale (from [SESH date], checkpoint is from [checkpoint date]). It may mislead teammates or LLMs opening this repo."
- If SESH.md timestamp matches or is newer: no issue, say nothing.
- If SESH.md has no parseable timestamp: flag it as "SESH.md has no timestamp, may be outdated."

This is informational only at /unwrap time — don't delete anything here. The user can deal with it when they /wrap at the end of this session.

### Step 3: Check for drift

```bash
git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "NO_GIT"
```

If git is available:
- Compare current branch to the checkpoint's `branch:` field. If different, note: "You're on `{current}` but the checkpoint was saved on `{saved}`. You may want to switch."
- Check for commits since the checkpoint timestamp:
  ```bash
  git log --oneline --since="[checkpoint timestamp]" 2>/dev/null | head -10
  ```
  If commits found, note: "Repo has changed since this checkpoint. N commits since wrap."

If not a git repo: note "No git history available, skipping drift detection."

### Step 4: Ready to work

Tell the user what the first Resume Here item is and ask if they want to start there. No formal AskUserQuestion needed here, just a natural "First up: [item]. Ready to go?"

If the checkpoint only has Remaining Work and Parking Lot (no Resume Here items), say: "No top-priority items from last session. Here's what's in flight: [list remaining]. What do you want to pick up?"
