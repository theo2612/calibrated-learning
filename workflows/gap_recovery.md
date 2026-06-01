---
name: Gap Recovery
description: |-
  Handle 🔴 RED signal during walkthrough. Stop forward motion, trace the
  gap back to a foundational layer the learner does own, rebuild forward
  from there with a different angle, capture the gap as a study artifact,
  and return to the walkthrough.
license: MIT
compatibility: Claude Code
metadata:
  author: theo2612
  usage: workflow file for the calibrated-learning skill
  version: 0.3.0
  related files: "workflows/walkthrough.md, workflows/export_notes.md"
  creation date: 2026-05-11
  last modified: 2026-06-01
---

# Gap Recovery Workflow

## Context Budget Plan

| File | Load At | ~Tokens | Purpose |
|------|---------|---------|---------|
| This workflow | Step 0 (already loaded) | ~1,500 | Recovery instructions |
| `cauldron/<session-id>/baseline.md` | Step 1 | ~1,000 | Lookup which prereqs are 🟢 (safe foundations) |
| `cauldron/<session-id>/gap_map.md` | Step 5 (read+write) | ~500-2,000 | Accumulated gap entries |
| `cauldron/<session-id>/walkthrough_state.md` | Step 1 | ~800 | Where in the chain we are |

**Batch-Reading Guardrail:** Read baseline.md and walkthrough_state.md once at Step 1; don't reload.

---

## Resumption Procedure

### File Proof Table

| Checkpoint Status | Expected File Proof | Resume At |
|-------------------|---------------------|-----------|
| `gap_active` | `cauldron/<session-id>/walkthrough_state.md` (with `last_signal: red`) | Step 1 — Diagnose gap |
| `gap_traced` | `cauldron/<session-id>/gap_map.md` (with latest gap entry) | Step 3 — Begin rebuild |
| `gap_rebuild_in_progress` | `gap_map.md` (with rebuild approach noted) | Step 4 — Continue rebuild |
| `gap_resolved` | `gap_map.md` (with `status: resolved`) | Done — return to walkthrough |

### Resumption Steps

1. Read `checkpoint.json` and `walkthrough_state.md` from active session
2. Identify `gap_concept` (the concept that triggered 🔴)
3. Look up status in File Proof Table; resume at indicated step

---

## Trigger and Activation

This workflow activates when:
- Walkthrough workflow receives 🔴 signal and routes here (`status: gap_active`)
- Learner directly signals 🔴 during a re-explanation in walkthrough

---

## Purpose

A 🔴 signal means the current concept can't be patched — it needs a foundational rebuild. This workflow:

1. **Stops the walkthrough cleanly** so no momentum is wasted forcing the unclear concept
2. **Diagnoses where the gap actually is** — usually a concept further back than the one that triggered 🔴
3. **Rebuilds from a foundation the learner does own** using a different angle than the first attempt
4. **Captures the gap as an artifact** so the learner ends the session with a concrete study target
5. **Returns the learner to the walkthrough** with the gap resolved

Frame the entire workflow as productive calibration, not failure recovery.

---

## Workflow

### Step 1: Stop and Acknowledge

Acknowledge the signal explicitly, briefly, and without apology:

> ✅ Good signal — let me back up. Different angle.

**Do not:**
- ❌ Apologize ("sorry I lost you")
- ❌ Diminish ("let me dumb this down")
- ❌ Promise speed ("I'll get to the point faster")
- ❌ Add commentary on the concept's difficulty

The framing matters more than the explanation. The learner battled imposter syndrome to send the 🔴; reward the signal, don't lecture about it.

Update `checkpoint.json`: `"status": "gap_active"`, `"gap_concept": "<current concept>"`

### Step 2: Diagnose Where the Gap Actually Is

Ask one targeted question to locate the gap:

> What's the specific bit that's unclear?
> - The [mechanism A] piece?
> - How [mechanism A] connects to [prerequisite X]?
> - Something further back I should re-establish?

Listen for the answer. The gap is usually **not** the concept that triggered 🔴 — it's a prerequisite or sub-concept the current concept depends on.

#### Common Gap Patterns

| 🔴 trigger says | Real gap is usually |
|----------------|---------------------|
| "I don't get the syntax" | The mechanism the syntax implements |
| "I don't get why this works" | A prerequisite concept (often 🟢 at baseline but actually 🟡 in practice) |
| "I'm lost on what X means" | An undefined term — needs a name+definition pass (append the definition to `glossary.md`'s `## Added during the walkthrough` section once resolved) |
| "This is moving too fast" | Pace, not concept — slow down without re-explaining |
| "I don't see how this connects" | The previous concept's "Next" preview was too thin |

If the diagnosis identifies a pace gap (not a concept gap), skip Step 3-4 and slow the pace on the current concept instead.

#### Persist to File: Gap Diagnosis

Append to `cauldron/<session-id>/gap_map.md` (creating if needed):

```markdown
## Gap <N>: <concept name>
- **Triggered at:** <walkthrough concept that produced 🔴>
- **Triggered on:** <ISO timestamp>
- **Learner statement:** "<verbatim quote of what's unclear>"
- **Traced to:** <actual foundational gap>
- **Status:** diagnosing → rebuilding → resolved
```

Update `checkpoint.json`: `"status": "gap_traced"`

### Step 3: Find a Safe Foundation

Identify a concept the learner **already owns** that the gap depends on. Two sources:

1. **Baseline 🟢 prerequisites** — read `cauldron/<session-id>/baseline.md` and find a 🟢 prereq that connects to the gap
2. **Universal foundations** — concepts most technical learners own (variables, functions, files, processes, HTTP requests, etc.)

The foundation must be:
- **Genuinely owned** — don't assume; if uncertain, ask
- **Connected to the gap** — there must be a path from foundation → gap, otherwise the rebuild won't bridge anything
- **Concrete** — abstract foundations don't anchor; pick something the learner has done with their hands

State the foundation explicitly:
> Let's start from [foundation] — that's the piece this all builds on.

### Step 4: Rebuild Forward with a Different Angle

Walk from foundation → gap using a different approach than the first attempt:

| First attempt was | Try instead |
|-------------------|-------------|
| Abstract definition | Concrete example with named values |
| Technical mechanism | Real-world analogy (then back to technical) |
| Procedure (steps) | Mechanism (what's actually happening) |
| Whiteboard / diagram-style | Hands-on / "if you ran this command, you'd see..." |
| One long explanation | Many small explanations with verification between each |

Don't try to land the concept in one go. Use multiple short attempts with mini-checkpoints:

> Foundation: [X] does [Y]. ✓?
> Step up: [Y] enables [Z] because [mechanism]. ✓?
> Connect to gap: that's why [original concept] does [what it does]. ✓?

If the learner signals 🟢 at any mini-checkpoint, continue building. If they signal 🟡, slow down. If they signal 🔴 again, recurse into Step 2 (re-diagnose; the gap is deeper than first thought).

Update `checkpoint.json`: `"status": "gap_rebuild_in_progress"`

#### Persist to File: Rebuild Approach

Append to the current gap entry in `gap_map.md`:

```markdown
- **Foundation used:** <foundation concept>
- **Rebuild approach:** <which angle from the table>
- **Mini-checkpoints:** <count of green confirmations during rebuild>
```

### Step 5: Confirm Resolution

When the learner reaches 🟢 on the rebuilt concept, confirm:
> Solid? Want to keep going from where we paused, or do you want to consolidate this further?

#### Persist to File: Gap Resolution

Update the gap entry in `gap_map.md`:

```markdown
- **Status:** resolved
- **Resolved at:** <ISO timestamp>
- **Final signal:** 🟢
- **Notes:** <one-line takeaway for the learner's notes>
```

Update `checkpoint.json`: `"status": "gap_resolved"`, `"gaps_resolved_count": <incremented>`

### Step 6: Return to Walkthrough

Hand control back to `workflows/walkthrough.md` at the concept that triggered the 🔴. Brief the walkthrough workflow:

> Returning to [concept]. Gap on [foundation] resolved — referencing freely now.

The walkthrough workflow re-checkpoints on the original concept before advancing.

---

## Handling Unresolved Gaps

If after 2-3 rebuild attempts the gap remains, do not force resolution. Offer to:

1. **Park the gap** — mark in gap_map.md as `status: parked-for-future-session`, skip the concept in the current walkthrough (note that the chain may have reduced detail downstream), continue
2. **End the session** — mark gap as `status: needs-dedicated-session`, route to export_notes for partial-completion artifact
3. **Spawn a research subagent** — if the gap is research-shaped (definitions, history, official docs), suggest invoking the research skill to produce a study artifact, then resume in a future session

Let the learner choose. **Do not present these as failure modes** — they're calibration options.

---

## State Transition Chain (Pattern E)

**State transitions:**
`gap_active` → `gap_traced` → `gap_rebuild_in_progress` → (`gap_rebuild_in_progress` ⇄ self for recurse)* → `gap_resolved` → (return to walkthrough)

Alternative terminal: `gap_active` → `gap_traced` → `gap_parked` (handed to export_notes for partial-completion artifact)
