# Microsoft Agent Framework Workflow API Deep-Dive

## Research Status: Complete

## Research Topics

1. Workflow API Architecture
2. Executors (Node Types)
3. Edges and Routing
4. Checkpointing and State
5. Human-in-the-Loop
6. Built-in Orchestration Patterns
7. Code Examples (Python and C#)

---

## 1. Workflow API Architecture

### What Is the Workflow Abstraction?

A **Workflow** is a directed graph that ties **executors** (processing nodes) and **edges** (connections) together, managing execution, message routing, and event streaming. Unlike an agent (which is LLM-driven and takes dynamic steps), a workflow is a **predefined sequence of operations** that can include AI agents as components. The execution path is explicitly defined, giving developers full control.

> "Agent Framework combines AutoGen's simple agent abstractions with Semantic Kernel's enterprise features — session-based state management, type safety, middleware, telemetry — and adds graph-based workflows for explicit multi-agent orchestration."
> — [Agent Framework Overview](https://learn.microsoft.com/en-us/agent-framework/overview/)

### Data-Flow vs Control-Flow (AutoGen Comparison)

| Aspect | AutoGen GraphFlow | Agent Framework Workflow |
|---|---|---|
| **Flow Type** | Control flow (edges are transitions) | Data flow (edges route messages) |
| **Node Types** | Agents only | Agents, functions, sub-workflows |
| **Activation** | Message broadcast | Edge-based activation |
| **Type Safety** | Limited | Strong typing throughout |
| **Composability** | Limited | Highly composable |

Key differences:

- **GraphFlow** broadcasts messages to all agents; edges represent conditional transitions.
- **Workflow** routes typed messages along specific edges. Executors are activated only when edges deliver messages to them.
- **Workflow** supports request/response pause for external input and checkpointing for persistence and resume.

Source: [AutoGen Migration Guide — Multi-Agent Feature Mapping](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/#multi-agent-feature-mapping)

### WorkflowBuilder API

The `WorkflowBuilder` class provides a **fluent API** for constructing workflow graphs programmatically. You incrementally add executors and edges, then call `.Build()` / `.build()` to produce an immutable `Workflow` instance.

#### Python

```python
from agent_framework import WorkflowBuilder, Executor, WorkflowContext, handler
from typing_extensions import Never

class UpperCaseExecutor(Executor):
    @handler
    async def process(self, text: str, ctx: WorkflowContext[str]) -> None:
        await ctx.send_message(text.upper())

class ReverseExecutor(Executor):
    @handler
    async def process(self, text: str, ctx: WorkflowContext[Never, str]) -> None:
        await ctx.yield_output(text[::-1])

upper = UpperCaseExecutor(id="upper")
reverse = ReverseExecutor(id="reverse")

workflow = (
    WorkflowBuilder(start_executor=upper)
    .add_edge(upper, reverse)
    .build()
)

events = await workflow.run("hello")
print(events.get_outputs())  # ['OLLEH']
```

#### C\#

```csharp
using Microsoft.Agents.AI.Workflows;

var processor = new DataProcessor();
var validator = new Validator();
var formatter = new Formatter();

WorkflowBuilder builder = new(processor); // Set starting executor
builder.AddEdge(processor, validator);
builder.AddEdge(validator, formatter);
var workflow = builder.Build();
```

**Python constructor parameters:**

- `start_executor` (required) — the entry-point executor
- `max_iterations` — safety limit for execution cycles
- `name`, `description` — metadata
- `checkpoint_storage` — optional `CheckpointStorage` for persistence
- `output_executors` — optional list to constrain which executors' outputs are collected

**C# constructor** takes the starting `ExecutorBinding` as the sole required parameter.

### Typed Edges

Edges route **typed messages** between executors. When an executor emits a message via `send_message()` or a return value, the framework routes it along the edges defined in the graph. Edges can be **typed** — e.g., `AddEdge<T>()` in C# — so that only messages of a specific type trigger that edge. This enables conditional routing based on message type rather than just conditions.

### Execution Model: Supersteps (Pregel/BSP)

The framework uses a modified **Pregel** (Bulk Synchronous Parallel) execution model with superstep-based processing:

1. Collect all pending messages from the previous superstep
2. Route messages to target executors based on edge definitions
3. Run all target executors concurrently within the superstep
4. Wait for all executors to complete (synchronization barrier)
5. Queue new messages for the next superstep

This provides: **deterministic execution**, **reliable checkpointing** at superstep boundaries, and **no race conditions** between supersteps.

Source: [Workflow Builder & Execution](https://learn.microsoft.com/en-us/agent-framework/workflows/workflows)

---

## 2. Executors (Node Types)

Executors are **autonomous processing units** that receive typed messages, perform operations, and produce output messages or events. Each executor has a unique identifier and can handle specific message types.

### Types of Executors

1. **Custom logic components** — process data, call APIs, transform messages
2. **AI agents** — use LLMs to generate responses (agents are auto-wrapped as executors)
3. **Sub-workflows** — nested workflows wrapped as `WorkflowExecutor`

### Custom Executor (C#)

```csharp
using Microsoft.Agents.AI.Workflows;

internal sealed partial class UppercaseExecutor() : Executor("UppercaseExecutor")
{
    [MessageHandler]
    private ValueTask<string> HandleAsync(string message, IWorkflowContext context)
    {
        return ValueTask.FromResult(message.ToUpperInvariant());
    }
}
```

C# uses `[MessageHandler]` attribute with source generation on `partial` classes for compile-time validation and AOT compatibility.

### Custom Executor (Python)

```python
from agent_framework import Executor, WorkflowContext, handler

class UpperCaseExecutor(Executor):
    @handler
    async def process(self, text: str, ctx: WorkflowContext[str]) -> None:
        await ctx.send_message(text.upper())
```

Python uses the `@handler` decorator with type hints for input/output routing.

### Function-Based Executors (C#)

```csharp
Func<string, string> uppercaseFunc = s => s.ToUpperInvariant();
var uppercase = uppercaseFunc.BindExecutor("UppercaseExecutor");
```

### Function-Based Executors (Python)

```python
from agent_framework import executor, WorkflowContext
from typing_extensions import Never

@executor(id="my_func")
async def my_func(input_msg: str, ctx: WorkflowContext[Never, str]) -> None:
    await ctx.yield_output(input_msg.upper())
```

### Multiple Input Types (C#)

```csharp
internal sealed partial class SampleExecutor() : Executor("SampleExecutor")
{
    [MessageHandler]
    private ValueTask<string> HandleStringAsync(string message, IWorkflowContext context)
        => ValueTask.FromResult(message.ToUpperInvariant());

    [MessageHandler]
    private ValueTask<int> HandleIntAsync(int message, IWorkflowContext context)
        => ValueTask.FromResult(message * 2);
}
```

### Agent as Executor (C# — Direct Usage)

AI agents can be passed directly to `WorkflowBuilder` without wrapping:

```csharp
var workflow = new WorkflowBuilder(frenchAgent)
    .AddEdge(frenchAgent, spanishAgent)
    .AddEdge(spanishAgent, englishAgent)
    .Build();
```

Agents cache messages and only process when they receive a `TurnToken`.

### Sub-Workflow Executor (Python)

```python
from agent_framework import WorkflowExecutor, WorkflowBuilder

sub_workflow = (
    WorkflowBuilder(start_executor=specialist1_executor)
    .add_edge(specialist1_executor, specialist2_executor)
    .build()
)

sub_workflow_executor = WorkflowExecutor(workflow=sub_workflow, id="sub_process")

parent_workflow = (
    WorkflowBuilder(start_executor=coordinator_executor)
    .add_edge(coordinator_executor, sub_workflow_executor)
    .add_edge(sub_workflow_executor, reviewer_executor)
    .build()
)
```

Sub-workflow nesting characteristics:

- **Isolated input/output** through `WorkflowExecutor`
- **No message broadcasting** — data flows through specific connections
- **Independent state management** for each workflow level

### The IWorkflowContext / WorkflowContext Object

Provides methods for interacting with the workflow during execution:

- `SendMessageAsync` / `send_message()` — send messages to connected executors
- `YieldOutputAsync` / `yield_output()` — produce workflow outputs returned/streamed to the caller
- `QueueStateUpdateAsync` / `set_state()` — write to shared state
- `ReadStateAsync` / `get_state()` — read shared state
- `request_info()` — request external/human input (pause workflow)

Source: [Executors](https://learn.microsoft.com/en-us/agent-framework/workflows/executors)

---

## 3. Edges and Routing

Edges define how messages flow between executors. They represent connections in the workflow graph and determine data flow paths.

### Edge Types

| Type | Description | Use Case |
|---|---|---|
| **Direct** | Simple one-to-one connections | Linear pipelines |
| **Conditional** | Edges with predicates | Binary routing (if/else) |
| **Switch-Case** | Route to different executors based on conditions | Multi-branch routing |
| **Multi-Selection (Fan-out)** | One executor → multiple targets | Parallel processing |
| **Fan-in (Barrier)** | Multiple executors → single target | Aggregation |

### Direct Edges

```csharp
// C#
WorkflowBuilder builder = new(sourceExecutor);
builder.AddEdge(sourceExecutor, targetExecutor);
```

```python
# Python
workflow = (
    WorkflowBuilder(start_executor=executor_a)
    .add_edge(executor_a, executor_b)
    .build()
)
```

### Conditional Edges

```csharp
// C# — condition on typed message
builder.AddEdge(spamDetectionExecutor, emailAssistantExecutor,
    condition: GetCondition(expectedResult: false));
builder.AddEdge(spamDetectionExecutor, handleSpamExecutor,
    condition: GetCondition(expectedResult: true));
```

Condition function:

```csharp
private static Func<object?, bool> GetCondition(bool expectedResult) =>
    detectionResult => detectionResult is DetectionResult result
                       && result.IsSpam == expectedResult;
```

### Switch-Case Edges (C#)

```csharp
builder.AddSwitch(spamDetectionExecutor, switchBuilder =>
    switchBuilder
    .AddCase(GetCondition(SpamDecision.NotSpam), emailAssistantExecutor)
    .AddCase(GetCondition(SpamDecision.Spam), handleSpamExecutor)
    .WithDefault(handleUncertainExecutor)
);
```

### Switch-Case Edges (Python)

```python
from agent_framework import Case, Default

builder.add_switch_case_edge_group(
    source=analysis_executor,
    cases=[
        Case(condition=lambda msg: msg.decision == "safe", target=safe_handler),
        Case(condition=lambda msg: msg.decision == "spam", target=spam_handler),
        Default(target=uncertain_handler),
    ],
)
```

### Fan-Out (Parallel Processing)

```csharp
// C# — unconditional fan-out
builder.AddFanOutEdge(source, [target1, target2, target3]);

// C# — with custom target selector
builder.AddFanOutEdge(
    emailAnalysisExecutor,
    targets: [handleSpamExecutor, emailAssistantExecutor, emailSummaryExecutor, handleUncertainExecutor],
    targetSelector: GetTargetAssigner()
);
```

```python
# Python — unconditional fan-out
builder.add_fan_out_edges(source, [val_a, val_b])

# Python — with selection function
builder.add_multi_selection_edge_group(
    source=analysis_executor,
    targets=[handler1, handler2, handler3],
    selection_func=my_selector,
)
```

### Fan-In (Barrier Join)

```csharp
// C# — barrier: waits for ALL sources before forwarding
builder.AddFanInBarrierEdge(sources: [worker1, worker2, worker3], target: aggregatorExecutor);
```

```python
# Python
builder.add_fan_in_edges([executor_c, executor_d], executor_e)
```

### Chain Helper (C#)

```csharp
builder.AddChain(source, [step1, step2, step3]);
```

```python
# Python
builder.add_chain([executor_a, executor_b, executor_c])
```

Source: [Edges](https://learn.microsoft.com/en-us/agent-framework/workflows/edges)

---

## 4. Checkpointing and State

### Shared State

State allows multiple executors to share data without direct message passing.

```csharp
// C# — write state
await context.QueueStateUpdateAsync(fileID, fileContent, scopeName: "FileContent");

// C# — read state
var content = await context.ReadStateAsync<string>(fileID, scopeName: "FileContent");
```

```python
# Python — executor-local state
prev_state = await ctx.get_executor_state() or {}
count = prev_state.get("count", 0) + 1
await ctx.set_executor_state({"count": count, "last_input": data})

# Python — shared state
ctx.set_state("original_input", data)
```

### FileCheckpointStorage

Built-in checkpointing captures:

- **Executor state** — local state per executor
- **Shared state** — cross-executor state
- **Message queues** — pending messages between executors
- **Workflow position** — current execution progress

```python
from agent_framework import FileCheckpointStorage, WorkflowBuilder

checkpoint_storage = FileCheckpointStorage(storage_path="./checkpoints")

workflow = (
    WorkflowBuilder(
        start_executor=processing_executor,
        checkpoint_storage=checkpoint_storage
    )
    .add_edge(processing_executor, finalize_executor)
    .build()
)
```

### Resuming from Checkpoints

```python
# List available checkpoints
checkpoints = await checkpoint_storage.list_checkpoints()

for checkpoint in checkpoints:
    summary = get_checkpoint_summary(checkpoint)
    print(f"Checkpoint {summary.checkpoint_id}: iteration={summary.iteration_count}")

# Resume from a specific checkpoint
new_workflow = create_workflow(checkpoint_storage)
async for event in new_workflow.run_stream(
    checkpoint_id=chosen_checkpoint_id,
    checkpoint_storage=checkpoint_storage
):
    print(f"Resumed event: {event}")
```

### Key Benefits

- **Automatic persistence** — no manual state management
- **Granular recovery** — resume from any superstep boundary
- **State isolation** — separate executor-local and shared state
- **Human-in-the-loop integration** — seamless pause-resume
- **Fault tolerance** — robust recovery from failures

Source: [State Management](https://learn.microsoft.com/en-us/agent-framework/workflows/state), [AutoGen Migration — Checkpointing](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/#checkpointing-and-resuming-workflows)

---

## 5. Human-in-the-Loop

### ctx.request_info() and @response_handler

Agent Framework provides **built-in request-response capabilities** where any executor can pause the workflow and wait for external/human input.

#### Python

```python
from agent_framework import (
    RequestInfoEvent, Executor, handler, response_handler, WorkflowContext
)
from dataclasses import dataclass

@dataclass
class ApprovalRequest:
    content: str = ""
    agent_name: str = ""

class ReviewerExecutor(Executor):

    @handler
    async def review_content(self, agent_response: str, ctx: WorkflowContext) -> None:
        approval_request = ApprovalRequest(
            content=agent_response,
            agent_name="writer_agent"
        )
        await ctx.request_info(request_data=approval_request, response_type=str)

    @response_handler
    async def handle_approval_response(
        self,
        original_request: ApprovalRequest,
        decision: str,
        ctx: WorkflowContext
    ) -> None:
        if decision.strip().lower() == "approved":
            await ctx.yield_output(f"APPROVED: {original_request.content}")
        else:
            await ctx.yield_output(f"REVISION NEEDED: {decision}")
```

### Running Human-in-the-Loop Workflows

```python
async def run_with_human_input():
    pending_responses = None
    completed = False

    while not completed:
        stream = (
            workflow.send_responses_streaming(pending_responses)
            if pending_responses
            else workflow.run_stream("initial input")
        )

        events = [event async for event in stream]
        pending_responses = None

        for event in events:
            if event.type == "request_info":
                request_data = event.data
                print(f"Review needed: {request_data.content}")
                human_response = input("Enter 'approved' or revision notes: ")
                pending_responses = {event.request_id: human_response}
            elif event.type == "output":
                print(f"Final result: {event.data}")
                completed = True
```

### Checkpointing + Human-in-the-Loop

When resuming from a checkpoint that contains pending requests, those requests are re-emitted as events, allowing workflows to be paused across server restarts.

Source: [AutoGen Migration — Human-in-the-Loop](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/#human-in-the-loop-with-request-response)

---

## 6. Built-in Orchestration Patterns

The `agent_framework.orchestrations` package (Python) and `AgentWorkflowBuilder` (C#) provide pre-built workflow patterns that internally use `WorkflowBuilder`:

```python
from agent_framework.orchestrations import (
    SequentialBuilder,
    ConcurrentBuilder,
    HandoffBuilder,
    GroupChatBuilder,
    MagenticBuilder,
)
```

### SequentialBuilder

Chains agents in sequence with shared conversation context. Equivalent to AutoGen's `RoundRobinGroupChat`.

```python
# Python
workflow = SequentialBuilder(participants=[agent1, agent2, agent3]).build()

async for event in workflow.run("Create a summary about EVs", stream=True):
    if event.type == "output":
        print(event.data)
```

```csharp
// C# — via AgentWorkflowBuilder
// Sequential is built using AddChain or sequential AddEdge calls
```

Internally wires: `input → InputToConversation → participant1 → ResponseToConversation → participant2 → … → End`

### ConcurrentBuilder

Fan-out to multiple agents in parallel, then aggregate results.

```python
# Python
workflow = ConcurrentBuilder(participants=[agent1, agent2, agent3]).build()

# With custom aggregator
def summarize(results):
    return " | ".join(r.agent_response.messages[-1].text for r in results)

workflow = ConcurrentBuilder(participants=[agent1, agent2, agent3]).with_aggregator(summarize).build()
```

Internally wires: `dispatcher → fan-out → [participants] → fan-in → aggregator`

### HandoffBuilder

Decentralized agent routing where agents decide handoff targets using tools.

```python
# Python
workflow = (
    HandoffBuilder()
    .participants([triage, billing, support])
    .with_start_agent(triage)
    .add_handoff(triage, [billing, support])
    .build()
)
```

```csharp
// C# — via HandoffsWorkflowBuilder
var workflow = new HandoffsWorkflowBuilder()
    .WithInitialAgent(triageAgent)
    .AddAgentWithHandoffs(triageAgent, [billingTarget, supportTarget])
    .Build();
```

Internally creates a fully connected graph with `HandoffAgentExecutor` instances that use `AddSwitch` for routing.

### GroupChatBuilder

Orchestrator-directed multi-agent conversations with pluggable speaker selection.

```python
workflow = GroupChatBuilder(
    participants=[agent1, agent2],
    selection_func=my_selector,
).build()
```

### MagenticBuilder

Sophisticated orchestration using the **Magentic One** pattern: an LLM-powered manager plans, selects agents, monitors progress, and replans.

```python
workflow = MagenticBuilder(
    participants=[researcher, coder, reviewer],
    manager_agent=manager_agent,
    max_round_count=10,
    max_stall_count=3,
    max_reset_count=2,
    enable_plan_review=True,  # Human reviews plans
).build()
```

```csharp
// C# — similar builder pattern available
```

Features:

- Dynamic task planning and replanning
- Progress tracking via ledgers
- Human-in-the-loop: plan review, stall intervention, tool approval
- Checkpointing support

Internally wires: `orchestrator ↔ participant` (bidirectional edges for each participant)

Source: [Orchestrations README](https://github.com/microsoft/agent-framework/tree/main/python/packages/orchestrations/README.md), [AutoGen Migration — Group Chat Patterns](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/#group-chat-patterns)

---

## 7. Key Features Summary

| Feature | Description |
|---|---|
| **Type Safety** | Strong typing ensures messages flow correctly between components |
| **Flexible Control Flow** | Graph-based architecture with conditional routing, parallel processing |
| **External Integration** | Built-in request/response patterns for APIs and HITL |
| **Checkpointing** | Save/resume workflow states at superstep boundaries |
| **Multi-Agent Orchestration** | Sequential, concurrent, handoff, group chat, magentic patterns |
| **Superstep Execution** | Modified Pregel BSP model for deterministic, checkpointable execution |
| **Validation** | Type compatibility, graph connectivity, executor binding, edge validation |
| **Streaming** | Real-time event streaming during workflow execution |
| **Observability** | OpenTelemetry integration with spans for build, execution, and agents |

---

## References

| Source | URL |
|---|---|
| Agent Framework Landing Page | <https://learn.microsoft.com/en-us/agent-framework/> |
| Overview | <https://learn.microsoft.com/en-us/agent-framework/overview/> |
| Workflows Overview | <https://learn.microsoft.com/en-us/agent-framework/workflows/> |
| Executors | <https://learn.microsoft.com/en-us/agent-framework/workflows/executors> |
| Edges | <https://learn.microsoft.com/en-us/agent-framework/workflows/edges> |
| Workflow Builder & Execution | <https://learn.microsoft.com/en-us/agent-framework/workflows/workflows> |
| State Management | <https://learn.microsoft.com/en-us/agent-framework/workflows/state> |
| Agents in Workflows | <https://learn.microsoft.com/en-us/agent-framework/workflows/agents-in-workflows> |
| AutoGen Migration Guide | <https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/> |
| GitHub — WorkflowBuilder.cs | <https://github.com/microsoft/agent-framework/tree/main/dotnet/src/Microsoft.Agents.AI.Workflows/WorkflowBuilder.cs> |
| GitHub — _workflow_builder.py | <https://github.com/microsoft/agent-framework/tree/main/python/packages/core/agent_framework/_workflows/_workflow_builder.py> |
| GitHub — Orchestrations Package | <https://github.com/microsoft/agent-framework/tree/main/python/packages/orchestrations/> |
| GitHub — C# Workflow Samples | <https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/03-workflows> |
| GitHub — Python Workflow Samples | <https://github.com/microsoft/agent-framework/tree/main/python/samples/03-workflows> |
| WorkflowBuilder Python API Ref | <https://learn.microsoft.com/python/api/agent-framework-core/agent_framework.workflowbuilder> |
| WorkflowBuilderExtensions .NET API | <https://learn.microsoft.com/dotnet/api/microsoft.agents.ai.workflows.workflowbuilderextensions> |
| NuGet Package | `Microsoft.Agents.AI.Workflows` (prerelease) |
| PyPI Package | `agent-framework` / `agent-framework-orchestrations` |

---

## Discovered Topics

1. **Superstep execution model** — modified Pregel/BSP with synchronization barriers
2. **TurnToken pattern** — agents cache messages and only process after receiving a TurnToken
3. **WorkflowExecutor** — wraps sub-workflows as executor nodes for composition
4. **DeclarativeWorkflowBuilder** — YAML/declarative workflow definition (Python)
5. **GroupChatBuilder** — orchestrator-directed multi-agent conversations with pluggable selection
6. **HandoffsWorkflowBuilder** (C#) — specialized builder for handoff workflows
7. **Workflow validation** — type compatibility, graph connectivity, executor binding, duplicate edge detection
8. **Workflow immutability** — workflows are immutable after build; builders are mutable
9. **State isolation** — recommended to create fresh workflow instances per task/request
10. **Agent-as-a-Tool pattern** — `agent.as_tool()` for hierarchical architectures
11. **Middleware** — cross-cutting concerns (logging, security, performance) in Agent Framework
12. **MAUI integration** — Agent Framework workflow usage in .NET MAUI apps
13. **Azure Functions / Durable Task integration** — hosting workflows in serverless environments
14. **A2A Protocol integration** — agent-to-agent communication across boundaries
15. **DevUI** — development UI for workflow visualization and debugging

---

## Next Research (Not Completed This Session)

- [ ] Deep-dive into **Events** system (`WorkflowEvent`, `WorkflowOutputEvent`, `ExecutorCompletedEvent`, custom events)
- [ ] Research **DeclarativeWorkflowBuilder** for YAML-based workflow definitions in detail
- [ ] Investigate **Azure Functions / Durable Task** integration for serverless hosting
- [ ] Research **A2A Protocol** integration details
- [ ] Explore **DevUI** for workflow visualization capabilities
- [ ] Research **Workflow.as_agent()** pattern — exposing workflows as agents
- [ ] Look into **C# orchestration builders** (`AgentWorkflowBuilder.BuildSequential`, `BuildConcurrent`, `BuildHandoffs`) in detail
- [ ] Research **GroupChatBuilder** with custom selection functions in depth
- [ ] Investigate **tool approval** patterns within orchestration builders
- [ ] Research **observability** (OpenTelemetry spans, workflow telemetry context)

---

## Clarifying Questions

None — all provided research questions were answerable through documentation and source code.
