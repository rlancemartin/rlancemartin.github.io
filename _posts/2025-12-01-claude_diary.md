---
layout: post
comments: false
title:  "Claude Diary"
excerpt: "Creating a memory system for Claude Code."
date:   2025-12-01 
---

[Lance Martin](https://x.com/RLanceMartin)

Humans refine their skills and learn preferences through experience. But most AI agents lack this capacity for ["continual learning"](https://www.dwarkesh.com/p/timelines-june-2025). For example, Claude Code relies on users to manually update its [CLAUDE.md memory files](https://code.claude.com/docs/en/memory). I created a [plugin](https://code.claude.com/docs/en/plugins) called [Claude Diary](https://github.com/rlancemartin/claude-diary) that gives Claude Code the ability to learn from experience and update its own memory.

<figure>
<img src="/assets/claude_diary.png" width="90%">
<figcaption>
</figcaption>
</figure>

## Ways To Think About Agent Memory

It's useful to first define what experiences and memory mean for agents. The [CoALA paper](https://arxiv.org/pdf/2309.02427) by Sumers et al. (2023) proposes a framework separating "procedural memory" (e.g., instructions in the system prompt) from "episodic memory" (e.g., experiences like past decisions and actions). 

Claude Code stores its procedural memory in `CLAUDE.md` files, which are always loaded into context when a session starts. A history of Claude Code's decisions and actions (e.g., tool call results) persists as messages in context during sessions. Full session histories are also logged to `~/.claude/projects/` in JSONL format. 

The core challenge is episodic-to-procedural conversion: how do we transform specific past decisions and actions into persistent, general rules that guide future behavior?

The [Generative Agents paper](https://arxiv.org/pdf/2304.03442) by Park et al. (2023) shows one approach. Their agents maintain a memory stream of observations, like our Claude Code sessions logs. They periodically synthesize these into higher-level reflections that generalize across experiences. The reflections then inform future planning and decisions. Reflection converts granular episodic memories (what happened) into abstract procedural knowledge (how to behave). 

<figure>
<img src="/assets/gen_agents.png" width="90%">
<figcaption>
</figcaption>
</figure>

In a [recent interview](https://www.youtube.com/watch?v=IDSAMqip6ms&t=352s), I thought it was very interesting that Cat Wu (product lead on Claude Code) mentioned some Anthropic staff have adopted a similar pattern with Claude Code: create diary entries from Claude Code sessions that summarize key decisions and actions. Then perform reflection over the diary entries to identify patterns that warrant updating long-term memory in `CLAUDE.md`. 

## Implementing Memory For Claude Code

I decided to implement a memory system for Claude Code that follows this reflection-based approach, which required answering a few questions:

1. **What source to use?** (session logs vs. loaded context)
2. **What to capture in diary entries?** (decisions, preferences, patterns)
3. **When to create diary entries?** (manual, automatic, hooks)
4. **What patterns to extract in reflections?** (signal vs. noise filtering)
5. **How to track processed diary entries?** (avoiding duplicates)
6. **Which memory files to update?** (user-level vs. project-level) 

#### What to use to create diary entries?

I initially had Claude Code parse JSONL session logs (e.g., with `jq`), but this required dozens of error-prone bash tool invocations. Instead, I generate diary entries in-session from the conversation context already loaded in Claude Code. The trade-off is that diary generation must happen during the Claude Code session, not afterward. 

#### What to capture in diary entries?

I created a `/diary` [slash command](https://code.claude.com/docs/en/slash-commands) that prompts Claude Code to capture key session details like what was accomplished, design decisions, challenges and solutions, user preferences (commit style, testing, code quality), and PR feedback. Entries save to `~/.claude/memory/diary/YYYY-MM-DD-session-N.md`.

#### When to create diary entries?

I use a hybrid approach to create diary entries: manual `/diary` invocation for meaningful sessions, plus automatic generation via the [PreCompact hook](https://code.claude.com/docs/en/hooks-guide#hook-events-overview) when conversations trigger compaction. This avoids cluttering the diary logs with trivial sessions, allows me to choose when to create diary entries, and will automatically generate entries for longer running sessions that use compaction.
 
#### What to capture in reflections?

The `/reflect` command instructs Claude Code to analyze unprocessed diary entries and synthesizes patterns into CLAUDE.md updates. It first reads the CLAUDE.md file, checks for violations to rules in the diary entries, and prioritizes strengthening weak rules before adding new ones. It also looks across diary entries to identify recurring patterns. Since `CLAUDE.md` loads into every session, updates proposed by reflection are formatted as one-line bullets with an imperative tone and no explanations. The reflection process saves analysis to `~/.claude/memory/reflections/YYYY-MM-reflection-N.md`, automatically updates CLAUDE.md with synthesized rules, and logs processed entries to prevent duplicate analysis.

#### How to track processed entries?

A `processed.log` file at `~/.claude/memory/reflections/processed.log` prevents duplicate analysis. The format is `[diary-entry] | [reflection-date] | [reflection-file]`. The reflection command checks this log first and skips already-processed entries to avoid duplicate analysis.

#### When to perform reflection?

I kept reflection manual because it's a high-impact operation that can shape all future behavior. Unlike diary creation, reflection updates CLAUDE.md directly. 

#### What memory files to update?

For now, I only have Claude Code update its user-level file `~/.claude/CLAUDE.md`. Most patterns (commit style, testing, code quality) apply universally. But, I leave supporting Claude Code updates across [multiple memory tiers](https://code.claude.com/docs/en/memory) as future work.

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

Claude Diary is just one simple attempt to convert raw Claude Sessions into long-term memory updates. The decisions summarized above capture a first attempt at converting episodic memories into procedural knowledge, but there is considerable room for improvement as noted [here](https://github.com/rlancemartin/claude-diary?tab=readme-ov-file#future-work). The code is available at [here](https://github.com/rlancemartin/claude-diary). 
