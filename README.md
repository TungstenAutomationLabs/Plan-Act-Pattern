## Plan-Act Agent Pattern

### What Is It?

The **Plan-Act Agent** implements a planning pattern where an LLM generates a complete reasoning and action plan *before* any tools are executed. Unlike reactive agent loops (e.g., ReAct) that interleave thinking and acting one step at a time, Plan-Act **separates planning from execution entirely**. The planner produces a structured sequence of steps — each with defined reasoning, required tools, and intermediate evidence variables (`#E1`, `#E2`, etc.) — and then a loop executor carries out each step in order.

This makes agent workflows more **controllable, efficient, and auditable** because:

- **Fewer LLM calls** — the plan is created once, then executed deterministically step-by-step.
- **Traceability** — explicit evidence variables (`#E1`, `#E2`, …) link outputs from earlier steps to inputs of later steps.
- **Deterministic processing** where appropriate (data extraction, formatting), with **judgment-based decisions** where necessary (source selection).
- **Modular task decomposition** — each step calls a registered tool or worker agent, enabling reuse and composability.
- **Auditability** — every plan step and its output is saved as a case note, providing a full execution trace.

### Prerequisites

- **TotalAgility 2026.1+** — You need access to a TotalAgility 2026.1 (or later) environment to import and run the Plan-Act Agent package.
- **You.com Connector** *(recommended for the tutorial use case)* — Install the [TotalAgility connector for You.com](https://github.com/TungstenAutomationLabs/TotalAgility-connector-for-you.com-) to enable web search, deep research, and web content retrieval tools referenced in the tutorial below.

### How It Works

The agent follows this high-level flow:

```
User Prompt
    │
    ▼
┌──────────────────────┐
│  Get Session & Seed  │  ← Authenticate / reuse session
└──────────┬───────────┘
           ▼
┌───────────────────────────────┐
│  Fast Analysis of Attachments │  ← Optional: analyse any uploaded document
└──────────┬────────────────────┘
           ▼
┌──────────────────────────┐
│  Task Decomposer         │  ← Pre-planner: break complex prompts into
│  (Pre-Planner Node)      │    manageable sub-tasks
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│  Create Plan (GenAI)     │  ← LLM generates a full Plan-Act plan:
│                          │    Steps, Tools, Evidence (E1→E2→…)
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│  Loop Plan Steps         │  ← Iterate over each planned step
│  ┌────────────────────┐  │
│  │ Call Worker Agent  │  │  ← Dispatch to the tool/agent specified
│  │ or Tool (Sub-job)  │  │    in the plan step
│  ├────────────────────┤  │
│  │ Record plan step   │  │  ← Store output as evidence (#E1, #E2…)
│  │ outputs            │  │
│  ├────────────────────┤  │
│  │ Update Plan Output │  │  ← Feed evidence into subsequent steps
│  │ History            │  │
│  └────────────────────┘  │
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│  Summarise Results       │  ← LLM synthesises all evidence into
│  (GenAI)                 │    a final response
└──────────┬───────────────┘
           ▼
       Final Output
```

**Key components**:

| Component | Role |
|---|---|
| **Task Decomposer** | Pre-planner GenAI node that breaks complex prompts into sub-tasks |
| **Create Plan** | GenAI node that produces the full Plan-Act plan with steps, tools, and evidence mapping |
| **Loop Plan Steps** | Iterates through each plan step sequentially |
| **Call Worker Agent or Tool** | Sub-job that dispatches to the appropriate tool/agent from the registry |
| **Evidence tracking** | Data list activities that record `#E1`, `#E2`, etc. and inject them into downstream steps |
| **Summarise Results** | GenAI node that synthesises all collected evidence into the final answer |
| **Error / Human-in-the-loop** | Max loop guard and error routing to a synchronization point for human escalation |

### When Should You Use It?

| Use Plan-Act When… | Use a Simpler Pattern (e.g., ReAct / Tool-Use) When… |
|---|---|
| The task requires **multiple coordinated steps** (research → extract → format) | A single tool call or direct LLM reply is sufficient |
| You need a **full audit trail** of reasoning and actions | Low-stakes, conversational Q&A |
| You want to **minimise LLM calls** (plan once, execute many) | The task is unpredictable and requires dynamic re-planning after each observation |
| **Deterministic, repeatable** workflows are important | Exploratory tasks where the next step depends entirely on the previous result |
| You're building **multi-agent orchestration** or composite "Russian Doll" agents | Simple single-agent scenarios |

---

### Tutorial: Getting Started with Plan-Act and You.com

This tutorial walks you through importing the Plan-Act Agent, installing the You.com connector for web search tools, configuring the tool registry, and running a test prompt to validate the full plan-act cycle.

#### Step 1 — Import the Plan-Act Agent Package

1. In TotalAgility Designer, navigate to **Advanced > Import Package**.
2. Select the Plan-Act Agent package file (`.pkg`) from this repository's `packages/` directory.
3. Follow the import wizard to completion, resolving any dependency prompts as needed.
4. Once imported, the Plan-Act Agent process and its supporting sub-processes will be available in your environment.

#### Step 2 — Install the You.com Connector

The [TotalAgility connector for You.com](https://github.com/TungstenAutomationLabs/TotalAgility-connector-for-you.com-) provides three tool agents that give the Plan-Act Agent web access capabilities:

| Tool Agent | Capability |
|---|---|
| **youcom_web_search** | Real-time web search via You.com Smart Search |
| **youcom_research** | Deep research synthesising multiple web sources |
| **youcom_get_web_content** | Full text extraction from a specific URL |

1. Download the You.com connector package from the [connector repository](https://github.com/TungstenAutomationLabs/TotalAgility-connector-for-you.com-).
2. Import the package into TotalAgility using **Advanced > Import Package**.
3. Configure your You.com API key as described in the connector's README.
4. Note the **Process IDs** assigned to each of the three You.com tool processes — you will need these in the next step.

#### Step 3 — Configure the Tool Registry

The Plan-Act agent dispatches work to tools and worker agents defined in an **Agent Tool Registry** JSON file. Update the `Set Agent Registry & Reporting Data` expression activity to include the You.com tools alongside the built-in `direct_reply` tool.

Below is a sample registry configuration with the You.com agents included:

```json
{
  "agents": [
    {
      "agent_name": "direct_reply",
      "description": "Use this when the request can be answered directly from the LLM's own knowledge without needing any external tools, searches, or data lookups. Suitable for general knowledge questions, explanations, creative writing, summarisation of provided text, or conversational replies.",
      "process_id": "",
      "intent": "direct_reply"
    },
    {
      "agent_name": "youcom_web_search",
      "description": "Performs a real-time web search using You.com Smart Search to find current information, news, facts, or references from across the internet. Returns a set of ranked search result snippets with source URLs. Use this when the user needs up-to-date information, wants to verify a claim, or needs to discover authoritative sources on a topic.",
      "process_id": "<YOUR_YOUCOM_SEARCH_PROCESS_ID>",
      "intent": "sync_agent"
    },
    {
      "agent_name": "youcom_research",
      "description": "Performs deep research using You.com Research mode. Synthesises information from multiple web sources into a comprehensive, cited research summary. Use this when the user needs an in-depth analysis, a literature review, a comparison of multiple perspectives, or a thorough investigation of a complex topic — not just a quick factual lookup.",
      "process_id": "<YOUR_YOUCOM_RESEARCH_PROCESS_ID>",
      "intent": "sync_agent"
    },
    {
      "agent_name": "youcom_get_web_content",
      "description": "Retrieves and extracts the full text content from a specific web page URL using You.com's web content retrieval service. Use this when a previous step has identified a specific URL and the full page content is needed for extraction, analysis, or summarisation. Requires a URL as input.",
      "process_id": "<YOUR_YOUCOM_WEBCONTENT_PROCESS_ID>",
      "intent": "sync_agent"
    },
    {
      "agent_name": "document_analyser",
      "description": "Analyses an uploaded document (PDF, image, etc.) to extract its content, metadata, and structure. Use this when the user has attached a file and the task requires understanding or extracting information from that document.",
      "process_id": "<YOUR_DOC_ANALYSIS_PROCESS_ID>",
      "intent": "sync_agent"
    }
  ]
}
```

> **Note:** Replace the `<YOUR_..._PROCESS_ID>` placeholders with the actual TotalAgility process IDs from your imported You.com connector package.

#### Step 4 — Test the Agent

Use the following prompt to test the Plan-Act agent with the You.com tools registered above. This prompt exercises the full plan-act cycle: web search → source evaluation → content retrieval → data extraction → structured output.

> **Prompt:**
>
> *"Research the top 5 large language models released in 2025–2026 by market adoption. For each model, find the developer, release date, parameter count (if public), and primary use case. Use web search to discover the models, retrieve the content from the most authoritative source for each, and compile the results into a structured markdown table. Cite your sources."*

**Expected Plan-Act behaviour:**

1. **Step 1 — Web Search** (`youcom_web_search`): Search for "top large language models 2025 2026 by market adoption" → Evidence `#E1` (list of search results with URLs)
2. **Step 2 — Source Evaluation** (`direct_reply`): Evaluate `#E1` and select the 3–5 most authoritative URLs → Evidence `#E2` (selected URLs)
3. **Step 3 — Content Retrieval** (`youcom_get_web_content`): Retrieve full page content from each URL in `#E2` → Evidence `#E3` (raw page content)
4. **Step 4 — Data Extraction** (`direct_reply`): Extract model name, developer, release date, parameter count, and use case from `#E3` → Evidence `#E4` (structured data)
5. **Step 5 — Format Output** (`direct_reply`): Compile `#E4` into a markdown table with source citations → Final output

---

### Integration Options

The Plan-Act Agent can be invoked in multiple ways:

| Method | Description |
|---|---|
| **Generative AI Chat Control** | Map the process to a "Custom LLM" configuration under *Integration > Generative AI* and use it from a TotalAgility Form |
| **OpenAPI REST Interface** | Call via TotalAgility's REST API (e.g., from MS Teams using the sample Teams project) |
| **MCP (Model Context Protocol)** | Expose via the example MCP Proxy Server for TotalAgility |
| **Multi-Agent Composition** | Call from other TotalAgility Agents, Cases, or Processes to create "Russian Doll" style composite agent workflows |

### Key Configuration Notes

- **Tool Registry**: Update the `Set Agent Registry & Reporting Data` expression activity to point to your registry variable/file.
- **Max Loop Count**: The `At Max Loop Count?` decision node prevents runaway execution. If the loop limit is reached, the case is routed to the `Pass to Human / Error` synchronization point for human escalation.
- **Session Management**: The `Get Session & Set Seed` activity retrieves or creates a user session based on their email profile, propagating identity to all downstream agent/tool calls.
- **Error Handling**: Configure the `Pass to Human / Error` synchronization step to launch a sub-process or case for human-in-the-loop review. This must run as a separate process since the agent runs synchronously to support real-time chat responses.
- **Document Support**: If a document is attached, the `Fast Analysis of Attachments` sub-job analyses it before planning begins, making document content available to the planner and all subsequent steps.

### Additional Resources

For more agentic design patterns and reference implementations for TotalAgility, see the **Agentic Design Patterns for TotalAgility** repository:

🔗 **[Agentic Design Patterns for TotalAgility](https://github.com/TungstenAutomationLabs/Agentic_Design_Patterns_For_TotalAgility)**

This companion repository provides a catalogue of reusable agent design patterns built on TotalAgility, including:

- **Pattern overviews** — Descriptions and rationale for each agentic pattern (e.g., Tool-Use, ReAct, Plan-Act, Reflection, Multi-Agent Orchestration).
- **Reference implementations** — Importable TotalAgility packages demonstrating each pattern.
- **Prompt templates** — Reusable system and planner prompts tailored to each pattern.
- **Data models** — JSON schemas for tool registries, plan structures, and evidence tracking.
- **Integration examples** — Samples for connecting agents via REST APIs, MCP, Generative AI Chat Controls, and multi-agent composition.

Use it as a starting point for exploring which agentic patterns best fit your automation use cases, and as a library of building blocks for constructing your own TotalAgility agent workflows.

---
