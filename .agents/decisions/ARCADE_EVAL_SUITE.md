# Decision: Use Arcade Eval Suite as the Core Evaluation Engine

**Status:** Accepted  
**Date:** 2026-02-17

## Context

ToolsBench needs an evaluation engine that sends prompts with tool definitions to LLMs, extracts tool calls from responses, matches expected vs actual tool calls, and scores tool selection and argument correctness. The Arcade MCP repository (`arcade-evals` library) already implements this exact workflow.

Since this is an Arcade-driven initiative, using `arcade-evals` as a dependency is a natural fit. The library is MIT-licensed and maintained by the same organization.

## Decision

Use `arcade-evals` (from the `arcade-mcp` package) as the core evaluation engine. Build the surrounding infrastructure (CLI, storage, publishing, tool ingestion) on top of it.

## What Works Out of the Box

- **`EvalSuite` + `EvalCase` + `ExpectedMCPToolCall`** for defining and running evaluations.
- **`add_tool_definitions()`** for registering tool schemas (fetched from Arcade/Composio) as JSON.
- **`BinaryCritic`** for exact matching (v1 requirement). Additional critics (`NumericCritic`, `SimilarityCritic`, `DatetimeCritic`) available when needed.
- **Rubric system** (`EvalRubric`) with separate tool-selection vs argument-correctness scoring and configurable pass/fail/warning thresholds.
- **Hungarian matching** (optimal assignment via `scipy.optimize.linear_sum_assignment`) for matching expected vs actual tool calls.
- **OpenAI and Anthropic provider adapters** with automatic format conversion and tool name normalization (dots/underscores/hyphens).
- **Multi-run support** with configurable seeds, statistics (mean, std deviation), and pass rules (last, mean, majority).
- **JSON output** of evaluation results.

## What Needs Extension (Subclass or Upstream PR)

- **Provider response metadata**: Surface request IDs and response headers from LLM API calls.
- **Token usage capture**: Extract and store token counts from API responses.
- **Cost computation**: Calculate cost from token usage and a local pricing snapshot.
- **Rate limiting**: Token-bucket style throttling per provider around the request loop.
- **Retry policy**: Wait and retry once on failure; store partial results; allow run resumption.
- **Run-level metadata**: Attach `run_id`, `started_at`, `config_hash`, prompt version, and tool schema hashes to results.

The subclass approach is preferred for most extensions -- override run/request methods to capture metadata and inject rate limiting without modifying upstream. PRs make sense for features that benefit the broader Arcade ecosystem (e.g., exposing token usage).

## What Is Built Separately (Outside arcade-evals)

- **Click CLI** that constructs `EvalSuite` instances programmatically and orchestrates runs.
- **Tool definition fetching and versioning** pipeline (Arcade API, Composio API) with metadata storage (version, source URL, download datetime).
- **Results storage layer** that consumes `arcade-evals` JSON output, enriches it with run metadata, and persists it.
- **System prompt registry** with versioning, hashing, and per-task override support.
- **Pricing snapshots** stored in-repo for cost reproducibility.
- **Static HTML generation** from results JSON and templates for publishing via GitHub Pages or S3 + CloudFront.

## Risks and Mitigations

- **Reproducibility gap from dependency drift.** If `arcade-evals` scoring logic or matching behavior changes between versions, results from different runs are not comparable. Mitigation: pin the exact `arcade-mcp` version (or commit hash) in `pyproject.toml`, and record the pinned version in every run's metadata. Since `external/arcade-mcp` is vendored in the repo, pinning to a specific commit is straightforward.

## Key Library Details

- **Location in repo:** `external/arcade-mcp/libs/arcade-evals/`
- **Install:** `pip install 'arcade-mcp[evals]'`
- **License:** MIT (Copyright 2025, Arcade AI)
- **Public API:** `EvalSuite`, `EvalCase`, `ExpectedMCPToolCall`, `BinaryCritic`, `NumericCritic`, `SimilarityCritic`, `DatetimeCritic`, `EvalRubric`, `FuzzyWeight`, `tool_eval`, `add_tool_definitions`, among others.
