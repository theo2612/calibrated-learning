# calibrated-learning

A Claude Code skill that turns Claude into a personal technical coach using a Red/Yellow/Green comprehension signaling system.

Built for career-changers, learners battling imposter syndrome, and anyone who wants structured technical mentorship that doesn't lose them in the first paragraph.

## What It Does

When you ask Claude to walk you through a deep technical topic — an attack chain, a protocol, an unfamiliar codebase, a tricky concept — the default behavior is to keep going at whatever depth Claude assumed appropriate at the start. If you get lost mid-explanation, you have three bad options:

1. Stay quiet and pretend you're tracking (gap compounds, chain becomes meaningless)
2. Say "wait, I'm lost" (high social cost; risks feeling like failure)
3. Try to catch up by rereading (slow, often unsuccessful, breaks pace)

This skill gives you a fourth option: **a low-cost, fast, non-judgmental signal that the assistant must respond to immediately.**

## The Traffic-Light System

| Signal | Meaning | What happens next |
|--------|---------|-------------------|
| 🟢 **GREEN** | Tracking. Keep going at this depth and pace. | Continue. No added scaffolding. |
| 🟡 **YELLOW** | Slow down. I need more "why" / context / fundamentals on the current concept. | Pause forward motion. Re-explain with a concrete example or analogy. Confirm before continuing. |
| 🔴 **RED** | Stop. I'm lost. Restart from first principles. | Stop completely. Find the foundational layer you do own. Rebuild forward. Capture the gap. |

You can signal in any natural form: `green`, `🟡`, `yellow on Rhino`, `🔴 lost on reflection`, `I'm red on this part`.

### Signal Quick Reference

Print this out or keep it in a tab. You can use it at any checkpoint.

---

**Simple signals (just the color):**
```
green
🟢
yellow
🟡
red
🔴
```

**Signal + specific part:**
```
🟡 on the Rhino part
🔴 — lost on what reflection means
green on the concept, yellow on the command syntax
```

**Signal + terms you had to look up (bundles into the response):**
```
🟢 / but what is: ClassShutter
🟡 on Rhino, and can you define "safe-mode scope"?
🔴 + I didn't follow these terms: (1) reflection  (2) ClassShutter  (3) JVM
green — but I had to google "ephemeral" while reading, here's what I found: [paste it]
yellow — and: PBKDF2, AES-CBC (what are these?)
```

**When you bundle terms:**
1. The color is honored first — I adjust depth and pace
2. Each term gets a compact definition anchored to how it was just used
3. Terms you googled or didn't know are added to your gap map
4. The session notes capture what you had to look up, not just what you understood immediately

**The goal: zero loss to silent googling.** Anything you would have quietly looked up can come into the conversation and become part of your session record.

---

### What the Vocab Footer Is

At the end of every concept block with 3+ new technical terms, you'll see:

```
**Terms used in this concept**
- *Reflection* — calling Java methods by name at runtime, without knowing the type at compile time
- *ClassShutter* — Rhino API that blocks cross-package class loading; the missing control in CVE-2024-1882
- *Rhino* — Mozilla's JavaScript-in-JVM engine, used by PaperCut for device scripting
```

This means you **don't have to scroll back up** to re-read a term definition. The reference is right above the checkpoint.

### The Glossary Pre-Flight

Before the walkthrough starts, the skill scans the planned content for the jargon it's about to lean on and hands you a one-screen glossary in three buckets:

- **Acronyms & Abbreviations** — `TGT`, `KDC`, `EKU`, `ADCS`, `RCE`…
- **Notation, Syntax & Conventions** — adapts to the topic: crypto notation (`K_s`, `⊕`), path syntax (`\\host\share`, the `$` hidden-share suffix, `\pipe\samr`), and the recurring command-flag shapes a tool family uses
- **Technical Vocabulary** — loaded terms whose technical meaning differs from the everyday one: *ephemeral*, *idempotent*, *null session*, *salted*…

Every entry is **what it is + what it's tied to** — a definition that only says what a term *is* floats free and won't stick, so each one is bound to the topic's spine (and, where possible, to a fundamental you already own). A ⚑ marks the terms tied to your **weakest-signalled** prerequisites this session (🔴-tied if you flagged any 🔴, else 🟡-tied) — the ones most worth a closer look. You skim it, flag anything fuzzy, and dive in knowing nothing will show up undefined.

**Why it exists:** a capable model reaches for precise jargon fluently — and the more fluent it is, the more invisible its own jargon becomes to *it*. `K_s` and "ephemeral" are the "obviously correct words," so they get dropped in cold with no definition. (This skill's own Kerberos session caught exactly this twice.) The pre-flight front-loads the definitions so "define on first use" becomes reinforcement instead of first contact. On a topic with no real jargon, the scan honestly reports empty rather than padding with `HTTP — a web protocol`.

The glossary stays alive through the session — terms that surface in Vocab Footers or that you bundle into a signal get appended to it — and it ships in your exported notes as a standalone reference.

### The Teach-Back Gate

A 🟢 you type yourself is *unverified*. The classic failure: you pattern-match "yeah, that tracks," signal green, and move on — but the understanding was thinner than the signal claimed, and the gap compounds invisibly. A false 🟢 looks identical to a real one.

At the **load-bearing concepts** (the 1–2 things everything downstream depends on) and **after every 🔴 rebuild**, the checkpoint upgrades: instead of a bare signal, you explain it back in one line (or predict the next step). Producing the explanation is the cheapest way to tell a real 🟢 from a hopeful one.

- **The burden is on the explanation, not you.** The prompt asks you to check *the assistant's* explanation landed — a gap reads "my explanation missed X," never "you got it wrong."
- **Mechanism, not phrasing.** A rough, in-your-own-words explanation that captures the point is a pass. No vocabulary nitpicking.
- **A surfaced misconception is the *best* outcome** — a wrong anchor caught here and corrected sticks harder than a fact you never mis-held. These land in your notes as a "Misconceptions Corrected" list.
- **On by default, always skippable** — a bare color at a gate point advances without nagging, for the days you genuinely just know it.

Selective by design: it fires only where a false 🟢 is most expensive, never on every concept.

## Installation

Drop this folder into `~/.claude/skills/`:

```bash
git clone https://github.com/theo2612/calibrated-learning ~/.claude/skills/calibrated-learning
```

That's it. The skill will be auto-discovered by Claude Code.

## Usage

Trigger the skill by asking for a coaching session on any technical topic:

- "Walk me through the PaperCut RCE chain — coach mode."
- "I want to learn how Kerberos authentication actually works. Calibrated learning please."
- "Teach me ADCS exploitation step by step."
- "Deep dive on Rhino sandbox escapes — I'm a beginner."

The skill will:

1. **Baseline-check** prerequisite concepts with R/Y/G signals before diving in
2. **Pre-flight the jargon** — scan the planned content for acronyms, notation, and loaded vocabulary, define each, and present it up front so nothing appears undefined
3. **Walk through** the topic in a Command → Output → Analysis → Next loop with R/Y/G checkpoints between concepts
4. **Verify with a teach-back** at the load-bearing concepts (and after every 🔴 rebuild) — instead of a self-assessed 🟢, you explain it back in one line so a false 🟢 can't slip through and compound
5. **Recover** when you signal 🔴 — backing up to a foundational layer and rebuilding, capturing the gap as a study artifact
6. **Export** session notes (topic, walkthrough chain, R/Y/G transitions, gap map, glossary, corrected misconceptions) on request

## Why This Approach

Three observations driving the design:

**1. Three states is the right resolution.** Two states (got it / didn't) loses the "slow down a bit" middle. Five+ states require cognitive overhead to choose. Three is the minimum useful resolution and the maximum frictionless one.

**2. Color symbolism is universal.** 🟢 = go, 🔴 = stop is pre-cognitive. The learner doesn't have to translate.

**3. Gaps are artifacts, not failures.** Every 🔴 produces a gap-map entry. The learner ends a session with a concrete list of things to revisit, framed as productive output rather than a remediation backlog.

## Pairing With Other Tools

This skill plays well with:

- **`research`-style workflows** — when 🔴 traces a gap to something needing offline research, the recovery phase can suggest spawning a research subagent
- **Code-walking tools** — wrap their output in the R/Y/G loop for guided code explanations
- **Note-taking systems** (Obsidian, etc.) — the export phase writes markdown that drops cleanly into a vault

## Anti-Patterns

Don't use this skill for:

- Quick factual questions ("what's the syntax for X")
- Multi-source research
- Memorization drills
- Time-pressured walkthroughs where you just need the answer

## License

MIT
