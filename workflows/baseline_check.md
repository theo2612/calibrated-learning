---
name: Baseline Check
description: |-
  Pre-walkthrough R/Y/G assessment on prerequisite concepts for a given topic.
  Decomposes the topic into 3-6 prerequisites, asks the learner to signal
  R/Y/G on each, and builds a walkthrough plan from highest to lowest
  confidence — capturing any 🔴 prerequisites as the first gap-map entries.
license: MIT
compatibility: Claude Code
metadata:
  author: theo2612
  usage: workflow file for the calibrated-learning skill
  version: 0.4.0
  related files: "workflows/router.md, workflows/walkthrough.md, workflows/gap_recovery.md"
  creation date: 2026-05-11
  last modified: 2026-06-02
---

# Baseline Check Workflow

## Context Budget Plan

| File | Load At | ~Tokens | Purpose |
|------|---------|---------|---------|
| This workflow | Step 0 (already loaded) | ~1,500 | Baseline check instructions |
| `cauldron/<session-id>/topic.md` | Step 1 | ~300 | User-supplied topic and context |
| `cauldron/<session-id>/checkpoint.json` | Step 1 | ~300 | Session state |
| `cauldron/<session-id>/glossary.md` | Step 5 (write) | ~600 | Pre-flight jargon artifact (acronyms, notation, vocab) |
| `cauldron/<session-id>/baseline.md` | Step 6 (write) | ~1,000 | Output artifact |

**Batch-Reading Guardrail:** No external files loaded — this workflow is self-contained except for cauldron state.

---

## Resumption Procedure

> **PURPOSE:** Resume the baseline check after session death.

### File Proof Table

| Checkpoint Status | Expected File Proof | Resume At |
|-------------------|---------------------|-----------|
| `initialized` | `cauldron/<session-id>/topic.md` | Step 1 — Decompose topic |
| `prerequisites_drafted` | `cauldron/<session-id>/prerequisites_draft.md` | Step 2 — Present to learner |
| `prerequisites_signaled` | `cauldron/<session-id>/baseline_signals.md` | Step 3 — Process signals |
| `glossary_built` | `cauldron/<session-id>/glossary.md` | Step 6 — Write baseline artifact |
| `baseline_complete` | `cauldron/<session-id>/baseline.md` | Done — route back to router |

### Resumption Steps

1. Read `checkpoint.json` from active session directory
2. Look up `status` in File Proof Table above
3. Verify proof file exists; if missing, regress to previous status
4. Resume at indicated step

---

## Trigger and Activation

This workflow activates when:
- Router dispatches a new session (status `initialized`)
- Learner explicitly invokes "baseline" or "calibrate first"

---

## Purpose

Front-loads R/Y/G calibration so the bulk of the walkthrough runs at the right depth from the start. Reactive-only signaling forces the learner to interrupt mid-explanation, which is high-cost. The baseline check is low-cost and catches most calibration needs before they become disruptive.

---

## Workflow

### Step 1: Decompose the Topic into Prerequisites

Read `cauldron/<session-id>/topic.md` to load the topic and any learner-supplied context.

Identify 3-6 **prerequisite concepts** the walkthrough will rely on. Good prerequisites are:
- **Foundational** — concepts that appear multiple times in the walkthrough
- **Distinct** — not overlapping with each other
- **Named precisely** — use the exact terminology (the learner can ask "what's that?" if unfamiliar)
- **Ordered** — earliest prerequisites first (the ones other concepts depend on)

**Examples:**

| Topic | Prerequisites |
|-------|--------------|
| PaperCut RCE chain | (1) XML-RPC, (2) SHA-256 hashing as auth, (3) Java reflection, (4) JavaScript-in-Java sandboxes (Rhino), (5) Windows service contexts (LocalSystem) |
| Kerberos authentication | (1) symmetric encryption, (2) tickets vs sessions, (3) KDC/AS/TGS roles, (4) PAC structure, (5) constrained delegation |
| ADCS exploitation | (1) PKI fundamentals, (2) certificate templates, (3) EKU semantics, (4) NTAuth and CA cert stores, (5) S4U2self/proxy |

#### Persist to File: Prerequisites Draft

> **RESILIENCE (Pattern A):** Write the prerequisite list to disk before presenting
> to the learner. If session dies mid-baseline, this file preserves the analysis.

Write to:
```
~/.claude/skills/calibrated-learning/cauldron/<session-id>/prerequisites_draft.md
```

Contents (~500 tokens):
- Topic name
- Numbered list of 3-6 prerequisites with one-line description each
- Brief note on why each prerequisite matters for the topic

Update `checkpoint.json`: `"status": "prerequisites_drafted"`

### Step 2: Present to the Learner

Present the prerequisite list in a single message with this structure:

```
Before we dive into [topic], let's calibrate. R/Y/G each of these:

1. [Prerequisite name] — [one-line description]
2. [Prerequisite name] — [one-line description]
3. [Prerequisite name] — [one-line description]
...

🟢 means tracking. 🟡 means you want a "why" refresher. 🔴 means we need to start from fundamentals on that one.
```

**DO NOT explain the prerequisites in this step.** The whole point is to find out what the learner already owns. Premature explanation defeats the calibration.

### Step 3: Capture Signals

After the learner responds, parse their signals. Accept these formats:
- "🟢 1, 🟡 2, 🟢 3, 🔴 4, 🟢 5"
- "1 green, 2 yellow, 3 green, 4 red, 5 green"
- "all green except 4"
- "red on Rhino, yellow on reflection, green on the rest"
- Free-form descriptions ("I know XML-RPC, kinda know reflection, never heard of Rhino")

Map each prerequisite to a color. If a signal is ambiguous, ask one clarifying question — don't guess.

#### Persist to File: Baseline Signals

Write to:
```
~/.claude/skills/calibrated-learning/cauldron/<session-id>/baseline_signals.md
```

Contents (~400 tokens):
- Prerequisite name → signal (🟢/🟡/🔴)
- Raw learner response (for audit)

Update `checkpoint.json`: `"status": "prerequisites_signaled"`

### Step 4: Process Signals and Build Walkthrough Plan

For each prerequisite, decide the action:

| Signal | Action in Walkthrough |
|--------|----------------------|
| 🟢 | Reference freely without re-explanation; assume ownership |
| 🟡 | Provide a brief "why this matters" framing the first time it appears |
| 🔴 | Pre-explain from fundamentals BEFORE introducing it in the walkthrough; capture as first gap-map entry |

For any 🔴 prerequisites:
1. Plan a brief foundational explanation to deliver before the main walkthrough starts
2. Add an entry to `gap_map.md` (creating the file if needed):

```markdown
## Gap 1: <prerequisite name>
- **Signal:** 🔴 at baseline
- **Foundation needed:** <what the rebuild should target>
- **Status:** pre-walkthrough explanation queued
```

Plan the walkthrough order:
- Start from the **highest-confidence** prerequisites (🟢) so the learner builds momentum
- Introduce 🟡 prerequisites with their "why" framing
- Hold 🔴 prerequisites for the foundational explanation step

#### Tag Load-Bearing Concepts (Teach-Back Gate)

Mark the **1–2 concepts everything downstream depends on** as `load-bearing`. These are where the walkthrough fires the Teach-Back Gate (see `walkthrough.md`) — a verified explain-back instead of a self-assessed 🟢, because a false 🟢 *here* poisons every concept after it.

A concept is load-bearing if:
- **Multiple later concepts reference it** — it's a hub, not a leaf, in the chain.
- **It's a central 🔴/🟡 prerequisite** the rest of the topic is built on.

Keep it to 1–2. Tagging everything defeats the point — the gate is selective by design. (Example, SMB enumeration: "MS-RPC over named pipes" was load-bearing — RID cycling and the whole identity layer hung off it; the file-discovery concepts were leaves.)

Record the tags in `baseline.md` (the walkthrough reads them to know where to fire the gate). 🔴-rebuild concepts are implicitly gated post-rebuild and don't need a separate tag.

### Step 5: Glossary Pre-Flight Scan

> **WHY THIS STEP EXISTS:** A capable model reaches for precise technical
> vocabulary fluently — and the more fluent it is, the more invisible its own
> jargon becomes to it. Acronyms (TGT, EKU), notation (`K_s`), and loaded
> adjectives (ephemeral, idempotent) get dropped into explanations as the
> "obviously correct words," with no definition, because to the explainer they
> need none. This is a documented repeat-failure mode for this learner: the
> Kerberos session's gap map flagged it **twice** (`K_s` undefined, "ephemeral"
> undefined) and both gap entries self-prescribed a pre-flight pass. This step
> front-loads that pass so the silent jargon is defined *before* it appears.

Scan the **anticipated walkthrough content** — every concept in the plan from
Step 4, plus the foundational rebuilds queued for 🔴 prerequisites — and surface
the terminology the walkthrough will lean on. Sort each item into one of three
classes:

| Class | What to catch | Examples |
|-------|---------------|----------|
| **Acronyms & Abbreviations** | Any initialism or shortened form the walkthrough will use | TGT, TGS, KDC, PAC, ADCS, EKU, NTAuth, RCE, P2P, OPSEC, MFD |
| **Notation, Syntax & Conventions** | Whatever symbolic/structural convention the topic asks the learner to *read*: math/crypto notation **and** path syntax **and** recurring command-line patterns. This bucket adapts to the topic. | Crypto: `K_s`, `K_u`, `H(x)`, `⊕`, `\|\|` (concat). Paths: `\\host\share`, `$`-suffix (hidden share), `\pipe\samr`. Commands: the recurring flag/argument shape a tool family uses (e.g. `-u '' -p ''` = null auth; `target[:port]`) |
| **Technical Vocabulary** | Loaded adjectives and domain terms whose everyday meaning ≠ technical meaning | ephemeral, idempotent, deterministic, symmetric, salted, derived, canonical, null session |

**Scan heuristics:**
- **Catch the silent ones.** The terms most worth defining are the ones the model would *not* think to define — the fluent, precise words. If a term feels too basic to define, that is often exactly the one that bites.
- **Every definition needs a "ties-to" anchor — not just a "what-it-is."** This is the load-bearing heuristic. A definition that only says *what a term is* floats free and does not stick; the learner cannot retain an untethered fact. Each entry must bind the term to the topic's spine — *how does this connect to the thing we're actually walking through?* Two parts:
  - **What it is** — one clause.
  - **What it's tied to** — what it connects to *in this topic*, and ideally to a fundamental the learner already owns (pull from their 🟢 prerequisites).
  - **Failure example:** "*SID* — the unique ID for every Windows principal, looks like `S-1-5-21-…`" ← what-it-is only; floats free.
  - **Fixed:** "*SID* — a Windows account's permanent fingerprint; **you read it over SMB by riding IPC$ to the LSARPC pipe** — SMB is the transport, not where SIDs live." ← anchored to the topic spine.
  - The learner will reliably flag the entries you failed to anchor and ignore the ones you anchored. If an entry has no plausible tie to the topic, it probably does not belong in this glossary.
- **⚑ flags the weakest tier present, not every non-🟢 term.** Cross-reference each term against the prerequisite signals from Step 3, then flag by the *weakest tier that exists in this session*:
  - If any prerequisite is 🔴 → ⚑ marks only the 🔴-tied terms (highest priority; don't dilute).
  - If the weakest signal is 🟡 (no 🔴s) → ⚑ marks the 🟡-tied terms.
  - If everything is 🟢 → no ⚑ flags (the glossary is pure reference).
  - **Rationale (dry-run finding, SMB-enum session):** when *every* prerequisite came back 🟡/🔴, a literal "flag anything non-green" marked all 25 terms — useless. ⚑ only carries signal if it points at the few terms that need it most. Flag the floor, not the field.
- **Define on first use still applies.** This pre-flight does not replace inline definition; it backstops it. The walkthrough still defines notation/adjectives on first appearance — but now there's a reference the learner pre-read, so first-use definition becomes reinforcement, not first contact.

#### Skip-When-Empty

If the topic genuinely has no specialized acronyms, notation, or loaded vocabulary (rare for security/technical topics, but possible), write a stub glossary recording that the scan ran and found nothing notable, and note it for the walkthrough so it skips presentation. **Never pad the glossary with trivial terms to look thorough** — an empty scan honestly reported is better than a glossary full of "HTTP — a web protocol."

#### Persist to File: Glossary

> **RESILIENCE (Pattern A):** The glossary is the single living source of
> defined terms for the whole session. The walkthrough presents it before the
> chain starts and appends to it as new terms surface; export_notes folds it
> into the final notes.

Write to:
```
~/.claude/skills/calibrated-learning/cauldron/<session-id>/glossary.md
```

Contents (~600 tokens):
```markdown
# Glossary Pre-Flight — <topic>

> Skim this before we start. Flag (🟡/🔴) anything still fuzzy and I'll expand it.
> ⚑ = tied to your weakest-signalled prerequisite tier this session — the highest-priority "look closer" items.
> Every entry is **what it is + what it's tied to** — the tie is what makes it stick.

## Acronyms & Abbreviations
- **TGT** ⚑ — Ticket-Granting Ticket; the credential the KDC issues after first auth **so you stop re-sending your password on every request** — it's the thing every later Kerberos step trades on.
- **<acronym>** — <what it is + what it connects to in this topic>

## Notation, Syntax & Conventions
- **K_s** ⚑ — session key; subscript `s` = "session." **The key the client and service actually encrypt their conversation with** — generated fresh, discarded after.
- **`\\host\share`** — UNC path syntax: how you address any SMB resource; the address bar for everything that follows.
- **`-u '' -p ''`** — the recurring null-auth flag shape across the SMB tool family; reading it once means you recognize it in every command below.
- **<symbol/path/flag>** — <what it denotes + how to read it + where it shows up in this topic>

## Technical Vocabulary
- **ephemeral** ⚑ — short-lived; generated fresh and thrown away after one use **(vs. the long-term password-derived keys it protects)** — that contrast is the whole point of the design.
- **<term>** — <what it is + what it's tied to in this topic, contrasted with the everyday meaning if they differ>

## Added during the walkthrough
<!-- The walkthrough appends terms here as they surface from Vocab Footers and bundled vocab questions. Same what-it-is + what-it's-tied-to format. -->
```

If the scan was empty, write instead:
```markdown
# Glossary Pre-Flight — <topic>

Scan ran; no specialized acronyms, notation, or loaded vocabulary anticipated for this topic. Inline definition on first use still applies during the walkthrough.

## Added during the walkthrough
<!-- Terms that surface mid-session land here. -->
```

Update `checkpoint.json`: `"status": "glossary_built"`

### Step 6: Write Baseline Artifact

#### Persist to File: Baseline

> **RESILIENCE (Pattern A):** This is the output artifact of the baseline check
> phase. The walkthrough workflow reads this to build its explanation plan.

Write to:
```
~/.claude/skills/calibrated-learning/cauldron/<session-id>/baseline.md
```

Contents (~1,000 tokens):
```markdown
# Baseline: <topic>

## Prerequisites and Signals
| # | Prerequisite | Signal | Action |
|---|--------------|--------|--------|
| 1 | <name> | 🟢 | Reference freely |
| 2 | <name> | 🟡 | Framing on first use |
| 3 | <name> | 🔴 | Pre-explain from fundamentals |

## Walkthrough Plan
1. <Concept order based on signals>
2. ...

## Load-Bearing Concepts (Teach-Back Gate fires here)
- <concept name> — <why it's a hub: which later concepts depend on it>
- (1–2 max; 🔴-rebuild concepts are gated post-rebuild automatically)

## Pre-Walkthrough Foundations (🔴 prerequisites)
- <name>: <approach for foundational rebuild>

## Glossary Pre-Flight
- Built: yes | empty-scan
- Path: cauldron/<session-id>/glossary.md
- Counts: <N acronyms, N notation, N vocab> (<M flagged ⚑>)

## Notes
- Learner's raw signal response (verbatim)
- Any clarifying questions asked and answered
```

Update `checkpoint.json`: `"status": "baseline_complete"`

### Step 7: Hand Off to Walkthrough

Return control to the router with status `baseline_complete`. The router will dispatch to `workflows/walkthrough.md`.

Brief the learner:
> Baseline complete. I've built a quick glossary of the acronyms, notation, and jargon this topic uses — you'll see it first so nothing shows up undefined. We'll start with [first concept] and build up. I've queued a foundational explanation on [🔴 prereqs] before we touch them in the chain. Ready?

Wait for confirmation before transitioning to the walkthrough workflow.

---

## State Transition Chain (Pattern E)

**State transitions:**
`initialized` → `prerequisites_drafted` → `prerequisites_signaled` → `glossary_built` → `baseline_complete`
