# Tools Benchmark

An open source service that benchmarks agentic tools by evaluating how well LLMs perform at both picking the right tool and calling it correctly. The benchmark results are fully transparent, verifiable, and reproducible. We compare tool vendors in the most neutral and fair way, on equal footing. Anyone can re-run the evals (using their own `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.) and confirm that published results are truthful.

Our benchmarks evaluate tools in three dimensions:

1. How well LLMs can select the correct tool?
2. How well LLMs can call the tool correctly, passing the expected parameters and values?
3. How much does it cost (in terms of LLM token usage) to run the tool?

## Scope

**LLM providers:** Anthropic (Haiku, Sonnet, Opus) and OpenAI (GPT models) for v1. Other providers and open source LLMs may follow.

**Tool providers:** Arcade.dev and Composio.dev for v1. Public MCP servers in v2.

**Tool definitions** are fetched from each provider and committed to the repo along with metadata (version, source URL, download datetime).

## Deliverables

1. **Eval suite** — runs the benchmark tasks against configured LLMs and stores results.
2. **CLI app** — allows anyone to download the package and run the full or partial eval suite locally.
3. **Public website** — publishes benchmark results for browsing.

## Relevant documents

### Plan Issues

We are tracking plan issues in the `./.agents/plans/PLAN_ISSUES.md` markdown document. "Plan issues" are topics we need to reflect upon, discuss, and decide before we move into writing a detailed plan for the implementation of this project.
