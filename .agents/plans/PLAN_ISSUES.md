# Plan Issues

Open questions, discussion topics, and decisions that need resolution before we write a concrete implementation plan.

Each issue is tagged with its status:
- **OPEN** — needs discussion or a decision
- **DECIDED** — resolved, decision recorded inline
- **DEFERRED** — acknowledged but intentionally postponed

---

## 1. Response Fixtures & Capture Pipeline

How we capture, store, and maintain realistic tool responses for use in multi-turn evaluations and response efficiency analysis.

### 1.1 Fixture schema and format

**Status: OPEN**

Each stored response fixture needs to carry enough metadata to be self-describing and reproducible. Candidate fields:

- Tool call that produced the response (tool name + arguments)
- Raw response body (as returned by the tool provider)
- Token count (tiktoken `cl100k_base`)
- Byte size
- Capture datetime
- Tool provider, toolkit, and tool version at capture time
- Source (who captured it: maintainer, contributor, provider)
- Anonymization notes (what was redacted, if anything)

Need to decide: file layout (one file per response? grouped by toolkit?), naming conventions, and whether we store responses as standalone JSON or embedded in a larger fixture file that also describes the triggering call.

### 1.2 Reproducible capture process

**Status: DECIDED (direction), OPEN (design details)**

**Decision:** The response capture process must be reproducible. We will build a CLI command (e.g., `toolsbench capture-responses`) that, given a configured integration, runs a defined set of canonical queries and writes the response fixtures to the repo in a structured format.

Open questions:
- How do we define the "canonical queries" per toolkit? Are they derived from the eval tasks, or maintained as a separate catalog?
- How do we handle integrations that require specific data to exist in the upstream system (e.g., a CRM must have companies, deals, contacts populated)?
- Do we provide seed data instructions or scripts per integration?

### 1.3 PII and data anonymization

**Status: OPEN**

Live tool responses will contain real data from whoever runs the capture. We need a strategy for:
- What gets redacted (emails, names, IDs, monetary values?)
- Whether redaction is automatic (regex-based), manual, or both
- Whether redacted values are replaced with realistic synthetic data (to preserve response structure) or masked with placeholders
- How to ensure anonymized responses still exercise the same context window pressure as real ones (token count shouldn't drop significantly after redaction)

### 1.4 Fixture versioning and staleness

**Status: OPEN**

Tool providers update their APIs and response formats over time. Need to decide:
- How we detect that stored fixtures are stale (compare against a fresh capture?)
- Whether we maintain multiple fixture versions per tool (tied to tool definition versions)
- Policy for when a fixture refresh is required (on every tool definition update? periodically?)

---

## 2. Multi-Turn Evaluation Design

How we use response fixtures to simulate realistic multi-turn agent interactions and evaluate tool selection under context pressure.

### 2.1 Context injection mechanics

**Status: OPEN**

For multi-turn evals, we inject simulated prior turns (tool calls + responses) into the conversation before the eval prompt. Need to decide:
- Format of injected turns (assistant message with tool_use + tool result, matching each provider's API format?)
- Whether we inject the *same* prior responses across providers (Arcade vs Composio) to test how response quality affects the *next* tool call
- How many prior turns to simulate (fixed per task? variable to test degradation?)

### 2.2 Dual-track scoring: ship together

**Status: DECIDED**

**Decision:** Response efficiency scoring (token count, cost) and tool selection correctness under context pressure will ship together, not as separate tracks. Cost analysis alone is shallow — tools must be both cost-efficient *and* well-designed for LLMs to use correctly. A cheap tool that LLMs can't use reliably is worthless; an accurate tool that consumes 59% of the context window per call is impractical.

Open design questions:
- How do we present combined results? Separate scores on the same page, or a composite metric?
- Do we define "context pressure tiers" (e.g., 10%, 25%, 50% of context window consumed by prior responses) and test each?

### 2.3 Measuring response quality impact on downstream tool selection

**Status: OPEN**

The Attio benchmark shows that bloated responses degrade follow-up accuracy. We should design eval tasks that explicitly test this:
- Same user prompt, same expected tool call, but different prior-turn response sizes (compact vs bloated)
- Measure whether tool selection accuracy drops as prior context grows
- This directly answers "does response quality matter?" with hard data per model

Need to decide whether this is a v2 feature or part of the initial multi-turn implementation.

---

## 3. Benchmark Integrity & Verification

How we ensure published benchmark results are trustworthy and not manipulable.

### 3.1 Verifying provider-submitted response fixtures

**Status: DECIDED (principle), OPEN (mechanism)**

**Decision:** We cannot blindly trust response fixtures submitted by tool providers. Providers have incentive to submit optimized or unrepresentative responses. All published fixtures must be verified against live tool calls made by a ToolsBench maintainer or trusted contributor.

Open questions:
- What does the verification process look like? Run the capture pipeline independently and diff against the submission?
- What tolerance do we allow for natural variation in live responses (timestamps, IDs will differ)?
- Do we publish a "verified" badge or status per fixture set?

### 3.2 End-to-end reproducibility of published benchmarks

**Status: OPEN**

Anyone should be able to clone the repo, set their API keys, and reproduce our published eval results (within statistical tolerance for non-deterministic models). This requires:
- All inputs are committed: tool definitions, response fixtures, prompts, eval tasks, model parameters
- Results include enough metadata to reconstruct the exact configuration
- A CLI command that takes a published run's config and re-executes it

This is largely addressed by existing design decisions (deterministic defaults, stored prompts, etc.) but needs explicit validation once the system is built.

---

## 4. Integration Setup Documentation

Structured metadata about the effort required to set up each tool provider integration.

### 4.1 Integration complexity metadata schema

**Status: DECIDED (direction), OPEN (schema)**

**Decision:** When we implement an integration, we record the setup process in a structured, public format. This is useful to agent developers evaluating tool providers.

Candidate fields to capture per integration:
- Authentication method (OAuth, API key, token)
- OAuth flow complexity (number of steps, scopes required)
- Whether a paid plan is required (and minimum tier)
- Whether a sandbox/test environment is available
- Estimated setup time
- Required upstream data (does the target system need to be pre-populated?)
- Any provider-specific gotchas or limitations

Need to decide: where this lives in the repo (alongside tool definitions? separate `integrations/` directory?), and the exact schema.

---

## 5. Tool Set Size per Task

How we select and record which tools are provided to the model for each eval case.

### 5.1 Tool selection policy

**Status: DECIDED**

**Decision:** By default, we provide the entire toolkit relevant to the prompt (e.g., the provider's full Slack toolkit for a Slack-related prompt). If the toolkit exceeds the LLM API's max tool count, we randomize a subset while ensuring the expected tool is always included. Tool definitions live in a central location; eval cases reference the tools provided by name and/or hash, and we store the final list actually sent to the model.

Open detail:
- The randomization of subsets should use a seeded RNG for reproducibility. Need to decide whether the seed is per-run, per-task, or configurable.

---

## 6. Rate Limiting

How we throttle requests to LLM providers to stay within API limits without affecting eval results.

### 6.1 Rate limiting strategy

**Status: DECIDED**

**Decision:** Per-provider token-bucket rate limiting, configured via `.env` variables. Each provider defines `RATE_LIMIT_RPS` (requests per second) and `RATE_LIMIT_BURST` (max immediate burst). This throttles how fast requests are started, even when concurrency is higher. On 429 or rate-limit errors, we pause the provider queue for `Retry-After` (if present) or a default backoff with jitter, then resume. Rate limiting is purely operational and does not affect evaluation scoring or results beyond pacing.

---

## 7. Run Identification

How we identify, link, and trace eval runs for reproducibility and comparison.

### 7.1 Run identification scheme

**Status: DECIDED**

**Decision:** Each eval run has a unique `run_id` (UUID) plus a `started_at` timestamp for ordering. Re-runs create a new `run_id` and store `rerun_of` pointing to the original run. We also store a `config_hash` for parity checks and allow an optional `run_label` for convenience.

Open details:
- **`config_hash` composition:** need to define what inputs feed into the hash (model params, prompt version, tool definition versions, seed policy, eval task set).
- **Seed tracking:** the Arcade eval SDK (`arcade-evals`) already implements seed management with a `seed_policy` (constant, random, or custom integer) and per-run `run_seeds` list, plus multi-run statistics (mean score, std deviation, pass rules). Our run metadata should capture seed info as well for reproducibility. Worth evaluating whether we build on top of Arcade's eval SDK (wrapping its results with our persistence layer) or build our own -- in either case, seed tracking should be part of run identification.
- **API request IDs:** each eval call should capture the provider-specific response/request ID (e.g., OpenAI `response.id`, Anthropic `message.id`) and the timestamp when the request was sent. Exact fields to capture can be determined per provider during implementation.

### 7.2 Model parameters and determinism

**Status: DECIDED**

**Decision:** Deterministic defaults for reproducibility, user-overridable via `.env`. Configurable parameters: `temperature`, `top_p`, `max_tokens`, `tool_choice`, `concurrency`, and `reasoning_effort` (for thinking models). No per-model overrides. Whatever values are used for a run get stored in the run metadata and feed into the `config_hash`.

---

## 8. Tool Definition Fetching

How we retrieve, store, and adapt tool definitions from tool providers.

### 8.1 Fetching strategy

**Status: DECIDED**

**Decision:** A CLI command (e.g., `toolsbench fetch-tools`) programmatically retrieves tool definitions from providers and stores the raw MCP definitions along with metadata (version, source URL, download datetime). Only raw definitions are committed to the repo; adaptation to model-specific formats (Anthropic tool use, OpenAI function calling) happens at eval runtime (see 8.2).

For v1 we target only tools served through the MCP protocol. This means we implement a single tool definition retrieval mechanism (MCP `tools/list`) that applies uniformly to any MCP-compatible server (Arcade, Composio, or others). Provider-specific fetching logic is not needed.

### 8.2 Storage layout and schema adaptation

**Status: DECIDED**

**Decision:** Only raw MCP definitions are stored in the repo. Adaptation to model-specific formats happens at eval runtime via a lightweight adapter registry.

**Storage layout:** One JSON file per tool, grouped by provider and toolkit:

```
tools/
  <provider>/
    _meta.json                          # provider-level metadata (source URL, fetch datetime)
    <toolkit>/
      _meta.json                        # toolkit-level metadata (tool count, version)
      <ToolName>.json                   # raw MCP tool definition
      ...
```

File names match the tool's canonical name as returned by the MCP server.

**Adaptation mechanism:** Delegated to the Arcade evals SDK. The SDK's `EvalSuiteToolRegistry` stores tools in MCP format and converts them on demand via `list_tools_for_model(tool_format=...)`, dispatching to built-in converters for OpenAI (strict mode, `additionalProperties: false`, optional params unioned with `null`) and Anthropic (`inputSchema` → `input_schema`, standard JSON Schema preserved). It also handles tool name normalization (dots vs underscores) and other edge cases. No custom adapter code is needed in ToolsBench.

**When adaptation happens:** At eval runtime. The eval runner loads raw MCP definitions for the relevant toolkit from disk, feeds them into the SDK's tool registry, and lets the registry handle conversion when making LLM API calls. The adapted definitions are stored in the eval results JSON for provenance (per decision 13.3).

**Rationale:** Storing adapted schemas on disk would create a sync problem (raw definitions change but adapted versions don't get regenerated) for minimal benefit. The Arcade evals SDK already implements correct, tested converters for each supported model API, so there is no reason to duplicate that logic.

### 8.3 Refresh policy

**Status: DECIDED**

**Decision:** Tool definitions are refreshed manually, only when a provider updates its tools. No automated or scheduled refresh.

### 8.4 Redistribution policy

**Status: DECIDED**

**Decision:** We only publish tool definitions that are already publicly advertised by the providers (e.g., on their public documentation pages). If a provider publicly documents tool interfaces (name, parameters, types), redistributing them falls under fair use. If a tool definition is not publicly advertised, we do not include it.

---

## 9. System Prompt Management

How we maintain, version, and trace system prompts used in evaluations.

### 9.1 Prompt registry and versioning

**Status: DECIDED (direction), OPEN (format)**

**Decision:** A default system prompt lives in a dedicated `prompts/` registry with version metadata. Each eval case can reference a `system_prompt_id` and optionally override it with a task-specific prompt. Results store the exact system prompt text plus a `prompt_hash` for traceability. A single unified system prompt is used across all LLM providers (no provider-specific prompts).

Open details:
- File format inside `prompts/` (plain text, markdown, structured YAML/JSON with metadata?)
- Versioning scheme (sequential numbers, semantic versioning, content-based hashes?)
- How task-specific overrides are defined and stored alongside eval cases

---

## 10. Correctness Matching and Scoring

How we determine whether the model selected the correct tool and provided correct arguments.

### 10.1 Tool choice scoring

**Status: DECIDED**

**Decision:** Tool choice and argument correctness are scored separately. Tool choice is an exact match on the tool name.

Note: the Arcade eval SDK scores both as separate weighted components within a single `EvaluationResult.results` list (`field="tool_selection"` for tool choice, individual `critic_field` entries for each argument). However, it only produces a single combined normalized score. To report two independent top-level scores we need a thin post-processing layer that splits and aggregates those results. Also, `fail_on_tool_selection` (default `True`) must be set to `False` if we want argument scoring to proceed even when tool selection fails -- useful for diagnostics.

### 10.2 Argument matching

**Status: DECIDED**

**Decision:** Exact match for v1. Other matchers (regex, enum membership, numeric tolerance, custom functions) can be added on-demand as the eval catalog grows and we encounter cases that need them.

### 10.3 Missing and extra arguments

**Status: DECIDED**

**Decision:** If an argument is mandatory (no default value) and missing from the model's tool call, it's a failure. If an argument has a default value in the tool interface, omitting it is not a failure — unless the eval case explicitly expects the model to override the default, in which case the expected value must match.

---

## 11. Error Handling and Retries

How we handle API failures during eval runs.

### 11.1 Retry policy

**Status: DECIDED**

**Decision:** On a model API error, wait a few seconds and retry once. If the retry also fails, the eval case fails. No further retries.

### 11.2 Run granularity and isolation

**Status: DECIDED**

**Decision:** Eval runs are scoped per toolkit (e.g., Slack tools, MS Teams tools). A failure in one toolkit does not require re-running previously completed toolkits.

### 11.3 Partial results and continuation

**Status: DEFERRED**

Not needed for v1 given per-toolkit isolation. If a toolkit eval fails, re-run that toolkit only. May revisit if eval suites grow large enough that re-running a single toolkit becomes expensive.

---

## 12. Public Website and Hosting

How we publish benchmark results for public consumption.

**Status: DEFERRED**

The publishing layer (frontend, backend/hosting, UX, templating) is deferred. The priority is the eval machinery: defining evals, running them, and storing results in a structured format that any future frontend/backend can consume. Decisions about static HTML generation, templating engine, hosting (S3 + CloudFront, GitHub Pages, etc.), and UX features (charts, filters, comparisons) will be made later.

---

## 13. Results Storage

How we persist eval results for consumption by future tooling and frontends.

### 13.1 Storage format

**Status: DECIDED**

**Decision:** JSON only, no SQLite or other databases. Results must be structured well enough that any future frontend or analysis tool can consume them directly.

### 13.2 File layout and schema

**Status: OPEN**

Need to decide:
- File layout: one JSON file per run, per task, per toolkit, or a combination (with an index file?)
- Minimal JSON schema for runs, tasks, tool calls, and scores
- How to ensure the schema supports future needs (comparisons across runs, filtering by model/provider/toolkit) without overengineering for v1

### 13.3 Provenance: full prompts and raw outputs

**Status: DECIDED**

**Decision:** Store full prompts and raw model outputs in the results JSON, not just hashes or IDs. This ensures full provenance and allows anyone to inspect exactly what was sent and received without needing to re-run the eval.

---

## 14. Cost and Token Usage Tracking

How we measure and estimate the cost of running evaluations.

### 14.1 Token usage capture

**Status: DECIDED**

**Decision:** Capture token counts (input, output) per eval call from the LLM API response. Store them alongside the eval results.

### 14.2 Pricing table

**Status: DECIDED (direction), OPEN (schema)**

**Decision:** Maintain a versioned JSON file in the repo with per-model prices sourced from the API providers' pricing pages. Each entry includes the datetime when prices were collected. Used to compute cost estimates for eval runs.

Open details:
- Pricing file schema (per-model input/output token prices, currency, effective date?)
- Whether we store a single pricing file with historical entries or one file per collection date
- How cost estimates are presented in the results (per eval case, per toolkit, per run total?)

---

## 15. Concurrency and Execution Model

How eval cases and suites are executed.

### 15.1 Concurrency strategy

**Status: DECIDED**

**Decision:** Eval runs execute sequentially (one toolkit at a time). Within a run, eval cases and suites are parallelized using the Arcade eval SDK's built-in concurrency control. No custom concurrency infrastructure needed.

---

## 16. Eval Case Authoring

How new eval cases are contributed and organized.

### 16.1 Contribution workflow

**Status: DECIDED**

**Decision:** New eval cases are contributed via pull requests on GitHub. The eval case definition format is dictated by the Arcade eval SDK (`EvalCase`, `ExpectedMCPToolCall`, critics). No custom format needed.

### 16.2 Initial eval catalog scope

**Status: DEFERRED**

Which toolkits and tools to evaluate first, and how many cases per tool, will be defined when ToolsBench is ready to run.
