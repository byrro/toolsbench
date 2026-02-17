# Brainstorming

I want to build an open source service that benchmarks Agentic tools evaluating how LLMs perform in both picking the right tool and calling the tool correctly. At first, I'd like to test agains Anthropic Haiku, Sonnet & Opus and OpenAI GPT models. In the future we may expand to other providers, including open source LLMs.

We need to build a public HTTP server and frontend website to publish the results and an eval suite to run the tests and store the results.

Everything will be public.

We'll also need a CLI app to easily run the tests, so that anyone can download our package and re-run the same eval suite to verify the results we published.

We'll download the tool definitions from each provider and store them locally, committed in the repo for future reference. The first tool service providers that come to my mind are Arcade.dev and Composio.dev. I also plan on benchmarking public MCP servers. Along with the tool definitions, we also need to store the version (if available), the source and datetime when we downloaded the tool definitions.

Let's start by brainstorming what we need to clarify in terms of requirements, architectural / design decisions, third party libraries, and anything else you deem necessary for us to determine / clarify before we start getting our hands dirty.

Feel free to research anything on the web, in case you deem appropriate or necessary.

## Definitions

### Tool Set Size per Task
For each eval case, we store the list of tools provided to the model. By default, we provide the entire toolkit relevant to the prompt (e.g., the provider's full Slack toolkit for a Slack-related prompt). If the toolkit exceeds the LLM API's max tool count, we randomize a subset while ensuring the expected tool is included. Tool definitions live in a central location; eval cases reference the tools provided by name and/or hash, and we store the final list actually sent to the model.

### Rate Limiting
We will implement per-provider rate limiting using a token-bucket style configuration, specified via `.env` variables. Each provider can define a `RATE_LIMIT_RPS` (requests per second) and a `RATE_LIMIT_BURST` (max immediate burst). This throttles how fast requests are started, even when concurrency is higher. On 429 or rate-limit errors, we will pause the provider queue for `Retry-After` (if present) or a default backoff with jitter, then resume. Rate limiting is separate from scoring and does not affect evaluation results beyond pacing.

### Run Identification
Each eval run has a unique `run_id` (UUID) plus a `started_at` timestamp for ordering. Re-runs create a new `run_id` and store `rerun_of` pointing to the original run. We also store a `config_hash` for parity checks and allow an optional `run_label` for convenience.

### Tool Definition Fetching
We need a reproducible, programmatic way to retrieve tool definitions from Arcade and Composio. This should be implemented as a CLI command that fetches tools, stores the raw definitions and metadata (version, source URL, download datetime), and writes adapted schemas for model APIs when needed.

### System Prompt Management
We will maintain a default system prompt in a dedicated `prompts/` registry with version metadata. Each eval case can reference a `system_prompt_id` and optionally override it with a task-specific prompt. Results should store the exact system prompt text plus a `prompt_hash` for traceability.



## Questions

1. Who is the primary audience for the benchmark results (researchers, vendors, engineers, general public)?

A: Primarily AI agent developers.

---

2. What is the core promise we want to measure: tool selection, tool calling correctness, end-to-end task success, or all three?

A: I'd like to test both tool selection and tool calling correctness, in both one-shot and multi-shot scenarios. Some prompts may require the LLM to reason across multiple tool calls, so I'd like to test each stage: the first tool call expected, then we simulate a real response and check whether the LLM calls the expected second tool, for example.

---

3. Do we publish full prompts, model outputs, tool schemas, and tool responses publicly, or redact any parts?

A: We're not going to actually call the tools, just provide a prompt and a list of tool definitions to the LLM and check whether they select and provide the call parameters correctly. We will publish all prompts, tool definitions / schemas, and the model outputs. We need to store some sort of ID returned by the Anthropic and OpenAI APIs, as well as a timestamp when we sent the request. The Anthropic and OpenAI API keys should be stored in a .env file, so that anyone can set their own and run the evaluations themselves. The entire project should be implemented in Python, by the way. Let's use 3.14.

3.1. Python 3.14 may be pre-release; do you want to target the latest stable Python (with 3.14 compatibility), or require 3.14 specifically?

A: Python 3.14 is not pre-release.

3.2. Which provider request IDs should we capture explicitly (e.g., OpenAI `response.id` and/or `x-request-id`, Anthropic `message.id` and/or request-id header)?

A: Whatever they provide, we can determine this later.

---

4. Should runs be deterministic (fixed temperature/seed) or allow stochastic settings for realism?

A: I'm leaning more towards deterministic, since the purpose is a public reproducible benchmark.

---

5. Are tool calls executed against real third-party services or mocked/simulated for repeatability?

A: We won't execute the tools, just check the model response and see if it selects the correct tool and is capable of providing the expected parameters (or close enough, when non-exact values are acceptable).

---

6. What open source license should we use for the codebase, and is there a separate license for the results dataset/DB?

A: MIT. The repo already advertises this license in `./LICENCE`.

---

7. What is the unit of evaluation: a single tool call, a multi-step agent workflow, or both?

A: Both, answered above.

7.1. For multi-step tasks, is any extra/unexpected tool call an automatic failure, or do we allow extra calls if the expected sequence appears?

A: Actually, let's start only with single tool call in v1. We can expand with multi-step workflows in v2.

7.2. Are tasks provider-specific (Arcade-only vs Composio-only), or should we create equivalent tasks across providers when possible?

A: No, tasks should be identical regardless of who is the tool supplier. The evals should be 100% fair across service providers and not benefit any tool supplier in any way.

---

8. How do we define “correct tool choice” and “correct arguments” (exact match vs semantic/graded)?

A: It depends on the case. When it's a user ID or username, for example, we should check for an exact value match. In some cases, we may accept some variability.

8.1. What matchers should we support per argument (exact, regex, enum membership, numeric tolerance, custom function)?

A: Let's start with exact. As we expand our catalog of eval cases, prompts, and tools evaluated, if we find the need for different matchers we can implement them on-demand.

8.2. How do we handle missing/extra arguments or optional defaults (auto-fail vs ignore vs warning)?

A: If the argument has a default value in the tool interface, it's not a fail, but we should check the case when it's expected for the LLM to override the default. If the argument is mandatory (has no default value) and is missing in the tool call, then it's a tool call failure.

---

9. Should we score tool choice separately from argument correctness, or compute a combined score?

A: Separately.

---

10. Will we normalize tool definitions across providers into a canonical schema, and if so, what schema?

A: No, we will use the definition provided by the service provider. If the API (Anthropic or OpenAI) requires some specific format that is not provided by the tool source, we will adapt it, striving to keep the tools as close as possible to the original schema / definition.

10.1. When we adapt a tool schema for a model API, should we store both the original and adapted versions on disk?

A: Yes.

---

11. How do we handle tool errors or failures: model error, tool error, or inconclusive?

A: Model errors should be retried. We won't run the tools, so there won't be any tool errors. In case of multi-turn evaluations, we will mock tool responses.

11.1. What retry policy should we use (max retries, backoff, and which errors are retryable)?

A: Let's wait a few seconds and retry once. If it fails again, the eval run fails. We should store partial results, so that the user can request a continuation from where it stopped.

---

12. Do we use a single, unified system prompt across all providers, or provider-specific prompts?

A: Single unified.

---

13. Should tool responses be cached/snapshotted to guarantee replayable runs?

A: In multi-turn evaluations, we will mock tool responses. The mock responses should be stored in the repo.

---

14. Confirm initial providers: Arcade.dev, Composio.dev, and public MCP servers. Any others for v1?

A: v1 will be Arcade and Composio. Let's leave public MCP servers for v2.

---

15. What metadata must we store for tool definitions (version, source URL, datetime, hash)?

A: version, source URL, datetime

15.1. What file layout and format should we use to store tool definitions and their metadata (e.g., `tools/<provider>/<tool>.json` plus `metadata.json`)?

A: I'll leave this for you to determine what you think is best.

---

16. How often should tool definitions be refreshed, and who triggers updates?

A: it should be updated manually; we will only update when the service provider changes its tools.

---

17. Do we need to consider licensing or ToS constraints for redistributing tool definitions?

A: Good question. Let's only publish what is already published by the service providers. If they have a public documentation page advertising their tool interfaces (name, parameters, types), I think it's fine for us to replicate it, since it falls under fair use. If it's not advertised, then we can't even evaluate them, so it doesn't apply.

---

18. Do we want a monolithic service (API + UI + worker) or separated services?

A: Separate services. One thing is the eval suite, another is the CLI app that will take some parameters and run the eval suite, another is the public HTTP/HTML website to publish the results generated by the eval suite. Since we may re-run the eval suite as new models are published, we'll need to store multiple runs. Each run will be identified by a datetime.

---

19. Should the frontend be static (Vite) or SSR (Next/SvelteKit), and what UX is required (charts, filters, comparisons)?

A: Good question. Let's generate static HTML files based on templates, so that we can push it to S3 + CloudFront, for example, and have a very cheap hosting.

---

20. Is the eval runner only a local CLI, or also a server-side scheduled job?

A:  Just a local CLI.

---

21. Besides SQLite, do we publish CSV/Parquet exports for analysis?

A: Actually, maybe we don't even need SQLite, just store JSON files and export to static HTML, perhaps?

21.1. Should results be stored as one JSON file per run, one file per task, or both (with an index file)?

A: I'll leave this for you to determine what's best.

---

22. Where will we host the public DB snapshots (GitHub Releases, S3, etc.)?

A: Probably S3 + CloudFront or, perhaps Github Pages would be even better.

---

23. What core entities should the SQLite schema include (models, tasks, runs, steps, tool_calls, scores)?

A: Let's keep it simple and use only JSON.

23.1. What is the minimal JSON schema we want to standardize on (top-level fields for runs, tasks, steps, tool_calls, scores)?

A: You can decide this.

---

24. What run metadata must we persist (model params, prompt version, tool schema hash, git commit, runtime env)?

A: Model params, prompt version, tool schema hash, datetime, some request ID returned by the LLM model API, if any.

24.1. What are the exact model IDs for v1 (Anthropic Haiku/Sonnet/Opus and specific OpenAI GPT variants)?

A: Let's make the list of models configurable for each eval run.

---

25. Should we store exact prompts/responses and system prompt hashes for provenance?

A: I don't know. What do you think?

25.1. Do you want to store full prompts and raw model outputs in the results JSON (recommended for provenance), or only store hashes/IDs to reduce size?

A: Let's store everything.

---

26. What CLI features are required (run full suite, subset, compare runs, export artifacts)?

A: Full suite or subset. Also a separate command to export the static HTML page(s).

---

27. How will API keys and provider credentials be configured for the CLI (env vars, config file, keychain)?

A: .env file.

---

28. Should the CLI enforce fixed temperatures/params to guarantee repeatability?

A: No, we'll have a default, but the user may configure it through the .env file.

28.1. Which parameters should be configurable via `.env` (temperature, top_p, max_tokens, tool_choice, concurrency), and do we need per-model overrides?

A: Apart from those, we'll need reasoning effort as well for the "thinking" models. No per-model overrides is necessary, I think.

---

29. How do we guarantee re-run parity (tool version pinning, prompt snapshots, task versioning)?

A: We should store all parameters used to run an eval suite and store them with the eval results, so that someone can select a previous eval run and re-rerun it with the same parameters (prompt versions, tool versions, etc).

---

30. Are models allowed to see full tool schemas, or do we limit tool visibility?

A: Yes, we should provide everything is necessary for the model to select and call the tool correctly.

---

31. Should we track cost and token usage as part of the benchmark?

A: Yes, this is good to give people an estimate of how much it'd cost to re-run a full or partial eval.

31.1. Should we compute cost from a versioned pricing table stored in the repo, and if so, which source should we use for prices?

A: Yes, let's store costs in a JSON file locally in the repo. We can check the API providers pricing pages to get the prices and replicate locally for future reference. We'll need to store the datetime when we collected the prices.

---

32. Preferred stack for backend (FastAPI vs Express) and ORM (SQLModel/SQLAlchemy vs Prisma/Drizzle)?

A:  Actually just static HTML pages generated from a template with the eval results should be fine. We can then serve them through GitHub Pages or S3+CloudFront.

---

33. Preferred stack for CLI (Python Typer/Click vs Node Commander/Clipanion)?

A: Python Click should be fine.

---

34. Preferred hosting targets for API/UI (Vercel/Netlify, Fly.io/Render, etc.)?

A: Already answered.

---

35. Should we automate eval runs in CI (nightly) and publish versioned DB snapshots with checksums?

A: No, it should be manual.
