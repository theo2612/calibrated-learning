---
name: Router
description: |-
  Routes user requests to the appropriate workflow for the calibrated-learning
  skill. Detects session state (new vs continuing), parses R/Y/G signals,
  and dispatches to baseline_check, walkthrough, gap_recovery, or export_notes.
license: MIT
compatibility: Claude Code
metadata:
  author: theo2612
  usage: workflow file for the calibrated-learning skill
  version: 0.1.0
  related files: "workflows/baseline_check.md, workflows/walkthrough.md, workflows/gap_recovery.md, workflows/export_notes.md"
  creation date: 2026-05-08
  last modified: 2026-05-08
---

# Router Workflow

## Context Budget Plan

| File | Load At | ~Tokens | Purpose |
|------|---------|---------|---------|
| This workflow | Step 0 (already loaded) | ~1,200 | Routing instructions |
| `cauldron/<session-id>/checkpoint.json` | Step 1 | ~300 | Session state (if continuing session) |
| Target sub-workflow | Step 3 | ~1,500-3,000 | Loaded only after route is determined |

**Batch-Reading Guardrail:** Only ever load ONE sub-workflow at a time after routing. Never pre-load all sub-workflows — defeats the just-in-time loading pattern.

---

## Resumption Procedure

> **PURPOSE:** Single entry point for resuming a coaching session after session death.
> Read `checkpoint.json` from the active session directory, find the current status
> in the File Proof Table below, verify the expected file artifact(s) exist, and
> resume at the indicated step.

### File Proof Table

| Checkpoint Status | Expected File Proof | Resume At |
|-------------------|---------------------|-----------|
| `initialized` | `cauldron/<session-id>/checkpoint.json` | Step 2 — Determine route |
| `baseline_complete` | `cauldron/<session-id>/baseline.md` | Step 3 — Route to walkthrough |
| `walkthrough_in_progress` | `cauldron/<session-id>/walkthrough_state.md` | Step 3 — Resume walkthrough |
| `gap_active` | `cauldron/<session-id>/gap_map.md` | Step 3 — Route to gap_recovery |
| `notes_exported` | `cauldron/<session-id>/notes.md` | Done |

### Resumption Steps

1. List session directories in `cauldron/` to find active session
2. Read most recent `checkpoint.json`
3. Look up `status` in File Proof Table above
4. **Verify proof files exist** — run `ls` on the expected file(s)
5. If proof files are present → resume at the indicated step
6. If proof files are MISSING → regress to the previous status with valid proof

---

## Trigger and Activation

This workflow activates when:
- The user invokes any quick activation pattern from SKILL.md (e.g., "walk me through", "coach me", "deep dive on")
- The user signals R/Y/G mid-session (🟢/🟡/🔴 or "green"/"yellow"/"red")
- The user asks to save, export, or wrap up a session

---

## Purpose

The router is the single entry point for all calibrated-learning interactions. It distinguishes between four scenarios and dispatches accordingly:

1. **New session** — start with baseline_check
2. **Continuing session** — read checkpoint and resume the active workflow
3. **R/Y/G signal mid-walkthrough** — route to walkthrough (for 🟢/🟡) or gap_recovery (for 🔴)
4. **Wrap-up request** — route to export_notes

---

## Workflow

### Step 1: Detect Session State

Check whether an active coaching session exists:

```bash
ls ~/.claude/skills/calibrated-learning/cauldron/ 2>/dev/null
```

- If no session directories exist → this is a NEW session. Skip to Step 2.
- If a session directory exists with a `checkpoint.json` whose `status` is not `notes_exported` → this is a CONTINUING session. Read the checkpoint and skip to Step 3.

### Step 2: Initialize New Session

Create a session directory and checkpoint:

```bash
SESSION_ID="$(date +%Y-%m-%d_%H%M%S)_<topic-slug>"
mkdir -p ~/.claude/skills/calibrated-learning/cauldron/$SESSION_ID
```

Write initial `checkpoint.json`:

```json
{
  "session_id": "<SESSION_ID>",
  "topic": "<user-supplied topic>",
  "status": "initialized",
  "started_at": "<ISO timestamp>",
  "transitions": []
}
```

Write extracted topic and learner-supplied context (if any) to `cauldron/<SESSION_ID>/topic.md`.

### Step 3: Determine Route

Match the user input against routing rules:

| Input Pattern | Route To | Rationale |
|---------------|----------|-----------|
| New topic request, status `initialized` | `workflows/baseline_check.md` | Always start with baseline assessment |
| Status `baseline_complete` | `workflows/walkthrough.md` | Baseline done, begin guided walkthrough |
| Mid-session 🟢 GREEN signal | `workflows/walkthrough.md` (continue) | Maintain depth and pace |
| Mid-session 🟡 YELLOW signal | `workflows/walkthrough.md` (re-explain branch) | Pause forward motion, re-explain current concept |
| Mid-session 🔴 RED signal | `workflows/gap_recovery.md` | Stop, rebuild from foundation, capture gap |
| "save notes", "export", "wrap up" | `workflows/export_notes.md` | End-of-session artifact creation |

### Step 4: Hand Off to Target Workflow

Load the target workflow file and execute its Step 1. Pass session state via:
- `session_id`
- Path to `checkpoint.json`
- The full user input (so the target workflow can re-parse signals if needed)

Update `checkpoint.json` to reflect routing decision in the `transitions` array:

```json
{
  "from_status": "<previous>",
  "to_status": "<new>",
  "routed_to": "<workflow_filename>",
  "timestamp": "<ISO>",
  "trigger": "<user input snippet>"
}
```

---

## State Transition Chain (Pattern E)

**State transitions:**
`initialized` → `baseline_complete` → `walkthrough_in_progress` → (`gap_active` ⇄ `walkthrough_in_progress`)* → `notes_exported` → `complete`

The router never sets `complete` itself — that is the export_notes terminal state.
