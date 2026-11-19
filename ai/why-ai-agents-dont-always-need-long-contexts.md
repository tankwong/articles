# Why AI agents don't always need long contexts

*Sunday Deep Dive*

**Source:** AlphaSignal, 16 November 2026\
**Author:** Ben Dickson

> Ben Dickson is a veteran software engineer and former CTO known as the "Engineer's Journalist" for translating technical AI research into practical, production-focused insights. He is a lead contributor to TechCrunch and VentureBeat.

In agentic tasks, the context of the underlying LLM might grow into hundreds of thousands of tokens. The agent spends many steps searching repositories, modifying files, and running tests.

Each of these interactions becomes part of the next request to the model. The agent's context continues to grow, even when much of that history is no longer useful.

In a new study, researchers at Nvidia tested whether AI models really need those long contexts. Across different long-context tasks, they found that LLMs can often work with much shorter contexts while maintaining their accuracy.

For developers, the findings suggest that instead of stuffing everything into the model's context, only keep the minimal information the agent needs for the next step.

## Why long contexts are expensive

When an LLM generates a token, it must attend to all the tokens that precede it. Even when storing attention values in the KV cache, the attention tax increases as the context grows because attending to previous tokens takes more computation and memory.

The researchers measured the impact on three open-weight models running on NVIDIA H100 GPUs. Reducing the configured context window from 256,000 to 16,000 tokens increased output-token throughput by 3.6 times for GLM 5.2, 5.6 times for Kimi K2.6 and seven times for Qwen 3 Coder Next.

![Cost per 1M output tokens across context windows for GLM 5.2, Kimi K2.6 and Qwen 3 Coder Next](images/why-ai-agents-dont-always-need-long-contexts/long-contexts-cost-per-context-window.png)

The study identifies two ways different types of long-context tasks:

- **Long-horizon tasks:** An agent adds commands, tool outputs, and decisions as it works through a multi-step job.
- **Long-input tasks:** A model receives a large collection of documents or other source material to carry out one task.

## How much context do tasks actually need?

The researchers tested 13 benchmarks, including SWE-bench Verified, Terminal-Bench, LongBench v2 and LOONG. Their main accuracy experiments used Claude Opus 4.5 and GPT-5.6 Sol. They compared a baseline that retains the full conversation with an approach that compacts the agent's history when it reaches a predefined threshold.

Their baseline was a plain technique that retains the full conversation history. Their compaction technique triggers when the agent's history reaches a predefined threshold (e.g., 16K tokens).

At that point, the agent compacts its history and starts a fresh conversation with the original instructions and that state. The researchers also tested a more aggressive method that resets the conversation after every environment action.

## Long-running agents can forget much of their history

On long-horizon benchmarks, threshold-based compaction generally kept accuracy close to the full-history baseline. On SWE-bench Verified, Claude Opus 4.5 solved 70% of tasks without compaction and 74.8% at its best compaction setting. GPT 5.6 Sol improved from 84.4% to 88.8% at its best setting.

![Accuracy change and cost savings under compaction on long-horizon benchmarks for Claude Opus 4.5 and GPT 5.6 Sol](images/why-ai-agents-dont-always-need-long-contexts/long-contexts-long-horizon-results.png)

## Long documents can be read selectively

For long-input tasks, the researchers used a different strategy. Instead of placing entire documents in the prompt, they stored them as files. Agents could search those files with commands such as `grep` and import relevant passages into the model's context.

This approach matched or improved accuracy across almost all seven long-input benchmarks. In many tasks, agents finished without performing a single compaction. Most of the benefit came from deciding what to read, rather than summarizing what had already been read.

![Accuracy change and cost savings with file-based retrieval on long-input benchmarks for Claude Opus 4.5 and GPT 5.6 Sol](images/why-ai-agents-dont-always-need-long-contexts/long-contexts-long-input-results.png)

## One action at a time

The researchers also tested single-action compaction. After each tool action, the agent produced an updated description of its task state. That state became the starting point for its next action, while the previous conversation was discarded.

On SWE-bench Verified, both Claude Opus 4.5 and GPT-5.6 Sol almost matched the full-history baseline. Across the two evaluated benchmarks, the median peak context was just 7,000–9,000 tokens.

These results led the researchers to propose Semantic State Space Models, which would be trained to carry an explicit, human-readable task state between steps, rather than a conventional state-space model's internal numerical state.

## What makes a good compaction?

The researchers found that conventional summaries can preserve the wrong information: they recount prominent events in an agent's history, even when those events no longer matter.

A good compaction is a briefing for the agent that only preserves what the next step requires, which they describe as "the gist of the agentic trace."

Their compaction instructions focus on the original objective, established facts, relevant files and results, completed changes, failed approaches, remaining work and the immediate next action.

## What developers should do differently

The first caveat is that models respond differently to compaction. In the paper's open-weight experiments, Qwen 3 Coder Next fell from 53% accuracy on SWE-bench Verified to 3.6% under a 16K-token context limit. The findings show that the collapse is due to the model's verbose outputs, which left little room for progress between compactions.

Other models such as Qwen3.6-27B and MiniMax M2.7 held up much better and showed very little accuracy drops. Developers should therefore run their own evaluations across different tasks and compaction thresholds instead of assuming that one setting will suit every model.

Also worth considering are API costs when using cloud-hosted models. When successive requests share an unchanged prefix, prompt caching can reuse earlier computation at a discounted input-token rate.

Compaction changes part of that prefix and invalidates the cache. As a result, the next request might become more expensive even though it contains fewer tokens.

For example, on SWE-bench Verified, GPT 5.6 Sol's cost under standard per-token API pricing rose from $167.68 without compaction to $203.12 at the 16K limit.

The practical choice depends on the workload:

- **Long-running agents:** Compact into a structured task handoff. Keep full logs, tool results and artifacts available for retrieval. Measure cache hits, output tokens, latency and cost per completed task before tightening context limits.
- **Document-processing applications:** Store the source material externally and let the agent retrieve the passages it needs. Compact only if the resulting investigation accumulates a long interaction history.
- **Tasks requiring exact information:** Avoid relying on lossy summaries for detailed records, instructions or audit trails. Keep the original information available, and use full context when the input is manageable and the additional machinery offers little benefit.

As context windows expand, developers have more room to work with. But as the study shows, for many applications, the better optimization is deciding what belongs in working memory, and what can be retrieved later.
