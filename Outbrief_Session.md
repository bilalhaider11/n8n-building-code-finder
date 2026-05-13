# Outbrief Session — AI-Powered Building Code Lookup

## Overview
This document summarizes the workflow design, the architectural decisions taken during the build, and the recommended next steps for the **AI-Powered Building Code Lookup** prototype delivered to Greyfibre.

The deliverable is a single n8n workflow (`AI-Powered Building Code Lookup.json`) that, given a US location and a code type, returns the currently adopted code, the adopting authority, the state administrative-code reference, any local amendments, and 3–6 official sources.

---

## Workflow Flow

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | User Input Form | `formTrigger` | Collects `Location` and `Code Type` (6 dropdown options). |
| 2 | Validate Input | `code` | Length check (2–200) and code-type allowlist. |
| 3 | Resolve City & State | `langchain.openAi` (GPT-4o-mini, temp 0) | Resolves raw location → `City, State`, or `UNKNOWN`. |
| 4 | Parse City/State | `code` | Falls back to raw location if resolver returned `UNKNOWN`. |
| 5 | Build Agent Prompt | `set` | Composes the research brief for the agent. |
| 6 | Building Code Research Agent | `langchain.agent` (GPT-4o) | Function-calling agent with the search tool attached. Caps at 6 iterations. |
| 7 | OpenAI Chat Model (GPT-4o) | `lmChatOpenAi` | Backing LLM for the agent (temp 0.1). |
| 8 | Web Search Tool (Serper) | `httpRequestTool` | Tool exposed to the agent. POSTs `{q, num:10}` to `google.serper.dev/search`. |
| 9 | Output Schema | `outputParserStructured` | Enforces the JSON contract for the rendering step. |
| 10 | Format HTML Output | `code` | Escapes all strings, validates URLs (`^https?://`), renders safe HTML. |
| 11 | Show Results to User | `form` (completion) | Renders the HTML on the form completion page. |

---

## APIs Used
| API | Purpose | Endpoint | Auth |
|-----|---------|----------|------|
| **OpenAI** | City/state resolution + research agent | OpenAI API | Greyfibre `OpenAi account` credential |
| **Serper** | Google search results as an agent tool | `https://google.serper.dev/search` | HTTP Header Auth (`X-API-KEY`) |

No paid permit-data APIs are used. The agent relies entirely on grounded web search against official municipal, state, and recognized publisher domains.

---

## Search Strategy (System Prompt)
The agent is instructed to run targeted, sequential searches rather than one broad query:
1. `<city> <state> <code type> adoption ordinance` — city-level adoption.
2. `<state> <code type> administrative code` — state base code (e.g., WAC, CCR, NYCRR).
3. `<city> <state> <code type> amendments` — local modifications.

Additional searches only if the first three are insufficient. Hard stop at 5 searches.

**Source priority:** official municipal / `.gov` / `.us` → recognized publishers (`codepublishing.com`, `municode.com`, `ecode360.com`, `iccsafe.org`) → state agency PDFs. Blogs, contractor marketing, Wikipedia, and third-party summaries are explicitly excluded.

**Grounding rule:** every URL in the output must come from a `web_search` tool result returned in the current run; no fabrication.

---

## Key Architectural Decisions

### Why an Agent node, not a plain LLM + tool
n8n's `langchain.openAi` completion node does not invoke attached tools — tool calling requires the `langchain.agent` node. An earlier draft attached the Serper tool to a completion node and the tool was never called. The Agent node fixes this.

### Why separate GPT-4o-mini for geocoding
The agent runs GPT-4o (more expensive, function-calling capable). Resolving "Renton WA" → "Renton, Washington" is a trivial deterministic task — running it on a cheap GPT-4o-mini call at temperature 0 keeps the high-cost agent run focused on the research step.

### Why a Structured Output Parser
The HTML formatter assumes a specific shape (`executive_summary`, `base_code`, etc.). The parser is wired as the agent's `ai_outputParser` so any malformed agent output is rejected before reaching the formatter.

### HTML safety
The `Format HTML Output` node escapes `& < > "` and validates every URL against `^https?://` before embedding it. The completion page is rendered inline, so injection-safe output matters.

### Cost & latency profile
- Geocoding: ~1 GPT-4o-mini call.
- Research: 1 GPT-4o agent run with up to 5 Serper calls.
- End-to-end latency: ~15–45 seconds typical, depending on agent search count.

---

## Known Limitations
- The agent can return "Not identified in search results" for fields it cannot ground — by design, to avoid hallucinated authorities or section references.
- Hard cap at 5 searches means very obscure jurisdictions may yield partial results.
- The result page is HTML on the form completion view only — no persistent storage of past lookups.
- The current workflow does not retry on transient Serper failures; an upstream error surfaces as a workflow execution error.

---

## Recommended Next Steps
1. **Persistence** — log each lookup (inputs + structured output + sources) to a database or sheet for audit and re-use. Useful for QA and for spotting jurisdictions where the agent struggles.
2. **Retry policy** — add a small retry on the Serper HTTP node for 5xx / timeout responses.
3. **Caching** — cache by `(city, codeType)` for 24–72 hours. Building-code adoptions move on a years-scale timeline, so cache hits are cheap and dramatically reduce per-lookup cost.
4. **Source ranking refinement** — track which `source_type`s users actually click and use that signal to tune the prompt.
5. **Expansion to permit data** — layer in a separate workflow that queries city open-data portals (e.g., `data.sfgov.org`, NYC Open Data, Chicago Data Portal) for active permits in the same jurisdiction, sharing the resolved city/state from this workflow.
6. **CSV / MinIO export** — per Greyfibre deployment standards, optionally export the structured JSON to MinIO for downstream consumption.

---

**Prepared by:** Bilal Haider
**Date:** 13 May 2026
**For:** Greyfibre
