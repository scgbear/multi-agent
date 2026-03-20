# Multi-Agent Libraries and Frameworks Research (.NET and Python)

**Research Date:** 2026-03-19
**Status:** Complete

## Research Topics

1. LangChain / LangGraph (Python) — orchestration capabilities, patterns, maturity, Azure integration
2. Semantic Kernel (.NET + Python) — multi-agent orchestration, GA status, Azure AI Foundry integration
3. Microsoft Agent Framework (AutoGen) — branding, team types, patterns, .NET/Python support
4. Other notable libraries — CrewAI, OpenAI Agents SDK, etc.
5. Pattern-to-Library mapping matrix

---

## 1. LangChain / LangGraph (Python)

### Overview

LangGraph is a low-level orchestration framework built by LangChain Inc. for building, managing, and deploying long-running, stateful agents. It is focused entirely on agent orchestration and does not abstract prompts or architecture. Trusted by companies including Klarna, Replit, and Elastic.

- **Latest version:** `langgraph 1.1.3` (released 2026-03-18)
- **Language:** Python (primary), JavaScript/TypeScript (separate package: `langgraphjs`)
- **License:** Open source
- **.NET equivalent:** None. Python and JS/TS only.
- **PyPI:** <https://pypi.org/project/langgraph/>
- **GitHub:** <https://github.com/langchain-ai/langgraph>

### Architecture

LangGraph is inspired by [Pregel](https://research.google/pubs/pub37252/) (Google's graph processing model) and [Apache Beam](https://beam.apache.org/). The public interface draws inspiration from [NetworkX](https://networkx.org/documentation/latest/).

Core primitives:

- **`StateGraph`**: The primary graph builder class. Nodes are functions/agents, edges define control flow.
- **`MessagesState`**: A common state type for message-based workflows.
- **Conditional edges**: `add_conditional_edges()` enables dynamic routing based on state.
- **`Send`**: Fan-out mechanism for parallel task dispatching.
- **Subgraphs**: Compiled graphs can be used as nodes within parent graphs for hierarchical composition.
- **Checkpointing**: Built-in persistence with `InMemorySaver`, Postgres, SQLite, etc.
- **`Command`**: A mechanism for state updates and routing within nodes.

### Multi-Agent Patterns Supported

LangGraph provides the low-level graph primitives to implement various multi-agent patterns. It does not provide pre-built orchestration patterns like Semantic Kernel does — instead, you compose them from `StateGraph`, conditional edges, and subgraphs.

| Pattern | How Implemented in LangGraph |
|---|---|
| **Sequential / Pipeline** | Linear chain of nodes with `add_edge()` |
| **Parallel / Fan-out** | `Send()` API to dispatch work to multiple subgraphs concurrently |
| **Supervisor** | A supervisor node uses an LLM to decide which worker node to invoke next via conditional edges |
| **Hierarchical** | Subgraphs composed as nodes within parent graphs, multi-level supervisor trees |
| **Handoff / Routing** | Conditional edges route to different agent nodes based on state/LLM decisions |
| **Reflection / Critic** | Loop back from evaluator node to generator node via conditional edges |
| **Swarm** | Agents hand off via tool calls; modeled as conditional edges with state transitions |
| **Human-in-the-loop** | `interrupt_before`/`interrupt_after` checkpoints allow injecting human input |
| **Map-Reduce** | Fan-out with `Send()` to subgraphs, then join results in an aggregator node |

### Prebuilt Components

- **`create_react_agent`**: Prebuilt ReAct agent with tool calling loop
- **`ToolNode`**: Prebuilt node that executes tool calls from LLM responses

### Integration with Azure OpenAI

LangGraph integrates via **LangChain** model adapters. The `langchain_openai` package provides:

- `AzureChatOpenAI` — for Azure OpenAI endpoints
- `ChatOpenAI` — for direct OpenAI API

LangGraph can use any LangChain-compatible chat model. No native Azure AI Foundry integration.

### Core Benefits

- **Durable execution**: Persists through failures, resumes from checkpoints
- **Human-in-the-loop**: Inspect/modify agent state at any point
- **Comprehensive memory**: Short-term working memory + long-term persistent memory
- **Streaming**: Built-in streaming support for all graph operations
- **Debugging**: LangSmith integration for traces, evaluation, and observability
- **Production deployment**: LangGraph Cloud / LangSmith Deployment platform

### Maturity

- Very mature, widely adopted in the Python agent ecosystem
- Version 1.1.3 as of March 2026
- Active development, large community
- Well-documented with extensive tutorials and examples

---

## 2. Semantic Kernel (.NET + Python)

### Overview

Semantic Kernel is Microsoft's open-source SDK for AI application development. Its **Agent Framework** provides multi-agent orchestration patterns directly in the SDK for both .NET and Python (and experimentally Java).

- **GitHub:** <https://github.com/microsoft/semantic-kernel>
- **.NET package:** `Microsoft.SemanticKernel.Agents.Orchestration` (prerelease)
- **Python package:** `semantic-kernel` (includes agents module)
- **Status:** Agent Orchestration features are **experimental** (under active development, may change significantly)

### Agent Types in Semantic Kernel

| Agent Type | Description |
|---|---|
| `ChatCompletionAgent` | Based on chat completion AI services (OpenAI, Azure OpenAI, etc.) |
| `AzureAIAgent` | Designed for Azure AI Foundry (Microsoft Foundry Agent Service). Automates tool calling, manages conversation history via threads. |
| `OpenAIAssistantAgent` | Uses OpenAI Assistants API |
| `OpenAIResponsesAgent` | Uses OpenAI Responses API |
| `CopilotStudioAgent` | Integration with Microsoft Copilot Studio |

### Multi-Agent Orchestration Patterns

All patterns share a **unified interface** for construction and invocation:

| Pattern | Class | Description | Use Case |
|---|---|---|---|
| **Concurrent** | `ConcurrentOrchestration` | Broadcasts task to all agents, collects results independently | Parallel analysis, ensemble decision-making |
| **Sequential** | `SequentialOrchestration` | Passes output from one agent to the next in order | Pipelines, multi-stage processing |
| **Handoff** | `HandoffOrchestration` | Dynamically passes control between agents based on context | Escalation, fallback, expert routing |
| **Group Chat** | `GroupChatOrchestration` | Agents participate in group conversation, coordinated by manager | Brainstorming, collaborative problem solving |
| **Magentic** | `MagenticOrchestration` | Inspired by MagenticOne research — manager builds/refines task ledger | Complex, generalist multi-agent collaboration |

### Key API (C#)

```csharp
// Choose orchestration pattern
SequentialOrchestration orchestration = new(agentA, agentB);

// Start runtime
InProcessRuntime runtime = new();
await runtime.StartAsync();

// Invoke
OrchestrationResult<string> result = await orchestration.InvokeAsync(task, runtime);
string text = await result.GetValueAsync();
```

### Key API (Python)

```python
orchestration = SequentialOrchestration(members=[agent_a, agent_b])
runtime = InProcessRuntime()
runtime.start()
result = await orchestration.invoke(task="Your task here", runtime=runtime)
final_output = await result.get()
await runtime.stop_when_idle()
```

### NuGet Packages (C#)

```powershell
dotnet add package Microsoft.SemanticKernel.Agents.Orchestration --prerelease
dotnet add package Microsoft.SemanticKernel.Agents.Runtime.InProcess --prerelease
```

### Azure AI Foundry Integration

- **`AzureAIAgent`**: Direct integration with Azure AI Foundry Agent Service
- **`AzureChatCompletion`**: Service connector for Azure OpenAI models
- Supports Azure AI Search, Bing, Azure Functions, and OpenAPI tools natively through `AzureAIAgent`
- Requires a Microsoft Foundry Project

### .NET vs Python Feature Parity

| Feature | .NET | Python |
|---|---|---|
| `ChatCompletionAgent` | Yes | Yes |
| `AzureAIAgent` | Yes | Yes |
| `OpenAIAssistantAgent` | Yes | Yes |
| `OpenAIResponsesAgent` | Yes | Yes |
| `CopilotStudioAgent` | Yes | Yes |
| Concurrent Orchestration | Yes | Yes |
| Sequential Orchestration | Yes | Yes |
| Handoff Orchestration | Yes | Yes |
| Group Chat Orchestration | Yes | Yes |
| Magentic Orchestration | Yes | Yes |
| Declarative Specs | Coming soon | Coming soon |
| Java support | Partial (agent orchestration not yet available) | N/A |

### Maturity

- Agent orchestration is **experimental** (pre-release NuGet packages)
- The older `AgentGroupChat` is deprecated; migration to `GroupChatOrchestration` recommended
- Semantic Kernel itself is GA; the Agent Framework is the newer addition
- Being superseded by **Microsoft Agent Framework** for new development

---

## 3. Microsoft Agent Framework (AutoGen)

### Branding / Naming History

| Name | Status | Notes |
|---|---|---|
| **AutoGen** | Original name (Microsoft Research) | Pioneered multi-agent concepts; still on GitHub at `microsoft/autogen` |
| **AG2** | Community fork | Maintained by open-source community; not the Microsoft-endorsed version |
| **Microsoft Agent Framework** | Current official name | Direct successor to both AutoGen and Semantic Kernel; in **public preview** |

**Key insight:** Microsoft Agent Framework is the **next generation of both Semantic Kernel and AutoGen**, created by the same teams. It combines AutoGen's simple abstractions with Semantic Kernel's enterprise features.

- **GitHub (AutoGen):** <https://github.com/microsoft/autogen>
- **GitHub (Agent Framework):** <https://github.com/microsoft/agent-framework>
- **Docs:** <https://learn.microsoft.com/agent-framework/overview/>

### Language Support

- **.NET:** Yes — `Microsoft.Agents.AI.OpenAI` and related packages (prerelease)
- **Python:** Yes — `agent-framework` package (`pip install agent-framework --pre`)

### AutoGen (Current) — Team Types

The AutoGen `autogen-agentchat` package provides these team types:

| Team Type | Pattern | Description |
|---|---|---|
| `RoundRobinGroupChat` | Round-robin / Sequential | Agents take turns in a fixed order |
| `SelectorGroupChat` | Supervisor / LLM-selected | An LLM-based model selects the next speaker |
| `Swarm` | Handoff / Swarm | Agents hand off via `HandoffMessage`; agent-to-agent delegation |
| `MagenticOneGroupChat` | Magentic / Adaptive planning | Orchestrator builds task/progress ledgers, manages complex tasks dynamically |
| `GraphFlow` | Graph-based | Experimental — agents as nodes with conditional edge transitions |

### AutoGen Key Agents and Tools

- `AssistantAgent`: LLM-powered agent with tool calling
- `UserProxyAgent`: Human-in-the-loop proxy
- `MultimodalWebSurfer`: Browser agent (MagenticOne)
- `FileSurfer`: File reading agent (MagenticOne)
- `MagenticOneCoderAgent`: Code writing agent (MagenticOne)
- `AgentTool`: Wraps an agent as a tool for another agent to call
- `TeamTool`: Wraps a team as a tool for agent orchestration
- `HandoffMessage`: Message type for swarm-style agent handoffs

### AutoGen Termination Conditions

`MaxMessageTermination`, `TextMentionTermination`, `StopMessageTermination`, `TokenUsageTermination`, `HandoffTermination`, `TimeoutTermination`, `ExternalTermination`, `SourceMatchTermination`, `TextMessageTermination`, `FunctionCallTermination`, `FunctionalTermination`

### AutoGen Design Patterns (Core API)

Beyond teams, AutoGen's `autogen-core` supports lower-level patterns:

- **Reflection**: Critic agent evaluates primary agent responses
- **Mixture of Agents**: Feed-forward neural network-inspired multi-layer agent architecture
- **Society of Mind**: Nested team as inner agent

### Microsoft Agent Framework (New) — Workflow Model

Agent Framework introduces a **unified `Workflow` abstraction** that replaces both AutoGen's `Team` and `autogen-core`:

| Feature | AutoGen | Agent Framework |
|---|---|---|
| **Flow Type** | Control flow (edges are transitions) | Data flow (edges route messages) |
| **Node Types** | Agents only | Agents, functions, sub-workflows |
| **Activation** | Message broadcast | Edge-based activation |
| **Type Safety** | Limited | Strong typing throughout |
| **Composability** | Limited | Highly composable |
| **Checkpointing** | Not built-in | Built-in with `FileCheckpointStorage` |
| **Human-in-the-loop** | Custom implementation | Built-in `ctx.request_info()` + `@response_handler` |

### Agent Framework Orchestration Patterns

Mapped from AutoGen to Agent Framework:

| Pattern | AutoGen | Agent Framework |
|---|---|---|
| Sequential | `RoundRobinGroupChat` | Sequential orchestration workflow |
| Concurrent | N/A (manual) | Concurrent orchestration workflow |
| Group Chat | `SelectorGroupChat` | Group chat orchestration |
| Handoff | `Swarm` | Handoff orchestration |
| Magentic | `MagenticOneGroupChat` | Magentic orchestration workflow |
| Swarm | `Swarm` | In development (roadmap) |
| SelectorGroupChat | `SelectorGroupChat` | In development (roadmap) |

### Azure AI Foundry Integration

- `AzureOpenAIChatCompletionClient` — for Azure OpenAI models
- `AzureOpenAIResponsesClient` — for Azure OpenAI Responses API
- Agent Framework supports Azure OpenAI, OpenAI, Anthropic, Ollama, and more via provider model
- `AzureAIAgentClient` — for Foundry Agent Service
- Enterprise features: Microsoft Entra security, OpenTelemetry observability
- Standards: A2A protocol, MCP support

### Relationship Between AutoGen and Semantic Kernel

- **AutoGen** pioneered multi-agent concepts (GroupChat, event-driven runtime)
- **Semantic Kernel** provided enterprise AI features (Kernel, plugins, connectors)
- **Microsoft Agent Framework** is the **direct successor to both**, combining:
  - AutoGen's simple agent abstractions
  - Semantic Kernel's enterprise features (state management, type safety, middleware, telemetry)
  - New graph-based workflows for explicit multi-agent orchestration
- Migration guides exist for both: [from AutoGen](https://learn.microsoft.com/agent-framework/migration-guide/from-autogen/) and [from Semantic Kernel](https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/)

---

## 4. Other Notable Libraries

### OpenAI Agents SDK

- **Version:** `openai-agents 0.12.5` (released 2026-03-19)
- **Language:** Python (primary), JavaScript/TypeScript (separate package)
- **GitHub:** <https://github.com/openai/openai-agents-python>
- **Docs:** <https://openai.github.io/openai-agents-python/>
- **Production-ready upgrade of OpenAI's experimental [Swarm](https://github.com/openai/swarm)**

#### Core Primitives

- **Agents**: LLMs equipped with instructions and tools
- **Handoffs**: Agents delegate to other agents for specific tasks
- **Agents as Tools**: `Agent.as_tool()` — a manager agent calls specialist agents as tools
- **Guardrails**: Input/output validation running in parallel with agent execution
- **Sessions**: Persistent memory layer for conversation history
- **Tracing**: Built-in tracing for debugging and monitoring

#### Multi-Agent Orchestration Patterns

| Pattern | Mechanism |
|---|---|
| **Agents as Tools (Manager)** | Manager agent keeps control, calls specialists via `Agent.as_tool()` |
| **Handoffs (Routing)** | Triage agent routes conversation to specialist; specialist becomes active agent |
| **Sequential (Code)** | Chain agents by transforming output of one into input of next |
| **Parallel (Code)** | `asyncio.gather` for independent parallel tasks |
| **Evaluator loop (Code)** | Agent + evaluator in a `while` loop until quality criteria met |
| **Combined** | Triage handoffs + specialists calling sub-agents as tools |

#### Azure Integration

Provider-agnostic via model interface. Supports:

- OpenAI Responses API and Chat Completions API
- 100+ LLMs via LiteLLM extension
- No native Azure OpenAI integration (would use LiteLLM or custom `ModelProvider`)

### CrewAI

- **Version:** `crewai 1.11.0` (released 2026-03-18)
- **Language:** Python only (no .NET)
- **GitHub:** <https://github.com/crewAIInc/crewAI>
- **Docs:** <https://docs.crewai.com/>
- **License:** MIT
- **Standalone framework** — completely independent from LangChain

#### Architecture

- **Crews**: Teams of autonomous, role-playing agents
- **Flows**: Event-driven, stateful workflows with precise control
- **Combined**: Crews within Flows for production-grade applications

#### Process Types

| Process | Description |
|---|---|
| `Process.sequential` | Agents execute tasks in defined order |
| `Process.hierarchical` | Auto-assigns a manager agent to coordinate via delegation |

#### Key Features

- YAML-based agent/task configuration
- Role-playing agents with goals and backstories
- `@start`, `@listen`, `@router` decorators for Flows
- `or_` and `and_` operators for conditional flow logic
- Human-in-the-loop support
- 100,000+ certified developers community
- Enterprise version (CrewAI AMP) with control plane and observability

---

## 5. Pattern-to-Library Mapping Matrix

| Pattern | LangGraph | Semantic Kernel | AutoGen (Current) | Agent Framework (New) | OpenAI Agents SDK | CrewAI |
|---|---|---|---|---|---|---|
| **Sequential / Pipeline** | Yes (edges) | Yes (`SequentialOrchestration`) | Yes (`RoundRobinGroupChat`) | Yes (sequential workflow) | Yes (code chaining) | Yes (`Process.sequential`) |
| **Concurrent / Parallel** | Yes (`Send()`) | Yes (`ConcurrentOrchestration`) | Partial (manual) | Yes (concurrent workflow) | Yes (`asyncio.gather`) | Partial (task parallelism) |
| **Supervisor / Selector** | Yes (conditional edges + LLM) | Yes (Group Chat) | Yes (`SelectorGroupChat`) | In development | Yes (`Agent.as_tool()`) | Yes (`Process.hierarchical`) |
| **Handoff / Routing** | Yes (conditional edges) | Yes (`HandoffOrchestration`) | Yes (`Swarm`) | Yes (handoff workflow) | Yes (native handoffs) | No (not explicit) |
| **Group Chat** | Yes (shared state graph) | Yes (`GroupChatOrchestration`) | Yes (`RoundRobinGroupChat`, `SelectorGroupChat`) | Yes (group chat workflow) | No | No |
| **Magentic / Adaptive** | No (build custom) | Yes (`MagenticOrchestration`) | Yes (`MagenticOneGroupChat`) | Yes (magentic workflow) | No | No |
| **Reflection / Critic** | Yes (loop edges) | Possible (custom) | Yes (core design pattern) | Possible (custom) | Yes (evaluator loop) | Possible (custom) |
| **Swarm** | Yes (via state) | Via Handoff | Yes (`Swarm` team) | In development | Yes (native handoffs) | No |
| **Graph / DAG** | Yes (native `StateGraph`) | No | Experimental (`GraphFlow`) | Yes (`Workflow` graph) | No | Yes (`Flow` decorators) |
| **Human-in-the-loop** | Yes (interrupts) | Yes (supported) | Yes (`UserProxyAgent`) | Yes (built-in request/response) | Yes (built-in) | Yes |
| **Nested / Hierarchical** | Yes (subgraphs) | Yes (orchestration nesting) | Yes (team nesting, `SocietyOfMindAgent`) | Yes (sub-workflows) | Yes (agents as tools) | Yes (Crews in Flows) |
| **Mixture of Agents** | Yes (custom graph) | No | Yes (core design pattern) | No | No | No |

### Language Support Matrix

| Library | Python | .NET | JavaScript/TypeScript | Java |
|---|---|---|---|---|
| **LangGraph** | Yes (primary) | No | Yes (LangGraphJS) | No |
| **Semantic Kernel** | Yes | Yes (primary) | No | Yes (partial) |
| **AutoGen** | Yes (primary) | Yes (partial) | No | No |
| **Agent Framework** | Yes | Yes | No | No |
| **OpenAI Agents SDK** | Yes (primary) | No | Yes | No |
| **CrewAI** | Yes | No | No | No |

### Maturity Matrix

| Library | Version | Stage | Multi-Agent Status |
|---|---|---|---|
| **LangGraph** | 1.1.3 | GA / Production | Mature, widely adopted |
| **Semantic Kernel** | GA (core); Agents prerelease | Orchestration: Experimental | Being superseded by Agent Framework |
| **AutoGen** | 0.x (agentchat) | Research-to-production | Stable teams; being migrated to Agent Framework |
| **Microsoft Agent Framework** | Preview | Public Preview | Active development; successor to both SK and AutoGen |
| **OpenAI Agents SDK** | 0.12.5 | Production-ready | Mature handoff/tool patterns |
| **CrewAI** | 1.11.0 | GA / Production | Mature Crews + Flows |

---

## References and Evidence

1. **LangGraph PyPI**: <https://pypi.org/project/langgraph/> — v1.1.3, released 2026-03-18
2. **LangGraph Docs**: <https://docs.langchain.com/oss/python/langgraph/overview>
3. **Semantic Kernel Agent Orchestration**: <https://learn.microsoft.com/semantic-kernel/frameworks/agent/agent-orchestration/>
4. **Semantic Kernel Agent Architecture**: <https://learn.microsoft.com/semantic-kernel/frameworks/agent/agent-architecture>
5. **Microsoft Agent Framework Overview**: <https://learn.microsoft.com/agent-framework/overview/>
6. **AutoGen to Agent Framework Migration**: <https://learn.microsoft.com/agent-framework/migration-guide/from-autogen/>
7. **Semantic Kernel to Agent Framework Migration**: <https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/>
8. **AI Agent Orchestration Patterns (Azure Architecture)**: <https://learn.microsoft.com/azure/architecture/ai-ml/guide/ai-agent-design-patterns>
9. **AutoGen GitHub (teams `__init__.py`)**: `microsoft/autogen` — `RoundRobinGroupChat`, `SelectorGroupChat`, `Swarm`, `MagenticOneGroupChat`, `GraphFlow`
10. **OpenAI Agents SDK Docs**: <https://openai.github.io/openai-agents-python/> — v0.12.5
11. **OpenAI Agents SDK Multi-Agent**: <https://openai.github.io/openai-agents-python/multi_agent/>
12. **CrewAI PyPI**: <https://pypi.org/project/crewai/> — v1.11.0
13. **CrewAI Docs**: <https://docs.crewai.com/introduction>
14. **.NET AI Ecosystem**: <https://learn.microsoft.com/dotnet/ai/dotnet-ai-ecosystem>

---

## Discovered Research Topics

1. **Microsoft Agent Framework `Workflow` abstraction** — the new data-flow-based graph model replacing AutoGen's `Team` and `GraphFlow`. Uses typed edges, executors (agents/functions/sub-workflows), with checkpointing and human-in-the-loop built in.
2. **A2A Protocol (Agent-to-Agent)** — Microsoft's open protocol for agent discovery and interoperability; supported in Agent Framework.
3. **MCP (Model Context Protocol)** — Standardized tool integration supported across Agent Framework, OpenAI Agents SDK, and others.
4. **Foundry Agent Service** — Azure's managed, no-code approach to agent chaining via "connected agents." Primarily nondeterministic workflows.
5. **GraphFlow (AutoGen)** — Experimental graph-based orchestration in AutoGen; control-flow based. Predecessor to Agent Framework's `Workflow`.
6. **LangGraph `Command`** — New mechanism for routing and state updates from within nodes.

---

## Next Research / Outstanding Questions

- [ ] Deep-dive into Agent Framework `Workflow` API (executors, edges, `WorkflowBuilder`)
- [ ] LangGraph `Command` API and its impact on multi-agent patterns
- [ ] Azure AI Foundry "connected agents" capabilities and limitations
- [ ] .NET AutoGen status — `Microsoft.AutoGen.AgentChat` namespace has `ITeam` interface
- [ ] Semantic Kernel Java support timeline for agent orchestration
- [ ] Performance benchmarks comparing these frameworks
- [ ] A2A protocol details and cross-framework interoperability

---

## Clarifying Questions

None — all primary research questions have been answered through documentation and source review.
