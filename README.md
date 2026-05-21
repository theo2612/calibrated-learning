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
2. **Walk through** the topic in a Command → Output → Analysis → Next loop with R/Y/G checkpoints between concepts
3. **Recover** when you signal 🔴 — backing up to a foundational layer and rebuilding, capturing the gap as a study artifact
4. **Export** session notes (topic, walkthrough chain, R/Y/G transitions, gap map) on request

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
