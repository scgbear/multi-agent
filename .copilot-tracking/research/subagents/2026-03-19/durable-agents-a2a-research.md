# Azure Functions Durable Agents with A2A Hosting (Serverless)

## Research Status: Complete

## Research Topics and Questions

1. What are Durable Agents? How does Azure Durable Functions relate to agent hosting?
2. Azure Functions + A2A Protocol integration
3. Durable Functions orchestration with AI agents
4. Architecture patterns for serverless agent hosting
5. Code examples and project structure
6. Limitations and considerations

---

## Findings

### 1. What Are Durable Agents?

**Durable Agents** is the term for AI agents hosted in Azure Functions using the **Durable Task extension** for the Microsoft Agent Framework. They combine the Agent Framework's `AIAgent` abstraction with Azure Durable Functions to provide stateful, serverless agent hosting.

#### Core Capabilities

Durable agents provide four key capabilities:

1. **Persist state automatically** across function invocations — conversation history is stored durably
2. **Resume after failures** without losing conversation context
3. **Scale automatically** based on demand (including scale-to-zero)
4. **Orchestrate multi-agent workflows** with reliable execution guarantees

#### How It Works

The `Microsoft.Agents.AI.Hosting.AzureFunctions` NuGet package provides a `ConfigureDurableAgents()` extension method on `FunctionsApplicationBuilder`. When you register agents, the durable task extension:

- Automatically creates HTTP endpoints for each agent (`/api/agents/{AgentName}/run`)
- Registers each agent as a **durable entity** with the Durable Task runtime
- Manages conversation state persistence via the **Durable Task Scheduler** (DTS)
- Handles concurrent requests and multi-agent coordination

Each agent gets a persistent **thread** identified by a `thread_id`. The thread stores complete conversation history in durable storage. State survives process crashes, restarts, and scaling events.

#### When to Use Durable Agents

Choose durable agents when you need:

- **Full code control**: Deploy and manage your own compute while keeping serverless benefits
- **Complex orchestrations**: Coordinate multiple agents with deterministic, reliable workflows (can run for days or weeks)
- **Event-driven orchestration**: Integrate with Azure Functions triggers (HTTP, timers, queues, etc.)
- **Automatic conversation state**: No explicit state handling code needed

#### How It Differs from ASP.NET Core A2A Hosting

| Aspect | ASP.NET Core + A2A | Azure Functions (Durable) |
|---|---|---|
| **Hosting model** | Long-running web app process | Serverless, event-driven |
| **Scaling** | Manual or autoscale rules | Automatic, including scale-to-zero |
| **Protocol** | A2A protocol (standardized) | Custom HTTP endpoints (``/api/agents/{name}/run``) |
| **State management** | In-memory session store or custom | Automatic durable state via DTS |
| **Cost model** | Always-on compute | Pay-per-execution |
| **Agent registration** | `builder.AddAIAgent()` + `app.MapA2A()` | `ConfigureDurableAgents(options => options.AddAIAgent())` |
| **Workflow coordination** | `AddWorkflow()` + `AddAsAIAgent()` | Durable Functions orchestrations |
| **Cold starts** | None (always running) | Possible (mitigated by always-ready instances) |
| **Long-running tasks** | Limited by request timeout | Native support via orchestrations |

> **Source**: [Microsoft Agent Framework Hosting Docs](https://learn.microsoft.com/en-us/agent-framework/get-started/hosting), [Azure Functions (Durable) Integration](https://learn.microsoft.com/en-us/agent-framework/integrations/azure-functions)

---

### 2. Azure Functions + A2A Protocol

#### Critical Finding: A2A and Durable Agents Are Currently Separate Hosting Paths

The A2A protocol integration and the Durable Agents hosting are **currently separate integration libraries** in the Agent Framework:

- **A2A Protocol**: Uses `Microsoft.Agents.AI.Hosting.A2A.AspNetCore` — works with ASP.NET Core apps via `app.MapA2A()`
- **Durable Agents**: Uses `Microsoft.Agents.AI.Hosting.AzureFunctions` — works with Azure Functions via `ConfigureDurableAgents()`

There is **no direct "A2A over Azure Functions" integration** documented. The Azure Functions hosting exposes agents through its own HTTP endpoint pattern (`/api/agents/{name}/run`), not the A2A protocol.

#### A2A Protocol in ASP.NET Core (for Reference)

The A2A (Agent-to-Agent) protocol is a standardized protocol supporting:

- **Agent discovery** through agent cards (`GET /a2a/{agent}/v1/card`)
- **Message-based communication** between agents (`POST /a2a/{agent}/v1/message:stream`)
- **Long-running agentic processes** via tasks
- **Cross-platform interoperability** between different agent frameworks

##### ASP.NET Core A2A Setup

```csharp
// NuGet: Microsoft.Agents.AI.Hosting.A2A.AspNetCore
var pirateAgent = builder.AddAIAgent("pirate",
    instructions: "You are a pirate. Speak like a pirate.");

builder.Services.AddA2AServer();
var app = builder.Build();
app.MapA2AServer(); // or app.MapA2A(pirateAgent, "/a2a/pirate")
app.Run();
```

##### AgentCard Configuration

```csharp
app.MapA2A(agent, "/a2a/my-agent", agentCard: new()
{
    Name = "My Agent",
    Description = "A helpful agent.",
    Version = "1.0",
});
```

##### Multiple A2A Agents

```csharp
var mathAgent = builder.AddAIAgent("math", instructions: "You are a math expert.");
var scienceAgent = builder.AddAIAgent("science", instructions: "You are a science expert.");
app.MapA2A(mathAgent, "/a2a/math");
app.MapA2A(scienceAgent, "/a2a/science");
```

#### Durable Agents HTTP Endpoint Pattern

Durable Agents expose a different endpoint shape:

```text
POST /api/agents/{AgentName}/run              # New thread
POST /api/agents/{AgentName}/run?thread_id=X  # Continue thread
```

Response includes `x-ms-thread-id` header for thread continuity.

#### Bridging A2A and Durable Agents

While there's no built-in A2A adapter for Azure Functions, the samples show how to consume A2A agents from other agents:

- **Sample `A2AAgent_AsFunctionTools`**: Resolves an A2A agent card, wraps each skill as an `AITool`, and registers those tools with a local agent. This enables a Durable Agent to call out to external A2A agents.

```csharp
A2ACardResolver agentCardResolver = new(new Uri(a2aAgentHost));
AgentCard agentCard = await agentCardResolver.GetAgentCardAsync();
AIAgent a2aAgent = agentCard.AsAIAgent();
// Use a2aAgent skills as function tools for another agent
```

> **Source**: [A2A Integration Docs](https://learn.microsoft.com/en-us/agent-framework/integrations/a2a), [GitHub: A2AAgent_AsFunctionTools sample](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/A2A/A2AAgent_AsFunctionTools)

---

### 3. Durable Functions Integration with AI Agents

#### Key Packages

**C# (.NET)**:

```bash
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Microsoft.Agents.AI.Hosting.AzureFunctions --prerelease
```

Requires `Microsoft.Azure.Functions.Worker` version 2.2.0 or later.

**Python**:

```bash
pip install agent-framework-azurefunctions --pre
```

#### Core API: ConfigureDurableAgents

The `ConfigureDurableAgents()` extension method:

1. Calls `services.ConfigureDurableAgents()` to register agent services
2. Registers `IFunctionsAgentOptionsProvider` singleton
3. Adds `DurableAgentFunctionMetadataTransformer` to generate function endpoints
4. Installs `BuiltInFunctionExecutionMiddleware` for HTTP, MCP tool, and entity invocations

#### Orchestration Patterns

##### Sequential Orchestration

Agents execute in order, with output flowing to the next agent:

```csharp
[Function(nameof(SpamDetectionOrchestration))]
public static async Task<string> SpamDetectionOrchestration(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    Email email = context.GetInput<Email>();

    DurableAIAgent spamDetectionAgent = context.GetAgent("SpamDetectionAgent");
    AgentSession spamSession = await spamDetectionAgent.CreateSessionAsync();
    AgentResponse<DetectionResult> result = await spamDetectionAgent.RunAsync<DetectionResult>(
        message: $"Analyze this email for spam: {email.EmailContent}",
        session: spamSession);

    if (result.Result.IsSpam)
        return await context.CallActivityAsync<string>(nameof(HandleSpamEmail), result.Result.Reason);

    DurableAIAgent emailAssistant = context.GetAgent("EmailAssistantAgent");
    AgentSession emailSession = await emailAssistant.CreateSessionAsync();
    AgentResponse<EmailResponse> response = await emailAssistant.RunAsync<EmailResponse>(
        message: $"Draft response to: {email.EmailContent}",
        session: emailSession);

    return await context.CallActivityAsync<string>(nameof(SendEmail), response.Result.Response);
}
```

##### Parallel (Fan-Out/Fan-In) Orchestration

Multiple agents run concurrently:

```csharp
[Function(nameof(ResearchOrchestration))]
public static async Task<string> ResearchOrchestration(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    string topic = context.GetInput<string>();

    DurableAIAgent technicalAgent = context.GetAgent("TechnicalResearchAgent");
    DurableAIAgent marketAgent = context.GetAgent("MarketResearchAgent");
    DurableAIAgent competitorAgent = context.GetAgent("CompetitorResearchAgent");

    Task<AgentResponse<TextResponse>> t1 = technicalAgent.RunAsync<TextResponse>($"Research technical aspects of {topic}");
    Task<AgentResponse<TextResponse>> t2 = marketAgent.RunAsync<TextResponse>($"Research market trends for {topic}");
    Task<AgentResponse<TextResponse>> t3 = competitorAgent.RunAsync<TextResponse>($"Research competitors in {topic}");

    await Task.WhenAll(t1, t2, t3);

    string allResearch = string.Join("\n\n", t1.Result.Result.Text, t2.Result.Result.Text, t3.Result.Result.Text);

    DurableAIAgent summaryAgent = context.GetAgent("SummaryAgent");
    AgentResponse<TextResponse> summary = await summaryAgent.RunAsync<TextResponse>($"Summarize:\n{allResearch}");
    return summary.Result.Text;
}
```

##### Human-in-the-Loop Orchestration

Orchestrations can pause for human input without consuming compute:

```csharp
[Function(nameof(ContentApprovalWorkflow))]
public static async Task<string> ContentApprovalWorkflow(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    string topic = context.GetInput<string>();

    DurableAIAgent contentAgent = context.GetAgent("ContentGenerationAgent");
    AgentResponse<GeneratedContent> content = await contentAgent.RunAsync<GeneratedContent>($"Write about {topic}");

    await context.CallActivityAsync(nameof(NotifyReviewer), content.Result);

    HumanApprovalResponse approval;
    try
    {
        approval = await context.WaitForExternalEvent<HumanApprovalResponse>(
            eventName: "ApprovalDecision",
            timeout: TimeSpan.FromHours(24));
    }
    catch (OperationCanceledException)
    {
        return await context.CallActivityAsync<string>(nameof(EscalateForReview), content.Result);
    }

    if (approval.Approved)
        return await context.CallActivityAsync<string>(nameof(PublishContent), content.Result);

    return "Content rejected";
}
```

External event to approve:

```csharp
await client.RaiseEventAsync(instanceId, "ApprovalDecision", new HumanApprovalResponse
{
    Approved = true,
    Feedback = "Looks great!"
});
```

#### DurableAIAgent vs AIAgent

- `context.GetAgent("AgentName")` returns a `DurableAIAgent` — a special subclass of `AIAgent`
- `DurableAIAgent` wraps a registered agent and ensures calls are tracked/checkpointed by the durable orchestration framework
- `CreateSessionAsync()` creates a durable session for the agent
- `RunAsync<T>()` runs the agent with structured output and checkpointing

#### Durability and Checkpointing

- The orchestration framework automatically **replays** the orchestration history on resume
- Completed agent executions are **not repeated** — their results are replayed from checkpoints
- State is managed by the **Durable Task Scheduler** (recommended backend)
- During `WaitForExternalEvent`, all compute resources are **spun down** — zero cost

> **Source**: [Azure Functions (Durable) Integration](https://learn.microsoft.com/en-us/agent-framework/integrations/azure-functions), [GitHub Samples](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions)

---

### 4. Architecture Patterns

#### Hosting Decision Matrix

| Scenario | Recommended Host | Why |
|---|---|---|
| Multi-agent A2A interop | ASP.NET Core + A2A | Standardized protocol, agent discovery |
| Serverless single agent | Azure Functions (Durable) | Pay-per-use, auto-scale, stateful threads |
| Complex long-running workflows | Azure Functions (Durable) | Orchestrations run for days/weeks, checkpointed |
| Human-in-the-loop approval | Azure Functions (Durable) | Zero cost while waiting; scale to zero |
| Low-latency always-on API | App Service / Container Apps | No cold starts, full control |
| Web-based agent UI | ASP.NET Core + AG-UI | Real-time streaming to browser |
| Managed service (no infra) | Foundry Agent Service | Fully managed, no deployment |

#### Azure Functions Plan Comparison for Agent Hosting

| Aspect | Consumption | Flex Consumption | Premium | Dedicated |
|---|---|---|---|---|
| **Scale to zero** | Yes | Yes | No | No |
| **Max scale-out** | 200 instances | 1,000 instances | 100 instances | Per plan |
| **Cold start** | Seconds | Reduced (always-ready) | Minimal | None |
| **VNet support** | No | Yes | Yes | Yes |
| **Billing** | Execution only | Execution + always-ready | Always-on + execution | Always-on |
| **Instance sizes** | Fixed | 512MB, 2048MB, 4096MB | Flexible | Flexible |
| **Durable Functions** | Yes | Yes (DTS recommended) | Yes | Yes |
| **Per-function scaling** | No | Yes | No | No |

**Recommendation for agents**: **Flex Consumption** plan is the recommended serverless option — it provides:

- Scale to 1,000 instances or to zero
- Always-ready instances to mitigate cold starts
- Per-function scaling (Durable Functions get their own scale group)
- VNet integration for secure connectivity
- Pay only for compute used

#### Durable Task Scheduler Architecture

The Durable Task Scheduler (DTS) is the recommended durable backend:

- **Separate managed resource** from your Function App — independent scaling, fault isolation
- **gRPC connectivity** via TLS — push model for work items (no polling)
- **Internal state management** — no separate storage account needed for state
- **In-memory + persistent storage** — short-lived state in memory, recovery data persisted
- **Dashboard** — built-in UI for monitoring orchestrations, conversations, and agent sessions
- **Multiple task hubs** — partition state for different workloads, teams, environments
- **Local emulator** — Docker-based emulator for development (port 8080: gRPC, port 8082: dashboard)
- **Autopurge** — configurable retention policies for stale orchestration data

DTS Billing SKUs:

- **Consumption**: Up to 10 schedulers, 5 task hubs per region/subscription
- **Dedicated**: Up to 25 schedulers, 25 task hubs per region/subscription

#### Cold Start Implications for Agent Scenarios

Cold starts on Consumption plan can be several seconds. For agent scenarios:

- **Flex Consumption always-ready instances** eliminate cold starts for the "durable" function group
- The 30-second host initialization timeout can be a concern for heavy agent apps
- For latency-sensitive agent interactions, configure at least 1 always-ready instance

#### Cost Model for Agent Workflows

Human-in-the-loop example: a workflow waiting 24 hours for approval:

- **Pay for**: A few seconds of execution (generate content, send notification, process response)
- **Not charged for**: The 24 hours of waiting
- During wait: no compute resources consumed, orchestration state persisted by DTS

> **Source**: [Flex Consumption Plan](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan), [Durable Task Scheduler](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler)

---

### 5. Code Examples

#### Example 1: Single Durable Agent (C#)

**Program.cs**:

```csharp
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Hosting.AzureFunctions;
using Microsoft.Azure.Functions.Worker.Builder;
using Microsoft.Extensions.Hosting;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("AZURE_OPENAI_ENDPOINT is not set.");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT") ?? "gpt-4o-mini";

AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a helpful assistant.",
        name: "MyDurableAgent");

using IHost app = FunctionsApplication
    .CreateBuilder(args)
    .ConfigureFunctionsWebApplication()
    .ConfigureDurableAgents(options => options.AddAIAgent(agent))
    .Build();
app.Run();
```

**Invocation**:

```bash
# New thread
curl -i -X POST http://localhost:7071/api/agents/MyDurableAgent/run \
  -H "Content-Type: text/plain" \
  -d "What are three popular programming languages?"

# Continue thread
curl -X POST "http://localhost:7071/api/agents/MyDurableAgent/run?thread_id=@dafx-mydurableagent@263fa..." \
  -H "Content-Type: text/plain" \
  -d "Which one is best for beginners?"
```

#### Example 2: Multi-Agent Orchestration with Fan-Out/Fan-In (C#)

**Program.cs** — registering multiple agents:

```csharp
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Hosting.AzureFunctions;
using Microsoft.Azure.Functions.Worker.Builder;
using Microsoft.Extensions.Hosting;
using OpenAI.Chat;

string endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("AZURE_OPENAI_ENDPOINT is not set.");
string deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT") ?? "gpt-4o-mini";

AzureOpenAIClient client = new(new Uri(endpoint), new DefaultAzureCredential());
ChatClient chatClient = client.GetChatClient(deploymentName);

AIAgent mainAgent = chatClient.AsAIAgent(
    instructions: "You are a helpful assistant.",
    name: "MyDurableAgent");

AIAgent frenchAgent = chatClient.AsAIAgent(
    instructions: "Translate text to French. Return only the translation.",
    name: "FrenchTranslator");

AIAgent spanishAgent = chatClient.AsAIAgent(
    instructions: "Translate text to Spanish. Return only the translation.",
    name: "SpanishTranslator");

using IHost app = FunctionsApplication
    .CreateBuilder(args)
    .ConfigureFunctionsWebApplication()
    .ConfigureDurableAgents(options =>
    {
        options.AddAIAgent(mainAgent);
        options.AddAIAgent(frenchAgent);
        options.AddAIAgent(spanishAgent);
    })
    .Build();
app.Run();
```

**AgentOrchestration.cs** — the orchestration function:

```csharp
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.DurableTask;
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask;

namespace MyDurableAgent;

public static class AgentOrchestration
{
    public sealed record TextResponse(string Text);

    [Function("agent_orchestration_workflow")]
    public static async Task<Dictionary<string, string>> AgentOrchestrationWorkflow(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        var input = context.GetInput<string>();

        // Step 1: Main agent
        DurableAIAgent mainAgent = context.GetAgent("MyDurableAgent");
        AgentResponse<TextResponse> mainResponse = await mainAgent.RunAsync<TextResponse>(input);
        string agentResponse = mainResponse.Result.Text;

        // Step 2: Fan out to translators
        DurableAIAgent frenchAgent = context.GetAgent("FrenchTranslator");
        DurableAIAgent spanishAgent = context.GetAgent("SpanishTranslator");

        Task<AgentResponse<TextResponse>> frenchTask = frenchAgent.RunAsync<TextResponse>(agentResponse);
        Task<AgentResponse<TextResponse>> spanishTask = spanishAgent.RunAsync<TextResponse>(agentResponse);

        await Task.WhenAll(frenchTask, spanishTask);

        // Step 3: Fan in results
        return new Dictionary<string, string>
        {
            ["original"] = agentResponse,
            ["french"] = (await frenchTask).Result.Text,
            ["spanish"] = (await spanishTask).Result.Text
        };
    }
}
```

**Invocation**:

```bash
# Start orchestration
curl -X POST http://localhost:7071/runtime/webhooks/durabletask/orchestrators/agent_orchestration_workflow \
  -H "Content-Type: application/json" \
  -d '"\"What are three popular programming languages?\""'

# Poll for completion using statusQueryGetUri from response
curl http://localhost:7071/runtime/webhooks/durabletask/instances/{instanceId}
```

#### Example 3: Single Durable Agent (Python)

```python
from agent_framework.azure import AgentFunctionApp, AzureOpenAIChatClient
from azure.identity import AzureCliCredential

def _create_agent():
    return AzureOpenAIChatClient(
        credential=AzureCliCredential(),
    ).as_agent(
        name="Joker",
        instructions="You are good at telling jokes.",
    )

app = AgentFunctionApp(agents=[_create_agent()], enable_health_check=True, max_poll_retries=50)
```

**Invocation**:

```bash
curl -X POST http://localhost:7071/api/agents/Joker/run \
  -H "Content-Type: text/plain" \
  -d "Tell me a short joke about cloud computing."
```

#### Example 4: Agent as MCP Tool

```csharp
options
    .AddAIAgent(agent1)  // HTTP trigger by default
    .AddAIAgent(agent2, enableHttpTrigger: false, enableMcpToolTrigger: true)
    .AddAIAgent(agent3, agentOptions =>
    {
        agentOptions.McpToolTrigger.IsEnabled = true;
    });
// MCP endpoint: /runtime/webhooks/mcp
```

#### Project Structure (Typical .NET Durable Agents)

```text
MyDurableAgent/
├── Program.cs                     # Agent registration + host config
├── AgentOrchestration.cs          # Orchestration functions (optional)
├── FunctionTriggers.cs            # Custom HTTP triggers (optional)
├── local.settings.json            # Local dev settings
├── host.json                      # Functions host config
├── MyDurableAgent.csproj          # Project file
└── infra/                         # Azure infra-as-code (azd templates)
```

#### Available GitHub Samples

| Sample | Description |
|---|---|
| [01_SingleAgent](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions/01_SingleAgent) | Single conversational agent via HTTP |
| [02_AgentOrchestration_Chaining](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions/02_AgentOrchestration_Chaining) | Sequential chaining in orchestration |
| [03_AgentOrchestration_Concurrency](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions/03_AgentOrchestration_Concurrency) | Parallel agent execution |
| [04_AgentOrchestration_Conditionals](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions/04_AgentOrchestration_Conditionals) | Conditional branching (spam detection) |
| [05_AgentOrchestration_HITL](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions/05_AgentOrchestration_HITL) | Human-in-the-loop approval |
| [06_LongRunningTools](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions/06_LongRunningTools) | Agent tools that start orchestrations |
| [07_AgentAsMcpTool](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions/07_AgentAsMcpTool) | Expose agents as MCP tools |
| [08_ReliableStreaming](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions/08_ReliableStreaming) | SSE streaming with Redis, reconnection support |

> **Source**: [GitHub: microsoft/agent-framework (dotnet/samples)](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples)

---

### 6. Limitations and Considerations

#### Execution Time Limits

- **Consumption plan**: 5-minute default, configurable up to 10 minutes per invocation
- **Flex Consumption**: Same execution limits per invocation
- **Premium plan**: 30-minute default, unlimited configurable
- **Durable orchestrations are NOT subject to single-invocation limits** — they checkpoint between steps, so total workflow time is unbounded

#### Memory Constraints

- Flex Consumption: 512 MB, 2048 MB (default), or 4096 MB per instance
- DTS max payload sizes: 1 MB for orchestrator inputs/outputs, activity inputs/outputs, external events, entity state, and custom status
- Large payload support available for C# (.NET isolated) when using DTS

#### State Management

- Conversation history is automatically managed by the durable entity pattern
- No need for explicit session store configuration (unlike ASP.NET Core which needs `.WithInMemorySessionStore()`)
- Thread TTL configurable: `options.AddAIAgent(agent, timeToLive: TimeSpan.FromHours(1))`
- DTS emulator stores state in memory (not for production)
- Production DTS uses persistent internal storage, not a separate Azure Storage account

#### Streaming Support (SSE from Functions)

- **Sample 08 (ReliableStreaming)** demonstrates SSE streaming from Azure Functions using Redis Streams
- Pattern: fire-and-forget agent run + SSE endpoint that reads from Redis
- Supports client disconnect/reconnect with cursor-based resumption
- Inspired by OpenAI's background mode for the Responses API

#### Authentication Patterns

- Local dev: Azure CLI Credential or API key
- Production: `DefaultAzureCredential`, `ManagedIdentityCredential` recommended
- Azure Functions HTTP triggers support function key authentication (`?code=...`)
- DTS system key for orchestration management endpoints (`durabletask_extension` system key)

#### Host Initialization

- 30-second timeout for app initialization — heavy agent apps may hit this
- Cannot currently configure this timeout ([GitHub issue #10482](https://github.com/Azure/azure-functions-host/issues/10482))

#### Durable Task Scheduler Limitations

- Max payload 1 MB (use large payload support for bigger data)
- Instance ID max 100 characters, printable ASCII only
- IDs starting with `@` reserved for entities
- Extended sessions not yet available
- Consumption SKU: max 10 schedulers, 5 task hubs per region/subscription
- Dedicated SKU: max 25 schedulers, 25 task hubs per region/subscription

#### Flex Consumption Plan Considerations

- Linux only (no Windows)
- Only one app per Flex Consumption plan
- 30-second host initialization timeout
- Durable Functions limited to Azure Storage and DTS backends
- Deployment slots not supported (use rolling updates)
- No NFS file shares
- Scale range: 1 to 1,000 instances

#### A2A Protocol Not Directly Available in Azure Functions

- A2A uses `Microsoft.Agents.AI.Hosting.A2A.AspNetCore` which depends on ASP.NET Core middleware (`MapA2A()`)
- Azure Functions durable agents use their own HTTP endpoint pattern
- To interoperate: use A2A client to call external A2A agents from within durable agent tools, or host A2A endpoint in separate ASP.NET Core service

---

## References

| Source | URL |
|---|---|
| Agent Framework Hosting Docs | https://learn.microsoft.com/en-us/agent-framework/get-started/hosting |
| Azure Functions (Durable) Integration | https://learn.microsoft.com/en-us/agent-framework/integrations/azure-functions |
| A2A Integration Docs | https://learn.microsoft.com/en-us/agent-framework/integrations/a2a |
| Durable Functions Overview | https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview |
| Durable Task Scheduler | https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler |
| Flex Consumption Plan | https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan |
| GitHub: Agent Framework .NET Samples | https://github.com/microsoft/agent-framework/tree/main/dotnet/samples |
| GitHub: DurableAgents AzureFunctions Samples | https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions |
| GitHub: A2A Samples | https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/A2A |
| GitHub: Python Azure Functions Samples | https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/azure_functions |
| A2A Protocol Specification | https://a2a-protocol.org/latest/ |
| `Microsoft.Agents.AI.Hosting.AzureFunctions` NuGet | https://www.nuget.org/packages/Microsoft.Agents.AI.Hosting.AzureFunctions |
| `Microsoft.Agents.AI.DurableTask` NuGet | https://www.nuget.org/packages/Microsoft.Agents.AI.DurableTask |

---

## Discovered Topics

1. **Durable Workflows vs Durable Agents**: The framework also provides `ConfigureDurableWorkflows()` for hosting Agent Framework `Workflow` objects (sequential/concurrent graphs) in Azure Functions, separate from orchestration-based coordination.
2. **Agent as MCP Tool**: Durable agents can be exposed as MCP (Model Context Protocol) tools via `enableMcpToolTrigger: true`, generating a remote MCP endpoint at `/runtime/webhooks/mcp`.
3. **Reliable Streaming**: Fire-and-forget + Redis Streams enables SSE with reconnection support for long-running agent tasks.
4. **DurableAgentsOptions.AddAIAgentFactory()**: Factory method for creating agents with DI-resolved dependencies (tools, loggers).
5. **DurableAgentRunOptions.IsFireAndForget**: Supports fire-and-forget pattern returning 202 Accepted immediately.
6. **Durable Workflows in Azure Functions**: `ConfigureDurableWorkflows()` hosts graph-based workflows with auto-generated HTTP + orchestration endpoints.

---

## Next Research (Not Completed This Session)

- [ ] Investigate whether A2A protocol support for Azure Functions is on the roadmap (check GitHub issues)
- [ ] Research AG-UI protocol integration with Durable Agents
- [ ] Deep-dive into the Reliable Streaming sample (08) architecture and Redis Streams integration
- [ ] Explore Durable Workflows (`ConfigureDurableWorkflows`) as an alternative to orchestration-based multi-agent coordination
- [ ] Research Container Apps hosting for Agent Framework as a middle ground between Functions and App Service
- [ ] Investigate Foundry Agent Service for fully managed agent hosting comparison
- [ ] Research authentication and authorization patterns for multi-tenant durable agent scenarios
- [ ] Explore the DTS dashboard agent pages (preview feature) in more detail

---

## Clarifying Questions

1. **A2A + Azure Functions**: Is the goal to expose durable agents via A2A protocol, or to have durable agents consume external A2A agents? The current framework supports the latter (via A2A client) but not the former directly.
2. **Python vs C# Priority**: The Python Azure Functions SDK (`agent-framework-azurefunctions`) has a different API surface (`AgentFunctionApp`) — should we research Python-specific patterns in more depth?
3. **Production vs Demo**: Are the Durable Agent samples intended for production architecture reference, or primarily for demonstration understanding?
