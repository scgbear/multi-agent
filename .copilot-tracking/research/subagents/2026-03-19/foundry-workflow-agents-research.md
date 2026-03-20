# Foundry Workflow Agents vs Code-Based Frameworks — Research Document

> Research completed: 2026-03-19
> Status: **Complete**

---

## Research Topics and Questions

1. What are Foundry Workflow Agents? (definition, deployment, agent types, status, vs connected agents)
2. Workflow Agent Capabilities (orchestration patterns, tools, state management, limitations, routing)
3. YAML/Declarative Definition (schema, format, version control, vs DeclarativeWorkflowBuilder)
4. Comparison: Foundry Workflow Agents vs Code-Based Frameworks (SK, Agent Framework, LangGraph)
5. Enterprise Considerations (infrastructure, observability, security, cost, limitations)

---

## 1. What Are Foundry Workflow Agents?

### Definition

Workflow agents are one of three agent types in Microsoft Foundry Agent Service (alongside **prompt agents** and **hosted agents**). They orchestrate a sequence of actions or coordinate multiple agents using **declarative definitions**. Users build workflows visually in the Foundry portal or define them in YAML through Visual Studio Code.

Workflow agents are described as:

> "UI-based tools in Microsoft Foundry. Use them to create declarative, predefined sequences of actions that orchestrate agents and business logic in a visual builder."

### How They Are Defined

Workflow agents can be defined through three interfaces:

1. **Foundry Portal Visual Designer**: A visual node-based workflow builder. Select templates (sequential, group chat, human-in-the-loop), add nodes (agent, logic, data transformation, basic chat), and wire them together.
2. **YAML Editor (in Foundry Portal)**: Toggle "YAML Visualizer View" to edit the workflow as a YAML file. Changes bidirectionally sync between visual and YAML views. Each save creates an immutable version.
3. **VS Code for the Web**: Open workflow YAML or code from the Foundry portal into VS Code for the Web. Edit YAML on the left with a visual graph on the right. Deploy changes back to Foundry.

### Deployment Model

- **Fully managed**: Foundry handles hosting, scaling, and runtime. No container management required.
- **Versioning**: Each save creates a new immutable version with full version history.
- **Publishing**: Workflows are published as agent applications with stable endpoints. Distributable through Microsoft 365 Copilot, Teams, and the Entra Agent Registry.

### Agent Types That Can Participate

Any **Foundry agent** from the project can participate in a workflow, including:

- Prompt agents (existing or newly created during workflow setup)
- Other workflow agents (nested)
- Agents configured with tools (file search, web search, code interpreter, MCP servers, custom functions)

### Current Status

- **Workflow agents**: **Preview** (as of March 2026)
- **Prompt agents**: Generally Available
- **Hosted agents**: Preview

### How Workflow Agents Differ from Connected Agents (Classic)

| Aspect | Connected Agents (Classic) | Workflow Agents |
|---|---|---|
| **Status** | Deprecated (retiring March 31, 2027). API `2025-05-15-preview` | Active preview. API `2025-11-15-preview` |
| **Orchestration** | Primary agent uses NL to route to subagents nondeterministically | Declarative, predefined sequences with deterministic flow control |
| **Flow Control** | No explicit branching or conditionals | If/else, for-each, go-to, condition groups |
| **Human-in-the-Loop** | Not built-in | Built-in (questions, approvals, external input) |
| **Visual Builder** | No | Yes (Foundry portal + VS Code) |
| **Definition** | SDK/API or portal configuration | YAML or visual builder |

Microsoft explicitly recommends migrating from connected agents to workflows:

> "We highly recommend you to migrate to use the `2025-11-15-preview` API version workflows for multi-agent orchestration."

---

## 2. Workflow Agent Capabilities

### Orchestration Patterns

Foundry provides templates for three common patterns:

| Pattern | Description | Use Cases |
|---|---|---|
| **Human in the Loop** | Asks the user a question and awaits user input to proceed | Approval requests, obtaining information from the user |
| **Sequential** | Passes the result from one agent to the next in a defined order | Step-by-step workflows, pipelines, multi-stage processing |
| **Group Chat** | Dynamically passes control between agents based on context or rules | Dynamic workflows, escalation, fallback, expert handoff |

### Node Types

Nodes are the building blocks of a workflow. Common node types:

- **Agent**: Invoke an agent (existing or newly created)
- **Logic**: If/else, go to, for each
- **Data Transformation**: Set a variable, parse a value
- **Basic Chat**: Send a message or ask a question

### Tools Available to Workflow Agents

All Foundry agent tools are available to agents participating in workflows:

- Web search
- File search (vector stores)
- Code interpreter
- Memory (managed, preview)
- MCP servers (Remote Model Context Protocol)
- Custom functions
- Image generation

### State/Context Management

- **Variables**: Workflows use **Power Fx** (Excel-like formula language) for expressions and variable manipulation.
  - **System variables**: `System.LastMessage`, `System.ConversationId`, `Conversation.Id`, `User.Language`, etc.
  - **Local variables**: `Local.*` namespace for workflow-scoped state.
- **JSON Output**: Agent nodes can return structured JSON output via configurable response format schemas. Output is saved to named variables for downstream consumption.
- **Conversation Context**: Conversation IDs flow through the workflow, enabling agents to share conversation history.

### Limitations

- **No auto-save**: Foundry does not save workflows automatically; must select Save after every change.
- **Workflow timeouts**: Complex workflows can time out; recommendation is to break into smaller segments.
- **Preview constraints**: Workflow agents are in preview; no SLA, not recommended for production workloads.
- **Pattern limitations**: Only three built-in patterns (sequential, group chat, HITL). No explicit concurrent/parallel pattern in the portal (though group chat enables dynamic delegation).
- **Service limits**: Max 128 tools per agent, 100K messages per thread, 512 MB file size.

### Routing/Branching

- **If/Else**: Build conditions using Power Fx expressions referencing system or local variables.
- **Go To**: Jump to specific nodes in the workflow.
- **For Each**: Iterate over collections.
- **Condition Groups**: No explicit switch/case in portal, but achievable via nested if/else.

---

## 3. YAML/Declarative Definition

### Portal YAML Format

In the Foundry portal, toggling "YAML Visualizer View" exposes the workflow as a YAML file. The portal YAML format is tightly integrated with the visual designer. Changes in either view sync bidirectionally. The portal YAML uses Power Fx for expressions and supports the same node types as the visual builder.

### VS Code Integration

The **Microsoft Foundry for Visual Studio Code extension** provides:

1. **View & Edit**: Open workflow YAML from Foundry portal in VS Code for the Web. YAML on the left, visual graph on the right.
2. **Test**: Remote Agent Playground in VS Code for testing workflows.
3. **Convert to Code**: "Generate Code" button converts YAML workflow to Agent Framework code (Python or C#) using GitHub Copilot. Generated code can run locally with a visualizer, then deploy back to Foundry.

### Version Control

- Each save in Foundry creates an immutable version with version history.
- Workflows are YAML files, so they can be stored in Git via the VS Code integration.
- The Copilot Studio VS Code extension supports Git integration for agent definitions.

### Agent Framework Declarative Workflows (`DeclarativeWorkflowBuilder`)

The **Microsoft Agent Framework** provides a separate (but related) declarative workflow system using YAML. This is a code-based SDK feature, not the Foundry portal feature.

**YAML Schema (C#):**

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: my_workflow
  actions:
    - kind: ActionType
      id: unique_action_id
      displayName: Human readable name
      # Action-specific properties
```

**Action Types Available:**

| Category | Actions | C# | Python |
|---|---|---|---|
| Variable Management | SetVariable, SetMultipleVariables, ResetVariable | ✅ | ✅ |
| Variable Management | SetTextVariable, ClearAllVariables, ParseValue, EditTableV2 | ✅ | ❌ |
| Control Flow | If, ConditionGroup, Foreach, BreakLoop, ContinueLoop, GotoAction | ✅ | ✅ |
| Control Flow | RepeatUntil | ❌ | ✅ |
| Output | SendActivity | ✅ | ✅ |
| Agent Invocation | InvokeAzureAgent | ✅ | ✅ |
| Tool Invocation | InvokeFunctionTool | ✅ | ✅ |
| Tool Invocation | InvokeMcpTool | ✅ | ❌ |
| Human-in-the-Loop | Question, RequestExternalInput | ✅ | ✅ |
| Workflow Control | EndWorkflow, EndConversation, CreateConversation | ✅ | ✅ |
| Conversation | AddConversationMessage, CopyConversationMessages, RetrieveConversationMessage(s) | ✅ | ❌ |

**Key Features:**

- **Checkpointing**: `CheckpointManager` supports in-memory and file-based checkpoints. Workflows can resume from saved checkpoints for fault tolerance.
- **Streaming Execution**: `InProcessExecution.RunStreamingAsync` with event-based processing.
- **Expression Language**: Power Fx (same as Foundry portal), prefixed with `=`.
- **Agent Provider**: `AzureAgentProvider` connects to Azure AI Foundry agents.
- **MCP Integration**: `InvokeMcpTool` action for calling external MCP servers.

**Relationship between Foundry YAML and Agent Framework YAML:**

- Foundry portal workflows can be **converted** to Agent Framework code via the VS Code "Generate Code" feature.
- The Agent Framework `DeclarativeWorkflowBuilder` loads standalone YAML files and executes them locally or in hosted agent containers.
- The YAML schemas share concepts (Power Fx, action types) but the Agent Framework YAML is more code-oriented with `kind: Workflow` and `trigger` structures.

---

## 4. Comparison: Foundry Workflow Agents vs Code-Based Frameworks

### Overview Comparison Table

| Dimension | Foundry Workflow Agents | Agent Framework (Programmatic) | Agent Framework (Declarative YAML) | Semantic Kernel Orchestration |
|---|---|---|---|---|
| **Code Required** | No (YAML optional) | Yes (C#/Python) | YAML + host code | Yes (C#/Python/Java) |
| **Hosting** | Fully managed | Self-hosted or Container Apps | Self-hosted or hosted agents | Self-hosted |
| **Orchestration Patterns** | Sequential, Group Chat, HITL | Sequential, Concurrent, Handoff, Group Chat, Magentic | Sequential, Conditional, Loops, HITL | Sequential, Concurrent, Handoff, Group Chat, Magentic |
| **Graph Flexibility** | Linear + branching | Full directed graph (executors + edges) | Action sequences with control flow | Pattern-based (not free-form graph) |
| **Checkpointing** | No (managed state) | Yes (in-memory, file-based, custom) | Yes (CheckpointManager) | No built-in |
| **Human-in-the-Loop** | Built-in (portal) | Built-in (all orchestrations) | RequestExternalInput, Question | Supported (some patterns) |
| **Visual Designer** | Yes (Foundry portal) | No | VS Code graph view | No |
| **Expression Language** | Power Fx | N/A (code) | Power Fx | N/A (code) |
| **Version Control** | Foundry versioning + Git via VS Code | Standard Git | Standard Git + Foundry versioning | Standard Git |
| **Deployment Target** | Foundry Agent Service | Container Apps, any host | Foundry Hosted Agents or local | Any host |
| **Status** | Preview | GA (open-source) | Preview (prerelease packages) | Experimental |

### Foundry Workflow Agents vs Semantic Kernel Orchestration

| Aspect | Foundry Workflow Agents | Semantic Kernel Orchestration |
|---|---|---|
| **Pattern Support** | Sequential, Group Chat, HITL | Sequential, Concurrent, Handoff, Group Chat, Magentic |
| **Concurrent Execution** | Not explicit (group chat can simulate) | Native concurrent pattern |
| **Magentic (Manager Agent)** | Not available | MagenticOne-inspired manager orchestration |
| **Flexibility** | Constrained to supported patterns and Power Fx | Full programmatic flexibility |
| **Foundry Integration** | Native | Agents can be Foundry agents, but orchestration runs externally |
| **Learning Curve** | Low (visual builder, Power Fx) | Medium-High (C#/Python SDK) |
| **Migration Path** | Convert YAML to Agent Framework code; SK → Agent Framework migration guide exists | Direct API; migrating to Agent Framework recommended |

> **Note**: Semantic Kernel agent orchestration is experimental and Microsoft recommends migrating to Agent Framework. A migration guide exists for `AgentGroupChat` → `GroupChatOrchestration`.

### Foundry Workflow Agents vs Agent Framework Workflow API

| Aspect | Foundry Workflow Agents | Agent Framework Workflows |
|---|---|---|
| **Graph Model** | Node sequence with branching | Directed graph (executors + edges) |
| **Checkpointing** | Managed (no user control) | Full checkpoint control (in-memory, file, custom) |
| **Resumability** | No explicit resume | Resume from checkpoint on any machine |
| **Human-in-the-Loop** | Portal-native questions/approvals | RequestExternalInput event pattern |
| **Tool Integration** | Foundry built-in tools | InvokeFunctionTool, InvokeMcpTool, custom |
| **Orchestration Patterns** | 3 templates | 5 built-in + custom graph patterns |
| **State Management** | Variables (Power Fx) | Variables (Power Fx) + code-level state |
| **Deployment** | Foundry-managed | Container Apps, hosted agents, or any host |
| **Observability** | Foundry tracing + App Insights | OpenTelemetry + custom |

### Foundry Workflow Agents vs LangGraph

| Aspect | Foundry Workflow Agents | LangGraph |
|---|---|---|
| **Graph Flexibility** | Pre-defined patterns, branching | Arbitrary cyclic/acyclic graphs |
| **Deployment Model** | Fully managed (Foundry) | LangGraph Cloud or self-hosted (or Foundry hosted agents) |
| **Code Required** | No | Yes (Python) |
| **Checkpointing** | Managed | Built-in persistent checkpointing |
| **Human-in-the-Loop** | Built-in | Supported via interrupt/resume |
| **Streaming** | Via chat window | Native streaming support |
| **Foundry Integration** | Native | Deploy as hosted agent in Foundry |
| **Complexity** | Low | High (full graph programming) |

### When to Use Each Approach

| Scenario | Recommended Approach |
|---|---|
| Rapid prototyping of multi-agent workflows | **Foundry Workflow Agents** |
| Non-developers need to create/modify workflows | **Foundry Workflow Agents** |
| Simple sequential or approval workflows | **Foundry Workflow Agents** |
| Complex custom logic or dynamic graphs | **Agent Framework (Programmatic)** |
| Workflows that change frequently but don't need code | **Agent Framework (Declarative YAML)** |
| Maximum flexibility and control | **Agent Framework (Programmatic)** or **LangGraph** |
| Existing Semantic Kernel workloads | **Semantic Kernel** → migrate to **Agent Framework** |
| Deterministic orchestration required | **Agent Framework** or **Foundry Workflow Agents** |
| Need checkpointing and fault tolerance | **Agent Framework** |
| Mixed: start no-code, graduate to code | **Foundry Workflows** → "Generate Code" → **Agent Framework** |

---

## 5. Enterprise Considerations

### Managed Infrastructure

- **Foundry Workflow Agents**: Fully managed. No container management, server provisioning, or scaling configuration. Foundry handles runtime, identity, and lifecycle.
- **Hosted Agents**: Container-based but Foundry manages provisioning, autoscaling (0-5 replicas during preview), and lifecycle. Developers manage container images.
- **Agent Framework (self-hosted)**: Developers manage infrastructure (Container Apps, VMs, etc.).

### Observability and Monitoring

| Capability | Foundry (Workflow + Prompt) | Foundry (Hosted) | Agent Framework (Self-hosted) |
|---|---|---|---|
| End-to-end tracing | ✅ Foundry traces + App Insights | ✅ OpenTelemetry + App Insights | Manual setup |
| Agent metrics dashboard | ✅ Built-in | ✅ Built-in | Custom |
| Visual workflow monitoring | ✅ Each node shows completion status | ❌ | ❌ |
| Version comparison | ✅ | ✅ | Standard Git diff |
| Evaluation (quality/safety) | ✅ Built-in evaluators | ✅ SDK evaluators | Custom |

### Security

| Capability | Foundry Agent Service |
|---|---|
| Identity | Microsoft Entra identity per agent; dedicated identity for published agents |
| Network isolation | Virtual network support for prompt and workflow agents; **not** for hosted agents during preview |
| RBAC | Azure RBAC (Contributor+ for workflow creation) |
| Content safety | Integrated content filters, prompt injection mitigation (XPIA) |
| Guardrails | Built-in guardrail framework |
| Data residency | Bring-your-own resources (storage, AI Search, Cosmos DB) |

### Cost Model

- **Prompt Agents + Workflow Agents**: Pay for model token consumption (Azure OpenAI/model deployment quotas). No separate compute charge for the agent runtime itself.
- **Hosted Agents**: Billing for managed hosting runtime enabled no earlier than April 1, 2026 (currently free during preview). Plus model token costs.
- **Agent Framework (self-hosted)**: Standard Azure compute costs (Container Apps, VMs) plus model token costs.

### Limitations for Complex Scenarios

| Limitation | Detail |
|---|---|
| No parallel execution in workflows | Workflow agents don't support explicit parallel/concurrent patterns |
| Preview status | Workflow agents and hosted agents are preview; no SLA |
| Hosted agent network isolation | Not supported during preview |
| Hosted agent replica limits | Max 5 replicas, max 4 CPU / 8 GiB memory per replica (preview) |
| No custom checkpoint control | Foundry workflow agents don't expose checkpoint/resume APIs |
| Portal-dependent | Workflows must be initially created in Foundry portal (then edited in VS Code) |
| Agent Framework declarative is prerelease | NuGet packages are `--prerelease` |

---

## References

| Source | URL |
|---|---|
| Build a workflow in Microsoft Foundry | <https://learn.microsoft.com/azure/foundry/agents/concepts/workflow> |
| What is Microsoft Foundry Agent Service? | <https://learn.microsoft.com/azure/foundry/agents/overview> |
| Add declarative agent workflows in VS Code | <https://learn.microsoft.com/azure/foundry/agents/how-to/vs-code-agents-workflow-low-code> |
| Agent Framework Declarative Workflows | <https://learn.microsoft.com/agent-framework/workflows/declarative> |
| Agent Framework Workflows Overview | <https://learn.microsoft.com/agent-framework/workflows/> |
| Agent Framework Orchestrations | <https://learn.microsoft.com/agent-framework/workflows/orchestrations/> |
| Foundry limits, quotas, and regions | <https://learn.microsoft.com/azure/foundry/agents/concepts/limits-quotas-regions> |
| What are hosted agents? | <https://learn.microsoft.com/azure/foundry/agents/concepts/hosted-agents> |
| Agent development lifecycle | <https://learn.microsoft.com/azure/foundry/agents/concepts/development-lifecycle> |
| Connected agents (classic, deprecated) | <https://learn.microsoft.com/azure/foundry-classic/agents/how-to/connected-agents> |
| Semantic Kernel Agent Orchestration | <https://learn.microsoft.com/semantic-kernel/frameworks/agent/agent-orchestration/> |
| AI agent orchestration patterns (Azure Architecture) | <https://learn.microsoft.com/azure/architecture/ai-ml/guide/ai-agent-design-patterns> |
| Technology plan for AI agents | <https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/technology-solutions-plan-strategy> |
| Agent transparency note | <https://learn.microsoft.com/azure/foundry/responsible-ai/agents/transparency-note> |
| Multi-agent workflow automation architecture | <https://learn.microsoft.com/azure/architecture/ai-ml/idea/multiple-agent-workflow-automation> |

---

## Discovered Topics

1. **Power Fx expression language** in workflows — Excel-like formulas for variable manipulation and conditions.
2. **Foundry-to-code conversion** — VS Code "Generate Code" button uses GitHub Copilot to convert YAML workflows to Agent Framework Python/C#.
3. **Hosted agents as "graduation path"** — Start with workflow agents (no-code), convert to Agent Framework code, deploy as hosted agents.
4. **Copilot Studio VS Code extension** — Related YAML-based agent definition editing with Git integration (separate from Foundry extension but conceptually parallel).
5. **Agent Framework combines SK + AutoGen** — Agent Framework is described as combining Semantic Kernel and AutoGen orchestrators into a single orchestrator.
6. **`azd ai agent` CLI extension** — Azure Developer CLI extension for streamlined hosted agent provisioning and deployment.

---

## Key Discoveries

1. **Workflow agents are the official replacement for connected agents.** Connected agents are deprecated and retiring March 2027. Microsoft recommends the `2025-11-15-preview` API workflows.

2. **Three-tier agent architecture in Foundry**: Prompt (simple) → Workflow (multi-agent orchestration, no code) → Hosted (full code control). This creates a clear graduation path.

3. **YAML is a first-class citizen** in both Foundry workflows and Agent Framework declarative workflows. The Foundry portal YAML and Agent Framework YAML share concepts (Power Fx, action types) but have different schemas.

4. **Agent Framework Declarative Workflows have richer action types** than the Foundry portal: 30+ action types including InvokeMcpTool, InvokeFunctionTool, ConditionGroup, Foreach, GotoAction, with full checkpointing support.

5. **"Generate Code" is a key bridge**: The VS Code extension can convert Foundry YAML workflows to Agent Framework code (C#/Python) using GitHub Copilot, enabling teams to start no-code and graduate to code.

6. **Agent Framework orchestration patterns are richer**: 5 built-in patterns (Sequential, Concurrent, Handoff, Group Chat, Magentic) versus Foundry's 3 templates (Sequential, Group Chat, HITL).

7. **Checkpointing is a major differentiator**: Agent Framework supports save/resume from checkpoints (in-memory, file, custom). Foundry workflow agents have managed state but no user-controlled checkpointing.

8. **Foundry workflow agents support private networking** during preview, but hosted agents do **not**.

9. **Semantic Kernel orchestration is experimental** and Microsoft recommends migration to Agent Framework. Migration guides exist.

10. **Cost advantage of Foundry workflows**: No compute charges for the agent runtime; you only pay for model tokens. Hosted agent compute billing starts April 2026 at earliest.

---

## Clarifying Questions

_None — all research topics were addressable through documentation._
