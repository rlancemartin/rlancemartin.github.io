
---
layout: post
comments: false
title:  "Learning The Bitter Lesson"
excerpt: "The Bitter Lesson in AI application development."
date:   2025-08-01 
---

[Lance Martin](https://x.com/RLanceMartin)

### The Bitter Lesson in AI Research

> *The biggest lesson that can be read from 70 years of AI research is that general methods that leverage computation are ultimately the most effective, and by a large margin.
-* Rich Sutton [The Bitter Lesson](http://www.incompleteideas.net/IncIdeas/BitterLesson.html)
> 

[The Bitter Lesson](http://www.incompleteideas.net/IncIdeas/BitterLesson.html) has been learned over and over across [many domains](https://arxiv.org/html/2410.09649v1) of AI research, such as Chess, Go, speech, vision. Leveraging computation turns out to be the most important thing and the “structure” we impose on models often limits their ability to leverage growing computation.

What do we mean by "structure"? Often structure includes inductive biases about how we *expect* models to solve problems. Computer vision is a good example. For decades, researchers designed features (e.g., [SIFT](https://en.wikipedia.org/wiki/Scale-invariant_feature_transform) and [HOG](https://en.wikipedia.org/wiki/Histogram_of_oriented_gradients)) based upon domain knowledge. But these hand-crafted features restricted models to the patterns that we anticipated. As computation and data scaled, deep networks that learned features *directly* from pixels outperformed hand-crafted methods.

In a [talk](https://youtu.be/orDKvo8h71o?si=fsZesZuP25BU6SqZ) at Stanford, Hyung Won Chung (OpenAI) talks about his approach with this in mind:

> *Add structures needed for the given level of compute and data available. 
Remove them later, because these shortcuts will bottleneck further improvement.*
> 

### The Bitter Lesson in AI Engineering

I’ve found that the approach mentioned by Chung also applies to AI engineering, the craft of building applications on top of exponentially improving models. Below I’ll illustrate this with story about building [open-deep-research](https://github.com/langchain-ai/open_deep_research) over the past 9 months.

<figure>
<img src="/assets/bitter_lesson_timeline.png" width="90%">
<figcaption>
</figcaption>
</figure>

### Adding Structure

I had frustrating experiences with agents in 2023. It was hard to get LLMs to reliably call tools and context windows were small. In early 2024, I became interested in [workflows](https://www.anthropic.com/engineering/building-effective-agents). Rather than an LLM autonomously calling tools in a loop, workflows embed LLM calls in predefined code paths.

In late 2024, I [released](https://github.com/langchain-ai/open_deep_research/commit/50ce48753c5ff3d2e258af08dc69b7f4bcb62cf4) an [orchestrator-worker](https://langchain-ai.github.io/langgraph/tutorials/workflows/#orchestrator-worker) workflow for web research. The orchestrator was an LLM call that took a user’s request and returned a list of report sections to write. A set of workers parallelized research and writing all report sections. Then, I just combined them together.

So, what was the “structure”? I imposed a few specific assumptions about how LLMs should perform fast, reliable research: it should decompose the request into a set of report sections, research and write them in parallel to make it fast, and avoid tool calling to make it highly reliable.

<figure>
<img src="/assets/research_workflow.png" width="90%">
<figcaption>
</figcaption>
</figure>


### Bottlenecks

Things began to shift as 2024 came to a close. Tool calling [was getting better](https://www.anthropic.com/news/claude-3-7-sonnet). By winter 2025, [MCP](https://modelcontextprotocol.io/introduction) had gained significant momentum and it was clear that agents were [well](https://blog.google/products/gemini/google-gemini-deep-research/) [suited](https://openai.com/index/introducing-deep-research/) [to the research task](https://www.anthropic.com/engineering/built-multi-agent-research-system). But, the structure I imposed prevented me from leveraging these improvements. 

I did not use tool calling, so I could not take advantage of the growing set of MCP tools. The workflow always decomposed the request into sections, which was not appropriate for all cases. The reports also were sometimes disjoint because I forced workers to write sections in parallel.

### Removing Structure

I moved to an agent, which allowed me to use tools and iterative (deep) research. But I designed it such that each sub-agent *still* wrote its own report section. This ran into a problem that Walden Yan of [Cognition](https://cognition.ai/blog/dont-build-multi-agents) has called out: multi-agent systems are hard because the sub-agents don’t communicate effectively. Reports were *still* disjoint because agents wrote sections in parallel.

<figure>
<img src="/assets/multi_agent_v1.png" width="90%">
<figcaption>
</figcaption>
</figure>

This is one of the main points of Chung’s talk: we often [fail to remove](https://youtu.be/orDKvo8h71o?feature=shared&t=790) all the structure we add. In my case, I moved to an agent, but still was forcing each agent to write part of the report in parallel. I moved out the agents, into a final step. The system could now flexibly plan the research strategy and use [multi-agent context gathering](https://x.com/jxnlco/status/1945490018127987092), which has [benefits](https://www.anthropic.com/engineering/built-multi-agent-research-system) reported by Anthropic. It scores a 43.5 on [deep research bench](https://huggingface.co/spaces/Ayanami0730/DeepResearch-Leaderboard) (top 6), which is not bad for a small open source effort (and close to agents that use [RL](https://moonshotai.github.io/Kimi-Researcher/) or benefit from much larger-scale efforts).  

<figure>
<img src="/assets/multi_agent_final.png" width="90%">
<figcaption>
</figcaption>
</figure>

### Lessons

AI engineering can benefit from some simple lessons drawn from Chung’s talk:

1. **Understand your application structure**
2. **Re-evaluate your structure as models improve**
3. **Make it easy to remove structure**

On the first point, consider what LLM performance assumptions are baked into the design of your application. For my initial workflow, I avoided tool calling because (at the time) it was not reliable. This was no longer true a few months after I started the project! Jared Kaplan [co-founder of Anthropic](https://youtu.be/p8Jx4qvDoSo?si=giFZoWqhevhPn_qu) recently made the point that it can be beneficial to "build things that don't quite work" yet because the models will catch up (often quickly).

On the second point, I was a bit slow to re-evaluate my assumptions as tool calling improved. Even when I moved to multi-agent, I also carried over the bias to write sections in parallel. And, on the final point, I agree with [Walden Yan](https://cognition.ai/blog/dont-build-multi-agents) and [Harrison](https://blog.langchain.com/how-to-think-about-agent-frameworks/) that agent abstractions can pose risk because they can make it harder remove structure. I still use a framework (LangGraph) for its useful general features (e.g., checkpointing), but stick to its [low-level](https://langchain-ai.github.io/langgraph/concepts/low_level/#state) building blocks (e.g., nodes and edges) that I can easily (re-)configure.

The design philosophy for building AI applications is in its infancy. Still, as Hyung Won Chung said, it is helpful to focus on the driving force that we *can* predict: model will get much better. Designing AI applications to take advantage of this will probably be the most important thing.  

### Credits

Thanks to [Vadym Barda](https://x.com/vadymbarda) for initial evals, MCP support, and helpful discussion. Thanks to [Nick Huang](https://www.linkedin.com/in/ncchuang) for work on the multi-agent implementation as well as Deep Research Bench evals.