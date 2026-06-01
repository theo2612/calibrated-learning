---
name: Export Notes
description: |-
  Terminal workflow. Composes the session's cauldron state (topic, baseline,
  walkthrough chain, R/Y/G transitions, gap map) into a single markdown
  notes file the learner can keep for follow-up study.
license: MIT
compatibility: Claude Code
metadata:
  author: theo2612
  usage: workflow file for the calibrated-learning skill
  version: 0.3.0
  related files: "workflows/walkthrough.md, workflows/gap_recovery.md, templates/coaching_notes_template.md"
  creation date: 2026-05-11
  last modified: 2026-06-01
---

# Export Notes Workflow

## Context Budget Plan

| File | Load At | ~Tokens | Purpose |
|------|---------|---------|---------|
| This workflow | Step 0 (already loaded) | ~1,000 | Export instructions |
| `cauldron/<session-id>/topic.md` | Step 1 | ~300 | Topic and learner context |
| `cauldron/<session-id>/baseline.md` | Step 1 | ~1,000 | Prerequisite signals and plan |
| `cauldron/<session-id>/walkthrough_state.md` | Step 1 | ~800 | Chain of concepts walked |
| `cauldron/<session-id>/gap_map.md` | Step 1 | ~500-2,000 | Gap entries |
| `cauldron/<session-id>/glossary.md` | Step 1 | ~600 | Pre-flight + walkthrough-added terms |
| `templates/coaching_notes_template.md` | Step 2 | ~600 | Output format |

**Batch-Reading Guardrail:** Read all five cauldron files at Step 1 (combined ~4-5K tokens). Do not re-read after composition.

---

## Resumption Procedure

### File Proof Table

| Checkpoint Status | Expected File Proof | Resume At |
|-------------------|---------------------|-----------|
| `walkthrough_complete` | `cauldron/<session-id>/walkthrough_state.md` (with `complete: true`) | Step 1 — Load state |
| `notes_composed` | `cauldron/<session-id>/notes_draft.md` | Step 3 — Write final notes file |
| `notes_exported` | Notes file at final location | Done |

### Resumption Steps

1. Read `checkpoint.json` from active session
2. Look up status in File Proof Table
3. Resume at indicated step
4. If `notes_draft.md` exists but final notes file doesn't, just write the final file

---

## Trigger and Activation

This workflow activates when:
- Walkthrough workflow completes (`status: walkthrough_complete`)
- Learner says "save notes", "export this", "wrap up", "give me the notes"
- Session is being ended early and gap recovery routed here for partial artifact

---

## Purpose

Turn ephemeral session state into a durable study artifact. The notes file should be:

- **Self-contained** — readable later without needing the original session context
- **Honest about gaps** — gap map is a feature, not a hidden failure log
- **Action-oriented** — ends with concrete follow-up suggestions
- **Friendly to vault tools** — markdown that drops into Obsidian, Notion, etc. without massaging

---

## Workflow

### Step 1: Load All Session State

Read in one batch:
- `cauldron/<session-id>/topic.md`
- `cauldron/<session-id>/baseline.md`
- `cauldron/<session-id>/walkthrough_state.md`
- `cauldron/<session-id>/gap_map.md` (if exists)
- `cauldron/<session-id>/glossary.md` (if exists)

Extract:
- Topic name and original learner context
- Final list of concepts walked (from `concepts_complete`)
- Re-explanations used (from `re_explanations_used`)
- All gap entries with status
- The full glossary (pre-flight terms + terms added during the walkthrough) — this becomes the notes' standalone Glossary section, a study reference the learner keeps
- Session start and end timestamps

### Step 2: Compose Notes Draft

Read the template at `templates/coaching_notes_template.md`. Populate it with extracted data:

#### Persist to File: Notes Draft

> **RESILIENCE (Pattern A):** Compose to draft first, then move to final location.
> This way if the destination write fails (permissions, disk full), the draft survives.

Write to:
```
~/.claude/skills/calibrated-learning/cauldron/<session-id>/notes_draft.md
```

Contents (~2-4K tokens):
- See template structure (templates/coaching_notes_template.md)
- All gaps annotated with status and resolution approach
- Concept chain with R/Y/G transition annotations
- Follow-up study recommendations derived from parked/unresolved gaps

Update `checkpoint.json`: `"status": "notes_composed"`

### Step 3: Determine Final Destination

Default destination:
```
~/learning-notes/<YYYY-MM-DD>_<topic-slug>.md
```

If `~/learning-notes/` does not exist, create it:
```bash
mkdir -p ~/learning-notes
```

If the learner specified a different location (e.g., "save to my Obsidian vault"), use that path. Confirm before writing.

### Step 4: Write Final Notes File

Copy `cauldron/<session-id>/notes_draft.md` to the final destination.

Verify the file was written:
```bash
test -f <final-path> && wc -l <final-path>
```

Update `checkpoint.json`: `"status": "notes_exported"`, `"notes_path": "<final-path>"`

### Step 5: Report to Learner

Present a short summary:

```
✅ Notes saved → <final-path>

Session summary:
- Topic: <topic>
- Concepts walked: <count>
- Re-explanations: <count>
- Gaps captured: <count> (<resolved> resolved, <parked> parked for future study)

Suggested follow-up:
- <one-line recommendation per parked gap>
```

If gaps were parked, offer to:
- Spawn a research subagent on the parked gap topic now
- Queue the gap as a future calibrated-learning session
- Just close out and revisit later

Let the learner choose. Do not push.

---

## State Transition Chain (Pattern E)

**State transitions:**
`walkthrough_complete` → `notes_composed` → `notes_exported` → `complete`

The `complete` status is terminal for the session. The cauldron directory can be cleaned up at the learner's discretion (or via the `clean-cauldrons` skill).
