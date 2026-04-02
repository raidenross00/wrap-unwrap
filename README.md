# /wrap + /unwrap

Session handoff skills for Claude Code. One command to save your session state, one command to resume.

## What it does

**`/wrap`** — Gracefully close a session. Classifies everything from the conversation into confirmed work, reverted work, and ambiguous items. Lets you triage the ambiguous ones. Writes a tiered checkpoint and optionally a SESH.md for team visibility. Enters plan mode for instant same-session continuation.

**`/unwrap`** — Cold-start resume. Finds the latest checkpoint for your project, presents it with tiered priorities, checks for drift, and gets you back to work.

## Three handoff paths

| Path | Artifact | Where | Who it's for |
|------|----------|-------|-------------|
| Same-session | Plan mode + clear context | In-memory | You, continuing now |
| Cold start | `~/.wrap/checkpoints/` | Local filesystem | You, next session |
| Team / clone | `SESH.md` in repo | Git | Anyone opening the project |

## The checkpoint pollution problem

Session continuation tools typically summarize the conversation into "next steps." The problem: speculative remarks, reverted work, and offhand ideas get promoted to priorities that misdirect the next session.

/wrap solves this with evidence-based classification:

- **CONFIRMED** (auto-include) — git evidence, explicit user intent, repeated discussion, active WIP
- **REVERTED** (auto-exclude) — user said "undo", "revert", "nah", or git shows revert
- **AMBIGUOUS** (review gate) — mentioned once, speculative language, no clear go-ahead

Only ambiguous items need user input. Most sessions have zero friction.

## Tiered checkpoints

```
Resume Here      — top priority, evidence-backed
Remaining Work   — real work, not urgent
Parking Lot      — explicitly deprioritized, next session won't act on these
```

/unwrap only foregrounds Resume Here items. Parking Lot is a footnote.

## Install

Copy the skill folders into your Claude Code skills directory:

```bash
# Clone
git clone https://github.com/raidenross00/wrap-unwrap.git

# Copy to your skills directory
cp -r wrap-unwrap/wrap ~/.claude/skills/wrap
cp -r wrap-unwrap/unwrap ~/.claude/skills/unwrap
```

Then use `/wrap` and `/unwrap` in Claude Code.

## First use

On first `/wrap`, the skill automatically sets `showClearContextOnPlanAccept: true` in `~/.claude/settings.json`. This enables the "clear context" checkbox when accepting a plan, which is what makes instant same-session continuation work. This only happens once.

## Requirements

- Claude Code with plan mode support
- No external dependencies
