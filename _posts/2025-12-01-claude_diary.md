---
layout: post
comments: false
title:  "Claude Diary"
excerpt: "Creating a memory system for Claude Code."
date:   2025-12-01 
---

[Lance Martin](https://x.com/RLanceMartin)

Humans refine their skills and learn preferences through experience. But many AI agents lack this capacity for ["continual learning"](https://www.dwarkesh.com/p/timelines-june-2025). For example, Claude Code relies on users to manually update its [CLAUDE.md memory files](https://code.claude.com/docs/en/memory). I created a [plugin](https://code.claude.com/docs/en/plugins) called [Claude Diary](https://github.com/rlancemartin/claude-diary) that gives Claude Code the ability to learn from experience and update its own memory. You can check out the code [here](https://github.com/rlancemartin/claude-diary).

<figure>
<img src="/assets/claude_diary.png" width="90%">
<figcaption>
</figcaption>
</figure>

## Ways To Think About Agent Memory

It's useful to first define what "experiences" and "memory" might mean for agents. The [CoALA paper](https://arxiv.org/pdf/2309.02427) by Sumers et al. (2023) proposes a framework separating "procedural memory" (e.g., instructions in the system prompt) from "episodic memory" (e.g., experiences like past decisions and actions). 

Claude Code stores instructions in `CLAUDE.md` files. And full Claude Code session logs with decisions and actions are saved to `~/.claude/projects/` in JSONL format. But, how do we transform specific past decisions and actions into persistent, general rules that can be added to instructions? 

The [Generative Agents paper](https://arxiv.org/pdf/2304.03442) by Park et al. (2023) shows one approach. Their agents use a reflection step to synthesize past actions into general rules that inform future planning and decisions. 

<figure>
<img src="/assets/gen_agents.png" width="90%">
<figcaption>
</figcaption>
</figure>

In a [recent interview](https://www.youtube.com/watch?v=IDSAMqip6ms&t=352s), Cat Wu (product lead on Claude Code) mentioned some Anthropic staff have adopted a similar pattern with Claude Code: create diary entries from Claude Code sessions that summarize key decisions and actions. Then perform reflection over the diary entries to identify patterns. 

## Implementing Memory For Claude Code

I used this reflection-based approach to implement a memory system for Claude Code.

<figure>
<img src="/assets/claude_diary_flow.png" width="90%">
<figcaption>
</figcaption>
</figure>

#### What to use to create diary entries?

I initially had Claude Code parse JSONL session logs, but this required dozens of bash tool calls. I decided to generate diary entries using the context already loaded in a given Claude Code session. 

#### What to capture in diary entries?

I created a `/diary` [slash command](https://code.claude.com/docs/en/slash-commands) that prompts Claude Code to capture key session details like what was accomplished, design decisions, challenges, user preferences, and PR feedback. Entries save to `~/.claude/memory/diary/YYYY-MM-DD-session-N.md`.

#### When to create diary entries?

I use a hybrid approach to create diary entries: manual `/diary` invocation and / or automatic invocation via the [PreCompact hook](https://code.claude.com/docs/en/hooks-guide#hook-events-overview). This allows me to choose when to create diary entries, but will automatically generate entries for longer sessions that use compaction.
 
#### What to capture in reflections?

The `/reflect` command instructs Claude Code to analyze diary entries and generate CLAUDE.md updates. It reads the CLAUDE.md file, checks for rule violations in the diary entries, and strengthens weak rules. It also looks across diary entries to identify recurring patterns. 

Since `CLAUDE.md` loads into every session, updates proposed by reflection are formatted as one-line bullets. The reflection process saves analysis to `~/.claude/memory/reflections/YYYY-MM-reflection-N.md` and automatically updates CLAUDE.md with synthesized rules.

#### How to track processed entries?

A `processed.log` file at `~/.claude/memory/reflections/processed.log` prevents duplicate analysis of diary entries. The format is `[diary-entry] | [reflection-date] | [reflection-file]`. The reflection command checks this log first.

#### When to perform reflection?

I kept reflection manual because it updates CLAUDE.md directly. I wanted to review the proposed updates before writing them to the CLAUDE.md file.

#### What memory files to update?

I only have Claude Code update its user-level file `~/.claude/CLAUDE.md` because many patterns captured in diary entries (commit style, testing, code quality) apply universally. 

## Examples

Here are some examples areas that I've found Claude Diary to be helpful: 

**PR review feedback**: I found that PR comments (which can be loaded via Claude Code's `pr-comments` command) are a great source of feedback to update Claude Code's memory.  

**Git workflow**: I found that this system was good a picking up revealed preferences in my git workflow and adding them to Claude Code's memory. 

**Self-correction**: I found that this system was good at detecting when existing rules needed reinforcement, especially when overriding some of Claude Code's default behavior. 

## Conclusion

Claude Diary is just one simple attempt to convert raw Claude Sessions into long-term memory updates. The commands are just prompts, making them trivial to modify. I also limit automation, but it is trivial to further automate any of the commands using hooks. There is considerable room for improvement as noted [here](https://github.com/rlancemartin/claude-diary?tab=readme-ov-file#future-work). The code is available at [here](https://github.com/rlancemartin/claude-diary). 
