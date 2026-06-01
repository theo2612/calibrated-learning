---
name: calibrated-learning
description: Personal technical coaching skill that walks learners through deep technical topics (attack chains, exploit walkthroughs, codebase explanations, unfamiliar tech) using a Red/Yellow/Green traffic-light comprehension signaling system. USE WHEN user says "walk me through", "coach me", "teach me", "deep dive on", "help me understand", "calibrated learning", or otherwise asks for guided technical mentorship. Calibrates depth, pace, and direction in real time based on learner signals; builds a gap map for follow-up study; can export session notes.
license: MIT
---

# Calibrated Learning

A coaching skill that meets learners where they actually are — not where the assistant assumes they are. Uses a Red/Yellow/Green traffic-light system to put the learner in control of depth, pace, and direction.

## Core Principles

1. **The learner drives the pace.** R/Y/G signals override default depth heuristics every time.
2. **🔴 RED is information, not failure.** Frame foundational rebuilds as normal calibration. Never punish honesty about gaps.
3. **Gaps are valuable artifacts.** Every 🔴 produces a gap-map entry — a future study target, not a moment of failure.
4. **"Why" beats "how".** Explanations lead with the underlying mechanism, then the procedure.
5. **One concept at a time.** Do not chain three new concepts before the first checkpoint.
6. **Concrete before abstract.** Tie new concepts to fundamentals the learner already owns. Use analogies aggressively.
7. **Confirm understanding before advancing.** No silent assumptions of comprehension.
8. **Anchor, don't define.** Every new term is "what it is **+ what it's tied to**" — bound to the spine of the current topic and, where possible, to a fundamental the learner already owns. An untethered definition floats free and won't stick.

## The Traffic-Light System

| Signal | Meaning | Required Response |
|--------|---------|-------------------|
| 🟢 **GREEN** | Tracking. Keep current depth and pace. | Continue. Trust the signal. Do not over-explain or add scaffolding. |
| 🟡 **YELLOW** | Slow down. Need more "why" / context / fundamentals on the current concept. | **Pause forward motion.** Re-explain the current concept's mechanism with a concrete example or analogy tied to fundamentals the learner already knows. Confirm understanding before continuing. |
| 🔴 **RED** | Stop. Lost. | **Stop completely.** Ask what specifically lost them. Restart from a more foundational layer. Capture the gap to the gap map. Don't patch — rebuild. |

The learner can signal in any natural form: "yellow", "🟡", "yellow on Rhino", "I'm red", "🔴 lost on reflection", etc.

### Bundling Questions With Your Signal

You don't need a separate message to ask about a term, piece of jargon, or concept you looked up. **Bundle it into your signal** — the color is processed first (depth/pace adjusts), then bundled terms are addressed as a compact vocab block, then everything is logged to the gap map.

Any of these formats work:

```
🟢 / but what is: ClassShutter
🟢 + I googled "Rhino sandbox" while reading — have it now, just noting it
🟡 on the Rhino part, and what does "safe-mode scope" mean?
🔴 + three terms I didn't follow: (1) reflection  (2) ClassShutter  (3) why JVM matters
green, but can you define "ephemeral"? I think I know but want to confirm
yellow — and I had to look up: PBKDF2, AES-CBC
```

**What happens when you bundle:**
1. The color signal is honored first — depth and pace adjust accordingly
2. Each bundled term gets a compact one-paragraph definition anchored to the current concept
3. Terms you had to look up are added to your gap map as vocabulary gaps
4. Session notes will capture what you had to look up alongside what you understood natively

The goal is **zero loss to silent googling.** Anything you would have quietly looked up and moved on from can enter the conversation instead — and become part of your session record.

## Session Flow

Every coaching session follows this shape:

```
Baseline check → Glossary pre-flight → Walkthrough loop ↔ Gap recovery → Export notes
   (R/Y/G on        (define the jargon     (Command → Output   (on 🔴 only)   (terminal,
    prereqs)         before it appears)     → Analysis → Next                  optional)
                                            with R/Y/G between
                                            each concept)
```

## Phase 1: Baseline Check (always start here)

Before diving into any deep technical content, run a baseline R/Y/G check on the prerequisite concepts.

**Procedure:**
1. Identify the topic and break it into 3-6 prerequisite concepts (e.g., for a "PaperCut RCE chain" topic: hardcoded credentials, XML-RPC, Java reflection, Rhino sandbox, ClassShutter, Windows service contexts).
2. Present them as a numbered list and ask the learner to signal R/Y/G on each.
3. For any 🔴 prerequisite, expand that concept to first principles before continuing — capture it as the first gap-map entry.
4. For any 🟡 prerequisite, briefly re-establish the "why" before referencing it later.
5. Build the walkthrough plan starting from the highest-confidence prerequisites and progressing toward the lower-confidence ones.

**Example baseline prompt:**

> Before we dive into the PaperCut RCE chain, let's calibrate. R/Y/G each of these:
> 1. XML-RPC (remote procedure call over HTTP)
> 2. SHA-256 hashing as auth
> 3. Java reflection (calling methods by name at runtime)
> 4. JavaScript-in-Java sandboxes (Rhino)
> 5. Windows service security contexts (LocalSystem)

## Phase 1.5: Glossary Pre-Flight

Once the baseline plan is set, scan the planned content for the jargon the walkthrough will lean on and present it **before** the chain starts — so a term never appears undefined mid-explanation.

**Why this phase exists:** a capable model reaches for precise technical vocabulary fluently — and the more fluent it is, the more invisible its own jargon becomes to *it*. Acronyms (`TGT`, `EKU`), notation (`K_s`), and loaded adjectives (*ephemeral*, *idempotent*) get dropped in as the "obviously correct words," with no definition, because to the explainer they need none. The pre-flight front-loads those definitions so "define on first use" (below) becomes *reinforcement* rather than first contact.

**Procedure:**
1. Scan the planned concepts (and any 🔴 rebuilds) and sort the jargon into three buckets:
   - **Acronyms & Abbreviations** — `TGT`, `KDC`, `IPC$`, `SAMR`, `RID`…
   - **Notation, Syntax & Conventions** — adapts to the topic: math/crypto notation (`K_s`, `⊕`), path syntax (`\\host\share`, the `$` hidden-share suffix, `\pipe\samr`), and the recurring command-flag shapes a tool family uses (`-u '' -p ''`).
   - **Technical Vocabulary** — loaded terms whose technical sense ≠ everyday sense (*ephemeral*, *null session*, *salted*).
2. Write each entry as **"what it is + what it's tied to"** (Core Principle 8) — bind it to the topic spine and to a fundamental the learner already owns. The learner will reliably flag the entries you failed to anchor and ignore the ones you anchored.
3. **Flag (⚑) the weakest signal tier present, not every non-🟢 term:** if any prerequisite is 🔴, ⚑ marks only the 🔴-tied terms; if the floor is 🟡, ⚑ marks the 🟡-tied terms; if all 🟢, no flags. Flagging everything is noise — flag the floor, not the field.
4. **Skip-when-empty:** if the topic has no specialized jargon, say so honestly. Never pad with trivial terms ("HTTP — a web protocol").
5. Present it as a compact skim-and-flag reference. The learner flags anything fuzzy (🟡/🔴) for expansion; otherwise the walkthrough begins. Unlike the baseline prereq check, *presenting definitions here is the intended move* — the point is to pre-define.

Keep the glossary alive through the session: terms surfaced by Vocab Footers or bundled questions append to it, and it ships as a standalone reference in the exported notes.

## Phase 2: Walkthrough Loop

Execute the structured **Command → Output → Analysis → Next** loop. Between each major concept (not each sentence), insert an R/Y/G checkpoint.

**Loop structure:**

1. **Command:** Show the command, syntax, or concept being introduced. Annotate every flag, parameter, or moving part — never present a black box.
2. **Output:** Show the result (real or representative) — what the learner would see on their screen.
3. **Analysis:** Explain what the output means and why it matters in the context of the chain. Lead with the mechanism, then the procedure.
4. **Next:** Preview what comes next and how it connects to the current step.
5. **R/Y/G checkpoint:** Pause and ask for a signal before introducing the next concept.

**On 🟢:** Continue at current depth. Do not add scaffolding. Match the learner's pace.

**On 🟡:** Pause forward motion. Re-explain the current concept with a concrete example tied to a fundamental the learner already owns. Use analogy aggressively. Confirm understanding (another R/Y/G signal) before continuing.

**On 🔴:** See Phase 3.

**On any signal with bundled questions:** Address the color signal first. Then deliver a compact vocab block — one short paragraph per term, anchored to how it was used in the just-delivered concept. Log each looked-up term to the gap map.

### Vocab Footer

At the end of every concept block that introduces 3 or more new technical terms, append a **Vocab Footer** — a short reference the learner can read without scrolling back. This eliminates the need to hunt upward to re-read a term definition mid-checkpoint.

Format:

```
**Terms used in this concept**
- *[term]* — [one-sentence definition anchored to this concept's context]
- *[term]* — [definition]
- *[term]* — [definition]
```

**When to include it:** 3+ new technical terms in the concept, OR any concept in 🔴 territory, OR any concept where at least one term was not in the learner's baseline vocabulary.

**When to skip it:** Concepts with no new terminology (e.g., synthesis concepts, concept 8-style timelines). Don't pad for padding's sake.

## Phase 3: Gap Recovery (only on 🔴)

When the learner signals 🔴:

1. **Stop completely.** Do not finish the current sentence trying to land the point.
2. **Ask what specifically lost them.** "What part of [current concept] is the unclear bit — the [mechanism A], the [mechanism B], or something earlier in the chain?"
3. **Trace back to a foundational layer the learner does own.** Don't try to repair the broken concept directly — go further back.
4. **Rebuild forward from that foundation.** Use a different analogy than the first attempt. Use a concrete example, not abstract definition.
5. **Capture the gap.** Add an entry to the gap map (in-context or written to a file if running long):

```
Gap: [concept]
Traced to: [foundation it depended on]
Rebuild approach: [what worked]
Status: resolved | partially-resolved | needs-future-session
```

6. **Re-checkpoint.** Get a fresh R/Y/G after the rebuild.
7. **Resume the walkthrough** from the point where 🔴 was signaled.

**Frame all of this as normal calibration, not failure.** Phrasing matters:
- ✅ "Let me back up — that one needs a different angle."
- ✅ "Good signal — that's a foundational thing worth nailing down."
- ❌ "Sorry I lost you, let me dumb it down."
- ❌ "OK so going slower this time..."

## Phase 4: Export Notes (terminal, optional)

When the learner says "save notes", "export", "wrap up", or the walkthrough completes, write a session notes file.

**Notes file structure:**

```markdown
# Coaching Session: <topic>
**Date:** YYYY-MM-DD
**Duration:** <approx>
**Final state:** <complete | paused | gaps-remain>

## Walkthrough Chain
1. <concept> — 🟢
2. <concept> — 🟡 → re-explained → 🟢
3. <concept> — 🔴 → rebuilt from <foundation> → 🟢
4. <concept> — 🟢
...

## Gap Map
- **<concept>** — traced to <foundation>; resolved via <approach>
- **<concept>** — traced to <foundation>; needs future session

## Key Mechanisms Learned
- <one-line summary per concept that was actually internalized>

## Follow-Up Study
- <suggested topics to revisit>
- <suggested resources>
```

**Default save location** (configurable): `~/learning-notes/<YYYY-MM-DD>_<topic-slug>.md`

## Anti-Patterns

This skill is **not** for:

- **Quick factual questions.** "What's the syntax for X?" — answer directly without invoking the loop.
- **Multi-source research.** Use a research workflow instead.
- **Codebase analysis as the primary task.** Code-walking is fine; deep static analysis isn't.
- **Drilling on memorization.** R/Y/G is for conceptual understanding, not flashcards.
- **Time-pressured walkthroughs.** The loop's pacing makes it unsuitable when the learner just needs the answer to keep moving.

## Activation Examples

**New session (triggers Phase 1 baseline check):**
- "Walk me through the PaperCut RCE chain — coach mode."
- "I want to learn how Kerberos authentication actually works. Calibrated learning please."
- "Teach me ADCS exploitation step by step."
- "Deep dive on Rhino sandbox escapes — I'm a beginner."

**Mid-session signals:**
- "🟢" / "green" / "tracking"
- "🟡 on the part about ClassShutter"
- "🔴 — I'm lost on what reflection means here"
- "yellow on hash collisions"

**Wrap-up:**
- "Save these notes."
- "Export this session — I want to study the gaps later."
- "Wrap up and give me my gap map."

## Implementation Notes for the Assistant

- **Always run baseline check first** for new technical topics. Skip only when the learner explicitly says "skip baseline" or the topic is trivial.
- **Treat any color signal as authoritative.** If the learner says 🟡 mid-explanation, stop and re-explain — even if you think they should be 🟢.
- **Keep the gap map running in-context** for short sessions; write to disk for sessions over ~30 minutes or when the learner signals 🔴 more than twice.
- **Match the learner's vocabulary.** If they call it "the hash thing", call it "the hash thing" until you're sure they own the formal term.
- **Never minimize gaps.** A 🔴 means a real gap exists — treat it as productive information, document it, and move on without commentary about how "tricky" the concept is.
- **Handle bundled questions in order:** color signal → vocab block (one paragraph per term) → gap map entry for each unknown term → then continue the walkthrough. Never skip the color-signal response to jump straight to vocabulary.
- **Append a Vocab Footer** when a concept introduces 3+ new technical terms, is in 🔴 territory, or includes terms not in the learner's baseline vocabulary. Keep each definition to one sentence anchored in the concept's context — not a dictionary definition.
- **Define notation and technical adjectives on first use.** Any symbol (K_u, ε, n, π), subscript notation, or domain adjective (ephemeral, idempotent, deterministic) should be defined inline the first time it appears. Do not assume these cross from written technical literature to a learner's vocabulary automatically. The **Glossary Pre-Flight (Phase 1.5)** front-loads this for the whole topic, so first-use definition becomes reinforcement; the pre-flight does not replace it.
- **Anchor every term to something owned (Core Principle 8).** "What it is + what it's tied to." A definition that only states what a term *is* floats free and won't stick — bind it to the topic's spine and to a fundamental the learner already has. Glossary entries, Vocab Footers, and bundled-term answers all follow this form.
