---
name: Walkthrough
description: |-
  Main coaching loop. Executes Command → Output → Analysis → Next for each
  concept in the walkthrough plan, with R/Y/G checkpoints between concepts.
  Handles 🟢 (continue at depth) and 🟡 (pause and re-explain) signals
  in-loop; routes 🔴 signals to gap_recovery.
license: MIT
compatibility: Claude Code
metadata:
  author: theo2612
  usage: workflow file for the calibrated-learning skill
  version: 0.5.0
  related files: "workflows/baseline_check.md, workflows/gap_recovery.md, workflows/export_notes.md, workflows/ledger.md"
  creation date: 2026-05-11
  last modified: 2026-06-02
---

# Walkthrough Workflow

## Context Budget Plan

| File | Load At | ~Tokens | Purpose |
|------|---------|---------|---------|
| This workflow | Step 0 (already loaded) | ~2,000 | Loop instructions |
| `cauldron/<session-id>/baseline.md` | Step 1 | ~1,000 | Walkthrough plan from baseline phase |
| `cauldron/<session-id>/glossary.md` | Step 1 (read), Step 3e (append) | ~600 | Pre-flight jargon; living term source |
| `cauldron/<session-id>/checkpoint.json` | Step 1 | ~300 | Session state |
| `cauldron/<session-id>/walkthrough_state.md` | Step 1 (read), Step 4 (write) | ~800 | Current loop state |
| `cauldron/<session-id>/gap_map.md` | Step 4 (write on 🟡) | ~500 | Gap entries |

**Batch-Reading Guardrail:** Read baseline.md and glossary.md exactly once at Step 1; reference in-context for the rest of the workflow.

---

## Resumption Procedure

### File Proof Table

| Checkpoint Status | Expected File Proof | Resume At |
|-------------------|---------------------|-----------|
| `baseline_complete` | `cauldron/<session-id>/baseline.md` | Step 1 — Load plan |
| `concept_N_introduced` | `cauldron/<session-id>/walkthrough_state.md` (with `current_concept: N`) | Step 3 — Await learner signal |
| `concept_N_yellow_reexplained` | `walkthrough_state.md` (with `last_signal: yellow`) | Step 3 — Await fresh signal after re-explanation |
| `walkthrough_in_progress` (last concept) | `walkthrough_state.md` (with last concept reached) | Step 5 — Offer wrap |
| `walkthrough_complete` | `walkthrough_state.md` with `complete: true` | Done — route to export_notes |

### Resumption Steps

1. Read `checkpoint.json` from active session directory
2. Read `walkthrough_state.md` to recover `current_concept` and `last_signal`
3. Look up status in File Proof Table; resume at indicated step
4. If `last_signal: yellow` and re-explanation was incomplete → restart concept's re-explanation
5. If `last_signal: red` → route to `gap_recovery.md` instead

---

## Trigger and Activation

This workflow activates when:
- Baseline check completes with `status: baseline_complete`
- Learner returns from gap_recovery with `status: gap_resolved`
- Mid-session 🟢 or 🟡 signal received (router-dispatched)

---

## Purpose

Deliver the structured walkthrough one concept at a time with explicit R/Y/G checkpoints between concepts. The loop:

```
For each concept in walkthrough plan:
  1. Command — show the command/concept being introduced
  2. Output — show what the learner would see
  3. Analysis — explain mechanism (why), then procedure (how)
  4. Next — preview the next concept and how it connects
  5. Checkpoint — pause for R/Y/G signal
  6. Branch on signal:
     🟢 → continue to next concept
     🟡 → re-explain current concept with fresh angle
     🔴 → route to gap_recovery
```

---

## Workflow

### Step 1: Load Walkthrough Plan

Read `cauldron/<session-id>/baseline.md` and `cauldron/<session-id>/glossary.md`. Extract:
- Ordered list of concepts to walk through
- 🔴 prerequisites that need pre-explanation
- 🟡 prerequisites that need framing on first use
- The pre-flight glossary (acronyms, notation, vocab) to present in Step 1.5

If `walkthrough_state.md` does not exist, initialize it:

```markdown
# Walkthrough State

current_concept: 0
total_concepts: <N>
last_signal: none
concepts_complete: []
gaps_during_walkthrough: []
re_explanations_used: []
```

### Step 1.5: Present the Glossary Pre-Flight

**Run this before the 🔴 foundational rebuilds in Step 2** — the rebuilds themselves use the topic's jargon, so the glossary has to land first or the rebuild reintroduces the exact silent-jargon problem the pre-flight exists to prevent.

If `glossary.md` is the empty-scan stub, skip presentation silently and proceed to Step 2.

Otherwise, present the glossary as a **compact skim-and-flag reference**, not a calibration gate:

> Before we start, here's the jargon this topic leans on — acronyms, notation, and a few loaded terms. Skim it. The ⚑ marks ones tied to a prerequisite you flagged 🟡/🔴, so those are the likeliest to want a closer look. Flag anything still fuzzy (🟡/🔴) and I'll expand it; otherwise just say 🟢 and we'll dive in.

Then paste the three sections (Acronyms & Abbreviations; Notation, Syntax & Conventions; Technical Vocabulary) directly from `glossary.md`.

**Handling the response:**
- 🟢 / "looks good" → proceed to Step 2.
- 🟡/🔴 on specific terms → expand those terms with a concrete example (treat exactly like a bundled vocab question), confirm, then proceed. Do **not** route a glossary 🔴 to gap_recovery — a fuzzy term at pre-flight is a definition request, not a foundational collapse. Only escalate to gap_recovery if expanding the term reveals a missing prerequisite mechanism.

**Why present rather than quiz:** Unlike the baseline prerequisite check (where premature explanation defeats calibration), the glossary's whole job is to *pre-define* so jargon never appears cold. Presenting definitions here is the intended move, not a leak.

### Step 2: Pre-Walkthrough Foundations (🔴 prereqs only)

For each prerequisite signaled 🔴 in the baseline:

1. State the foundational concept being explained: *"Before we touch [prereq], here's the foundation it builds on..."*
2. Explain from first principles with a concrete example
3. Tie to the topic with a one-line preview: *"You'll see this come back in [concept N] when we get there."*
4. Get a fresh R/Y/G signal on the foundational explanation
5. If still 🔴 → route to `gap_recovery.md` for that prerequisite
6. If 🟢 or 🟡 → proceed

Update `walkthrough_state.md`: mark prereq as "foundation explained".

### Step 3: The Loop — Concept N

For each concept in the walkthrough plan (in order):

#### 3a. Command

Introduce the concept by showing its concrete form:
- For a command: the actual command line with every flag and parameter annotated
- For a code construct: the syntax with each piece labeled
- For an abstract concept: a minimal worked example

**No black boxes.** Every flag, every parameter, every variable gets a one-line explanation.

#### 3b. Output

Show what the learner would see:
- Real command output if you have it (or a representative example clearly labeled as such)
- For code: the result of executing it
- For abstract concepts: a concrete instance ("here's what this looks like in practice")

#### 3c. Analysis

Explain in this order:
1. **Mechanism (why)** — what is actually happening under the hood. Lead with this.
2. **Procedure (how)** — the steps the learner would take to do this themselves
3. **Connection** — how this concept ties to prerequisites and to the chain so far

**Lead with mechanism, not procedure.** Procedure without mechanism is memorization.

#### 3d. Next

Preview the next concept:
- Name it
- One-line statement of how it connects to the current concept

#### 3e. Vocab Footer (conditional)

Before the checkpoint, assess whether a Vocab Footer is warranted:

**Include when:**
- The concept introduced 3+ new technical terms
- The concept was in 🔴 territory per the baseline
- Any term used was not in the learner's baseline vocabulary

**Format:**
```
**Terms used in this concept**
- *[term]* — [one-sentence definition anchored to how it was just used, not a dictionary definition]
- *[term]* — [definition]
```

**Skip when:** The concept was synthesis-only or introduced no new terminology.

The vocab footer eliminates scroll-back friction — the learner can read term definitions right above the checkpoint without hunting upward through the concept body.

**Append to the living glossary:** Any footer term not already in `glossary.md` gets appended to its `## Added during the walkthrough` section (same one-sentence, topic-anchored format). This keeps `glossary.md` the single source of every defined term across the session, so a term defined in Concept 2's footer is still findable at Concept 7 — and export_notes picks it up automatically. A term already in the pre-flight glossary is **not** re-appended (no duplicates).

#### 3f. Checkpoint

Pause and ask for an R/Y/G signal. **Do not chain into the next concept without a signal.**

**Default checkpoint** (most concepts):

> R/Y/G on [concept]?

**Teach-Back Gate** — at concepts tagged `load-bearing` in `baseline.md` (and after every 🔴 rebuild, handled in `gap_recovery.md` Step 5): a bare 🟢 is self-assessed and unverified, which is the most expensive calibration error at exactly these concepts. So the checkpoint upgrades to signal **plus** a one-line teach-back:

> R/Y/G on [concept]? — and so I can check **my explanation** landed: in one line, [explain it back to me / what do you think the next [command/step] does]?

- Pick **explain-back** for conceptual material; **predict-the-next-step** for procedural/command material.
- The gate is **on by default but always skippable** (see Teach-Back Evaluation below for the skip path).
- Only load-bearing + post-🔴 fire the gate. A teach-back on every concept is friction the learner explicitly does not want.

Wait for the response, then evaluate per the next section.

### Teach-Back Evaluation

> **WHY THIS GATE EXISTS:** A false 🟢 looks identical to a real one. Self-assessed comprehension pattern-matches "yeah, that tracks" and advances — and the gap compounds invisibly. Asking the learner to *produce* the explanation (Feynman) is the cheapest way to tell a real 🟢 from a hopeful one. Evidence it works: this skill's own SMB-enumeration session caught two edges *because* the learner volunteered teach-backs ("if only 445 is open, may not be a path…" surfaced a correctable misconception). The gate makes that prompt explicit where it matters most.

**Framing rule (non-negotiable):** the burden is on **the explanation, never the learner.** The prompt asks the learner to check *the assistant's* explanation, not to prove themselves. A gap is phrased "my explanation under-specified X" — never "you got it wrong." This learner battles imposter syndrome; a teach-back framed as a test poisons the whole mechanic. Reward the attempt, always.

**Judge the mechanism/anchor, not the phrasing.** Rough, in-the-learner's-own-words explanations that capture the load-bearing point are 🟢. Do not nitpick vocabulary.

Three outcomes:

| Outcome | Trigger | Response |
|---------|---------|----------|
| **Confirmed** | Teach-back captures the load-bearing mechanism, even roughly | Acknowledge the signal, advance. No vocabulary nitpicking. Log `teach_back: confirmed`. |
| **Partial** | Captures most but misses one key piece | Fill **only** the missing piece (not a full re-explanation), re-confirm. Treat as a soft 🟡. Log `teach_back: partial`. |
| **Misconception** | Reveals a **wrong anchor** or broken mechanism (e.g. "445-only means no path") | The high-value catch — worth more than a 🟡. Correct gently ("my explanation let that slip — here's the fix…"), log to **gap_map.md** as a corrected misconception, re-confirm. Log `teach_back: misconception_corrected`. |

**Skip path:** a bare color at a fire point (`🟢`, `🟢 skip`, "green, skip the teach-back") is honored without nagging — advance, but record `teach_back: skipped` in state so that if a later gap traces back to this concept, we know verification was skipped here.

#### Persist to File: Teach-Back Log

Append to `cauldron/<session-id>/walkthrough_state.md` (in a `teach_backs` section):

```markdown
- Concept: <name>
  - Type: explain-back | predict
  - Outcome: confirmed | partial | misconception_corrected | skipped
  - Misconception (if any): "<learner's wrong anchor>" → "<correction>"
```

A `misconception_corrected` outcome also appends a gap-map entry (status `resolved`, tagged `corrected-misconception`) — these are the highest-value study artifacts the session produces.

### Handling Bundled Signals

When the learner bundles vocabulary questions into their signal (e.g., "🟡 on Rhino, and what does safe-mode scope mean?"), process in this order:

1. **Honor the color signal first** — adjust depth/pace per 🟢/🟡/🔴 semantics
2. **Deliver a compact vocab block** — one short paragraph per bundled term, anchored to how the term appeared in the concept just delivered
3. **Append to the living glossary** — add each bundled term to `glossary.md`'s `## Added during the walkthrough` section (if not already present). The glossary is the durable home for defined terms.
4. **Log to gap map** — treat each bundled unknown term as a vocabulary gap entry: `term → definition → context → resolved`. (Glossary = the definition reference; gap map = the "this was a gap" record. Both, because they serve different readers: the glossary is study-reference, the gap map is the calibration audit trail.)
5. **Do not reorder** — never skip the color response to jump straight to vocabulary definitions

### Step 4: Branch on Signal

#### 🟢 GREEN — Continue

Acknowledge briefly ("alright, moving on"). Update state:
- Mark concept N as complete in `walkthrough_state.md`
- Increment `current_concept`
- Update `checkpoint.json`: `"status": "concept_<N+1>_pending"`

**At a load-bearing concept:** only advance once the Teach-Back Gate returned **confirmed** or **skipped** (a `partial` re-confirms first; a `misconception_corrected` re-confirms after the correction). A plain-checkpoint concept advances on the bare 🟢 as before.

Return to Step 3 with the next concept.

#### 🟡 YELLOW — Re-explain

**Pause forward motion completely.** Do not finish the current sentence trying to land the point.

Ask one clarifying question: *"Which part of [concept] is the unclear bit — the [mechanism A], the [mechanism B], or the [procedure C]?"*

Re-explain the unclear part with a **different angle** than the first attempt:
- If first was abstract → use a concrete example
- If first was technical → use an analogy
- If first was procedural → lead with mechanism
- Tie to a fundamental the learner already owns (🟢 prereq from baseline)

#### Persist to File: Re-explanation Log

> **RESILIENCE (Pattern A):** Capture the re-explanation so subsequent sessions
> can see what angle worked.

Append to `cauldron/<session-id>/walkthrough_state.md` (in `re_explanations_used` section):

```markdown
- Concept: <name>
  - Unclear part: <as identified>
  - Re-explanation approach: <angle used>
  - Resolution: <signal after re-explanation>
```

Update `checkpoint.json`: `"status": "concept_<N>_yellow_reexplained"`

Get a fresh R/Y/G signal. Loop until 🟢 or 🔴.

If the learner stays 🟡 after 2 re-explanations on the same concept → propose escalating to 🔴 ("This might want a foundational rebuild — should we treat it as red?"). Let the learner choose.

#### 🔴 RED — Route to Gap Recovery

Update `checkpoint.json`: `"status": "gap_active"`, `"gap_concept": "<current concept>"`.

Hand off to `workflows/gap_recovery.md`. Pass:
- Current concept name
- The unclear part (if specified)
- Path to gap_map.md

Wait for gap_recovery to return with `status: gap_resolved` before resuming Step 3 on the same concept.

### Step 5: Walkthrough Complete

When all concepts are complete:

Summarize the chain in one short message:
> We've walked through: [concept 1] → [concept 2] → ... → [concept N]. End of chain.

Ask:
> Want to save notes from this session?

#### Persist to File: Final Walkthrough State

Update `walkthrough_state.md`:

```markdown
complete: true
concepts_complete: [all N concepts]
finished_at: <ISO timestamp>
```

Update `checkpoint.json`: `"status": "walkthrough_complete"`

#### Update the Learning Ledger (always — before export)

Run the **WRITE procedure** in `workflows/ledger.md`. This fires at completion **regardless of whether the learner exports notes**, so the session compounds into cross-session memory either way:
- Teach-back-verified concepts → **OWNED**
- Plain 🟢 concepts → **FAMILIAR**
- Parked gaps → **OPEN**; gaps closed this session leave OPEN
- A FAMILIAR concept that earned a teach-back this session upgrades to OWNED

Report the result in the completion summary: *"Ledger updated: +N owned, +M familiar, K gaps still open."* This is the payoff — next time a related topic's baseline runs, these won't be re-asked from scratch.

If learner says yes → route to `workflows/export_notes.md`.
If no → mark session as ended, leave cauldron state intact for later resumption. (The ledger is already updated, so the session is never "lost" even without an export.)

---

## State Transition Chain (Pattern E)

**State transitions:**
`baseline_complete` → (`concept_N_pending` → `concept_N_introduced` → [`concept_N_yellow_reexplained` ⇄ `concept_N_introduced`]* → `concept_N_complete`)* → `walkthrough_complete`

Gap recovery transitions: `concept_N_introduced` → `gap_active` → (gap_recovery.md) → `gap_resolved` → `concept_N_introduced` (resume)
