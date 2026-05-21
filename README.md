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

| Signal | Meaning |
|--------|---------|
| 🟢 **GREEN** | Tracking. Keep going at this depth and pace. |
| 🟡 **YELLOW** | Slow down. I need more "why" / context / fundamentals on the current concept. |
| 🔴 **RED** | Stop. I'm lost. Restart from first principles. |

You can signal in any natural form: `green`, `🟡`, `yellow on Rhino`, `🔴 lost on reflection`, `I'm red on this part`.

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
