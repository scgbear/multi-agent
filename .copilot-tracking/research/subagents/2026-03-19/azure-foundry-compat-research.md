# Azure AI Foundry Compatibility with Multi-Agent Libraries

## Research Topics and Questions

1. Azure AI Foundry overview and agent-related capabilities
2. Azure AI Agent Service (now "Microsoft Foundry Agent Service")
3. Semantic Kernel + Azure AI Foundry integration
4. Microsoft Agent Framework (AutoGen successor) + Azure AI Foundry integration
5. LangChain/LangGraph + Azure AI Foundry integration
6. Compatibility matrix and concerns summary

---

## 1. Azure AI Foundry Overview (Agent-Relevant)

### What Is Azure AI Foundry?

Azure AI Foundry (now branded **Microsoft Foundry**) is a unified platform for building, deploying, and managing AI applications. It provides:

- **Model catalog**: Access to OpenAI models (GPT-4o, GPT-4.1, GPT-5, o3, o4-mini), Meta Llama (Llama-4-Scout, Llama-4-Maverick, Llama-3.3-70B), Mistral AI (Mistral-Large-3, Mistral-Small, Codestral), DeepSeek (R1, V3), Grok (grok-4, grok-3), and more.
- **Foundry portal** (ai.azure.com): Visual environment for project management, model deployment, agent playground, evaluation, and observability.
- **Foundry SDK**: Python (`azure-ai-projects`, `azure-ai-agents`) and .NET SDKs for programmatic access.
- **Project-based organization**: Each project has an endpoint like `https://<resource>.services.ai.azure.com/api/projects/<project>`.

### What Is Microsoft Foundry Agent Service?

Foundry Agent Service is a **fully managed platform** for building, deploying, and scaling AI agents. It is now GA (Generally Available).

**Key components:**

| Component | Description |
|---|---|
| **Agent Runtime** | Hosts and scales prompt agents and hosted agents. Manages conversations, tool calls, and lifecycle. |
| **Tools** | Built-in: web search, file search, memory, code interpreter, MCP servers, Azure Functions, Logic Apps, OpenAPI tools, computer use, browser automation, image generation, Agent2Agent protocol. |
| **Models** | Works with many models from the Foundry model catalog. Swap models without changing agent code. |
| **Observability** | End-to-end tracing, metrics, Application Insights integration. |
| **Identity & Security** | Microsoft Entra identity, RBAC, content filters, virtual network isolation. |
| **Publishing** | Version agents, create stable endpoints, share through Teams, M365 Copilot, Entra Agent Registry. |

**Agent types:**

| Type | Code Required | Hosting | Orchestration | Best For |
|---|---|---|---|---|
| **Prompt agents** | No | Fully managed | Single agent | Prototyping, simple tasks |
| **Workflow agents** (preview) | No (YAML optional) | Fully managed | Multi-agent, branching | Multi-step automation |
| **Hosted agents** (preview) | Yes | Container-based, managed | Custom logic | Full control, custom frameworks |

### Connected Agents

Connected Agents let a primary agent delegate tasks to specialized sub-agents using `ConnectedAgentTool`. Key characteristics:

- No custom orchestrator needed; the main agent uses natural language routing.
- Configured via SDK (`ConnectedAgentToolDefinition`) or the Foundry portal.
- **Maximum depth of 2**: A parent can have multiple sub-agents, but sub-agents cannot have their own sub-agents.
- Connected agents **cannot call local functions** (use OpenAPI or Azure Functions tools instead).
- Citations from connected agents are **not guaranteed** to pass through.
- Note: Connected Agents (classic) are being replaced by **workflow agents** (API `2025-11-15-preview`).

**Python example:**

```python
from azure.ai.projects import AIProjectClient
from azure.ai.agents.models import ConnectedAgentTool

project_client = AIProjectClient(
    endpoint=os.environ["PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

stock_agent = project_client.agents.create_agent(
    model=os.environ["MODEL_DEPLOYMENT_NAME"],
    name="stock_price_bot",
    instructions="Get the stock price of a company.",
)

connected_agent = ConnectedAgentTool(
    id=stock_agent.id, name=stock_agent.name,
    description="Gets the stock price of a company"
)

main_agent = project_client.agents.create_agent(
    model=os.environ["MODEL_DEPLOYMENT_NAME"],
    name="my-agent",
    instructions="You are a helpful agent.",
    tools=connected_agent.definitions,
)
```

### Hosted Agents

Hosted agents allow deploying containerized agents built with **Microsoft Agent Framework**, **LangGraph**, or custom code onto Foundry-managed infrastructure.

- **Framework support**: Agent Framework (Python, C#), LangGraph (Python), Custom code (Python, C#).
- **Hosting adapter packages**: `azure-ai-agentserver-core`, `azure-ai-agentserver-agentframework`, `azure-ai-agentserver-langgraph` (Python); `Azure.AI.AgentServer.Core`, `Azure.AI.AgentServer.AgentFramework` (.NET).
- **API compatibility**: Hosted agents expose the Foundry Responses API.
- **Currently in preview** with limitations: no private networking support for hosted agents.

### Model Deployment for Agents

- Agent Service supports Azure OpenAI models (GPT-4o, GPT-4.1, GPT-5, o-series, etc.) and **models from the Foundry model catalog** (Llama, Mistral, DeepSeek, Grok, etc.).
- All models supporting **OpenAI-compatible APIs** can be used.
- Rate limiting is at the model deployment level, not the Agent Service level.
- Maximum 128 tools per agent; 10,000 files per agent/thread; 100,000 messages per thread.

---

## 2. Semantic Kernel + Azure AI Foundry

### Integration Approach: `AzureAIAgent`

Semantic Kernel provides a dedicated `AzureAIAgent` class that wraps Azure AI Foundry's Agent Service. This is the primary way SK connects to Foundry.

**Package:**
- Python: `pip install semantic-kernel` (as of v1.32.1, Azure AI dependencies are included)
- C#: `dotnet add package Microsoft.SemanticKernel.Agents.AzureAI --prerelease`

**Key classes:**
- `AzureAIAgent`: The agent wrapper that maps to a Foundry Agent Service agent
- `AzureAIAgentThread`: Manages conversation state (abstraction over Foundry's `PersistentAgentThread`)
- `AzureAIAgentSettings`: Configuration (reads from `AZURE_AI_AGENT_ENDPOINT` env var)
- `PersistentAgentsClient`: The underlying Azure SDK client for Foundry Agent Service

**Python creation pattern:**

```python
from azure.identity.aio import DefaultAzureCredential
from semantic_kernel.agents import AzureAIAgent, AzureAIAgentSettings

async with (
    DefaultAzureCredential() as creds,
    AzureAIAgent.create_client(credential=creds) as client,
):
    agent_definition = await client.agents.create_agent(
        model=AzureAIAgentSettings().model_deployment_name,
        name="<name>",
        instructions="<instructions>",
    )
    agent = AzureAIAgent(client=client, definition=agent_definition)
```

**C# creation pattern:**

```csharp
PersistentAgentsClient client = AzureAIAgent.CreateAgentsClient("<endpoint>", new AzureCliCredential());
PersistentAgent definition = await client.Administration.CreateAgentAsync(
    "<model>", name: "<name>", instructions: "<instructions>");
AzureAIAgent agent = new(definition, client);
```

### `AzureAIAgent` Capabilities

- Automates tool calling (no manual parsing/invocation)
- Manages conversation history via threads
- Supports built-in tools: file search, code interpreter, Bing Search, Azure AI Search, Azure Functions, OpenAPI
- Supports SK plugins for custom function calling
- Supports streaming responses
- Available in Python and C# (Java: **not yet available**)

### Multi-Agent Orchestration in SK

Semantic Kernel provides built-in orchestration patterns that work with `AzureAIAgent`:

| Pattern | Description |
|---|---|
| **Concurrent** | Broadcasts task to all agents, collects results independently |
| **Sequential** | Passes result from one agent to the next in order |
| **Handoff** | Dynamically passes control between agents |
| **Group Chat** | All agents participate in a conversation, coordinated by a manager |
| **Magentic** | MagenticOne-inspired orchestration |

SK also has `ChatCompletionAgent` which connects to Azure OpenAI endpoints directly (without Agent Service) and can be combined with `AzureAIAgent` in the same orchestration.

### Compatibility Concerns

1. **C# is experimental**: The `AzureAIAgent` package is marked `--prerelease` and in "experimental stage."
2. **Python GA migration**: SK Python 1.31.0+ requires the new Foundry GA endpoint format (`AZURE_AI_AGENT_ENDPOINT`). Projects created before May 19, 2025 use the old connection string format and require SK < 1.31.0.
3. **Java not available**: `AzureAIAgent` is not yet implemented for Java.
4. **Thread type limitation**: `AzureAIAgent` only supports `AzureAIAgentThread` (not interchangeable with other thread types).
5. **Model dependency**: The agent creation happens server-side in Foundry; model must be deployed in the Foundry project first.
6. **Migration path**: SK is being superseded by **Microsoft Agent Framework** (same team). SK users should consider migrating to Agent Framework for new projects. Migration guides are available.

### SK `ChatCompletionAgent` with Azure OpenAI (Direct Model Use)

For scenarios that don't need Agent Service features (file search, code interpreter, etc.), SK's `ChatCompletionAgent` can connect directly to Azure OpenAI deployments within Foundry:

```python
from semantic_kernel.agents import ChatCompletionAgent
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion

agent = ChatCompletionAgent(
    service=AzureChatCompletion(),
    name="<name>",
    instructions="<instructions>",
)
```

This uses standard `AZURE_OPENAI_ENDPOINT` and `AZURE_OPENAI_API_KEY` environment variables and works with any Azure OpenAI deployment.

---

## 3. Microsoft Agent Framework (AutoGen Successor) + Azure AI Foundry

### Background

**Microsoft Agent Framework** is a new open-source multi-language SDK (Python, C#) that is the **direct successor to both AutoGen and Semantic Kernel's agent capabilities**. Created by the same teams, it combines:

- AutoGen's simple abstractions for single- and multi-agent patterns
- Semantic Kernel's enterprise features (session management, type safety, middleware, telemetry)
- New graph-based **Workflows** for explicit multi-agent orchestration

### Model Clients for Azure AI Foundry

| Feature | AutoGen (Legacy) | Agent Framework |
|---|---|---|
| OpenAI | `OpenAIChatCompletionClient` | `OpenAIChatClient` |
| OpenAI Responses | Not available | `OpenAIResponsesClient` |
| Azure OpenAI | `AzureOpenAIChatCompletionClient` | `AzureOpenAIChatClient` |
| Azure OpenAI Responses | Not available | `AzureOpenAIResponsesClient` |
| Azure AI (Foundry Agent Service) | `AzureAIChatCompletionClient` | `AzureAIAgentClient` |
| Anthropic | `AnthropicChatCompletionClient` | Planned |
| Ollama | `OllamaChatCompletionClient` | Planned |

**Key integration code:**

```python
from agent_framework.azure import AzureOpenAIChatClient, AzureAIAgentClient

# Azure OpenAI (direct model access)
client = AzureOpenAIChatClient(model_id="gpt-5")

# Azure AI Foundry Agent Service
client = AzureAIAgentClient(...)
```

Agent Framework also supports creating agents directly from hosted agent clients:

```csharp
AIAgent azureFoundryAgent = await persistentAgentsClient.CreateAIAgentAsync(
    instructions: "You are helpful.");
// Or retrieve an existing agent:
AIAgent azureFoundryAgent = await persistentAgentsClient.GetAIAgentAsync(agentId);
```

### Hosted Agent Deployment

Agent Framework agents can be deployed as **hosted agents** on Foundry:

```python
from agent_framework import ai_function, ChatAgent
from agent_framework.azure import AzureAIAgentClient
from azure.ai.agentserver.agentframework import from_agent_framework

# Agent Framework code wrapped with hosting adapter
# Deployed as container to Foundry Agent Service
```

Adapter package: `azure-ai-agentserver-agentframework`

### Multi-Agent Orchestration

Agent Framework provides graph-based `Workflow` with:

- Sequential, concurrent, group chat, handoff, and magentic orchestration
- Nesting patterns and agent-as-a-tool patterns
- Human-in-the-loop with request-response API
- Checkpointing and resuming workflows
- Middleware for security, observability, rate limiting

### Compatibility Concerns

1. **Public preview**: Agent Framework is in public preview (not yet GA).
2. **Migration required**: Migrating from AutoGen or Semantic Kernel requires code changes (different class names, different orchestration model). Migration guides are available.
3. **Limited provider support**: Anthropic and Ollama clients are planned but not yet available.
4. **No distributed runtime yet**: Currently single-process composition only; distributed execution is planned.
5. **Seamless Foundry integration**: Agent Framework is a first-class citizen on Azure AI Foundry, with dedicated hosting adapter support.
6. **Wireline compatible**: Agent Service works with frameworks that are "wireline compatible" with the Responses API.

---

## 4. LangChain/LangGraph + Azure AI Foundry

### Official Integration: `langchain-azure-ai`

Microsoft provides the **`langchain-azure-ai`** package as the official integration point between LangChain/LangGraph and Azure AI Foundry. This is **co-developed with LangChain** (hosted at `langchain-ai/langchain-azure` on GitHub).

**Installation:**

```bash
pip install -U langchain-azure-ai azure-identity
# Optional extras:
pip install -U "langchain-azure-ai[tools]"          # Document Intelligence, Logic Apps
pip install -U "langchain-azure-ai[opentelemetry]"   # OpenTelemetry tracing
```

### Integration Building Blocks

| Capability | Namespace | Description |
|---|---|---|
| **Foundry Agent Service** | `langchain_azure_ai.agents` | Build managed agent nodes for LangGraph/LangChain |
| **Chat models** | `langchain_azure_ai.chat_models` | Call Azure OpenAI and model catalog chat models |
| **Embeddings** | `langchain_azure_ai.embeddings` | Call embedding models from the catalog |
| **Vector stores** | `langchain_azure_ai.vectorstores` | Azure AI Search and Cosmos DB |
| **Retrievers** | `langchain_azure_ai.retrievers` | Retrieval over Azure-backed indexes |
| **Chat history** | `langchain_azure_ai.chat_message_histories` | Persist/replay chat history, Foundry Memory |
| **Tools** | `langchain_azure_ai.tools` | Document Intelligence, Vision, health text analytics, Logic Apps |
| **Callbacks/Tracing** | `langchain_azure_ai.callbacks` | OpenTelemetry traces to Application Insights |
| **Query constructors** | `langchain_azure_ai.query_constructors` | Backend-specific query filters |

### Chat Model Access

**Key class:** `AzureAIOpenAIApiChatModel`

```python
from langchain.chat_models import init_chat_model

# Simplest approach - uses AZURE_AI_PROJECT_ENDPOINT env var
model = init_chat_model("azure_ai:gpt-4.1")

# Or configure directly
from langchain_azure_ai.chat_models import AzureAIOpenAIApiChatModel
model = AzureAIOpenAIApiChatModel(
    project_endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
    model="Mistral-Large-3",  # Non-OpenAI model from catalog!
)
```

**Connection patterns:**
1. **Project endpoint with Entra ID** (recommended): `AZURE_AI_PROJECT_ENDPOINT`
2. **Direct endpoint with API key**: `OPENAI_BASE_URL` + `OPENAI_API_KEY`

**Non-OpenAI model support:** All Foundry models with OpenAI-compatible APIs work, including Mistral-Large-3, DeepSeek-R1, Llama models. The `AzureAIOpenAIApiChatModel` uses the OpenAI Responses API by default, configurable with `use_responses_api=False`.

**Runtime model switching:**

```python
configurable_model = init_chat_model(model_provider="azure_ai", temperature=0)
configurable_model.invoke("hello", config={"configurable": {"model": "gpt-5-nano"}})
configurable_model.invoke("hello", config={"configurable": {"model": "Mistral-Large-3"}})
```

### Foundry Agent Service Integration (LangGraph)

The `AgentServiceFactory` class creates LangGraph-compatible nodes that run through Foundry Agent Service:

```python
from langchain_azure_ai.agents import AgentServiceFactory

factory = AgentServiceFactory(
    project_endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

# Create a prompt agent (managed by Foundry)
agent = factory.create_prompt_agent(
    name="my-agent",
    model=os.environ["MODEL_DEPLOYMENT_NAME"],
    instructions="You are a helpful assistant.",
    tools=[add, multiply],  # Local Python tools
)

# The agent is a LangGraph CompiledStateGraph
response = agent.invoke({"messages": [HumanMessage(content="Add 3 and 4")]})
```

**Tool support:**
- **Local tools**: Python functions that run where your code runs
- **Built-in tools**: Run server-side in Foundry (file search, code interpreter, Bing search)
- **Ecosystem tools**: LangChain tools like `AzureAIDocumentIntelligenceTool`

### Hosted Agent Deployment (LangGraph)

LangGraph agents can be deployed as hosted agents on Foundry:

```bash
pip install azure-ai-agentserver-langgraph
```

Framework support: LangGraph (Python only, no C#).

### Legacy: `AzureChatOpenAI` (langchain-openai)

The older `langchain-openai` package's `AzureChatOpenAI` class still works with Azure OpenAI endpoints but is **not the recommended path** for Foundry integration:

```python
from langchain_openai import AzureChatOpenAI
model = AzureChatOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    azure_deployment="gpt-4.1",
    api_version="2024-08-01-preview",
)
```

This class connects to Azure OpenAI endpoints directly but does **not** support Foundry project endpoints, non-OpenAI model catalog models, or Foundry Agent Service features.

### Compatibility Concerns

1. **SDK version split**: `langchain-azure-ai` v2 (new Foundry SDK) vs. `langchain-azure-ai[v1]` (legacy Azure AI Inference SDK for Foundry classic/Hubs). Must choose correctly.
2. **OpenAI-compatible API requirement**: Only models with OpenAI-compatible APIs work through `AzureAIOpenAIApiChatModel`. Models without this compatibility require direct Azure AI Inference SDK usage.
3. **LangGraph hosted agents are Python-only**: No C# support for LangGraph on Foundry.
4. **`AzureChatOpenAI` limitations**: The older `langchain-openai` connector doesn't support Foundry project endpoints or non-OpenAI models. Teams should migrate to `langchain-azure-ai`.
5. **Foundry Memory integration**: Available through `langchain_azure_ai.chat_message_histories` but focuses on long-term memory; short-term state remains in LangGraph runtime.
6. **Tracing**: Full OpenTelemetry integration via `AzureAIOpenTelemetryTracer` callback, traces flow to Application Insights.

---

## 5. Compatibility Concerns Summary

### Compatibility Matrix

| Feature | Semantic Kernel | Agent Framework | LangChain/LangGraph |
|---|---|---|---|
| **Azure OpenAI models** | Full support (`ChatCompletionAgent`, `AzureAIAgent`) | Full support (`AzureOpenAIChatClient`) | Full support (`AzureAIOpenAIApiChatModel`) |
| **Non-OpenAI catalog models** (Mistral, Llama, DeepSeek) | Via `AzureAIAgent` (server-side model selection) | Via `AzureAIAgentClient` | Full support (`init_chat_model("azure_ai:Mistral-Large-3")`) |
| **Foundry Agent Service** | `AzureAIAgent` class | `AzureAIAgentClient`, hosted agent deployment | `AgentServiceFactory`, hosted agent deployment |
| **Multi-agent orchestration** | Group Chat, Sequential, Concurrent, Handoff, Magentic | Workflow (graph-based), Group Chat, Handoff, Magentic | LangGraph (graph-based), Agent Service nodes |
| **Hosted agent deployment** | Not directly (use Agent Framework) | Yes (Python, C#) | Yes (Python only) |
| **Built-in Foundry tools** (file search, code interpreter) | Via `AzureAIAgent` | Via `AzureAIAgentClient` | Via `AgentServiceFactory` (server-side tools) |
| **Custom tools/plugins** | SK Plugins | `@tool` decorator, hosted tools | LangChain tools, local functions |
| **Observability/Tracing** | OpenTelemetry | OpenTelemetry | `AzureAIOpenTelemetryTracer` to App Insights |
| **Python support** | Yes (GA) | Yes (preview) | Yes (GA) |
| **C# support** | Yes (experimental/prerelease) | Yes (preview) | No |
| **Java support** | Not yet | Not yet | Not applicable |
| **Connected Agents** (multi-agent via Foundry) | Via `ConnectedAgentTool` in `AzureAIAgent` | Via `AzureAIAgentClient` | Via `AgentServiceFactory` |
| **Foundry Memory** | Via `AzureAIAgent` memory tool | Planned | `langchain_azure_ai.chat_message_histories` |

### Known Issues and Limitations

| Issue | Impact | Workaround |
|---|---|---|
| Semantic Kernel C# `AzureAIAgent` is prerelease | API may change | Pin package versions; watch for breaking changes |
| SK Java `AzureAIAgent` not available | Can't use SK Java with Foundry Agent Service | Use `ChatCompletionAgent` with Azure OpenAI directly |
| Agent Framework in public preview | Not yet production-supported | Use SK or direct Foundry SDK for production workloads |
| Connected Agents max depth of 2 | Cannot create deeply nested agent hierarchies | Flatten agent structures; use orchestration frameworks for deeper nesting |
| Connected Agents can't call local functions | Must use OpenAPI or Azure Functions tools | Wrap local functions as Azure Functions or OpenAPI endpoints |
| `langchain-azure-ai` v1 vs v2 confusion | Wrong version breaks Foundry classic vs. new Foundry | Check project type; use `[v1]` for classic, default for new Foundry |
| LangGraph hosted agents Python-only | C# LangGraph not supported | Use Agent Framework for C# hosted agents |
| Non-OpenAI models may lack tool calling | Some catalog models don't support function calling | Check model capabilities in catalog before building agents |
| `AzureChatOpenAI` doesn't support project endpoints | Can't use Foundry project auth with old connector | Migrate to `langchain-azure-ai` |
| Foundry classic deprecation (retiring March 2027) | Classic agents and classic portal being retired | Migrate to new Foundry and GA Agent Service |

### Friction Assessment for Existing Azure AI Foundry Investment

**Low friction:**
- All three frameworks have first-class Azure AI Foundry integration paths.
- Foundry's OpenAI-compatible API surface means most frameworks "just work" with deployed models.
- Microsoft actively maintains integrations for all three (SK, Agent Framework, langchain-azure-ai).

**Potential friction points:**
- **Framework diversity**: Choosing between SK, Agent Framework, and LangChain adds decision complexity. Microsoft's own stack is consolidating toward Agent Framework.
- **Preview/experimental status**: Agent Framework and SK's `AzureAIAgent` are still maturing. Production workloads should plan for API changes.
- **Model-specific quirks**: Not all catalog models support the same features (tool calling, structured output). Agent scenarios may work with GPT-4o/4.1 but fail with some OSS models.
- **Foundry classic to new migration**: Organizations on the classic portal need to migrate by March 2027.

---

## References

| Source | URL |
|---|---|
| What is Microsoft Foundry Agent Service? | https://learn.microsoft.com/azure/foundry/agents/overview |
| Transparency Note for Azure Agent Service | https://learn.microsoft.com/azure/foundry/responsible-ai/agents/transparency-note |
| Connected Agents (classic) | https://learn.microsoft.com/azure/foundry-classic/agents/how-to/connected-agents |
| Hosted Agents | https://learn.microsoft.com/azure/foundry/agents/concepts/hosted-agents |
| Deploy a Hosted Agent | https://learn.microsoft.com/azure/foundry/agents/how-to/deploy-hosted-agent |
| Foundry Agent Service Limits and Quotas | https://learn.microsoft.com/azure/foundry/agents/concepts/limits-quotas-regions |
| Semantic Kernel `AzureAIAgent` | https://learn.microsoft.com/semantic-kernel/frameworks/agent/agent-types/azure-ai-agent |
| SK `AzureAIAgent` GA Migration Guide | https://learn.microsoft.com/semantic-kernel/support/migration/azureagent-foundry-ga-migration-guide |
| SK Agent Orchestration | https://learn.microsoft.com/semantic-kernel/frameworks/agent/agent-orchestration/ |
| SK to Agent Framework Migration | https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/ |
| Microsoft Agent Framework Overview | https://learn.microsoft.com/agent-framework/overview/ |
| AutoGen to Agent Framework Migration | https://learn.microsoft.com/agent-framework/migration-guide/from-autogen/ |
| Get Started with LangChain/LangGraph + Foundry | https://learn.microsoft.com/azure/foundry/how-to/develop/langchain |
| Use LangChain with Foundry Models | https://learn.microsoft.com/azure/foundry/how-to/develop/langchain-models |
| Use Foundry Agent Service with LangGraph | https://learn.microsoft.com/azure/foundry/how-to/develop/langchain-agents |
| Foundry Memory with LangChain | https://learn.microsoft.com/azure/foundry/how-to/develop/langchain-memory |
| LangChain/LangGraph Tracing | https://learn.microsoft.com/azure/foundry/how-to/develop/langchain-traces |
| Foundry Model Catalog | https://learn.microsoft.com/azure/machine-learning/concept-models-featured |
| `langchain-azure-ai` GitHub | https://github.com/langchain-ai/langchain-azure/tree/main/libs/azure-ai |
| Agent Framework GitHub | https://github.com/microsoft/agent-framework |

---

## Discovered Topics

1. **Microsoft Foundry Workflows** (preview): Visual/YAML-based multi-agent orchestration in Foundry portal, as an alternative to code-based orchestration.
2. **Agent2Agent Protocol**: Foundry supports connecting agents via A2A protocol for cross-platform agent interop.
3. **MCP Server Integration**: Both Agent Service and Agent Framework support Model Context Protocol for tool interoperability.
4. **Foundry classic deprecation timeline**: March 2027 retirement for classic agents.
5. **SK being superseded by Agent Framework**: Same teams, Agent Framework is described as "the next generation of both Semantic Kernel and AutoGen."
6. **`langchain-azure-ai` v1 vs v2**: Critical distinction for Foundry classic vs new Foundry.

---

## Next Research (Not Completed)

- [ ] Deep-dive on Foundry Workflow Agents (YAML-based multi-agent) and how they compare to code-based frameworks
- [ ] Agent2Agent (A2A) protocol support across all frameworks
- [ ] MCP server integration patterns for each framework
- [ ] Performance benchmarks for Foundry Agent Service vs. direct model calls
- [ ] Specific model catalog models that do/don't support tool calling (detailed matrix)
- [ ] Foundry pricing comparison across agent types
- [ ] Agent Framework production readiness timeline (when will it exit preview?)

---

## Clarifying Questions

1. **Which framework(s) is the organization currently using or evaluating?** This would help focus the compatibility analysis.
2. **Is the organization on Foundry classic or new Foundry?** The integration path differs significantly.
3. **What programming language is primary (Python, C#, or both)?** Java support is limited across all frameworks.
4. **Are there specific non-OpenAI models from the catalog being used?** Some models have tool-calling limitations that affect agent scenarios.
5. **Is hosted agent deployment (containers on Foundry) a requirement, or is self-hosted acceptable?** This changes the framework recommendations significantly.
