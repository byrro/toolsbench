# Tool Registry Integration

---

Guru is building a **Tool Scoring** system — a static checker that grades MCP tools on quality without involving LLMs. His v1 (announced in `#proj-registry` on Feb 24) is scoped to two scoring dimensions: **MCP Spec Compliance** and **Definition Quality**, producing letter grades (A–F) per tool. It includes a tool leaderboard, per-tool trust reports, ecosystem-wide insights, and improvement guidance linked to Arcade's 54 tool patterns. He explicitly descoped **Agent Readiness (LLM eval)** and **Model Benchmarks (LLM comparison matrix)** from his v1 — those are exactly what ToolsBench covers.

The repo is `ArcadeAI/marketplace` (private, created March 1, currently empty). The URL will likely be `toolbench.arcade.dev` with `toolsbench.com` as a redirect.

Guru asked that what we're building is **extendable to what he has** (DM, Feb 27). This document tracks integration-relevant design issues we need to address so the two systems can compose cleanly.

---

## 1. No canonical tool identity scheme

Tools in ToolsBench are implicitly identified by the combination of provider + toolkit + tool name. Guru's system indexes tools by MCP server URL + tool name. There is no shared **tool definition hash or fingerprint** that can serve as a universal join key between the two systems.

When Guru eventually pulls LLM eval results from ToolsBench to attach them to his quality grades, he needs a reliable way to match "this tool in ToolsBench" to "this tool in my scoring system."

The schemas still OPEN in PLAN_ISSUES sections **8.2** (tool definition storage layout) and **13.2** (results file layout) are the right place to bake this in. A content-addressable identifier — e.g., a hash of the normalized tool definition — would let both systems reference the exact same tool definition unambiguously.

---

## 2. Results structured per-toolkit, not per-tool

PLAN_ISSUES section 11.2 scopes eval runs to toolkits (e.g., "Slack tools"), and section 13.2's file layout candidates include "one JSON file per run, per task, per toolkit." Guru's leaderboard ranks **individual tools**, not toolkits. If our results JSON nests everything under toolkit groupings without per-tool indexing, extracting individual tool scores for integration will require extra work.

The results schema (PLAN_ISSUES 13.2, still OPEN) should ensure results are queryable per individual tool, not just per toolkit.

---

## 3. v1 scope limited to two providers

ToolsBench v1 targets only Arcade and Composio. Guru's system indexes the entire MCP ecosystem — any GitHub repo that exposes an MCP server. Our tool definition fetching (PLAN_ISSUES 8.1) is already MCP-generic, which is good. The risk is that other parts of the design — eval task authoring, fixture capture, provider configuration — might accidentally become coupled to the two-provider model. If we ensure that tool definitions always flow through MCP `tools/list` and nothing else assumes a fixed provider list, any MCP server should slot in without architectural changes.

---

## 4. No definition quality signal in tool definition storage

Guru scores tool definitions on description quality, parameter naming, and spec compliance. Our tool definition storage (PLAN_ISSUES 8.2, still OPEN) captures the raw definition and fetch metadata but does not store a **content hash of the tool definition itself**.

Without that hash, the connection between "which exact definition was the LLM evaluated against" and "what was Guru's quality grade for that definition version" becomes fuzzy. Storing a hash of the normalized definition content alongside the raw definition would let Guru's system correlate: "this LLM eval was run against definition version X, which I scored as grade B."
