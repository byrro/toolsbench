## High-Level Plan

### 1) Repository Structure and Conventions
- Define top-level folders for `eval`, `tools`, `results`, `site`, and `cli`.
- Standardize JSON schemas for tasks, tool metadata, and run results.
- Add `.env.example`, configs, and documentation for local setup.

### 2) Tool Definitions and Metadata
- Implement a tool ingestion workflow for Arcade and Composio.
- Store original tool definitions + adapted schemas (for each model API).
- Capture tool metadata: version, source URL, download datetime.

### 3) Task Catalog and Ground Truth
- Create a task format for single-call evaluations.
- Define expected tool and required arguments with exact matching rules.
- Include mock tool responses for future multi-step expansion.

### 4) Model Provider Adapters
- Build Anthropic and OpenAI adapters with unified prompt format.
- Normalize tool schema to each provider API while preserving originals.
- Capture provider request IDs, timestamps, token usage, and cost.

### 5) Eval Runner Core
- Orchestrate tasks, model calls, and scoring.
- Implement deterministic defaults and retry policy (1 retry + backoff).
- Persist partial results and allow resume from last completed task.
- Record per-task completion state to support safe run resumption.

### 6) Scoring and Reporting
- Score tool choice and argument correctness separately.
- Store full prompts and raw model outputs in results JSON.
- Produce summary metrics per model, provider, and tool set.

### 7) Results Storage Format
- Decide on JSON layout (run-level index + per-task details).
- Store run metadata, parameters, and hashes for reproducibility.
- Include pricing snapshot references used to compute cost.

### 7.1) Pricing Data Collection
- Define a versioned pricing JSON schema (provider, model, input/output rates, timestamp/source).
- Maintain pricing snapshots in-repo for cost reproducibility.
- Document the manual update workflow for pricing data.

### 7.2) System Prompt Management
- Create a versioned prompt registry with IDs and hashes.
- Allow per-task prompt overrides while retaining a default prompt.
- Record prompt versions and hashes in run results.

### 8) Static Site Generation
- Generate static HTML from results JSON and templates.
- Provide pages for leaderboards, run details, and per-task inspection.
- Output assets suitable for GitHub Pages or S3 + CloudFront.

### 9) CLI Application
- Provide commands for running evals and exporting static HTML.
- Support full suite and subset selection, configurable via `.env`.
- Emit artifacts into versioned run directories.

### 10) Testing and CI (Manual-Run Friendly)
- Unit tests for schema validation, scoring, and adapters.
- Lightweight integration tests for a small task set.
- Document manual release steps for publishing results.

### 11) Documentation
- README with architecture overview, setup, and usage.
- Explain data formats, schema, and how to reproduce results.
- Add contribution guidelines for new tasks and tool providers.
