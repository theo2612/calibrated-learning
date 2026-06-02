---
name: Learning Ledger
description: |-
  Cross-session memory for the calibrated-learning skill. Persists what the
  learner OWNS (teach-back verified), is FAMILIAR with (self-reported 🟢), and
  has left OPEN (parked gaps) across sessions. Read by baseline_check to skip
  re-calibrating settled ground; written at walkthrough-complete so each
  session compounds instead of starting from zero.
license: MIT
compatibility: Claude Code
metadata:
  author: theo2612
  usage: workflow file for the calibrated-learning skill
  version: 0.5.0
  related files: "workflows/baseline_check.md, workflows/walkthrough.md, workflows/export_notes.md"
  creation date: 2026-06-02
  last modified: 2026-06-02
---

# Learning Ledger Workflow

## Purpose

Without a ledger, every session's cauldron is sealed — the skill is reborn
ignorant each time and re-asks the learner to calibrate ground they've already
earned. The ledger is the skill's long-term memory: it records what the learner
has demonstrated, so `baseline_check` can pre-fill or skip settled concepts and
surface gaps that were parked.

**The integrity rule:** only **teach-back-verified** concepts (from the v0.4.0
Teach-Back Gate) earn OWNED trust. A hopeful, self-assessed 🟢 lands in FAMILIAR
and is still re-asked. The teach-back gate is the ledger's quality gate — this is
why it was built first.

This workflow is not invoked on its own. It defines the ledger format and two
procedures consumed by other workflows:
- **READ** — called by `baseline_check.md` Step 1 to classify prerequisites.
- **WRITE** — called by `walkthrough.md` Step 5 (walkthrough-complete) to record outcomes.

---

## The Ledger File

Path (single file, shared across all sessions):
```
~/.claude/skills/calibrated-learning/ledger.md
```

**Privacy:** the ledger is personal learning history — git-ignored, never published, hand-editable. The learner can downgrade or delete any entry directly.

**Format** (create on first write if absent):

```markdown
# Calibrated Learning Ledger

> Cross-session memory. OWNED = teach-back verified. FAMILIAR = self-reported 🟢 (unverified). OPEN = parked gaps.
> Hand-editable — downgrade or delete anything that's faded.
> staleness_weeks: 8   ← OWNED entries older than this are flagged for re-confirmation (tunable)

## OWNED
| Concept | Topic context | Last confirmed |
|---------|--------------|----------------|
| <concept> | <topic it was learned in> | <YYYY-MM-DD> |

## FAMILIAR
| Concept | Topic context | Last seen |
|---------|--------------|-----------|
| <concept> | <topic> | <YYYY-MM-DD> |

## OPEN
| Gap | Topic context | Parked | Note |
|-----|--------------|--------|------|
| <gap> | <topic> | <YYYY-MM-DD> | <why parked / what's needed> |
```

The `staleness_weeks` header value is the decay threshold (default 8). Read it from the file; if absent, default to 8.

---

## READ Procedure (consumed by baseline_check Step 1)

Called **before** prerequisites are presented to the learner. Input: the topic and its draft prerequisite list. Output: a per-prerequisite classification that shapes how Step 2 presents each one.

1. If `ledger.md` does not exist → return "no ledger yet"; baseline proceeds normally (all prereqs are fresh R/Y/G questions).
2. Read `ledger.md`. Parse OWNED, FAMILIAR, OPEN and the `staleness_weeks` value.
3. Compute the staleness cutoff: `today − staleness_weeks`. (Use the session's current date.)
4. For each draft prerequisite, match it against ledger entries (by concept name / close synonym — judge meaning, not exact string) and classify:

| Match | Classification | How Step 2 presents it |
|-------|---------------|------------------------|
| In OWNED, `Last confirmed` ≥ cutoff | **owned-fresh** | Pre-fill 🟢. Confirm in one glance: "you've got this — still good?" |
| In OWNED, `Last confirmed` < cutoff | **owned-stale** | Pre-fill but flag: "⏳ you owned this ~N months ago — still solid, or re-walk it?" |
| In FAMILIAR (any age) | **familiar** | Pre-fill 🟢 but still ask: "you've seen this — still tracking?" (never was verified) |
| No match | **new** | Normal R/Y/G question |

5. Scan OPEN gaps for relatedness to the new topic. For any related gap, queue a surfacing line for Step 2: *"Last time we parked **<gap>** — want to fold it into this walkthrough?"*
6. Return the classification map + any related OPEN gaps to `baseline_check`.

**The learner can always override.** Pre-fill is a default, not a decision — a one-word downgrade ("actually 🟡 on that") wins. The ledger informs; the learner drives.

---

## WRITE Procedure (consumed by walkthrough.md Step 5)

Called at **walkthrough-complete** (fires whether or not the learner exports notes, so the ledger captures every session). Inputs: `baseline.md` (prereq signals), `walkthrough_state.md` (teach-back outcomes, concepts completed), `gap_map.md` (gap statuses).

1. If `ledger.md` does not exist → create it with the format above (default `staleness_weeks: 8`).
2. For each concept/prerequisite resolved this session, decide its destination:

| Session outcome | Ledger destination |
|-----------------|-------------------|
| Teach-back **confirmed** (or partial→confirmed, or misconception→corrected→confirmed) | **OWNED**, `Last confirmed` = today |
| Plain 🟢 (walkthrough concept or baseline prereq, no teach-back) | **FAMILIAR**, `Last seen` = today |
| Gap parked / `needs-dedicated-session` | **OPEN**, `Parked` = today |
| An OPEN gap from a prior session that was **closed** this session | Remove from OPEN; add to OWNED (if teach-backed) or FAMILIAR |

3. **Dedup and upgrade** (never create duplicate rows):
   - Concept already in the ledger → update its date in place.
   - Concept in FAMILIAR that earned a teach-back this session → **move it to OWNED** (upgrade wins; a verified concept is never demoted to FAMILIAR).
   - A concept is never listed in two sections at once. OWNED > FAMILIAR > OPEN in precedence.
4. Write the updated `ledger.md`.
5. Report to the learner in the completion summary: *"Ledger updated: +N owned, +M familiar, K gaps still open."*

---

## Resilience

The ledger is the one piece of state that must survive a corrupted/abandoned session. The WRITE only appends/updates verified outcomes at completion, so a session that dies mid-walkthrough leaves the ledger untouched (no half-learned concepts wrongly marked OWNED). If `ledger.md` is malformed on READ, treat as "no ledger" and proceed normally rather than blocking the session — never let a bad ledger stop a walkthrough.

---

## Why teach-back gates OWNED (design note)

A self-assessed 🟢 is exactly the signal the Teach-Back Gate exists to distrust. If plain 🟢s earned OWNED, the ledger would accumulate unverified "mastery" and then *skip re-asking it* — compounding a false 🟢 across every future session that depends on it. Restricting OWNED to teach-back-verified concepts keeps the ledger's trust honest; FAMILIAR is the holding tier for "seen it, not proven it."
