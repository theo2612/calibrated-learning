---
name: Full Coaching Session Playbook
description: |-
  End-to-end orchestration of a calibrated-learning session. Sequences
  baseline_check → walkthrough → (gap_recovery as needed) → export_notes.
  Handles state transitions, session boundaries, and partial-completion
  fallbacks.
license: MIT
compatibility: Claude Code
metadata:
  author: theo2612
  usage: playbook file for the calibrated-learning skill
  version: 0.3.0
  related files: "workflows/baseline_check.md, workflows/walkthrough.md, workflows/gap_recovery.md, workflows/export_notes.md"
  creation date: 2026-05-11
  last modified: 2026-06-01
---

# Full Coaching Session Playbook

## Trigger and Activation

This playbook activates when:
- Learner explicitly requests `/calibrated-learning <topic>`
- Learner asks for a "full coaching session" or "complete walkthrough"
- Router determines a topic merits the full end-to-end flow vs. just a quick re-explanation

The playbook is **opt-in** — by default, the router activates individual workflows. The playbook is used when the learner wants the whole structured experience start to finish.

---

## Purpose

Sequence all four workflows in the correct order with proper state hand-offs. Handle:

- Initial session setup
- Baseline → walkthrough transition
- Walkthrough ↔ gap_recovery cycling
- Walkthrough → export_notes transition
- Session boundary recommendations for long topics

---

## Workflows Used

1. `workflows/baseline_check.md` — Phase 1
2. `workflows/walkthrough.md` — Phase 2 (with gap_recovery cycling)
3. `workflows/gap_recovery.md` — Phase 2.5 (invoked from walkthrough on 🔴)
4. `workflows/export_notes.md` — Phase 3

---

## Resumption Procedure

### File Proof Table

| Playbook Status | Expected File Proof | Resume At |
|-----------------|---------------------|-----------|
| `session_initialized` | `cauldron/<session-id>/topic.md` | Phase 1 — Baseline check |
| `baseline_phase` | `cauldron/<session-id>/baseline.md` | Phase 2 — Walkthrough |
| `walkthrough_phase` | `cauldron/<session-id>/walkthrough_state.md` | Phase 2 (resume at concept_N from state file) |
| `gap_recovery_active` | `cauldron/<session-id>/gap_map.md` | Phase 2.5 (resume gap recovery for active gap) |
| `walkthrough_complete` | `walkthrough_state.md` (with `complete: true`) | Phase 3 — Export notes |
| `notes_exported` | Final notes file at recorded path | Done |

### Resumption Steps

1. Read `checkpoint.json` from active session
2. Look up `status` in File Proof Table
3. **For sub-workflow resumption, check the workflow's own File Proof Table** (each leaf workflow has detailed sub-state)
4. Resume at indicated phase / sub-step

---

## Playbook Phases

### Phase 1: Baseline Check

**Workflow:** `workflows/baseline_check.md`

**Entry condition:** `status: session_initialized`
**Exit condition:** `status: baseline_complete`

**Hand-offs:**
- Input: topic.md (the learner's request)
- Output: baseline.md (prerequisite signals and walkthrough plan) + glossary.md (pre-flight acronyms, notation, vocabulary)

**Stop conditions:**
- All prerequisites signaled 🔴 → suggest the topic is too advanced for the current baseline; offer to either (a) tackle one prereq as its own coaching session, or (b) proceed and accept extensive foundational rebuilds
- Learner declines the baseline → skip to Phase 2 with no prerequisite plan (assistant will infer depth from the walkthrough itself; expect more 🟡/🔴)

### Phase 2: Walkthrough

**Workflow:** `workflows/walkthrough.md`

**Entry condition:** `status: baseline_complete`
**Exit condition:** `status: walkthrough_complete`

**Hand-offs:**
- Input: baseline.md (plan and 🔴 prerequisites needing pre-explanation)
- Output: walkthrough_state.md (chain walked, re-explanations used, gaps encountered)

**Mid-phase routing:** Walkthrough ↔ gap_recovery cycling
- On 🔴 → walkthrough hands off to Phase 2.5
- After gap_resolved → walkthrough resumes at the same concept

**Session Boundary (Pattern F):** If the walkthrough has consumed significant context (e.g., 6+ concepts, multiple 🔴 events), offer a natural session boundary before continuing:

> We're 6 concepts in with [N] gaps captured. Want to save what we have and pick up in a fresh session? Or push through to the end?

Mandatory boundary at ~50% context consumption. Persistence is in cauldron state; resumption from a fresh session is supported via Pattern B.

### Phase 2.5: Gap Recovery (cyclic, invoked from Phase 2)

**Workflow:** `workflows/gap_recovery.md`

**Entry condition:** `status: gap_active` (set by walkthrough)
**Exit condition:** `status: gap_resolved` (returns control to Phase 2) OR `status: gap_parked` (routes to Phase 3 for partial-completion notes)

**Hand-offs:**
- Input: current concept name, learner's unclear-bit statement
- Output: gap_map.md entry (resolved or parked)

### Phase 3: Export Notes

**Workflow:** `workflows/export_notes.md`

**Entry condition:** `status: walkthrough_complete` OR `status: gap_parked` (partial completion path)
**Exit condition:** `status: notes_exported`

**Hand-offs:**
- Input: all cauldron state (topic, baseline, walkthrough_state, gap_map)
- Output: notes file at `~/learning-notes/<date>_<topic-slug>.md` (or learner-specified path)

**Stop conditions:**
- Learner declines export → mark session complete without notes; cauldron state remains for later manual export

---

## Quality Gates

### Gate 1: After Phase 1

Check before transitioning to Phase 2:
- [ ] baseline.md exists and contains at least 3 prerequisites
- [ ] Every prerequisite has a signal (🟢/🟡/🔴)
- [ ] Walkthrough plan is ordered
- [ ] glossary.md exists (populated, or the honest empty-scan stub)

If gate fails → loop back to Phase 1 (likely the learner's signal response was ambiguous and needs clarification).

### Gate 2: After Phase 2

Check before transitioning to Phase 3:
- [ ] walkthrough_state.md exists with `complete: true` OR `walkthrough_complete` checkpoint reached
- [ ] At least one concept walked (no zero-concept exports)
- [ ] gap_map.md is consistent (every gap has a status)

If gate fails → either resume walkthrough (state shows incomplete) or skip to Phase 3 in partial-completion mode.

### Gate 3: After Phase 3

Check before declaring `complete`:
- [ ] Notes file exists at the recorded path
- [ ] File is non-trivial (> 500 bytes)
- [ ] Learner has been notified of save location

---

## Output

**Primary artifact:** Notes file at `~/learning-notes/<YYYY-MM-DD>_<topic-slug>.md`

**Secondary artifacts (in cauldron):**
- Session state for resumption
- Gap map for later study
- Glossary (pre-flight + walkthrough-added terms) — folded into the exported notes
- Raw signal log for retrospective analysis (e.g., "which re-explanation approaches worked?")

**Optional follow-ups (offered by Phase 3):**
- Spawn research skill for parked gaps
- Queue future calibrated-learning sessions on parked gaps
- Clean cauldron via `clean-cauldrons` skill

---

## State Transition Chain (Pattern E)

**Playbook state transitions:**
`session_initialized` → `baseline_phase` → `baseline_complete` → `walkthrough_phase` → (`gap_recovery_active` ⇄ `walkthrough_phase`)* → `walkthrough_complete` → `notes_exported` → `complete`

Each phase's workflow has its own internal state transition chain (see the respective workflow file's State Transition Chain section).
