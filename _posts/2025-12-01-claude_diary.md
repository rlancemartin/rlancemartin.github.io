---
layout: post
comments: false
title:  "Giving Claude Code Memory"
excerpt: "Creating a memory system for Claude Code."
date:   2025-12-01 
---

[Lance Martin](https://x.com/RLanceMartin)

Humans refine their skills and learn preferences through experience. But most AI agents today lack this capacity for ["continual learning"](https://www.dwarkesh.com/p/timelines-june-2025). For example, Claude Code relies on users to manually update its [CLAUDE.md memory files](https://code.claude.com/docs/en/memory). I've been experimenting with ways to give Claude Code the ability to learn from experience and update its own memory. I created a sharable [plugin](https://code.claude.com/docs/en/plugins) called [Claude Diary](https://github.com/rlancemartin/claude-diary) that does this.

<figure>
<img src="/assets/claude_diary.png" width="90%">
<figcaption>
</figcaption>
</figure>

## Ways To Think About Agent Memory

To enable this learning, it's useful to first define what experiences and memory mean for agents. The [CoALA paper](https://arxiv.org/pdf/2309.02427) by Sumers et al. (2023) proposes a framework separating "procedural memory" (instructions in the system prompt) from "episodic memory" (past decisions and actions). 

Claude Code stores procedural memory in `CLAUDE.md` files (always loaded into context) and logs all sessions to `~/.claude/projects/` in JSONL format. The core challenge is episodic-to-procedural conversion: how do we transform specific past decisions and actions into persistent, general rules that guide future behavior?

The [Generative Agents paper](https://arxiv.org/pdf/2304.03442) by Park et al. (2023) demonstrates one approach. Their agents maintain a memory stream of observations and periodically synthesize these into higher-level reflections. The mechanism is straightforward: when the cumulative importance of recent observations exceeds a threshold, the agent generates abstract thoughts that generalize across experiences. These reflections then inform future planning and decisions.
 
The key insight: reflection converts granular episodic memories (what happened) into abstract procedural knowledge (how to behave). In a [recent interview](https://www.youtube.com/watch?v=IDSAMqip6ms&t=352s), Cat Wu (product lead on Claude Code) mentioned that some Anthropic staff have adopted this pattern—creating diary entries from sessions, then using reflection to identify patterns that warrant updating `CLAUDE.md`. 

<figure>
<img src="/assets/gen_agents.png" width="90%">
<figcaption>
</figcaption>
</figure>

## Implementing Memory For Claude Code

I decided to implement a memory system for Claude Code that follows this reflection-based approach, which required answering a few questions:

1. **What source to use?** (session logs vs. loaded context)
2. **What to capture in diary entries?** (decisions, preferences, patterns)
3. **When to create diary entries?** (manual, automatic, hooks)
4. **What patterns to extract in reflections?** (signal vs. noise filtering)
5. **How to track processed diary entries?** (avoiding duplicates)
6. **Which memory files to update?** (user-level vs. project-level) 

#### What to use to create diary entries?

I initially tried parsing JSONL session logs with `jq`, but this required dozens of bash tool invocations, complex escaping, and error-prone path hashing. Instead, I generate diary entries in-session from the conversation context already loaded in Claude Code. The trade-off: diary generation must happen during the session, not afterward. 

#### What to capture in diary entries?

I created a `/diary` [slash command](https://code.claude.com/docs/en/slash-commands) that prompts Claude to capture: what was accomplished, design decisions, challenges and solutions, user preferences (commit style, testing, code quality), and PR feedback. Entries save to `~/.claude/memory/diary/YYYY-MM-DD-session-N.md`.

#### When to create diary entries?

I use a hybrid approach: manual `/diary` invocation for meaningful sessions, plus automatic generation via the [PreCompact hook](https://code.claude.com/docs/en/hooks-guide#hook-events-overview) when conversations trigger compaction. This avoids cluttering the diary with trivial sessions while capturing important ones automatically.
 
#### What to capture in reflections?

The `/reflect` command analyzes unprocessed diary entries and synthesizes patterns into CLAUDE.md updates. Three design choices:

**First, pattern detection over one-offs**. Look for recurring patterns: 2+ occurrences suggests emergence, 3+ confirms strength. The threshold isn't rigid—a single critical preference (e.g., "never force push to main") can still be captured if clearly important. 

**Second, self-correction over accumulation**. Memory systems need to detect when existing rules aren't working, not just add new ones. After three consecutive sessions where Claude violated an existing rule, I added violation detection: the reflection command reads CLAUDE.md first, checks for violations, and prioritizes strengthening weak rules (top placement, "ZERO TOLERANCE" language, explicit overrides) before adding new ones.

**Third, terse formatting for token efficiency**. Since CLAUDE.md loads into every session, every token matters. Strict format: one-line bullets, imperative tone, no explanations. "git commits: use conventional format" not verbose rationale. Detailed reasoning stays in reflection documents, not operational memory.

The reflection process saves analysis to `~/.claude/memory/reflections/YYYY-MM-reflection-N.md`, automatically updates CLAUDE.md with synthesized rules, and logs processed entries to prevent duplicate analysis.

#### How to track processed entries?

A `processed.log` file at `~/.claude/memory/reflections/processed.log` prevents duplicate analysis. Format: `[diary-entry] | [reflection-date] | [reflection-file]`. The reflection command checks this log first and skips already-processed entries.

#### When to perform reflection?

I kept reflection manual. Unlike diary creation (low-risk observation, automated via PreCompact), reflection updates CLAUDE.md and shapes all future behavior—high-impact operations warrant user awareness. The asymmetry is intentional: automate observation, gate synthesis.

#### What memory files to update?

For now, just user-level `~/.claude/CLAUDE.md`. Most patterns (commit style, testing, code quality) apply universally. Claude Code supports [multiple memory tiers](https://code.claude.com/docs/en/memory)—project-level memory for context-appropriate rules is future work.

<figure>
<img src="/assets/claude_diary_flow.png" width="90%">
<figcaption>
</figcaption>
</figure>

## Examples

**PR review feedback**: PR comments (via Claude Code's `pr-comments` command) are rich memory sources. After Claude used excessive exception handling, reflection generated "avoid verbose exception handling; match existing code patterns"—correctly prioritizing one high-impact piece of feedback over more frequent but less important patterns.

**Git workflow**: Reflection synthesized patterns appearing 2-3 times into rules: isolated PR branches for review, local integration branches for testing, conventional commits (feat:, fix:), specific naming (rlm/feature-description).

**Self-correction**: Three consecutive entries documented the same violation—Claude adding attribution to commits despite an existing rule. Reflection detected this, strengthened the rule (top placement, "ZERO TOLERANCE," explicit system prompt override). The system wasn't just learning new things, but detecting when existing rules needed reinforcement. 

## Conclusion

Claude Diary is just one simple attempt to convert raw Claude Sessions into long-term memory updates. 

If you try it, run `/diary` after a substantive session, then `/reflect` after a few diary entries accumulate. Watch your `CLAUDE.md` evolve. 

The code is available at [github.com/rlancemartin/claude-diary](https://github.com/rlancemartin/claude-diary). 
