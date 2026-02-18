# Tools Benchmark

The idea behind this project is to offer a publicly accessible benchmark comparing different LLM toolkits / MCP servers. The benchmark results must be fully transparent, verifiable, and reproducible. We must compare different tool vendors in the most neutral and fair way, on equal footing. Anyone viewing the benchmark results must be able to re-run the evals (using their own `ANTHROPIC_API_KEY`, or `OPENAI_API_KEY`, for example), and confirm that what we advertise is indeed truthful.

Our benchmarks will evaluate tools in three dimensions:

1. How well LLMs can select the correct tool?
2. How well LLMs can call the tool correctly, passing the expected parameters and values?
3. How much does it cost (in terms of LLM token usage) to run the tool?

## Relevant documents

### Plan Issues

We are tracking plan issues in the `./.agents/plans/PLAN_ISSUES.md` markdown document. "Plan issues" are topics we need to reflect upon, discuss, and decide before we move into writing a detailed plan for the implementation of this project.
