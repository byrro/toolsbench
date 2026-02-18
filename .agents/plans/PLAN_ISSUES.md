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
