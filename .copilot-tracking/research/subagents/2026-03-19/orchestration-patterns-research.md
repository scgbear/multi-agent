# Multi-Agent Orchestration Patterns Research

## Research Status: Complete

## Research Topics and Questions

1. What are the standardized multi-agent orchestration patterns? (Sequential, Handoff, Parallel/Concurrent, Group Chat/Collaborative, Magentic)
2. What are "agentic workflows" as defined by the Microsoft Agent Framework (AutoGen/AG2)?
3. For each pattern: definition, use cases, benefits/trade-offs, conceptual diagrams
4. How do framework-specific patterns map to standard patterns?

---

## Complexity Spectrum (Before Choosing a Pattern)

Before adopting a multi-agent pattern, evaluate whether your scenario requires one. Microsoft's Azure Architecture guide defines a clear complexity spectrum:

| Level | Description | When to Use |
|---|---|---|
| **Direct model call** | Single LLM call with a well-crafted prompt. No agent logic, no tool access. | Classification, summarization, translation, single-step tasks |
| **Single agent with tools** | One agent that reasons and acts by selecting from available tools, knowledge sources, and APIs. Can loop through multiple model calls. | Varied queries within a single domain requiring dynamic tool use |
| **Multi-agent orchestration** | Multiple specialized agents coordinate to solve a problem. An orchestrator or peer-based protocol manages work distribution, context sharing, and result aggregation. | Cross-functional or cross-domain problems, distinct security boundaries, parallel specialization |

> **Key Guidance:** Use the lowest level of complexity that reliably meets your requirements. Decision-making and flow-control overhead often exceed the benefits of breaking tasks into multiple agents.

**Source:** [AI agent orchestration patterns - Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)

---

## Standardized Multi-Agent Orchestration Patterns

The following five patterns have been **standardized by Microsoft** in their Azure Architecture guidance and are implemented across the Microsoft Agent Framework (Semantic Kernel) and AutoGen. These patterns are technology agnostic.

### 1. Sequential Orchestration

**Also known as:** Pipeline, Prompt Chaining, Linear Delegation

**Definition:** Chains AI agents in a predefined, linear order. Each agent processes the output from the previous agent in the sequence, creating a pipeline of specialized transformations. The choice of which agent gets invoked next is deterministically defined as part of the workflow and is not a choice given to agents.

**Conceptual Diagram:**

```
Input → [Agent 1] → [Agent 2] → ... → [Agent N] → Result
              ↕              ↕                  ↕
          Model/Tools    Model/Tools        Model/Tools
         ←────────── Common State ──────────→
```

**When to Use:**
- Multistage processes with clear linear dependencies and predictable workflow progression
- Data transformation pipelines where each stage adds specific value the next stage depends on
- Workflow stages that cannot be parallelized
- Progressive refinement requirements (draft → review → polish)
- Systems where availability/performance of every agent in the pipeline is understood

**When to Avoid:**
- Stages are embarrassingly parallel
- Only a few stages that a single agent can accomplish
- Early stages might fail and produce low-quality output that cascades
- Agents need to collaborate rather than hand off work
- Workflow requires backtracking or iteration
- Dynamic routing based on intermediate results is needed

**Benefits:**
- Simple, predictable execution flow
- Easy to debug and test individual stages
- Clear data lineage through the pipeline
- Each stage can be independently optimized

**Trade-offs:**
- No parallelism (increased latency)
- Single point of failure—errors in early stages propagate
- Cannot handle dynamic routing or backtracking
- All stages must be known at design time

**Example:** Contract generation pipeline: Template Selection → Clause Customization → Regulatory Compliance → Risk Assessment

**Sources:**
- [Azure Architecture - Sequential orchestration](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#sequential-orchestration)
- [Semantic Kernel - Sequential Orchestration](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/sequential)

---

### 2. Concurrent Orchestration (Parallel / Fan-out/Fan-in)

**Also known as:** Parallel, Fan-out/Fan-in, Scatter-Gather, Map-Reduce

**Definition:** Runs multiple AI agents simultaneously on the same task. Each agent provides independent analysis or processing from its unique perspective or specialization. Agents operate independently and do not hand off results to each other. An initiator/collector agent dispatches the task and aggregates results.

**Conceptual Diagram:**

```
                    ┌→ [Agent 1] → Result 1 ─┐
                    │                         │
Input → [Initiator] ├→ [Agent 2] → Result 2 ─┼→ [Aggregated Result]
                    │                         │
                    └→ [Agent N] → Result N ─┘
```

**When to Use:**
- Tasks that can run in parallel (fixed or dynamically chosen agents)
- Tasks benefiting from multiple independent perspectives (technical, business, creative)
- Multi-agent decision-making: brainstorming, ensemble reasoning, quorum/voting-based decisions
- Time-sensitive scenarios where parallel processing reduces latency

**When to Avoid:**
- Agents need to build on each other's work or require cumulative context
- Task requires specific order of operations or deterministic reproducible results
- Resource constraints (model quota) make parallel processing inefficient
- Agents cannot reliably coordinate changes to shared state
- No clear conflict resolution strategy for contradictory results
- Result aggregation logic is too complex

**Benefits:**
- Reduced latency through parallelism
- Diverse perspectives on the same problem
- Natural fit for voting and ensemble methods
- Straightforward fan-out/fan-in architecture

**Trade-offs:**
- Resource-intensive (concurrent model invocations)
- Requires conflict resolution when results contradict
- No context sharing between concurrent agents
- Aggregation logic can lower quality if poorly designed

**Example:** Stock analysis: Fundamental Analysis Agent + Technical Analysis Agent + Sentiment Analysis Agent + ESG Agent → Combined Investment Recommendation

**Sources:**
- [Azure Architecture - Concurrent orchestration](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#concurrent-orchestration)
- [Semantic Kernel - Concurrent Orchestration](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/concurrent)

---

### 3. Handoff Orchestration

**Also known as:** Routing, Triage, Transfer, Dispatch, Delegation

**Definition:** Enables dynamic delegation of tasks between specialized agents. Each agent can assess the task at hand and decide whether to handle it directly or transfer it to a more appropriate agent based on context and requirements. Full control transfers from one agent to another—agents do not work in parallel. The optimal agent for a task is not known upfront; task requirements become clear only during processing.

**Conceptual Diagram:**

```
Input → [Triage Agent] ──→ [Specialist A] ──→ Result
              │                    │
              ├──→ [Specialist B] ─┤
              │         │          │
              └──→ [Specialist C] ─┘
                        │
                  [Human Agent]
        (curved arrows between agents—dynamic routing)
```

**When to Use:**
- Tasks requiring specialized knowledge/tools where agent count or order cannot be predetermined
- Expertise requirements emerge during processing (dynamic routing based on content analysis)
- Multiple-domain problems requiring different specialists operating one at a time
- Logical relationships and signals predetermining when one agent reaches its limit
- Customer support escalation scenarios

**When to Avoid:**
- Appropriate agent or sequence is identifiable from initial input (use deterministic routing)
- Task routing is deterministic and rule-based
- Suboptimal routing decisions might lead to poor user experience
- Multiple operations should run concurrently
- Avoiding infinite handoff loops is challenging

**Benefits:**
- Dynamic, context-aware routing
- Supports human-in-the-loop escalation
- Agents can specialize deeply
- Naturally handles multi-step conversations

**Trade-offs:**
- Risk of infinite handoff loops or excessive bouncing
- Unpredictable routing paths make testing harder
- Only one agent active at a time (no parallelism)
- Performance depends on quality of handoff decisions

**Example:** Telecom customer support: Triage Agent → Technical Infrastructure Agent → Financial Resolution Agent → Account Access Agent → Customer Support Employee

**Sources:**
- [Azure Architecture - Handoff orchestration](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#handoff-orchestration)
- [Semantic Kernel - Handoff Orchestration](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/handoff)

---

### 4. Group Chat Orchestration (Collaborative)

**Also known as:** Roundtable, Collaborative, Multi-Agent Debate, Council

**Definition:** Enables multiple agents to solve problems, make decisions, or validate work by participating in a shared conversation thread where they collaborate through discussion. A chat manager coordinates the flow by determining which agents can respond next and manages different interaction modes, from collaborative brainstorming to structured quality gates. All agents and humans emit output into a single accumulating thread.

**Conceptual Diagram:**

```
Input → [Chat Manager (LLM)] ←→ Accumulating Chat Thread → Result
              │
              ├──→ [Agent 1] (Model + Knowledge)
              ├──→ [Agent 2] (Model + Knowledge)
              ├──→ [Agent N] (Model + Knowledge)
              └──→ [Human Participant/Observer]
                    ↓
            Chat output feeds back into thread
```

**Sub-pattern: Maker-Checker Loop (Evaluator-Optimizer)**
A specific type of group chat where one agent (maker) creates/proposes something and another (checker) evaluates against defined criteria. The cycle repeats until the checker approves or a max iteration limit is reached.

**When to Use:**
- Creative brainstorming where agents with different perspectives build on each other
- Decision-making requiring debate and consensus-building
- Iterative refinement through discussion
- Quality assurance with structured review processes (maker-checker loops)
- Compliance/regulatory validation requiring multiple expert perspectives
- Human-in-the-loop scenarios where humans can guide conversations

**When to Avoid:**
- Basic task delegation or linear pipeline processing is sufficient
- Real-time processing requirements make discussion overhead unacceptable
- Clear hierarchical decision-making without discussion is more appropriate
- Chat manager has no objective way to determine if the task is complete
- More than three agents (control becomes difficult to maintain)

**Benefits:**
- Transparency and auditability (single accumulating thread)
- Supports human-in-the-loop naturally
- Flexible—supports brainstorming, debate, and formal review
- Iterative refinement through agent collaboration

**Trade-offs:**
- Conversation loops and infinite refinement risks
- Difficult to control with many agents
- Chat overhead increases latency
- Agents in read-only mode (typically do not use tools to make changes)

**Example:** City parks proposal review: Environmental Planning Agent + Community Engagement Agent + Budget/Operations Agent debate a park development proposal with a human parks department employee

**Sources:**
- [Azure Architecture - Group chat orchestration](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#group-chat-orchestration)
- [Semantic Kernel - Group Chat Orchestration](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/group-chat)

---

### 5. Magentic Orchestration (Adaptive Planning)

**Also known as:** Dynamic Orchestration, Task-Ledger-Based Orchestration, Adaptive Planning

**Definition:** Designed for open-ended and complex problems that have no predetermined plan of approach. A dedicated Magentic Manager agent coordinates a team of specialized agents, selecting which agent should act next based on evolving context, task progress, and agent capabilities. The manager maintains a task ledger that is dynamically built and refined through collaboration. Agents have tools that allow them to make direct changes in external systems. Inspired by the Magentic-One system from AutoGen research.

**Conceptual Diagram:**

```
Input → [Manager Agent (LLM)] ←→ Task & Progress Ledger → Result
              │                        │
              │                   [Human Participant]
              │
              ├──→ [Agent 1] (Model + Knowledge)
              ├──→ [Agent 2] (Model + Knowledge + Tools → External Systems)
              └──→ [Agent N] (Model + Tools → External Systems)
              
         ↻ Evaluate Goal Loop (Task Complete? → Yes → Result, No → Continue)
```

**When to Use:**
- Complex or open-ended use cases with no predetermined solution path
- Requirement for input and feedback from multiple specialized agents to develop a valid solution path
- Requirement for the AI system to generate a fully developed plan that a human can review
- Agents equipped with tools that interact with external systems, consume resources, or induce changes
- Need for documented plan showing how agents are sequenced (audit trail)

**When to Avoid:**
- Solution path is deterministic
- No requirement to produce a ledger/audit trail
- Task has low complexity (simpler pattern can solve it)
- Work is time-sensitive (pattern focuses on planning, not speed)
- Frequent stalls or infinite loops without clear resolution

**Benefits:**
- Handles truly complex, open-ended problems
- Produces auditable plans and progress tracking
- Dynamic adaptation as context evolves
- Combines research, reasoning, and tool execution

**Trade-offs:**
- Slow to converge; most expensive pattern
- Stalls on ambiguous goals
- Total cost difficult to predict (variable iteration count)
- Requires capable manager model (typically reasoning model)

**Example:** SRE incident response: Manager Agent coordinates Diagnostics Agent, Infrastructure Agent, Rollback Agent, and Communication Agent to dynamically investigate and resolve a service outage

**Sources:**
- [Azure Architecture - Magentic orchestration](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#magentic-orchestration)
- [Semantic Kernel - Magentic Orchestration](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/magentic)
- [Microsoft Research - Magentic-One](https://www.microsoft.com/research/articles/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/)

---

## Pattern Comparison Matrix

| Pattern | Communication | Agent Selection | Best For | Key Risk |
|---|---|---|---|---|
| **Sequential** | Linear pipeline; each agent processes previous output | Deterministic, predefined order | Step-by-step refinement with clear stage dependencies | Failures propagate; no parallelism |
| **Concurrent** | Parallel; agents work independently on same input | Deterministic or dynamic selection | Independent analysis from multiple perspectives; latency-sensitive | Contradictory results; resource-intensive |
| **Group Chat** | Conversational; agents contribute to shared thread | Chat manager controls turn order | Consensus-building, brainstorming, maker-checker validation | Conversation loops; hard to control with many agents |
| **Handoff** | Dynamic delegation; one active agent at a time | Agents decide when to transfer control | Tasks where right specialist emerges during processing | Infinite handoff loops; unpredictable routing |
| **Magentic** | Plan-build-execute; manager assigns and reorders tasks | Manager dynamically assigns tasks | Open-ended problems with no predetermined solution path | Slow to converge; stalls on ambiguous goals |

**Source:** [Azure Architecture - Choosing a pattern](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#choosing-a-pattern)

---

## Microsoft Agent Framework / AutoGen Agentic Workflows

### Overview

The **Microsoft Agent Framework** (formerly AutoGen, sometimes referenced as AG2) is an open-source SDK for building multi-agent orchestrations. It provides two API layers:

1. **AutoGen Core** (autogen-core): Low-level, event-driven programming model with maximum flexibility and control
2. **AutoGen AgentChat** (autogen-agentchat): High-level API for building multi-agent applications, built on top of Core

AutoGen AgentChat provides **Teams** as the primary multi-agent abstraction. Teams define how agents interact, share context, and coordinate.

### AutoGen Team Types (AgentChat High-Level API)

| Team Type | Pattern Category | Description |
|---|---|---|
| **RoundRobinGroupChat** | Group Chat (Sequential Turn-Taking) | Agents take turns in a fixed round-robin order. Simple and predictable. |
| **SelectorGroupChat** | Group Chat (Dynamic/Collaborative) | Model-based next-speaker selection. An LLM analyzes context to choose which agent speaks next. Supports custom selector functions and candidate filtering. |
| **Swarm** | Handoff | Agents hand off via tool calls. Each agent makes localized decisions about handoff. Inspired by OpenAI's Swarm pattern. Decentralized—no central orchestrator. |
| **GraphFlow** | Workflow (Configurable) | Directed graph-based execution. Supports sequential, parallel, conditional branching, and loops. Most deterministic and configurable approach. |
| **Magentic-One** | Magentic | Generalist multi-agent system with a manager agent coordinating specialized agents for complex, open-ended tasks. |

### Mapping AutoGen Patterns to Standard Patterns

| Standard Pattern | AutoGen Implementation | Key Difference |
|---|---|---|
| Sequential | GraphFlow (sequential edges) or RoundRobinGroupChat | GraphFlow offers strict control; RoundRobin is simpler |
| Concurrent/Parallel | GraphFlow (fan-out/fan-in edges) | Uses parallel edges in directed graph |
| Handoff | Swarm | Decentralized—agents decide handoff via tool calls, no central orchestrator |
| Group Chat | SelectorGroupChat or RoundRobinGroupChat | SelectorGroupChat uses LLM for speaker selection; RoundRobin is deterministic |
| Magentic | Magentic-One | Full manager-led adaptive orchestration |
| Workflow (Graph) | GraphFlow | Unique to AutoGen—supports conditional branching, loops, activation groups |

### GraphFlow Capabilities (Unique to AutoGen)

GraphFlow provides fine-grained workflow control through directed graphs:
- **Sequential chains:** Strict linear agent execution
- **Parallel fan-out/fan-in:** Multiple agents work simultaneously, results merged
- **Conditional branching:** Edge conditions determine next agent based on output
- **Loops with exit conditions:** Iterative refinement (e.g., maker-checker)
- **Message filtering:** Control which messages each agent receives
- **Activation groups:** Handle complex dependency patterns (all vs. any activation)

**Source:** [AutoGen - GraphFlow](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/graph-flow.html)

### Key AutoGen Concepts

- **Shared message context:** All agents in a team see all messages (broadcast model)
- **HandoffMessage:** Special message type used in Swarm for agent delegation
- **Termination conditions:** TextMentionTermination, HandoffTermination, MaxMessageTermination
- **Human-in-the-loop:** UserProxyAgent enables human participation in any team type
- **Serialization:** Teams can be serialized/deserialized for persistence

**Sources:**
- [AutoGen AgentChat](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html)
- [AutoGen Swarm](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/swarm.html)
- [AutoGen Selector Group Chat](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/selector-group-chat.html)
- [AutoGen GraphFlow](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/graph-flow.html)

---

## Semantic Kernel Agent Orchestration

### Overview

Semantic Kernel's Agent Orchestration framework (part of the **Microsoft Agent Framework**) implements the five standard patterns directly in the SDK as C#, Python, and Java APIs. All orchestration patterns share a unified interface.

### Supported Patterns

| Pattern | SK Class | Description |
|---|---|---|
| Concurrent | `ConcurrentOrchestration` | Broadcasts task to all agents, collects results independently |
| Sequential | `SequentialOrchestration` | Passes result from one agent to the next in defined order |
| Handoff | `HandoffOrchestration` | Dynamically passes control between agents based on context/rules |
| Group Chat | `GroupChatOrchestration` | Agents participate in group conversation, coordinated by manager |
| Magentic | `MagenticOrchestration` | Manager-led orchestration inspired by Magentic-One |

### Unified API

All patterns use the same invocation pattern:

```csharp
// Choose pattern
SequentialOrchestration orchestration = new(agentA, agentB);
// Start runtime
InProcessRuntime runtime = new();
await runtime.StartAsync();
// Invoke
OrchestrationResult<string> result = await orchestration.InvokeAsync(task, runtime);
string text = await result.GetValueAsync();
```

### Key Features
- **Agent types:** ChatCompletionAgent, OpenAIAssistantAgent, AzureAIAgent, OpenAIResponsesAgent, CopilotStudioAgent
- **Human-in-the-loop:** Supported across all patterns via InteractiveCallback
- **Custom Group Chat Managers:** Override selection, termination, and filtering logic
- **NuGet packages:** `Microsoft.SemanticKernel.Agents.Orchestration` + `Microsoft.SemanticKernel.Agents.Runtime.InProcess`

> **Note:** Agent Orchestration features are in the **experimental** stage as of the latest documentation.

**Source:** [Semantic Kernel - Agent Orchestration](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/)

---

## LangGraph/LangChain Multi-Agent Patterns

LangGraph supports multi-agent orchestration through its graph-based framework. The Azure Architecture guide confirms: "Other frameworks that support multi-agent orchestration include LangChain, CrewAI, and the OpenAI Agents SDK."

LangGraph's approach maps as follows:
- **Supervisor pattern** → Similar to Magentic/Group Chat (central coordinator routes to workers)
- **Network/Swarm** → Similar to Handoff (agents route to each other)
- **Hierarchical** → Nested supervisor patterns (supervisors managing sub-supervisors)
- **Sequential chains** → Sequential orchestration
- **Parallel execution** → Concurrent orchestration via graph fan-out

**Source:** [Azure Architecture - Other frameworks](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#other-frameworks)

---

## Implementation Considerations (Cross-Cutting)

From the Azure Architecture guide, key considerations across all patterns:

### Context and State Management
- Context windows grow rapidly—each agent adds reasoning, tool results, intermediate outputs
- Use compaction techniques (summarization, selective pruning) between agents
- Persist shared state externally for long-running tasks
- Scope persisted state to minimum necessary information

### Reliability
- Implement timeout and retry mechanisms
- Include graceful degradation for agent faults
- Validate agent output before passing to next agent
- Consider circuit breaker patterns for agent dependencies
- Ensure compute isolation between agents
- Use checkpoint features for recovery from interrupted orchestration

### Security
- Implement authentication and secure networking between agents
- Follow principle of least privilege
- Handle user identity propagation across agents (security trimming)
- Apply content safety guardrails at multiple points

### Cost Optimization
- Assign each agent a model matching task complexity (not every agent needs the most capable model)
- Monitor token consumption per agent and per orchestration run
- Apply context compaction between agents

### Observability and Testing
- Instrument all agent operations and handoffs
- Track performance and resource metrics per agent
- Use scoring rubrics or LLM-as-judge evaluations (not exact-match assertions)

### Human Participation
- Several patterns support human-in-the-loop (HITL)
- Identify mandatory vs. optional human input points
- Persist state at HITL checkpoints for resumption
- Scope HITL gates to specific tool invocations for sensitive operations

### Combining Patterns
- Applications sometimes require combining multiple orchestration patterns
- Use sequential for initial processing, then concurrent for parallelizable analysis
- Do not force one pattern when different stages have different characteristics

**Source:** [Azure Architecture - Implementation considerations](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns#implementation-considerations)

---

## Sources and References

### Primary Sources (Microsoft Official)

1. **Azure Architecture Center - AI Agent Orchestration Patterns** (Updated 2026-02-12)
   - https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns
   - Definitive reference for all five standard patterns with guidance, examples, comparison matrix, and implementation considerations

2. **Semantic Kernel - Agent Orchestration**
   - Overview: https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/
   - Sequential: https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/sequential
   - Concurrent: https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/concurrent
   - Handoff: https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/handoff
   - Group Chat: https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/group-chat
   - Magentic: https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/magentic
   - Architecture: https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-architecture

3. **AutoGen (Microsoft Agent Framework) - AgentChat**
   - AgentChat Overview: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html
   - Selector Group Chat: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/selector-group-chat.html
   - Swarm: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/swarm.html
   - Magentic-One: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html
   - GraphFlow: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/graph-flow.html
   - Core Design Patterns: https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/design-patterns/intro.html

4. **Microsoft Agent Framework Overview**
   - https://learn.microsoft.com/en-us/agent-framework/overview/agent-framework-overview

5. **Microsoft Research - Magentic-One**
   - https://www.microsoft.com/research/articles/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/

### Additional Sources

6. **AutoGen Research Paper** - https://aka.ms/autogen-paper
7. **LangChain Multi-Agent Docs** - https://docs.langchain.com/oss/python/langchain/multi-agent#patterns
8. **CrewAI Concepts** - https://docs.crewai.com/concepts/processes
9. **OpenAI Agents SDK** - https://openai.github.io/openai-agents-python/multi_agent/

---

## Discovered Topics and Questions

1. **Agent2Agent (A2A) Communication** - The presentation should also cover ASP.NET-based inter-process agent communication vs. in-process orchestration
2. **Microsoft Agent Framework migration** - Semantic Kernel orchestration is migrating to the Agent Framework SDK; a migration guide exists
3. **Foundry Agent Service** - Azure AI Foundry provides a managed, no-code approach to chaining agents (connected agents functionality); workflows are primarily nondeterministic
4. **Model Context Protocol (MCP)** - Standardizes how agents discover and invoke tools; relevant when considering single-agent-with-tools vs. multi-agent
5. **Declarative Spec** - Both Semantic Kernel and Agent Framework support declarative workflow definitions
6. **AgentGroupChat deprecation** - The older `AgentGroupChat` is deprecated in favor of `GroupChatOrchestration`; a migration guide is provided

---

## Next Research (Not Completed)

- [ ] Deep-dive on LangGraph-specific multi-agent patterns and implementation details (LangGraph docs were not accessible during this session)
- [ ] CrewAI process types mapping to standard patterns
- [ ] OpenAI Agents SDK multi-agent patterns comparison
- [ ] Microsoft Build 2025/2026 session content on multi-agent patterns
- [ ] Azure AI Foundry connected agents and compatibility with Semantic Kernel/AutoGen
- [ ] Agent2Agent (A2A) communication patterns via ASP.NET (in-process vs. out-of-process)

---

## Clarifying Questions

1. **Presentation depth:** Should the presentation focus on the five standardized patterns or also include framework-specific implementations (e.g., GraphFlow, SelectorGroupChat as separate topics)?
2. **Code examples:** Should the presentation include code snippets in C# (Semantic Kernel), Python (AutoGen), or both?
3. **Azure AI Foundry focus:** Should we research deeper into how Foundry Agent Service's connected agents work alongside Semantic Kernel orchestration?
