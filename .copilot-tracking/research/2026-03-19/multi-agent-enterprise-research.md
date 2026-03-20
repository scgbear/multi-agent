<!-- markdownlint-disable-file -->
# Task Research: Multi-Agent Enterprise Application Presentation

Research for designing a presentation on multi-agent enterprise application architecture, covering orchestration patterns, available libraries, and inter-agent communication strategies.

## Task Implementation Requests

* Research multi-agent orchestration patterns (sequential, handoff, parallel, collaborative) including agentic workflows supported by the Microsoft Agent Framework
* Catalog .NET and Python libraries for multi-agent orchestration (LangChain, Semantic Kernel, Microsoft Agent Framework) with pattern support mapping and Azure AI Foundry compatibility
* Research Agent2Agent (A2A) communication via ASP.NET — when/why to use it and differences from in-process agent communication

## Scope and Success Criteria

* Scope: Enterprise-grade multi-agent patterns, library ecosystem (.NET + Python), Azure AI Foundry integration, A2A protocol via ASP.NET
* Assumptions:
  * Target audience is an enterprise organization already using Azure AI Foundry
  * Presentation should cover both .NET and Python ecosystems
  * Focus on production-ready, Microsoft-aligned technologies
* Success Criteria:
  * Comprehensive catalog of orchestration patterns with descriptions and use cases
  * Library comparison matrix mapping patterns to libraries
  * Azure AI Foundry compatibility assessment for each library
  * Clear guidance on A2A vs in-process communication with ASP.NET implementation details

## Outline

1. Multi-Agent Orchestration Patterns
   - Complexity Spectrum (when to use multi-agent vs simpler approaches)
   - Sequential
   - Concurrent (Parallel)
   - Handoff
   - Group Chat (Collaborative)
   - Magentic (Adaptive Planning)
   - Cross-cutting concerns (context management, reliability, security, cost, observability)
2. Library Ecosystem
   - LangChain / LangGraph (Python)
   - Semantic Kernel (.NET + Python)
   - Microsoft Agent Framework (successor to AutoGen + SK)
   - Other notable (OpenAI Agents SDK, CrewAI)
   - Pattern-to-library mapping matrix
3. Azure AI Foundry Compatibility
   - Foundry Agent Service overview
   - Library-specific integration paths
   - Compatibility matrix
   - Known concerns or gaps
4. Agent2Agent Communication
   - A2A Protocol overview (Linux Foundation, v1.0.0)
   - A2A vs MCP (complementary protocols)
   - ASP.NET implementation with Microsoft Agent Framework
   - In-process vs inter-process communication comparison
   - When and why to use A2A
   - Hybrid architecture patterns

---

## 1. Multi-Agent Orchestration Patterns

### Complexity Spectrum — Start Simple

Before adopting a multi-agent pattern, evaluate whether the scenario requires one. Microsoft's Azure Architecture guide defines a clear complexity spectrum:

| Level | Description | When to Use |
|---|---|---|
| **Direct model call** | Single LLM call with a well-crafted prompt | Classification, summarization, translation, single-step tasks |
| **Single agent with tools** | One agent that reasons and acts by selecting from tools, knowledge, APIs | Varied queries within a single domain requiring dynamic tool use |
| **Multi-agent orchestration** | Multiple specialized agents coordinate to solve a problem | Cross-functional problems, distinct security boundaries, parallel specialization |

> **Key Guidance:** Use the lowest level of complexity that reliably meets your requirements. Decision-making overhead often exceeds the benefits of breaking tasks into multiple agents.

**Source:** [Azure Architecture Center — AI Agent Orchestration Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)

### Five Standardized Patterns

Microsoft has codified five orchestration patterns in their Azure Architecture Center guidance (updated Feb 2026). These are technology-agnostic and implemented across the Microsoft Agent Framework, Semantic Kernel, and AutoGen.

#### 1.1 Sequential Orchestration

**Also known as:** Pipeline, Prompt Chaining, Linear Delegation

Chains agents in a predefined, linear order. Each agent processes the output from the previous agent, creating a pipeline of specialized transformations. The next agent is deterministically defined.

```
Input → [Agent 1] → [Agent 2] → ... → [Agent N] → Result
```

| Aspect | Details |
|---|---|
| **When to use** | Multistage processes with clear linear dependencies; progressive refinement (draft → review → polish) |
| **When to avoid** | Stages are embarrassingly parallel; workflow requires backtracking or dynamic routing |
| **Benefits** | Simple, predictable; easy to debug; clear data lineage |
| **Trade-offs** | No parallelism (latency); single point of failure cascades; all stages must be known at design time |
| **Example** | Contract generation: Template Selection → Clause Customization → Regulatory Compliance → Risk Assessment |

**Sources:** [Azure Architecture — Sequential](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#sequential-orchestration), [Semantic Kernel — Sequential](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/sequential)

#### 1.2 Concurrent Orchestration (Parallel / Fan-out/Fan-in)

**Also known as:** Parallel, Scatter-Gather, Map-Reduce

Runs multiple agents simultaneously on the same task. Each agent provides independent analysis from its unique specialization. An initiator dispatches the task and aggregates results.

```
                    ┌→ [Agent 1] → Result 1 ─┐
Input → [Initiator] ├→ [Agent 2] → Result 2 ─┼→ [Aggregated Result]
                    └→ [Agent N] → Result N ─┘
```

| Aspect | Details |
|---|---|
| **When to use** | Tasks that can run in parallel; multi-perspective analysis; voting/ensemble decisions; time-sensitive scenarios |
| **When to avoid** | Agents need cumulative context; no conflict resolution strategy; resource constraints |
| **Benefits** | Reduced latency; diverse perspectives; natural fit for voting/ensemble |
| **Trade-offs** | Resource-intensive; requires conflict resolution; no context sharing between concurrent agents |
| **Example** | Stock analysis: Fundamental + Technical + Sentiment + ESG → Combined Recommendation |

**Sources:** [Azure Architecture — Concurrent](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#concurrent-orchestration), [Semantic Kernel — Concurrent](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/concurrent)

#### 1.3 Handoff Orchestration

**Also known as:** Routing, Triage, Transfer, Dispatch, Delegation

Enables dynamic delegation between specialized agents. Each agent assesses the task and decides whether to handle it or transfer to a more appropriate agent. Only one agent is active at a time.

```
Input → [Triage Agent] ──→ [Specialist A] ──→ Result
              ├──→ [Specialist B]
              └──→ [Specialist C] → [Human Agent]
```

| Aspect | Details |
|---|---|
| **When to use** | Expertise requirements emerge during processing; customer support escalation; multi-domain problems with one-at-a-time processing |
| **When to avoid** | Appropriate agent is identifiable from initial input; multiple operations should run concurrently; risk of infinite handoff loops |
| **Benefits** | Dynamic, context-aware routing; supports human-in-the-loop escalation; deep specialization |
| **Trade-offs** | Risk of infinite handoff loops; unpredictable routing paths; only one agent active at a time |
| **Example** | Telecom support: Triage → Technical → Financial → Account Access → Human Employee |

**Sources:** [Azure Architecture — Handoff](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#handoff-orchestration), [Semantic Kernel — Handoff](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/handoff)

#### 1.4 Group Chat Orchestration (Collaborative)

**Also known as:** Roundtable, Collaborative, Multi-Agent Debate, Council

Multiple agents solve problems by participating in a shared conversation thread. A chat manager coordinates turn order and interaction modes. All agents emit output into a single accumulating thread.

Sub-pattern: **Maker-Checker Loop** — one agent creates, another evaluates, repeating until approval.

```
Input → [Chat Manager (LLM)] ←→ Accumulating Chat Thread → Result
              ├──→ [Agent 1]
              ├──→ [Agent 2]
              └──→ [Human Participant]
```

| Aspect | Details |
|---|---|
| **When to use** | Creative brainstorming; consensus-building; iterative refinement; quality assurance (maker-checker); compliance validation |
| **When to avoid** | Linear pipeline is sufficient; real-time latency requirements; more than ~3 agents (control becomes difficult) |
| **Benefits** | Transparency/auditability; natural HITL support; flexible (brainstorm, debate, formal review) |
| **Trade-offs** | Conversation loops; difficult to control with many agents; chat overhead increases latency |
| **Example** | City parks proposal: Environmental + Community + Budget agents debate a park development proposal |

**Sources:** [Azure Architecture — Group Chat](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#group-chat-orchestration), [Semantic Kernel — Group Chat](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/group-chat)

#### 1.5 Magentic Orchestration (Adaptive Planning)

**Also known as:** Dynamic Orchestration, Task-Ledger-Based, Adaptive Planning

Designed for open-ended, complex problems with no predetermined plan. A Magentic Manager coordinates specialized agents, maintaining a task ledger that is dynamically built and refined. Agents have tools that make changes in external systems. Inspired by Microsoft Research's Magentic-One system.

```
Input → [Manager Agent] ←→ Task & Progress Ledger → Result
              ├──→ [Agent 1] (Model + Knowledge)
              ├──→ [Agent 2] (Model + Tools → External Systems)
              └──→ [Agent N]
         ↻ Evaluate Goal Loop
```

| Aspect | Details |
|---|---|
| **When to use** | Complex/open-ended problems with no predetermined solution path; need for documented audit trail; agents with external system tools |
| **When to avoid** | Solution path is deterministic; low complexity; time-sensitive tasks; frequent stalls |
| **Benefits** | Handles truly complex problems; auditable plans; dynamic adaptation; combines research, reasoning, and tool execution |
| **Trade-offs** | Slow to converge; most expensive pattern; total cost difficult to predict; requires capable manager model |
| **Example** | SRE incident response: Manager coordinates Diagnostics, Infrastructure, Rollback, and Communication agents |

**Sources:** [Azure Architecture — Magentic](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#magentic-orchestration), [Microsoft Research — Magentic-One](https://www.microsoft.com/research/articles/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/)

### Pattern Comparison Matrix

| Pattern | Communication | Agent Selection | Best For | Key Risk |
|---|---|---|---|---|
| **Sequential** | Linear pipeline | Deterministic, predefined | Step-by-step refinement | Failures propagate; no parallelism |
| **Concurrent** | Parallel, independent | Deterministic or dynamic | Multi-perspective analysis; latency-sensitive | Contradictory results; resource-intensive |
| **Handoff** | Dynamic delegation, one active | Agents decide when to transfer | Tasks where right specialist emerges during processing | Infinite handoff loops |
| **Group Chat** | Conversational, shared thread | Chat manager controls turns | Consensus-building, brainstorming, maker-checker | Conversation loops; hard to control with many agents |
| **Magentic** | Plan-build-execute | Manager dynamically assigns | Open-ended problems, no predetermined path | Slow to converge; stalls on ambiguous goals |

**Source:** [Azure Architecture — Choosing a pattern](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#choosing-a-pattern)

### AutoGen/Agent Framework Team Types Mapping

| Standard Pattern | AutoGen Implementation | Semantic Kernel Class |
|---|---|---|
| Sequential | `GraphFlow` (sequential edges) or `RoundRobinGroupChat` | `SequentialOrchestration` |
| Concurrent | `GraphFlow` (fan-out/fan-in edges) | `ConcurrentOrchestration` |
| Handoff | `Swarm` | `HandoffOrchestration` |
| Group Chat | `SelectorGroupChat` or `RoundRobinGroupChat` | `GroupChatOrchestration` |
| Magentic | `MagenticOneGroupChat` | `MagenticOrchestration` |
| Workflow (Graph) | `GraphFlow` (unique—conditional branching, loops, activation groups) | N/A |

**Sources:** [AutoGen AgentChat](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html), [SK Agent Orchestration](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/)

### Cross-Cutting Implementation Concerns

Key considerations across all patterns from the Azure Architecture guide:

* **Context/State Management:** Context windows grow rapidly—use compaction (summarization, selective pruning) between agents. Persist shared state externally for long-running tasks.
* **Reliability:** Implement timeout/retry mechanisms, circuit breaker patterns, output validation before passing to next agent, and checkpoint features for recovery.
* **Security:** Authentication between agents, least privilege, user identity propagation, content safety guardrails at multiple points.
* **Cost Optimization:** Assign each agent a model matching task complexity (not every agent needs the most capable model). Monitor token consumption per agent.
* **Observability:** Instrument all agent operations and handoffs. Use LLM-as-judge evaluations, not exact-match assertions.
* **Human-in-the-Loop:** Identify mandatory vs optional human input points. Persist state at HITL checkpoints. Scope HITL gates to sensitive operations.
* **Combining Patterns:** Applications may require multiple patterns—sequential for initial processing, then concurrent for parallelizable analysis. Match the pattern to each stage.

**Source:** [Azure Architecture — Implementation considerations](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#implementation-considerations)

---

## 2. Library Ecosystem

### 2.1 LangChain / LangGraph (Python)

**LangGraph** is a low-level graph orchestration framework from LangChain Inc. for building stateful, long-running agents.

| Attribute | Value |
|---|---|
| **Version** | 1.1.3 (March 2026) |
| **Languages** | Python (primary), JavaScript/TypeScript |
| **.NET** | Not available |
| **Maturity** | GA / Production, widely adopted |
| **Architecture** | `StateGraph` with conditional edges, `Send()` for fan-out, subgraphs for composition |

**Pattern support:** Sequential (edges), Parallel (`Send()`), Supervisor (conditional edges + LLM), Handoff (conditional routing), Reflection (loop edges), Swarm (state transitions), Hierarchical (subgraphs), Map-Reduce, Human-in-the-loop (interrupt checkpoints).

LangGraph provides low-level primitives to compose patterns rather than pre-built orchestration classes.

**Key strengths:** Durable execution with checkpointing, streaming, LangSmith integration for observability, LangGraph Cloud for deployment, large community.

**Azure integration:** Via `langchain-azure-ai` package (co-developed with LangChain). Supports `AzureAIOpenAIApiChatModel` for all OpenAI-compatible Foundry models (including Mistral, DeepSeek, Llama), `AgentServiceFactory` for Foundry Agent Service nodes, and hosted agent deployment via `azure-ai-agentserver-langgraph`.

**Sources:** [LangGraph Docs](https://docs.langchain.com/oss/python/langgraph/overview), [PyPI](https://pypi.org/project/langgraph/)

### 2.2 Semantic Kernel (.NET + Python)

Microsoft's open-source SDK for AI application development with a dedicated Agent Framework providing multi-agent orchestration.

| Attribute | Value |
|---|---|
| **Core** | GA |
| **Agent Orchestration** | **Experimental** (prerelease) |
| **Languages** | C# (primary), Python, Java (partial) |
| **Maturity** | Being superseded by Microsoft Agent Framework for new development |

**Five built-in orchestration patterns** with unified API:

| Pattern | Class |
|---|---|
| Concurrent | `ConcurrentOrchestration` |
| Sequential | `SequentialOrchestration` |
| Handoff | `HandoffOrchestration` |
| Group Chat | `GroupChatOrchestration` |
| Magentic | `MagenticOrchestration` |

**C# example:**

```csharp
SequentialOrchestration orchestration = new(agentA, agentB);
InProcessRuntime runtime = new();
await runtime.StartAsync();
OrchestrationResult<string> result = await orchestration.InvokeAsync(task, runtime);
string text = await result.GetValueAsync();
```

**Agent types:** `ChatCompletionAgent`, `AzureAIAgent` (Foundry), `OpenAIAssistantAgent`, `OpenAIResponsesAgent`, `CopilotStudioAgent`.

**Azure Foundry integration:** `AzureAIAgent` wraps Foundry Agent Service with full tool support (file search, code interpreter, Bing, Azure Functions, OpenAPI).

**Sources:** [SK Agent Orchestration](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/), [GitHub](https://github.com/microsoft/semantic-kernel)

### 2.3 Microsoft Agent Framework (Successor to AutoGen + SK)

The **next-generation SDK** that unifies AutoGen's simple abstractions with Semantic Kernel's enterprise features. Created by the same teams.

| Attribute | Value |
|---|---|
| **Status** | **Public preview** |
| **Languages** | Python, C# |
| **Predecessor** | AutoGen (research) + Semantic Kernel (enterprise) |
| **Key innovation** | Data-flow-based `Workflow` model with typed edges, checkpointing, built-in HITL |

**Naming history:**

| Name | Status |
|---|---|
| AutoGen | Original (Microsoft Research); still on GitHub at `microsoft/autogen` |
| AG2 | Community fork; not Microsoft-endorsed |
| Microsoft Agent Framework | Current official name; direct successor |

**AutoGen team types (current):** `RoundRobinGroupChat`, `SelectorGroupChat`, `Swarm`, `MagenticOneGroupChat`, `GraphFlow` (experimental).

**Agent Framework improvements over AutoGen:**

| Feature | AutoGen | Agent Framework |
|---|---|---|
| Flow Type | Control flow (edges are transitions) | Data flow (edges route messages) |
| Node Types | Agents only | Agents, functions, sub-workflows |
| Type Safety | Limited | Strong typing throughout |
| Checkpointing | Not built-in | Built-in |
| Human-in-the-loop | Custom implementation | Built-in `ctx.request_info()` |

**Azure Foundry integration:** `AzureOpenAIChatClient` for direct model access, `AzureAIAgentClient` for Foundry Agent Service, hosted agent deployment support, A2A protocol support.

**Sources:** [Agent Framework Overview](https://learn.microsoft.com/agent-framework/overview/), [AutoGen GitHub](https://github.com/microsoft/autogen), [Migration from AutoGen](https://learn.microsoft.com/agent-framework/migration-guide/from-autogen/), [Migration from SK](https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/)

### 2.4 Other Notable Libraries

**OpenAI Agents SDK** (v0.12.5): Lightweight, production-ready. Native handoffs, agents-as-tools, guardrails. Python + JS/TS only. No native Azure integration (use LiteLLM). [Docs](https://openai.github.io/openai-agents-python/)

**CrewAI** (v1.11.0): Standalone Python framework with role-playing Crews + event-driven Flows. Sequential and hierarchical processes. YAML-based config. Python only. [Docs](https://docs.crewai.com/)

### 2.5 Pattern-to-Library Mapping Matrix

| Pattern | LangGraph | Semantic Kernel | AutoGen | Agent Framework | OpenAI Agents SDK | CrewAI |
|---|---|---|---|---|---|---|
| **Sequential** | ✅ (edges) | ✅ (`SequentialOrchestration`) | ✅ (`RoundRobinGroupChat`) | ✅ | ✅ (code) | ✅ |
| **Concurrent** | ✅ (`Send()`) | ✅ (`ConcurrentOrchestration`) | Partial | ✅ | ✅ (`asyncio`) | Partial |
| **Handoff** | ✅ (conditional) | ✅ (`HandoffOrchestration`) | ✅ (`Swarm`) | ✅ | ✅ (native) | ❌ |
| **Group Chat** | ✅ (shared state) | ✅ (`GroupChatOrchestration`) | ✅ (multiple types) | ✅ | ❌ | ❌ |
| **Magentic** | ❌ (build custom) | ✅ (`MagenticOrchestration`) | ✅ (`MagenticOneGroupChat`) | ✅ | ❌ | ❌ |
| **Graph/DAG** | ✅ (native) | ❌ | Experimental (`GraphFlow`) | ✅ (`Workflow`) | ❌ | ✅ (Flows) |
| **HITL** | ✅ (interrupts) | ✅ | ✅ (`UserProxyAgent`) | ✅ (built-in) | ✅ | ✅ |

### Language & Maturity Matrix

| Library | Python | C# | JS/TS | Maturity | Multi-Agent Status |
|---|---|---|---|---|---|
| **LangGraph** | ✅ | ❌ | ✅ | GA | Mature, widely adopted |
| **Semantic Kernel** | ✅ | ✅ | ❌ | Core GA; Agents Experimental | Being superseded |
| **AutoGen** | ✅ | Partial | ❌ | Research → Production | Migrating to Agent Framework |
| **Agent Framework** | ✅ | ✅ | ❌ | Public Preview | Active development |
| **OpenAI Agents SDK** | ✅ | ❌ | ✅ | Production-ready | Mature handoff/tool |
| **CrewAI** | ✅ | ❌ | ❌ | GA | Mature Crews + Flows |

---

## 3. Azure AI Foundry Compatibility

### 3.1 Foundry Agent Service Overview

Microsoft Foundry Agent Service is a fully managed platform (GA) for building, deploying, and scaling AI agents.

**Agent types:**

| Type | Code Required | Best For |
|---|---|---|
| **Prompt agents** | No | Prototyping, simple tasks |
| **Workflow agents** (preview) | No (YAML optional) | Multi-step automation |
| **Hosted agents** (preview) | Yes (containerized) | Full control, custom frameworks |

**Supported tools:** Web search, file search, memory, code interpreter, MCP servers, Azure Functions, Logic Apps, OpenAPI, A2A protocol, browser automation, image generation.

**Model catalog:** GPT-4o, GPT-4.1, GPT-5, o3, o4-mini, Llama-4-Scout/Maverick, Mistral-Large-3, DeepSeek-R1, Grok-4, and more. All OpenAI-compatible API models work.

### 3.2 Compatibility Matrix

| Feature | Semantic Kernel | Agent Framework | LangChain/LangGraph |
|---|---|---|---|
| **Azure OpenAI models** | Full (`ChatCompletionAgent`, `AzureAIAgent`) | Full (`AzureOpenAIChatClient`) | Full (`AzureAIOpenAIApiChatModel`) |
| **Non-OpenAI catalog models** | Via `AzureAIAgent` (server-side) | Via `AzureAIAgentClient` | Full (`init_chat_model("azure_ai:Mistral-Large-3")`) |
| **Foundry Agent Service** | `AzureAIAgent` | `AzureAIAgentClient`, hosted deployment | `AgentServiceFactory`, hosted deployment |
| **Multi-agent orchestration** | 5 patterns (SDK) | Workflow (graph-based) | LangGraph (graph-based) |
| **Hosted agent deployment** | Not directly | Yes (Python, C#) | Yes (Python only) |
| **Built-in tools** | Via `AzureAIAgent` | Via `AzureAIAgentClient` | Via `AgentServiceFactory` |
| **Observability** | OpenTelemetry | OpenTelemetry | `AzureAIOpenTelemetryTracer` → App Insights |
| **Python** | ✅ (GA) | ✅ (preview) | ✅ (GA) |
| **C#** | ✅ (experimental) | ✅ (preview) | ❌ |

### 3.3 Friction Assessment for Existing Azure AI Foundry Users

**Low friction overall:** All three primary frameworks have first-class Azure AI Foundry integration. Foundry's OpenAI-compatible API surface means most frameworks "just work" with deployed models. Microsoft actively maintains integrations for all three.

**Potential friction points:**

| Issue | Impact | Mitigation |
|---|---|---|
| Framework choice complexity | Microsoft consolidating toward Agent Framework | Follow Microsoft guidance; start with SK or Agent Framework |
| Preview/experimental status | Agent Framework and SK `AzureAIAgent` are maturing | Pin package versions; plan for API changes |
| Model-specific tool-calling gaps | Not all catalog models support function calling | Check model capabilities before building agents |
| Foundry classic deprecation (March 2027) | Classic portal and agents retiring | Migrate to new Foundry |
| `langchain-azure-ai` v1 vs v2 | Wrong version breaks Foundry classic vs new | Use default (v2) for new Foundry |
| LangGraph hosted agents Python-only | No C# LangGraph on Foundry | Use Agent Framework for C# hosted agents |
| Connected Agents max depth 2 | Cannot create deep agent hierarchies | Flatten structures; use orchestration frameworks for deeper nesting |

**Sources:** [Foundry Agent Service Overview](https://learn.microsoft.com/azure/foundry/agents/overview), [SK AzureAIAgent](https://learn.microsoft.com/semantic-kernel/frameworks/agent/agent-types/azure-ai-agent), [LangChain + Foundry](https://learn.microsoft.com/azure/foundry/how-to/develop/langchain)

---

## 4. Agent2Agent (A2A) Communication

### 4.1 Protocol Overview

The **Agent2Agent (A2A) Protocol** is an open standard (v1.0.0) for agent-to-agent communication. Originally developed by Google, now governed by the Linux Foundation.

**Core concepts:**

| Concept | Description |
|---|---|
| **Agent Card** | JSON metadata document at `/.well-known/agent-card.json` describing identity, capabilities, skills |
| **Task** | Fundamental work unit with lifecycle: `submitted` → `working` → `completed`/`failed`/`canceled` |
| **Message** | Communication turn with `role` ("user"/"agent") containing Parts |
| **Part** | Content unit: `text`, `raw` (binary), `url` (file reference), or `data` (structured JSON) |
| **Artifact** | Output generated by the agent (documents, images, structured data) |
| **Streaming** | Real-time updates via SSE or gRPC streaming |

**Protocol bindings:** JSON-RPC 2.0, gRPC, HTTP+JSON/REST.

**Source:** [A2A Protocol Specification v1.0](https://a2a-protocol.org/latest/specification/)

### 4.2 A2A vs MCP — Complementary Protocols

| Aspect | A2A | MCP |
|---|---|---|
| **Focus** | Agent-to-agent communication | Agent-to-tool/resource communication |
| **Analogy** | The "public internet" between agents | The "how-to" for using a specific tool |
| **Interaction** | Task-based, multi-turn, potentially long-running | Request/response, function call style |

They work together: An A2A Client requests an A2A Server agent to perform a task. The Server agent may internally use MCP to invoke tools. MCP is inward-facing (tools); A2A is outward-facing (agents).

### 4.3 ASP.NET Implementation

Microsoft provides first-class A2A support through the Microsoft Agent Framework hosting libraries:

**Packages:**

```bash
dotnet add package Microsoft.Agents.AI.Hosting.A2A.AspNetCore --prerelease
```

**Minimal A2A server:**

```csharp
var pirateAgent = builder.AddAIAgent("pirate",
    instructions: "You are a pirate. Speak like a pirate.",
    description: "An agent that speaks like a pirate.");

builder.Services.AddA2AServer();
var app = builder.Build();
app.MapA2AServer();
app.Run();
```

**Multiple agents:**

```csharp
var mathAgent = builder.AddAIAgent("math", instructions: "You are a math expert.");
var scienceAgent = builder.AddAIAgent("science", instructions: "You are a science expert.");

app.MapA2A(mathAgent, "/a2a/math");
app.MapA2A(scienceAgent, "/a2a/science");
```

**Workflows as A2A agents (hybrid in-process + A2A):**

```csharp
var workflowAsAgent = builder
    .AddWorkflow("science-workflow", (sp, key) => { ... })
    .AddAsAIAgent();
app.MapA2A(workflowAsAgent, "/a2a/science-workflow");
```

**Key APIs:** `builder.AddAIAgent()`, `builder.Services.AddA2AServer()`, `app.MapA2AServer()`, `app.MapA2A()`, agent middleware pipeline.

**Sources:** [Agent Framework A2A Integration](https://learn.microsoft.com/agent-framework/integrations/a2a), [ASP.NET Core Hosting](https://learn.microsoft.com/agent-framework/get-started/hosting)

### 4.4 In-Process vs A2A Communication

| Aspect | In-Process (SK/Agent Framework) | A2A (Inter-Process) |
|---|---|---|
| **Latency** | Nanoseconds (direct calls) | Milliseconds to seconds (HTTP/gRPC) |
| **Reliability** | Single process failure = all agents fail | Independent failure domains |
| **Scalability** | Scales as one unit | Independently scalable per agent |
| **Framework** | Same framework/language required | Any framework, any language |
| **Deployment** | Single deployment unit | Independent deployment per agent |
| **State sharing** | Shared memory | Explicit via A2A messages/artifacts |
| **Security** | In-process trust boundary | Network-level auth (OAuth, mTLS, API keys) |
| **Debugging** | Standard debugger | Distributed tracing required |
| **Team ownership** | Same team | Different teams/orgs possible |

### 4.5 When to Use Each

**Use in-process when:** Same team, same technology stack, low-latency requirements, simple deployment, shared state essential, prototyping.

**Use A2A when:** Different teams/organizations own agents, polyglot systems (.NET + Python + Java), independent deployment/scaling, security boundaries between agents, long-running async tasks, enterprise governance, agents exposed as reusable services.

### 4.6 Hybrid Architecture

A2A and in-process orchestration combine naturally. A workflow within a single ASP.NET service orchestrates in-process agents, then exposes the entire workflow as a single A2A endpoint:

```
[User] → [Frontend App]
              ↓
     [Orchestrator Agent (ASP.NET)]
       ├── [In-Process Agent 1] (SK/Agent Framework)
       ├── [In-Process Agent 2] (SK/Agent Framework)
       └── A2A →→→ [External Agent A (Python)]
           A2A →→→ [External Agent B (Java)]
           A2A →→→ [Vendor Agent C (any)]
```

**Integrations:** Copilot Studio can connect to external A2A agents. Azure Foundry supports A2A as a tool type (`A2APreviewTool`).

**Sources:** [Copilot Studio A2A](https://learn.microsoft.com/microsoft-copilot-studio/add-agent-agent-to-agent), [Foundry A2A Tool](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/agent-to-agent)

---

## Key Discoveries

### 1. Microsoft Is Consolidating to One Framework

Microsoft Agent Framework is the direct successor to **both** AutoGen and Semantic Kernel's agent capabilities, created by the same teams. Migration guides exist for both paths. This is the strategic direction for new enterprise development.

### 2. Five Standardized Patterns Are Now Official

Microsoft codified five technology-agnostic orchestration patterns in Azure Architecture Center (Feb 2026): Sequential, Concurrent, Handoff, Group Chat, and Magentic. All are implemented across SK, AutoGen, and Agent Framework.

### 3. Azure AI Foundry Compatibility Is Strong Across All Frameworks

All three primary frameworks (SK, Agent Framework, LangChain/LangGraph) have first-class Foundry integration. The OpenAI-compatible API surface minimizes friction. An organization on Azure AI Foundry has no vendor lock-in concerns for library choice.

### 4. A2A Is the Standard for Cross-Service Agent Communication

The A2A protocol (v1.0.0, Linux Foundation) is the emerging standard for inter-agent communication. Microsoft provides first-class ASP.NET support. A2A and MCP are complementary: MCP for tools (inward), A2A for agents (outward).

### 5. In-Process + A2A Hybrid Is the Recommended Enterprise Architecture

The recommended approach is to use in-process orchestration (SK or Agent Framework) within each service, then expose services via A2A for cross-team/cross-language/cross-organization communication.

### 6. Everything Is in Preview/Experimental

Agent Framework is in public preview. SK Agent Orchestration is experimental. A2A ASP.NET packages are prerelease. Only LangGraph's core is GA. Plan for API changes.

### 7. Agent Framework Workflow API Uses a Superstep Execution Model

The Workflow API uses a modified Pregel (BSP) model with superstep-based processing. All triggered executors within a superstep run concurrently with synchronization barriers between supersteps. This provides deterministic execution and reliable checkpointing. Five built-in orchestration builders (`SequentialBuilder`, `ConcurrentBuilder`, `HandoffBuilder`, `GroupChatBuilder`, `MagenticBuilder`) generate `WorkflowBuilder` graphs internally.

### 8. Foundry Provides a Three-Tier Graduation Path

Foundry offers Prompt agents → Workflow agents → Hosted agents as a clear progression. The VS Code "Generate Code" button converts Foundry YAML workflows to Agent Framework code using GitHub Copilot, enabling teams to start no-code and graduate to code.

### 9. Durable Agents and A2A Are Currently Separate Hosting Paths

Azure Functions Durable Agents use their own HTTP endpoint pattern (`/api/agents/{name}/run`), not the A2A protocol. To expose agents via A2A, use ASP.NET Core hosting. Durable Agents can consume external A2A agents as tools but cannot natively serve the A2A protocol. The Flex Consumption plan provides optimal serverless agent hosting with scale-to-zero and always-ready instances.

---

## 5. Microsoft Agent Framework Workflow API (Deep-Dive)

### 5.1 Architecture: Data-Flow Graph Model

The Agent Framework `Workflow` is a **directed graph** of executors connected by typed edges. It uses a modified **Pregel (BSP — Bulk Synchronous Parallel)** superstep execution model:

1. Collect all pending messages from the previous superstep
2. Route messages to target executors based on edge definitions
3. Run all target executors **concurrently** within the superstep (synchronization barrier)
4. Queue new messages for the next superstep

This provides deterministic execution, reliable checkpointing at superstep boundaries, and no race conditions between supersteps.

| Aspect | AutoGen GraphFlow | Agent Framework Workflow |
|---|---|---|
| **Flow Type** | Control flow (edges are transitions) | Data flow (edges route typed messages) |
| **Node Types** | Agents only | Agents, functions, sub-workflows |
| **Activation** | Message broadcast to all agents | Edge-based activation (only targeted executors run) |
| **Type Safety** | Limited | Strong typing throughout |
| **Composability** | Limited | Highly composable (sub-workflows as nodes) |

**Sources:** [Agent Framework Workflows](https://learn.microsoft.com/en-us/agent-framework/workflows/workflows), [AutoGen Migration Guide](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/)

### 5.2 WorkflowBuilder API

Fluent builder in both Python and C# — incrementally add executors and edges, then build an immutable `Workflow`.

**Python:**

```python
from agent_framework import WorkflowBuilder, Executor, WorkflowContext, handler

class UpperCaseExecutor(Executor):
    @handler
    async def process(self, text: str, ctx: WorkflowContext[str]) -> None:
        await ctx.send_message(text.upper())

class ReverseExecutor(Executor):
    @handler
    async def process(self, text: str, ctx: WorkflowContext[Never, str]) -> None:
        await ctx.yield_output(text[::-1])

workflow = (
    WorkflowBuilder(start_executor=UpperCaseExecutor(id="upper"))
    .add_edge(upper, reverse)
    .build()
)
events = await workflow.run("hello")  # ['OLLEH']
```

**C#:**

```csharp
var processor = new DataProcessor();
var validator = new Validator();
var formatter = new Formatter();

WorkflowBuilder builder = new(processor);
builder.AddEdge(processor, validator);
builder.AddEdge(validator, formatter);
var workflow = builder.Build();
```

### 5.3 Executors (Node Types)

Three types of executors:

| Type | Description | Python | C# |
|---|---|---|---|
| **Custom logic** | Process data, call APIs, transform messages | `@handler` decorator on `Executor` subclass | `[MessageHandler]` attribute with source generation |
| **AI agents** | LLM-powered; auto-wrapped as executors | Pass agent directly to builder | Pass agent directly to builder |
| **Sub-workflows** | Nested workflows via `WorkflowExecutor` | `WorkflowExecutor(workflow=sub)` | Same pattern |

**Agents as executors (C#):**

```csharp
var workflow = new WorkflowBuilder(frenchAgent)
    .AddEdge(frenchAgent, spanishAgent)
    .AddEdge(spanishAgent, englishAgent)
    .Build();
```

**Function-based executors (C#):**

```csharp
Func<string, string> uppercaseFunc = s => s.ToUpperInvariant();
var uppercase = uppercaseFunc.BindExecutor("UppercaseExecutor");
```

**`WorkflowContext` methods:** `send_message()` (route to connected executors), `yield_output()` (produce workflow output), `set_state()` / `get_state()` (shared state), `request_info()` (pause for human input).

### 5.4 Edge Types and Routing

| Edge Type | Description | Use Case |
|---|---|---|
| **Direct** | One-to-one connection | Linear pipelines |
| **Conditional** | Edge with predicate function | Binary routing (if/else) |
| **Switch-Case** | Route to different executors by condition | Multi-branch routing |
| **Fan-Out** | One executor → multiple targets (with optional selector) | Parallel processing |
| **Fan-In (Barrier)** | Multiple executors → single target (waits for all) | Aggregation |

**Conditional edges (C#):**

```csharp
builder.AddEdge(spamDetector, emailAssistant, condition: msg => !(msg as DetectionResult).IsSpam);
builder.AddEdge(spamDetector, handleSpam, condition: msg => (msg as DetectionResult).IsSpam);
```

**Fan-out/Fan-in (C#):**

```csharp
builder.AddFanOutEdge(source, [worker1, worker2, worker3]);
builder.AddFanInBarrierEdge(sources: [worker1, worker2, worker3], target: aggregator);
```

**Chain helper (C#):** `builder.AddChain(source, [step1, step2, step3]);`

**Source:** [Agent Framework Edges](https://learn.microsoft.com/en-us/agent-framework/workflows/edges)

### 5.5 Checkpointing

Built-in `FileCheckpointStorage` captures executor state, shared state, message queues, and workflow position at superstep boundaries.

```python
from agent_framework import FileCheckpointStorage, WorkflowBuilder

checkpoint_storage = FileCheckpointStorage(storage_path="./checkpoints")
workflow = WorkflowBuilder(
    start_executor=processing_executor,
    checkpoint_storage=checkpoint_storage
).add_edge(processing_executor, finalize_executor).build()

# Resume from checkpoint
async for event in workflow.run_stream(checkpoint_id=chosen_id, checkpoint_storage=checkpoint_storage):
    print(event)
```

**Source:** [Agent Framework State Management](https://learn.microsoft.com/en-us/agent-framework/workflows/state)

### 5.6 Human-in-the-Loop

Built-in `ctx.request_info()` pauses the workflow; `@response_handler` processes the human response. Works seamlessly with checkpointing — workflows survive server restarts while waiting for human input.

```python
class ReviewerExecutor(Executor):
    @handler
    async def review(self, content: str, ctx: WorkflowContext) -> None:
        await ctx.request_info(request_data=ApprovalRequest(content=content), response_type=str)

    @response_handler
    async def handle_response(self, request: ApprovalRequest, decision: str, ctx: WorkflowContext) -> None:
        if decision.lower() == "approved":
            await ctx.yield_output(f"APPROVED: {request.content}")
```

### 5.7 Built-in Orchestration Pattern Builders

The `agent_framework.orchestrations` package provides pre-built patterns that internally construct `WorkflowBuilder` graphs:

| Builder | Pattern | Description |
|---|---|---|
| `SequentialBuilder` | Sequential | Chains agents in order with shared conversation context |
| `ConcurrentBuilder` | Concurrent | Fan-out to agents, aggregate with optional custom aggregator |
| `HandoffBuilder` | Handoff | Decentralized agent routing via tool-based handoff decisions |
| `GroupChatBuilder` | Group Chat | Orchestrator-directed conversation with pluggable speaker selection |
| `MagenticBuilder` | Magentic | LLM manager plans, selects agents, monitors progress, replans |

```python
# Sequential
workflow = SequentialBuilder(participants=[agent1, agent2, agent3]).build()

# Concurrent with aggregator
workflow = ConcurrentBuilder(participants=[a1, a2]).with_aggregator(summarize).build()

# Handoff
workflow = HandoffBuilder().participants([triage, billing, support]).with_start_agent(triage).build()

# Magentic
workflow = MagenticBuilder(participants=[researcher, coder], manager_agent=manager,
                           max_round_count=10, enable_plan_review=True).build()
```

**Sources:** [Agent Framework Orchestrations](https://learn.microsoft.com/agent-framework/workflows/orchestrations/), [GitHub Orchestrations](https://github.com/microsoft/agent-framework/tree/main/python/packages/orchestrations/)

---

## 6. Foundry Workflow Agents vs Code-Based Frameworks

### 6.1 What Are Foundry Workflow Agents?

Workflow agents are the **official replacement for deprecated Connected Agents** in Microsoft Foundry Agent Service. They orchestrate sequences of actions or coordinate multiple agents using declarative definitions. Defined via:

1. **Foundry Portal Visual Designer** — node-based workflow builder with templates
2. **YAML Editor** — toggle "YAML Visualizer View" in portal (bidirectional sync)
3. **VS Code for the Web** — YAML + visual graph side-by-side

**Status:** Preview. Three templates: Sequential, Group Chat, Human-in-the-Loop.

**Node types:** Agent (invoke existing agents), Logic (if/else, go-to, for-each), Data Transformation (variables, parse), Basic Chat (messages, questions).

**Expression language:** Power Fx (Excel-like formulas) for variables and conditions.

**Sources:** [Foundry Workflow Agents](https://learn.microsoft.com/azure/foundry/agents/concepts/workflow), [Foundry Agent Service Overview](https://learn.microsoft.com/azure/foundry/agents/overview)

### 6.2 Three-Tier Graduation Path

Foundry provides a clear progression: start simple, graduate to code as complexity grows:

| Tier | Type | Code Required | Patterns | Best For |
|---|---|---|---|---|
| 1 | **Prompt agents** (GA) | No | Single agent | Prototyping, simple tasks |
| 2 | **Workflow agents** (preview) | No (YAML optional) | Sequential, Group Chat, HITL | Multi-step automation, non-developers |
| 3 | **Hosted agents** (preview) | Yes (containerized) | All 5+ patterns | Full control, custom frameworks |

**Key bridge:** The VS Code "Generate Code" button converts Foundry YAML workflows to Agent Framework code (Python/C#) using GitHub Copilot — enabling teams to start no-code and graduate to code.

### 6.3 Comparison Matrix

| Dimension | Foundry Workflow Agents | Agent Framework (Code) | Agent Framework (Declarative YAML) | Semantic Kernel |
|---|---|---|---|---|
| **Code Required** | No | Yes (C#/Python) | YAML + host code | Yes (C#/Python) |
| **Hosting** | Fully managed | Self-hosted / Container Apps | Self-hosted / hosted agents | Self-hosted |
| **Patterns** | 3 (Sequential, Group Chat, HITL) | 5+ (all) | Sequential, Conditional, Loops, HITL | 5 (all, experimental) |
| **Graph Flexibility** | Linear + branching | Full directed graph | Action sequences with control flow | Pattern-based |
| **Checkpointing** | Managed (no user control) | Full (in-memory, file, custom) | `CheckpointManager` | None built-in |
| **HITL** | Built-in (portal) | Built-in (`request_info`) | `RequestExternalInput`, `Question` | Some patterns |
| **Visual Designer** | Yes | No | VS Code graph view | No |
| **Cost** | Token-only (no compute charge) | Azure compute + tokens | Compute + tokens (or hosted) | Compute + tokens |

### 6.4 Agent Framework Declarative Workflows

The Agent Framework's `DeclarativeWorkflowBuilder` provides a richer YAML schema than the Foundry portal — 30+ action types including `InvokeMcpTool`, `InvokeFunctionTool`, `ConditionGroup`, `Foreach`, `GotoAction`, with full checkpointing support. Both use Power Fx expressions.

### 6.5 When to Use Each

| Scenario | Recommended |
|---|---|
| Rapid prototyping | **Foundry Workflow Agents** |
| Non-developers creating workflows | **Foundry Workflow Agents** |
| Complex custom logic or dynamic graphs | **Agent Framework (Code)** |
| Workflows that change frequently without code | **Agent Framework (Declarative YAML)** |
| Maximum flexibility and control | **Agent Framework (Code)** or **LangGraph** |
| Start no-code, graduate to code | **Foundry → "Generate Code" → Agent Framework** |
| Checkpointing and fault tolerance | **Agent Framework** |

**Sources:** [Foundry Workflow Agents](https://learn.microsoft.com/azure/foundry/agents/concepts/workflow), [Agent Framework Declarative Workflows](https://learn.microsoft.com/agent-framework/workflows/declarative)

---

## 7. Azure Functions Durable Agents (Serverless Hosting)

### 7.1 What Are Durable Agents?

**Durable Agents** are AI agents hosted in Azure Functions using the Durable Task extension. They combine the Agent Framework's `AIAgent` abstraction with Durable Functions to provide:

* **Automatic state persistence** — conversation history stored durably across invocations
* **Failure recovery** — resume without losing context after crashes
* **Auto-scaling** — including scale-to-zero (pay-per-execution)
* **Deterministic multi-agent orchestrations** — sequential, parallel, conditional, HITL

**Package:** `Microsoft.Agents.AI.Hosting.AzureFunctions` (NuGet, prerelease)

**Core API:**

```csharp
using IHost app = FunctionsApplication
    .CreateBuilder(args)
    .ConfigureFunctionsWebApplication()
    .ConfigureDurableAgents(options => options.AddAIAgent(agent))
    .Build();
app.Run();
```

Each agent gets an HTTP endpoint at `/api/agents/{AgentName}/run` with thread continuity via `thread_id` query parameter.

**Source:** [Agent Framework Azure Functions Integration](https://learn.microsoft.com/en-us/agent-framework/integrations/azure-functions)

### 7.2 Orchestration Patterns in Durable Functions

**Sequential chaining:**

```csharp
[Function(nameof(SpamWorkflow))]
public static async Task<string> SpamWorkflow([OrchestrationTrigger] TaskOrchestrationContext context)
{
    DurableAIAgent spamAgent = context.GetAgent("SpamDetector");
    AgentResponse<DetectionResult> result = await spamAgent.RunAsync<DetectionResult>(input);

    if (result.Result.IsSpam) return "Spam detected";

    DurableAIAgent emailAgent = context.GetAgent("EmailAssistant");
    AgentResponse<EmailResponse> response = await emailAgent.RunAsync<EmailResponse>(input);
    return response.Result.Response;
}
```

**Parallel fan-out/fan-in:**

```csharp
DurableAIAgent french = context.GetAgent("FrenchTranslator");
DurableAIAgent spanish = context.GetAgent("SpanishTranslator");

Task<AgentResponse<TextResponse>> t1 = french.RunAsync<TextResponse>(text);
Task<AgentResponse<TextResponse>> t2 = spanish.RunAsync<TextResponse>(text);
await Task.WhenAll(t1, t2);  // Both run concurrently, checkpointed
```

**Human-in-the-loop:**

```csharp
var approval = await context.WaitForExternalEvent<HumanApprovalResponse>(
    "ApprovalDecision", timeout: TimeSpan.FromHours(24));
// Zero compute cost while waiting — orchestration state persisted by DTS
```

### 7.3 A2A and Durable Agents — Currently Separate Paths

**Critical finding:** A2A protocol (`Microsoft.Agents.AI.Hosting.A2A.AspNetCore`) and Durable Agents (`Microsoft.Agents.AI.Hosting.AzureFunctions`) are currently **separate hosting integrations**. There is no built-in "A2A over Azure Functions."

* Durable Agents expose `/api/agents/{name}/run` endpoints (not A2A protocol)
* A2A requires ASP.NET Core middleware (`MapA2A()`)
* **Workaround:** Durable Agents can *consume* external A2A agents as tools via `A2ACardResolver` and `AgentCard.AsAIAgent()`

### 7.4 Hosting Decision Matrix

| Scenario | Recommended Host |
|---|---|
| Multi-agent A2A interop | ASP.NET Core + A2A |
| Serverless single/multi-agent | Azure Functions (Durable) |
| Complex long-running workflows (days/weeks) | Azure Functions (Durable) |
| Human-in-the-loop approval (zero cost while waiting) | Azure Functions (Durable) |
| Low-latency always-on API | App Service / Container Apps |
| Managed service (no infra) | Foundry Agent Service |

**Recommended plan:** **Flex Consumption** — scale to 1,000 instances or zero, always-ready instances for cold start mitigation, per-function scaling, VNet integration.

**Durable Task Scheduler (DTS):** Recommended backend — managed service with gRPC connectivity, internal state management, built-in dashboard, and local Docker emulator for development.

### 7.5 Limitations

* A2A protocol not directly available from Azure Functions (separate hosting path)
* 1 MB max payload for orchestrator inputs/outputs
* Flex Consumption: Linux only, 30-second host init timeout
* Cold starts on consumption plan (mitigate with always-ready instances)
* DTS Consumption SKU: max 10 schedulers, 5 task hubs per region

**Eight official samples** available covering single agent, chaining, concurrency, conditionals, HITL, long-running tools, MCP tool exposure, and reliable streaming.

**Sources:** [Agent Framework Azure Functions](https://learn.microsoft.com/en-us/agent-framework/integrations/azure-functions), [Durable Task Scheduler](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler), [GitHub Samples](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions)

---

## Potential Next Research

* LangGraph `Command` API for multi-agent routing
* A2A authentication patterns (OAuth client credentials for M2M)
* Performance benchmarks comparing frameworks
* Model-by-model tool-calling support matrix for Foundry catalog
* Agent Framework `DeclarativeWorkflowBuilder` YAML schema details
* A2A protocol support roadmap for Azure Functions
* Container Apps hosting as middle ground between Functions and App Service

---

## Research Executed

### Subagent Research Documents

1. [orchestration-patterns-research.md](../subagents/2026-03-19/orchestration-patterns-research.md) — Five standardized patterns, AutoGen team types, SK orchestration classes, cross-cutting concerns
2. [libraries-frameworks-research.md](../subagents/2026-03-19/libraries-frameworks-research.md) — LangGraph, Semantic Kernel, Agent Framework, OpenAI Agents SDK, CrewAI, pattern matrix
3. [a2a-aspnet-research.md](../subagents/2026-03-19/a2a-aspnet-research.md) — A2A protocol, ASP.NET implementation, in-process vs A2A, enterprise patterns
4. [azure-foundry-compat-research.md](../subagents/2026-03-19/azure-foundry-compat-research.md) — Foundry Agent Service, SK/AF/LangChain integration, compatibility matrix
5. [agent-framework-workflow-api-research.md](../subagents/2026-03-19/agent-framework-workflow-api-research.md) — Workflow API architecture, executors, edges, checkpointing, HITL, orchestration builders
6. [foundry-workflow-agents-research.md](../subagents/2026-03-19/foundry-workflow-agents-research.md) — Foundry Workflow Agents, YAML definition, comparison with code-based frameworks
7. [durable-agents-a2a-research.md](../subagents/2026-03-19/durable-agents-a2a-research.md) — Durable Agents, Azure Functions hosting, orchestration patterns, hosting decision matrix

### Key Sources

| Source | URL |
|---|---|
| Azure Architecture — AI Agent Orchestration Patterns | https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns |
| Semantic Kernel — Agent Orchestration | https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/ |
| AutoGen — AgentChat User Guide | https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html |
| Microsoft Agent Framework Overview | https://learn.microsoft.com/agent-framework/overview/ |
| Agent Framework Workflows | https://learn.microsoft.com/en-us/agent-framework/workflows/workflows |
| Agent Framework Edges | https://learn.microsoft.com/en-us/agent-framework/workflows/edges |
| Agent Framework Executors | https://learn.microsoft.com/en-us/agent-framework/workflows/executors |
| Agent Framework Orchestrations | https://learn.microsoft.com/agent-framework/workflows/orchestrations/ |
| Agent Framework Azure Functions | https://learn.microsoft.com/en-us/agent-framework/integrations/azure-functions |
| A2A Protocol Specification v1.0 | https://a2a-protocol.org/latest/specification/ |
| Agent Framework A2A Integration | https://learn.microsoft.com/agent-framework/integrations/a2a |
| Foundry Agent Service Overview | https://learn.microsoft.com/azure/foundry/agents/overview |
| Foundry Workflow Agents | https://learn.microsoft.com/azure/foundry/agents/concepts/workflow |
| Agent Framework Declarative Workflows | https://learn.microsoft.com/agent-framework/workflows/declarative |
| Durable Task Scheduler | https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler |
| LangChain + Azure AI Foundry | https://learn.microsoft.com/azure/foundry/how-to/develop/langchain |
| LangGraph Docs | https://docs.langchain.com/oss/python/langgraph/overview |

---

## Technical Scenarios

### Scenario 1: Recommended Framework Selection Strategy

For an enterprise already on Azure AI Foundry:

**Selected Approach: Microsoft Agent Framework + Semantic Kernel bridge**

* **For new projects:** Start with Microsoft Agent Framework (public preview) if comfortable with preview status. It offers the most complete pattern support, first-class Foundry integration, A2A support, and represents Microsoft's strategic direction.
* **For production today:** Use Semantic Kernel for .NET or LangGraph for Python. Both are more mature and integrate well with Foundry.
* **For polyglot scenarios:** Combine in-process orchestration with A2A protocol. .NET services use SK/Agent Framework internally, Python services use LangGraph, and they communicate via A2A.

**Rationale:** Microsoft is consolidating toward Agent Framework as the successor to both AutoGen and SK. Starting there positions the team on the strategic path while SK provides a stable fallback for production-critical workloads today.

### Scenario 2: Hosting Strategy for Enterprise Agent Systems

**Selected Approach: Tiered hosting based on requirements**

| Requirement | Hosting Choice |
|---|---|
| Quick prototyping, non-developers | Foundry Workflow Agents (no code, managed) |
| Production multi-agent with A2A interop | ASP.NET Core + A2A (Container Apps or App Service) |
| Serverless, cost-optimized, event-driven | Azure Functions Durable Agents (Flex Consumption) |
| Long-running workflows (hours/days) | Azure Functions Durable Agents |
| HITL with zero wait-time cost | Azure Functions Durable Agents |
| Fully managed, no infrastructure | Foundry Agent Service (hosted agents) |

**Key insight:** A2A protocol and Durable Agents are currently separate paths. For A2A interop, use ASP.NET Core hosting. For serverless cost optimization, use Durable Agents. For maximum flexibility, start with Foundry Workflow Agents and graduate to Agent Framework code via "Generate Code."

### Scenario 3: No-Code to Full-Code Graduation Path

**Selected Approach: Foundry → Agent Framework declarative → Agent Framework programmatic**

1. **Start:** Foundry Workflow Agents (visual builder, 3 patterns, zero code)
2. **Graduate:** "Generate Code" → Agent Framework declarative YAML (30+ action types, checkpointing)
3. **Full control:** Agent Framework programmatic Workflow API (full graph, all patterns, custom executors)

This graduation path is officially supported by Microsoft tooling (VS Code extension, GitHub Copilot code generation).

### Considered Alternatives

* **LangGraph everywhere:** Excellent for Python-only teams but no .NET support. Would require A2A for any .NET agents.
* **Semantic Kernel only:** Mature but being superseded. Good bridge strategy but plan for migration.
* **AutoGen directly:** Still functional but actively being migrated to Agent Framework. Not recommended for new projects.
* **CrewAI:** Strong Python ecosystem but no .NET, no Foundry-native integration, limited pattern coverage.
