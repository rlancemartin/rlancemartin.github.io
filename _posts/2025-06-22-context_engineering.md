---
layout: post
comments: false
title:  "Context Engineering for Agents"
excerpt: "Patterns for managing the context needed for agents to perform their tasks."
date:   2025-06-24 
---

[Lance Martin](https://x.com/RLanceMartin)

### Context Engineering

As Andrej Karpathy puts it, LLMs are like a [new kind of operating system](https://www.youtube.com/watch?si=-aKY-x57ILAmWTdw&t=620&v=LCEmiRjPEtQ&feature=youtu.be). The LLM is like the CPU and its [context window](https://docs.anthropic.com/en/docs/build-with-claude/context-windows) is like RAM, serving as the model’s working memory. 

Context enters an LLM in several ways, including prompting (user, system instructions), retrieval (documents or files), and tool calls (calculators, APIs).

Just like RAM, the LLM context window has limited [“communication bandwidth](https://lilianweng.github.io/posts/2023-06-23-agent/)” to handle these various sources of context.

And just as an operating system curates what fits into a CPU’s RAM, we can think about “context engineering” as designing, packaging, and managing the context needed for an LLM [to solve a task](https://x.com/tobi/status/1935533422589399127).

<figure>
<img src="/assets/context_types.png" width="90%">
<figcaption>
</figcaption>
</figure>

### Phases of Context Engineering

Chatbots were the first mainstream LLM application. We can consider [prompt engineering](https://www.promptingguide.ai/) as the first phase of context engineering to steer the behavior of LLMs.

As interest grew in connecting LLMs to external datasources, [retrieval augmented generation](https://github.com/langchain-ai/rag-from-scratch) (RAG) marked the second phase of context engineering.

As LLMs get better at tool calling, [agents](https://www.anthropic.com/engineering/building-effective-agents) are becoming feasible. Agents interleave [LLM calls and tool calls](https://www.anthropic.com/engineering/building-effective-agents) for [long-running tasks](https://blog.langchain.com/introducing-ambient-agents/). Agents motivate what we might consider the third phase of context engineering. 

<figure>
<img src="/assets/agent_flow.png" width="90%">
<figcaption>
Agents interleave [LLM call and tool calls](https://www.anthropic.com/engineering/building-effective-agents), using tool feedback to decide the next step.
</figcaption>
</figure>

[Cognition](https://cognition.ai/blog/dont-build-multi-agents) called out the importance of context engineering with agents:

> *“Context engineering” … is effectively the #1 job of engineers building AI agents.*
> 

[Anthropic](https://www.anthropic.com/engineering/built-multi-agent-research-system) also laid it out clearly:

> *Agents often engage in conversations spanning hundreds of turns, requiring careful context management strategies.*
> 

This post is my attempt to break down some common strategies — **curate**, **persist**, and **isolate —** for agent context engineering.

### Context Engineering for Agents

The agent context is populated by feedback from tool calls. Accumulating feedback from tool calls presents a challenge for building agents. It can [exceed the size of the context window](https://cognition.ai/blog/kevin-32b), and balloon the cost and latency. 

<figure>
<img src="/assets/tool_context.png" width="90%">
<figcaption>
Context from tool calls accumulates over multiple agent turns.
</figcaption>
</figure>

I’ve been bitten by this many times. One incarnation of a [deep research agent](https://github.com/langchain-ai/open_deep_research) that I built had token-heavy tool calls, resulting in > 500k token and several dollars per run! 

Agent performance may also degrade as the context grows. [Google Research](https://research.google/blog/chain-of-agents-large-language-models-collaborating-on-long-context-tasks/#:~:text=example%2C%20Gemini%20is%20able%20to,architecture%20that%20underlies%20most%20LLMs) and [Percy Liang’s group](https://arxiv.org/abs/2307.03172) have described different flavors of “[context degradation syndrome](https://jameshoward.us/2024/11/26/context-degradation-syndrome-when-large-language-models-lose-the-plot)” since a long context can limit an LLMs ability to recall facts or follow task instructions clearly.  

Looking at many popular agents and reflecting on my own experience, I group agent context engineering into 3 buckets: curating, persisting, and isolating context. 

<figure>
<img src="/assets/context_eng_overview.png" width="90%">
<figcaption>
</figcaption>
</figure>

### Curating Context

Curating context involves managing the tokens that the agent sees at each turn.

**Context Summarization**

Agent interactions can span [hundreds of turns](https://www.anthropic.com/engineering/built-multi-agent-research-system) and may have token-heavy tool calls. Summarization is one way to manage this. 

If you’ve used Claude Code, you’ve seen this in action. Claude Code runs “[auto-compact](https://docs.anthropic.com/en/docs/claude-code/costs)” after you exceed 95% of the context window.

Summarization can be used in different places. You can apply to the agent trajectory with many [popular](https://python.langchain.com/api_reference/langchain/memory/langchain.memory.summary.ConversationSummaryMemory.html) tools and methods (e.g., [recursive](https://arxiv.org/pdf/2308.15022#:~:text=the%20retrieved%20utterances%20capture%20the,based%203) or [hierarchical](https://alignment.anthropic.com/2025/summarization-for-monitoring/#:~:text=We%20addressed%20these%20issues%20by,of%20our%20computer%20use%20capability) summarization). 

<figure>
<img src="/assets/context_curation.png" width="90%">
<figcaption>
</figcaption>
</figure>

I’ve also found it useful to [add summarization](https://github.com/langchain-ai/open_deep_research/blob/e5a5160a398a3699857d00d8569cb7fd0ac48a4f/src/open_deep_research/utils.py#L1407) to post-process tool calls (e.g., a token-heavy search tool). Similarly, [Anthropic’s multi-agent researcher](https://www.anthropic.com/engineering/built-multi-agent-research-system) uses summarization applied to completed work phases.

[Cognition](https://cognition.ai/blog/dont-build-multi-agents#a-theory-of-building-long-running-agents) called out that this can be tricky if you need specific events or decisions from agent trajectories. They use a fine-tuned model for this in Devin, which underscores how much work can go into refining this step. 

### Persisting Context

Persisting context involves systems to store, save, and retrieve context over time. 

**Storing context**

Files are a simple way to store context. We see this used with many popular agents: Claude Code uses [`CLAUDE.md`](http://CLAUDE.md). [Cursor](https://docs.cursor.com/context/rules) and [Windsurf](https://windsurf.com/editor/directory) use rules files, and some plugins (e.g., [Cursor Memory Bank](https://forum.cursor.com/t/managing-chat-context-in-cursor-ide-for-large-repositories-what-s-working-for-you/76391/2)) / [MCP servers](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem) help manage collections of memory files.

Some agents need to store information that can’t be easily be captured in a small set of files. For example, we may want to store large [collections](https://langchain-ai.github.io/langgraph/concepts/memory/#collection) of facts and / or relationships. A few libraries emerged to support this and showcase some common patterns. 

[Letta](https://docs.letta.com/concepts/memgpt), [Mem0](https://mem0.ai/research), and [LangGraph](https://langchain-ai.github.io/langgraph/concepts/memory/#long-term-memory) / [Mem](https://langchain-ai.github.io/langmem/) store collections of embedded documents. [Zep](https://arxiv.org/html/2501.13956v1#:~:text=In%20Zep%2C%20memory%20is%20powered,subgraph%2C%20and%20a%20community%20subgraph) and [Neo4J](https://neo4j.com/blog/developer/graphiti-knowledge-graph-memory/#:~:text=changes%20since%20updates%20can%20trigger,and%20holistic%20memory%20for%20agentic) use knowledge graphs for continuous indexing of facts or relationships.

**Saving context**

Claude Code entrusts the user to create / update memories (e.g., the `#` shortcut). But there are many cases where we want agents to autonomously create / update memories. 

The [Reflexion](https://arxiv.org/abs/2303.11366) paper introduced the idea of reflection following each agent turn and re-using these self-generated hints. [Generative Agents](https://ar5iv.labs.arxiv.org/html/2304.03442) created memories as higher-level summaries synthesized periodically from collections of past agent feedback.

These concepts made their way into popular products like [ChatGPT,](https://help.openai.com/en/articles/8590148-memory-faq) [Cursor](https://docs.cursor.com/context/rules#memories), and [Windsurf](https://docs.windsurf.com/windsurf/cascade/memories), which all have mechanisms to auto-generate memories based on user-agent interactions. 

<figure>
<img src="/assets/email_agent.png" width="90%">
<figcaption>
Agents can create or update memories based upon user feedback. 
</figcaption>
</figure>

Memory creation can also be done at specific points. One pattern I like: create / update memories based upon user feedback.

For example, human-in-the-loop review of tool calls is a good way to build confidence in your agent. But if you pair this with memory updating, then the agent can learn from your feedback over time! My [email assistant](https://github.com/langchain-ai/agents-from-scratch) does this with simple file based memory.

**Retrieving context**

The simplest approach is just to pull all memories into the agent’s context window. For example, Claude Code just reads all [`CLAUDE.md`](http://CLAUDE.md) files into context at the start of each session. In my [email assistant](https://github.com/langchain-ai/agents-from-scratch), I always load a set memories that provide email triage and response instructions into context.

But, mechanisms to fetch select memories are important if the collection is large. The store (e.g., embedding-based search or graph retrieval) will determine the approach. 

[In practice this can be a very deep topic](https://x.com/_mohansolo/status/1899630246862966837) because retrieval is tricky. [Generative Agents](https://ar5iv.labs.arxiv.org/html/2304.03442) scored memories not just on similarity, but also on recency and importance.

The UX of selective memory retrieval is also tricky: [Simon Willison has shared](https://simonwillison.net/2025/Jun/6/six-months-in-llms/) an example of memory retrieval gone wrong. GPT-4o injected location into an image based upon his memories, which was not desired. Users can feel that the context window “no longer belongs to them” if this is not done right!

### Isolating Context

Isolating context involves approaches to partition it across agents or environments.   

**Context Schema**

Oftentimes, [messages](https://python.langchain.com/docs/concepts/messages/) are used to structure agent context. The message list is just passed to the LLM at each agent turn. 

The problem is that a list can get bloated with token-heavy tool calls. A structured runtime state - defined via a [schema](https://langchain-ai.github.io/langgraph/concepts/low_level/#schema) (e.g., a [Pydantic](https://docs.pydantic.dev/latest/concepts/models/) model) - can often be more effective. 

Then, you can control what fields are passed to the LLM at each agent turn. For example, in one version of a deep research agent, I [save token-heavy completed sections](https://github.com/langchain-ai/open_deep_research/blob/e5a5160a398a3699857d00d8569cb7fd0ac48a4f/src/open_deep_research/multi_agent.py#L428) in one field of my schema, isolated from the LLM. When all sections are done, I fetch them from state and pass to the LLM for final writing.  

**Multi-agent**

One popular approach is to split context across sub-agents. A motivation for the OpenAI [Swarm](https://github.com/openai/swarm) library was “[separation of concerns](https://openai.github.io/openai-agents-python/ref/agent/)”, where a team of agents can handle sub-tasks and each agent has its own instructions and context window. 

<figure>
<img src="/assets/multi_agent.png" width="90%">
<figcaption>
</figcaption>
</figure>

Anthropic’s [multi-agent researcher](https://www.anthropic.com/engineering/built-multi-agent-research-system) makes a clear case for this: many agents with isolated context outperformed single-agent by 90.2%, with token usage by explaining 80% of the variance. As the blog said: 

> *[Subagents operate] in parallel with their own context windows, exploring different aspects of the question simultaneously.*
> 

The problems with multi-agent include token use (e.g., [15× more tokens](https://www.anthropic.com/engineering/built-multi-agent-research-system) than chat), the need for careful [prompt engineering](https://www.anthropic.com/engineering/built-multi-agent-research-system) / [context](https://cognition.ai/blog/dont-build-multi-agents) to plan sub-agent work, and coordination of sub-agents. Cognition argues [against](https://cognition.ai/blog/dont-build-multi-agents) multi-agent for these reasons.

I’ve also been bitten by this: one iteration of my deep research agent had a team of agents write sections of the final report. Sometimes the final report was disjointed because the agents did not coordinate or communicate with one another while writing.

One way to reconcile this is to ensure the task is parallelizable. A subtle point is that Anthropic’s deep research multi-agent system applied parallelization to *research.* This is easier than *writing*, which requires cohesion across the sections of a report.

**Context Isolation with Environments**

I thought HuggingFace’s [deep researcher](https://huggingface.co/blog/open-deep-research#:~:text=From%20building%20,it%20can%20still%20use%20it) is an interesting example context isolation. Most agents use [tool calling APIs](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview), which return JSON objects (tool arguments) that can be passed to tools (e.g., a search API) to get tool feedback (e.g., search results).

<figure>
<img src="/assets/isolation.png" width="90%">
<figcaption>
</figcaption>
</figure>

HuggingFace uses a [CodeAgent](https://huggingface.co/papers/2402.01030), which outputs code to invoke tools. The code then runs in a [sandbox](https://e2b.dev/). Selected context (e.g., stdout or return values) from code execution is then passed back to the LLM. 

> *[Code Agents allow for] a better handling of state … Need to store this image / audio / other for later use? No problem, just assign it as a variable in your state and you [use it later].*
> 

The sandbox stores objects generated during execution (e.g., images, docs), isolating them from the LLM context window. But the agent can still reference these objects with variables at each turn, as needed.

### Lessons

General principles for building agents are still in their infancy. Also, models are changing fast and the [Bitter Lesson](http://www.incompleteideas.net/IncIdeas/BitterLesson.html) warns us to avoid scaffolding that will become irrelevant as new model capabilities are released. For example, [continual learning](https://www.dwarkesh.com/p/timelines-june-2025) may let LLMs to [learn from feedback](https://www.wired.com/story/this-ai-model-never-stops-learning/?utm_source=chatgpt.com), limiting some of the need for external memory. With this and the patterns above in mine, I tried to distill a few general lessons and order them roughly by the amount of effort required. 

- **Instrument first:** Always [look at your data](https://hamel.dev/blog/posts/evals/)! Ensure you have a way to track token-usage and cost when building agents. This has allowed me to catch various cases of excessive token-usage and isolate token-heavy tool calls.
- **Define an agent state schema to isolate context:** Anthropic has called out the idea of “[thinking like your agent](https://www.youtube.com/watch?v=D7_ipDqhtwk).” One way to do this is to think through the information your agent needs to collect and use during runtime. A state object is easy to define and lets you control what is exposed to the LLM at what time during the agent trajectory. I use this with nearly every agent I build, rather than just blindly saving everything to a single message list.
- **Curate at tool boundaries:** Tools boundary are a natural place to add curation, if needed. The output of token-heavy tool calls can be easily summarized, for example, using a small LLM with basic prompting. This lets you quickly limit runaway context growth at the source without the need for more compression over the full agent trajectory.
- **Start simple with memory**: Memory can be a really powerful way to personalize an agent. But, it can be challenging to get right. I [often use](https://github.com/langchain-ai/agents-from-scratch) simple, file-based memory that tracks a narrow set of agent preferences that I want to save and improve. I load these preferences into context every time my agent runs. Based upon human-in-the-loop feedback, I use an LLM to update these preferences. This is a basic but effective way to use memory. Ramping up to higher complexity (e.g., search over large collections of memories) can be done, but requires careful tuning.
- **Consider multi-agent for easily parallelizable problems**: Agent-agent communication is still early. It’s hard to coordinate multi-agent teams. But that doesn’t mean you should abandon the idea of multi-agent. Instead, consider multi-agent in cases where the problem can be easily parallelized and tight coordination between sub-agents is not strictly required.