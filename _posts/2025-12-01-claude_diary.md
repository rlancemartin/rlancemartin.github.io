---
layout: post
comments: false
title:  "Claude Diary"
excerpt: "Creating a memory system for Claude Code."
date:   2025-12-01 
---

[Lance Martin](https://x.com/RLanceMartin)

Humans refine their skills and learn preferences through experience. But many AI agents lack this capacity for [continual learning](https://www.dwarkesh.com/p/timelines-june-2025). I created a [plugin](https://code.claude.com/docs/en/plugins) called [Claude Diary](https://github.com/rlancemartin/claude-diary) that gives Claude Code the ability to learn from experience and update its own memory. You can check out the code [here](https://github.com/rlancemartin/claude-diary).

<figure>
<img src="/assets/claude_diary.png" width="90%">
<figcaption>
</figcaption>
</figure>

## Agent Memory

The [CoALA paper](https://arxiv.org/pdf/2309.02427) by Sumers et al. (2023) proposes a framework for agent memory, including  "procedural memory" (e.g., prompt instructions) and "episodic memory" (e.g., past actions). 

Claude Code stores its system instructions in `CLAUDE.md` files and session logs are saved to `~/.claude/projects/`. But, how can we transform past actions from logs into persistent, general rules that can be added to instructions? 

The [Generative Agents paper](https://arxiv.org/pdf/2304.03442) by Park et al. (2023) shows one approach. Their agents use a reflection step to synthesize past actions into general rules that inform future planning and decisions. 

<figure>
<img src="/assets/gen_agents.png" width="90%">
<figcaption>
</figcaption>
</figure>

In a [recent interview](https://www.youtube.com/watch?v=IDSAMqip6ms&t=352s), Cat Wu (product lead on Claude Code) mentioned some Anthropic staff use a similar pattern with Claude Code: create diary entries from Claude Code sessions and reflect on them to identify patterns. 

## Implementing Claude Diary

I used this reflection-based approach with Claude Code, asking Claude to distill diary entries from sessions and performing reflection over collected entries to update `CLAUDE.md`.

<figure>
<img src="/assets/claude_diary_flow.png" width="90%">
<figcaption>
</figcaption>
</figure>

#### What to use to create diary entries?

I initially had Claude Code parse JSONL session logs, but this required dozens of bash tool calls. I decided to generate diary entries using the context already loaded in a given Claude Code session. 

#### What to capture in diary entries?

I created a `/diary` [slash command](https://code.claude.com/docs/en/slash-commands) that prompts Claude Code to capture key session details like what was accomplished, design decisions, challenges, user preferences, and PR feedback. Diary entries are saved to: 

`~/.claude/memory/diary/YYYY-MM-DD-session-N.md`

#### When to create diary entries?

I use a hybrid approach to create diary entries: manual `/diary` invocation and / or automatic invocation via the [PreCompact hook](https://code.claude.com/docs/en/hooks-guide#hook-events-overview). This allows me to choose when to create diary entries, but will automatically generate entries for longer sessions that use compaction.
 
#### What to capture in reflections?

The `/reflect` command instructs Claude Code to analyze diary entries and generate CLAUDE.md updates. It reads the CLAUDE.md file, checks for rule violations in the diary entries, and strengthens weak rules. It also looks across diary entries to identify recurring patterns. 

Since `CLAUDE.md` loads into every session, updates proposed by reflection are formatted as one-line bullets. The reflection process saves analysis to and automatically updates CLAUDE.md with synthesized rules. Reflections are saved to:

`~/.claude/memory/reflections/YYYY-MM-reflection-N.md`

#### How to track processed entries?

A `processed.log` file at prevents duplicate analysis of diary entries. The reflection command checks this log first. The log is saved to:

`~/.claude/memory/reflections/processed.log`

#### When to perform reflection?

I kept reflection manual because it updates CLAUDE.md directly. I wanted to review the proposed updates before writing them to the CLAUDE.md file.

#### What memory files to update?

I only have Claude Code update its user-level file `~/.claude/CLAUDE.md` because many patterns captured in diary entries (commit style, testing, code quality) apply universally. 

## Examples

I'ved used Claude Diary for the past month. I just run the `diary` command in sessions that I want to capture. Then I run `reflect` periodically to update my `CLAUDE.md`. Here are some examples where I've found Claude Diary to be helpful:

**PR review feedback**: PR comments (which can be loaded via Claude Code's `pr-comments` command) are a great source of feedback to update Claude Code's memory. 

**Git workflow**: The system excels at capturing revealed preferences in git workflow - from atomic commits and branch naming conventions to commit message formatting.

**Testing practices**: Reflection identified patterns like running targeted tests first for quick feedback, then comprehensive suites, and using specialized test libraries.

**Code quality**: The system learned to avoid anti-patterns like naming conflicts between files and package directories, leaving stale directories after refactoring, and unnecessarily verbose code.

**Agent design**: For AI agent work, reflection captured preferences around token efficiency, biasing toward single-agent delegation over premature parallelization, and using filesystem for context offloading.

**Self-correction**: Sometimes rules in CLAUDE.md need reinforcement; the system was great at finding cases where Claude did was not following instructions and reinforcing them. 

## Conclusion

Claude Diary is just a simple attempt to convert raw Claude Sessions into memory updates in `CLAUDE.md`. The commands are just prompts, making them easy to modify. I also limit automation, but it is easy to further automate any of the commands using hooks. There is considerable room for improvement as noted [here](https://github.com/rlancemartin/claude-diary?tab=readme-ov-file#future-work). The code is available as a Claude Code plugin [here](https://github.com/rlancemartin/claude-diary). 
