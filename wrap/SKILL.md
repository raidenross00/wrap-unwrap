---
name: wrap
description: Gracefully close out a session. Classifies session items by evidence, triages ambiguous ones with the user, then writes SESH.md (portable), a local checkpoint (recall via /unwrap), and enters plan mode with clear-context option for instant same-session continuation.
---

# /wrap

One command, three handoff paths. Type /wrap when you're done, context is filling up, or you need to hand off.

## Process

### Step 1: Ensure showClearContextOnPlanAccept

```bash
SETTINGS="$HOME/.claude/settings.json"
if [ -f "$SETTINGS" ]; then
  if grep -q '"showClearContextOnPlanAccept"' "$SETTINGS" 2>/dev/null; then
    echo "SETTING_EXISTS"
  else
    echo "NEEDS_SETTING"
  fi
else
  echo "NO_SETTINGS_FILE"
fi
```

- If `NEEDS_SETTING`: Read the file, add `"showClearContextOnPlanAccept": true` to the top-level object, write it back. Tell the user: "Enabled clear-context-on-plan-accept in your Claude settings."
- If `NO_SETTINGS_FILE`: Create `~/.claude/settings.json` with `{"showClearContextOnPlanAccept": true}`. Tell the user the same.
- If `SETTING_EXISTS`: Do nothing. Move on.
- If the file exists but cannot be parsed as JSON: warn "Could not update settings.json, skipping config step" and continue. This is non-blocking.

### Step 2: Gather git state

```bash
echo "=== PROJECT ==="
pwd
echo "=== BRANCH ==="
git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "NO_GIT"
echo "=== STATUS ==="
git status --short 2>/dev/null || echo "NO_GIT"
echo "=== RECENT LOG ==="
git log --oneline -10 2>/dev/null || echo "NO_GIT"
echo "=== SESSION DIFF ==="
git diff --name-only HEAD~10..HEAD 2>/dev/null || echo "NO_GIT"
```

If not a git repo, skip "Files Changed" in the outputs. Everything else still works.

### Step 3: Classify session items

Scan the full conversation and git state. Extract every item that was worked on, discussed, or mentioned. Classify each into one of three buckets:

#### CONFIRMED (auto-include, no review needed)

An item is CONFIRMED if it has **any** of these:
- **Git evidence**: code was committed this session, visible in diff
- **Explicit intent**: user used imperative language ("do X", "build X", "fix X", "write X") and never contradicted it
- **Repeated intent**: discussed 3+ times with consistent direction
- **Active work in progress**: partially built, files modified but not yet committed

CONFIRMED items go straight into the checkpoint. The user is told what was auto-included but doesn't need to approve each one.

#### REVERTED (auto-exclude, no review needed)

An item is REVERTED if **any** of these:
- User explicitly reversed it: "undo", "revert", "put it back", "nah", "actually no", "don't do that", "that's shit", or similar rollback language AFTER the item was discussed or worked on
- Git shows a revert of session work (file returned to pre-session state)
- Task was marked done then user requested undo

REVERTED items are excluded from the checkpoint entirely. The user is told what was auto-excluded.

#### AMBIGUOUS (review gate)

Everything that doesn't clearly fit CONFIRMED or REVERTED:
- Mentioned once with no follow-up
- Speculative language: "maybe", "could", "what about", "we should consider", "might be worth"
- Discussed but no clear go-ahead or rejection
- Ideas explored in conversation but no code written and no explicit "do this"

AMBIGUOUS items go through the review gate in Step 3.5.

### Step 3.5: Review gate (only if ambiguous items exist)

**If zero ambiguous items**: Skip the gate entirely. Show the user a summary of what was auto-included and auto-excluded, then proceed to Step 4.

```
Wrapping up. Here's what I captured:

CARRYING FORWARD:
  ✓ [item] (shipped, commit abc1234)
  ✓ [item] (you asked for this)

DROPPED:
  ✗ [item] (you reverted this)
  ✗ [item] (you said "nah" to this)

Nothing ambiguous — saving checkpoint.
```

**If 1+ ambiguous items exist**: Present each ambiguous item as a tab in AskUserQuestion. Each item gets 4 options representing its destination tier.

**Structure**: Each ambiguous item = 1 tab (question). Max 4 items per AskUserQuestion call. If more than 4 ambiguous items, chain multiple calls.

```
Tab header: short slug for the item (max 12 chars)
Question: full description of the item + context for why it's ambiguous

Options (always these 4, in this order):
  1. "Do first" / "Top priority for next session" — goes to Resume Here
  2. "Keep it" / "Real work, not urgent" — goes to Remaining Work
  3. "Park it" / "Maybe later, low priority" — goes to Parking Lot
  4. "Drop it" / "Don't save this anywhere" — killed entirely
```

**The recommended option MUST be listed first** — the UI auto-selects item 1, so the recommended choice must always be in position 1 for good UX. Put "(Recommended)" in that option's label. Reorder the 4 options so the recommended one is first, followed by the remaining three in their standard order. If the item was speculative with no follow-up, recommend "Park it" or "Drop it". If discussed with some intent, recommend "Keep it".

**After receiving responses**: Apply the user's selection for each item. Combined with the auto-classified items, this produces the final tiered list.

### Step 3.6: Build the final item list

Merge auto-classified and user-triaged items:

- **Resume Here**: CONFIRMED items with active work + any items user marked "Do first"
- **Remaining Work**: CONFIRMED items that shipped (context, not action) + items user marked "Keep it"
- **Parking Lot**: Items user marked "Park it"
- **Dropped**: REVERTED items + items user marked "Drop it" (not written anywhere)

### Step 3.7: SESH.md decision

Ask whether this session should produce a SESH.md in the repo. The checkpoint always gets written regardless — this decision only affects the team-facing repo artifact.

**Recommendation logic**: Check the git state from Step 2 and the conversation context.
- Code was committed or modified this session → recommend "Write SESH.md"
- Only design docs, reviews, brainstorming, conversation → recommend "Skip SESH.md"
- Mixed session (some code + some discussion) → recommend "Write SESH.md"
- Not a git repo → recommend "Skip SESH.md" (no repo to share)

Include the recommendation reason in the option description so the user understands WHY.

If there were ambiguous items in Step 3.5, add this as an extra tab in the LAST AskUserQuestion call from that step (if there's room, i.e., fewer than 4 ambiguous items in that batch). Otherwise, make a separate AskUserQuestion call.

If there were NO ambiguous items (gate was skipped), make a standalone AskUserQuestion call.

Use exactly 2 options (Write or Skip). **The recommended option MUST be listed first** — the UI auto-selects item 1, so the recommended choice must always be in position 1 for good UX. Put "(Recommended)" in the label of the first option. Tailor the description to explain WHY for this specific session.

```
Example when recommending Skip:
  1. "Skip SESH.md (Recommended)" / "Design/architecture session, no code changes — nothing for the repo to pick up"
  2. "Write SESH.md" / "Save it if you want team visibility on the discussion anyway"

Example when recommending Write:
  1. "Write SESH.md (Recommended)" / "Active code changes — teammates or LLMs opening this repo should know what's in flight"
  2. "Skip SESH.md" / "You can skip if this work isn't ready for team visibility yet"
```

If the user picks "Skip SESH.md": skip Step 5, BUT check if a SESH.md already exists in the project root:

```bash
[ -f SESH.md ] && echo "SESH_EXISTS" || echo "NO_SESH"
```

If `SESH_EXISTS`: Ask whether to delete the stale file. A stale SESH.md is worse than none — it misleads anyone who opens the repo.

```
Tab header: "Stale SESH"
Question: "There's an existing SESH.md in this repo. Since you're skipping the update, it's now outdated and could mislead teammates or LLMs. Delete it?"

Options (recommended first):
  1. "Delete it (Recommended)" / "Remove stale SESH.md so it doesn't mislead — checkpoint still has your state"
  2. "Leave it" / "Keep the old SESH.md as-is, even though it's outdated"
```

If the user picks "Delete it": remove the file (`rm SESH.md`). If "Leave it": do nothing.

### Step 4: Generate the briefing

Using the final item list, git state, and conversation context, produce the briefing content.

**Framing rules:**
- Forward-looking and positive. No "what failed" or "what went wrong" sections.
- If an approach was tried and abandoned, frame it as direction: "Use approach B because X" not "Approach A failed because Y." This avoids negative priming in the next session.
- Be specific. File paths, function names, line numbers where relevant.
- LLM-optimized structure. This is for Claude to read, not a human README.

**Briefing sections:**
1. **Working On** — the high-level goal (1-2 sentences)
2. **Current State** — what's done, what's in progress
3. **Direction** — key decisions, framed as forward instructions ("use X because Y")
4. **Resume Here** — 1-3 items, top priority, grounded in evidence
5. **Remaining Work** — items confirmed as real but not first priority
6. **Parking Lot** — items explicitly parked by user, low priority, next session should NOT act on these unless user promotes them
7. **Files Changed** — from git status (skip if non-git)

### Step 5: Write SESH.md

Write SESH.md to the current working directory (project root). This is the portable, slim format.

```markdown
## Status: CONTINUING

**Timestamp:** [ISO-8601]
**Branch:** [branch or "n/a"]

### Working On
[high-level goal]

### Current State
[what's done, what's in progress]

### Direction
[key decisions as forward instructions]

### Resume Here
1. [concrete action — top priority]

### Remaining Work
- [item]

### Parking Lot
- [item — do not act on unless user promotes]

### Files Changed
- path/to/file
```

### Step 6: Write checkpoint

The checkpoint is the complete record with frontmatter and a Notes section.

Generate the filename: `{YYYYMMDD-HHMMSS}-{title-slug}.md` where title-slug is the briefing title in lowercase, spaces to hyphens, non-alphanumeric characters stripped, max 40 chars.

```bash
mkdir -p ~/.wrap/checkpoints
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
echo "CHECKPOINT_DIR=$HOME/.wrap/checkpoints"
echo "TIMESTAMP=$TIMESTAMP"
```

Write the checkpoint file to `~/.wrap/checkpoints/{TIMESTAMP}-{title-slug}.md`:

```markdown
---
status: in-progress
branch: [branch or "n/a"]
timestamp: [ISO-8601]
project: [absolute path to project root]
files_modified:
  - path/to/file
---

## Working on: [title]

### Summary
[1-3 sentences covering goal and progress]

### Direction
[decisions framed as forward instructions]

### Resume Here
1. [top priority item]

### Remaining Work
- [confirmed real work, not urgent]

### Parking Lot
- [parked by user — do not act on unless explicitly promoted]

### Notes
[gotchas, open questions, context the next session needs — things not captured above]
```

### Step 7: Enter plan mode

Call the `EnterPlanMode` tool.

Write the briefing as the plan document. Use this format:

```markdown
# Session Continuation: [title]

Wrapped at [timestamp] on branch [branch].

## Working On
[from briefing]

## Current State
[from briefing]

## Direction
[from briefing]

## Resume Here
[from briefing — these are the top priority items for the next session]

## Remaining Work
[from briefing — real work, not urgent]

## Parking Lot
[from briefing — parked items, do NOT act on these unless the user explicitly says to]

## Files Changed
[from briefing]

## Notes
[from checkpoint notes section]
```

Then call `ExitPlanMode` to present the plan to the user.

The user will see:
- The plan content
- An "Accept" button
- A "Clear context" checkbox (enabled by showClearContextOnPlanAccept)

**That's it.** The user chooses:
- Accept + clear context → instant continuation in the same window with fresh context
- Accept without clearing → keeps current session going
- Close the session → SESH.md and checkpoint persist for later / for teammates / for /unwrap
