# Magentic Orchestration — Hands-On Lab

## Lab Overview

Build an **SRE incident response system** using the magentic orchestration pattern. A Manager Agent dynamically builds and maintains a Task & Progress Ledger — a living plan — and coordinates specialized agents to diagnose, mitigate, and resolve a production incident. After each step, the Manager evaluates progress and replans as findings emerge:

```text
                                ┌──→ [Diagnostics Agent] (Model + Knowledge)
Alert → [Manager Agent] ←──→   ├──→ [Infrastructure Agent] (Model + Tools)
         ↕                     ├──→ [Rollback Agent] (Model + Tools)
   Task & Progress Ledger      └──→ [Communication Agent] (Model + Knowledge)
         ↻ Evaluate Goal Loop
```

You will choose one of two implementation tracks:

| Track | Technology | Language |
|-------|-----------|----------|
| **Track A** | Azure AI Foundry Workflow Agents | No code (portal + YAML) |
| **Track B** | Microsoft Agent Framework | C# or Python |

Both tracks build the same Manager-led incident response system with a dynamic ledger. Pick the track that matches your team's skills and goals.

## Prerequisites

* An Azure subscription with access to Azure AI Foundry
* A deployed chat model (GPT-4o or equivalent) in your Foundry project — the Manager Agent benefits from a highly capable model
* **Track A only:** Access to the Azure AI Foundry portal
* **Track B only:** .NET 9+ SDK or Python 3.11+, and an IDE of your choice

## Scenario

Your platform engineering team responds to production incidents through ad-hoc Slack threads and manual runbooks. The process is slow, poorly documented, and the team often discovers mid-incident that their initial diagnosis was wrong — requiring a completely different mitigation strategy. You are building an adaptive incident response system where a Manager Agent creates an initial investigation plan, dispatches specialists, and replans as findings emerge:

| Agent | Type | Capabilities |
|-------|------|-------------|
| **Manager Agent** | Orchestrator | Creates and maintains the Task & Progress Ledger, assigns work, evaluates progress, replans when findings change the picture |
| **Diagnostics Agent** | Model + Knowledge | Analyzes logs, metrics, traces, and error patterns to identify root cause. Reads but does not modify systems |
| **Infrastructure Agent** | Model + Tools | Scales resources, adjusts load balancers, modifies network rules. Makes changes in external systems |
| **Rollback Agent** | Model + Tools | Reverts deployments, restores configurations, rolls back database migrations. Makes changes in external systems |
| **Communication Agent** | Model + Knowledge | Drafts stakeholder updates, status page entries, and post-incident summaries. Reads context but does not modify systems |

The key insight: the solution path is unknown at the start. The alert says "API latency spike" but the root cause might be a bad deployment, a database issue, a network problem, or a resource exhaustion scenario. The Manager's initial plan will almost certainly need revision — this adaptive replanning is what makes Magentic the right pattern.

Notice the two agent types from the slides: Diagnostics and Communication are **Model + Knowledge** agents that analyze and report. Infrastructure and Rollback are **Model + Tools** agents that take real actions in external systems. This distinction matters for security boundaries — agents with tools carry higher risk and need stricter guardrails.

---

## Track A — Azure AI Foundry Workflow Agents

This track uses the Foundry portal's visual workflow designer to build the magentic system without writing code.

### A1: Create the Four Specialist Agents

In the Azure AI Foundry portal, create four **prompt agents** — one for each specialist role. The Manager will be configured as part of the workflow orchestration.

**Diagnostics Agent**

* Name: `diagnostics-agent`
* Instructions:

  ```text
  You are an SRE diagnostics specialist (Model + Knowledge). You analyze
  production incidents by examining logs, metrics, traces, and error patterns.
  You DO NOT make changes to systems — you observe and report.

  Given an incident description and any prior findings from other agents:
  1. Identify the most likely root cause based on available evidence.
  2. Rate your confidence (high/medium/low) and explain what evidence supports
     or contradicts your hypothesis.
  3. List specific diagnostic steps that would increase confidence.
  4. If prior mitigation attempts have been reported, assess whether they
     addressed the root cause or just the symptoms.

  Format your output as a structured diagnostic report with: Hypothesis,
  Confidence, Supporting Evidence, Contradicting Evidence, and Recommended
  Next Steps.
  ```

**Infrastructure Agent**

* Name: `infrastructure-agent`
* Instructions:

  ```text
  You are an SRE infrastructure specialist (Model + Tools). You can make changes
  to production systems including: scaling compute resources, adjusting load
  balancer configurations, modifying network rules, and updating resource limits.

  Given a mitigation request from the Manager with diagnostic context:
  1. Propose the specific infrastructure change needed.
  2. Assess the risk of the change (blast radius, reversibility).
  3. Describe the exact steps you would take (simulated for this exercise).
  4. Report the expected outcome and how to verify the change worked.

  IMPORTANT: Always state what change you are making before executing it.
  Never make changes without explaining the rationale and risk first.

  Format your output as an action report with: Proposed Change, Risk Assessment,
  Execution Steps, Expected Outcome, and Verification Method.
  ```

**Rollback Agent**

* Name: `rollback-agent`
* Instructions:

  ```text
  You are an SRE rollback specialist (Model + Tools). You can revert production
  changes including: rolling back deployments, restoring previous configurations,
  reverting database migrations, and restoring from backups.

  Given a rollback request from the Manager with diagnostic context:
  1. Identify what needs to be rolled back and to which version/state.
  2. Assess rollback risk (data loss potential, dependency impacts, rollback
     duration).
  3. Describe the exact rollback steps (simulated for this exercise).
  4. Report the expected post-rollback state and verification steps.

  IMPORTANT: Rollbacks can cause data loss or break dependent services. Always
  assess cascading impacts before proceeding.

  Format your output as a rollback report with: Target Rollback, Risk Assessment,
  Execution Steps, Expected Post-Rollback State, and Verification Method.
  ```

**Communication Agent**

* Name: `communication-agent`
* Instructions:

  ```text
  You are an SRE communications specialist (Model + Knowledge). You draft
  stakeholder communications based on the current state of an incident. You
  DO NOT make changes to systems — you produce written updates.

  Given the current incident status, diagnostic findings, and mitigation actions:
  1. Draft a status page update appropriate for external customers.
  2. Draft an internal Slack update for the engineering team with technical detail.
  3. If the incident is resolved, draft a brief post-incident summary.

  Adjust tone by audience: external updates should be calm and factual with
  estimated resolution time. Internal updates should be technically detailed
  with current hypothesis and next steps.

  Format your output with clearly labeled sections: Status Page Update, Internal
  Engineering Update, and (if applicable) Post-Incident Summary.
  ```

### A2: Create the Workflow Agent with Magentic Orchestration

1. In the Foundry portal, create a new **Workflow Agent**.
2. Select the **Magentic** (or Adaptive Planning) template.
3. Configure the Manager Agent with the following instructions:

   ```text
   You are an SRE Incident Manager coordinating a production incident response.
   You maintain a Task & Progress Ledger — a living plan that evolves as findings
   emerge. Your available agents are:

   - diagnostics-agent: Analyzes logs/metrics to identify root cause (READ-ONLY)
   - infrastructure-agent: Scales resources, modifies infra (MAKES CHANGES)
   - rollback-agent: Reverts deployments and configurations (MAKES CHANGES)
   - communication-agent: Drafts stakeholder updates (READ-ONLY)

   Your workflow for each evaluation cycle:
   1. ASSESS: Review the current state of the incident and all agent reports so far.
   2. UPDATE LEDGER: Record what you've learned, what's changed, and what remains.
   3. DECIDE: Choose the next agent to dispatch and what to ask them, OR declare
      the incident resolved.
   4. EXPLAIN: State your reasoning for the chosen action.

   Rules:
   - Always start with diagnostics-agent to establish a hypothesis before acting.
   - Never dispatch infrastructure-agent or rollback-agent without a diagnostic
     hypothesis.
   - Dispatch communication-agent at least once during the incident and once at
     resolution.
   - If a mitigation doesn't work, re-dispatch diagnostics-agent to reassess
     before trying something else.
   - Declare the incident resolved only when the root cause is addressed AND
     service health is verified.

   Format your ledger as:
   ## Task & Progress Ledger
   **Incident:** [summary]
   **Status:** [investigating/mitigating/resolved]
   **Current Hypothesis:** [root cause theory]
   **Confidence:** [high/medium/low]
   **Actions Taken:** [numbered list]
   **Next Action:** [agent + task description]
   **Reasoning:** [why this action next]
   ```

4. Add the four specialist agents as available participants.
5. Set a **maximum iteration limit of 8** (to bound cost and prevent infinite loops).
6. Configure the termination condition: the Manager declares the incident resolved.

### A3: Test with the Incident Scenario

Send this incident alert to your workflow agent:

```text
INCIDENT ALERT — Severity 1

Service: Payment Processing API
Impact: 35% of payment transactions failing with HTTP 500 errors
Started: 12 minutes ago (14:23 UTC)
Environment: Production (US-East)

Recent changes:
- Deploy v2.47.3 (payment-service) completed at 14:15 UTC — added retry logic
  for downstream payment gateway timeouts
- Scheduled database maintenance window completed at 13:00 UTC — index rebuild
  on transactions table

Monitoring signals:
- API latency p99 spiked from 200ms to 4,800ms at 14:20 UTC
- Database connection pool utilization at 98% (normal: 40-60%)
- Payment gateway response times normal (no upstream issue)
- Memory usage on payment-service pods: 89% and climbing (normal: 45-55%)
- No network errors between services

Customer reports: "Payment page hangs for 30+ seconds then shows error"
Revenue impact: ~$12,000/minute in failed transactions
```

**Verify that the Manager:**

* Creates an initial Task & Progress Ledger with a starting hypothesis
* Dispatches the Diagnostics Agent first (before any infrastructure changes)
* Updates the ledger after each agent reports back
* Discovers the likely root cause (the retry logic in v2.47.3 is causing a connection pool exhaustion / memory leak under load)
* Replans when initial findings change the picture — the incident looks like a database issue at first glance (connection pool) but the root cause is the deployment
* Dispatches the appropriate mitigation agent (Rollback Agent for the deployment, Infrastructure Agent for immediate resource scaling)
* Sends stakeholder communications at least once during the incident
* Declares the incident resolved only after verifying service recovery

### A4: Inspect the Ledger Evolution

In the Foundry portal, open the run trace and read through the Manager's ledger at each iteration. Create a timeline of how the plan evolved:

* **Iteration 1** — What was the initial hypothesis? What agent did the Manager dispatch first?
* **Iteration 2-3** — How did the diagnostic findings change the hypothesis? Did the Manager replan?
* **Iteration 4-5** — What mitigation was attempted? Did it work the first time?
* **Final iteration** — What was the resolution? Does the ledger provide a clear audit trail?

Count the total number of LLM calls (Manager evaluations + specialist calls). This total is the "most expensive pattern by a significant margin" trade-off from the slides — compare it to what a simple sequential pipeline would have cost.

---

## Track B — Microsoft Agent Framework

This track builds the same Manager-led incident response system in code using the Agent Framework's `MagenticBuilder`.

Choose your language: **C#** or **Python**.

### B1: Project Setup

#### C\#

```bash
dotnet new console -n IncidentResponse
cd IncidentResponse
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Microsoft.AgentFramework --prerelease
dotnet add package Microsoft.AgentFramework.Orchestrations --prerelease
```

#### Python

```bash
mkdir incident-response && cd incident-response
python -m venv .venv && source .venv/bin/activate
pip install azure-ai-projects agent-framework agent-framework-orchestrations
```

### B2: Configure the Model Client

Set up your Azure OpenAI connection. Replace the placeholders with your Foundry project values.

#### C\#

```csharp
using Azure.AI.OpenAI;
using Azure.Identity;

var client = new AzureOpenAIClient(
    new Uri("https://<your-resource>.openai.azure.com/"),
    new DefaultAzureCredential());

var chatClient = client.GetChatClient("<your-deployment-name>");
```

#### Python

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project = AIProjectClient(
    credential=DefaultAzureCredential(),
    endpoint="https://<your-resource>.services.ai.azure.com/api",
    project_name="<your-project>"
)
```

### B3: Define the Specialist Agents

Create four specialist agents. Note the distinction between knowledge agents (read-only) and tool agents (make changes) — this mirrors the two agent types shown on the slide diagram.

#### C\#

```csharp
using Microsoft.AgentFramework;

// Model + Knowledge agents (read-only — analyze and report)
var diagnosticsAgent = new AIAgent(
    name: "DiagnosticsAgent",
    instructions: """
        You are an SRE diagnostics specialist (Model + Knowledge). You analyze
        production incidents by examining logs, metrics, traces, and error patterns.
        You DO NOT make changes to systems — you observe and report.

        Given an incident description and any prior findings from other agents:
        1. Identify the most likely root cause based on available evidence.
        2. Rate your confidence (high/medium/low) and explain what evidence supports
           or contradicts your hypothesis.
        3. List specific diagnostic steps that would increase confidence.
        4. If prior mitigation attempts have been reported, assess whether they
           addressed the root cause or just the symptoms.

        Format your output as a structured diagnostic report with: Hypothesis,
        Confidence, Supporting Evidence, Contradicting Evidence, and Recommended
        Next Steps.
        """,
    client: chatClient);

var communicationAgent = new AIAgent(
    name: "CommunicationAgent",
    instructions: """
        You are an SRE communications specialist (Model + Knowledge). You draft
        stakeholder communications based on the current state of an incident.
        You DO NOT make changes to systems — you produce written updates.

        Given the current incident status, diagnostic findings, and mitigation actions:
        1. Draft a status page update appropriate for external customers.
        2. Draft an internal Slack update for the engineering team with technical detail.
        3. If the incident is resolved, draft a brief post-incident summary.

        Adjust tone by audience: external updates should be calm and factual with
        estimated resolution time. Internal updates should be technically detailed
        with current hypothesis and next steps.

        Format your output with clearly labeled sections: Status Page Update,
        Internal Engineering Update, and (if applicable) Post-Incident Summary.
        """,
    client: chatClient);

// Model + Tools agents (make changes in external systems)
var infrastructureAgent = new AIAgent(
    name: "InfrastructureAgent",
    instructions: """
        You are an SRE infrastructure specialist (Model + Tools). You can make changes
        to production systems including: scaling compute resources, adjusting load
        balancer configurations, modifying network rules, and updating resource limits.

        Given a mitigation request from the Manager with diagnostic context:
        1. Propose the specific infrastructure change needed.
        2. Assess the risk of the change (blast radius, reversibility).
        3. Describe the exact steps you would take (simulated for this exercise).
        4. Report the expected outcome and how to verify the change worked.

        IMPORTANT: Always state what change you are making before executing it.
        Never make changes without explaining the rationale and risk first.

        Format your output as an action report with: Proposed Change, Risk Assessment,
        Execution Steps, Expected Outcome, and Verification Method.
        """,
    client: chatClient);

var rollbackAgent = new AIAgent(
    name: "RollbackAgent",
    instructions: """
        You are an SRE rollback specialist (Model + Tools). You can revert production
        changes including: rolling back deployments, restoring previous configurations,
        reverting database migrations, and restoring from backups.

        Given a rollback request from the Manager with diagnostic context:
        1. Identify what needs to be rolled back and to which version/state.
        2. Assess rollback risk (data loss potential, dependency impacts, rollback
           duration).
        3. Describe the exact rollback steps (simulated for this exercise).
        4. Report the expected post-rollback state and verification steps.

        IMPORTANT: Rollbacks can cause data loss or break dependent services. Always
        assess cascading impacts before proceeding.

        Format your output as a rollback report with: Target Rollback, Risk Assessment,
        Execution Steps, Expected Post-Rollback State, and Verification Method.
        """,
    client: chatClient);
```

#### Python

```python
from agent_framework import AIAgent

# Model + Knowledge agents (read-only — analyze and report)
diagnostics_agent = AIAgent(
    name="DiagnosticsAgent",
    instructions=(
        "You are an SRE diagnostics specialist (Model + Knowledge). You analyze "
        "production incidents by examining logs, metrics, traces, and error patterns. "
        "You DO NOT make changes to systems — you observe and report.\n\n"
        "Given an incident description and any prior findings from other agents:\n"
        "1. Identify the most likely root cause based on available evidence.\n"
        "2. Rate your confidence (high/medium/low) and explain what evidence supports "
        "or contradicts your hypothesis.\n"
        "3. List specific diagnostic steps that would increase confidence.\n"
        "4. If prior mitigation attempts have been reported, assess whether they "
        "addressed the root cause or just the symptoms.\n\n"
        "Format your output as a structured diagnostic report with: Hypothesis, "
        "Confidence, Supporting Evidence, Contradicting Evidence, and Recommended "
        "Next Steps."
    ),
    client=project,
)

communication_agent = AIAgent(
    name="CommunicationAgent",
    instructions=(
        "You are an SRE communications specialist (Model + Knowledge). You draft "
        "stakeholder communications based on the current state of an incident. "
        "You DO NOT make changes to systems — you produce written updates.\n\n"
        "Given the current incident status, diagnostic findings, and mitigation actions:\n"
        "1. Draft a status page update appropriate for external customers.\n"
        "2. Draft an internal Slack update for the engineering team with technical detail.\n"
        "3. If the incident is resolved, draft a brief post-incident summary.\n\n"
        "Adjust tone by audience: external updates should be calm and factual with "
        "estimated resolution time. Internal updates should be technically detailed "
        "with current hypothesis and next steps.\n\n"
        "Format your output with clearly labeled sections: Status Page Update, "
        "Internal Engineering Update, and (if applicable) Post-Incident Summary."
    ),
    client=project,
)

# Model + Tools agents (make changes in external systems)
infrastructure_agent = AIAgent(
    name="InfrastructureAgent",
    instructions=(
        "You are an SRE infrastructure specialist (Model + Tools). You can make changes "
        "to production systems including: scaling compute resources, adjusting load "
        "balancer configurations, modifying network rules, and updating resource limits.\n\n"
        "Given a mitigation request from the Manager with diagnostic context:\n"
        "1. Propose the specific infrastructure change needed.\n"
        "2. Assess the risk of the change (blast radius, reversibility).\n"
        "3. Describe the exact steps you would take (simulated for this exercise).\n"
        "4. Report the expected outcome and how to verify the change worked.\n\n"
        "IMPORTANT: Always state what change you are making before executing it. "
        "Never make changes without explaining the rationale and risk first.\n\n"
        "Format your output as an action report with: Proposed Change, Risk Assessment, "
        "Execution Steps, Expected Outcome, and Verification Method."
    ),
    client=project,
)

rollback_agent = AIAgent(
    name="RollbackAgent",
    instructions=(
        "You are an SRE rollback specialist (Model + Tools). You can revert production "
        "changes including: rolling back deployments, restoring previous configurations, "
        "reverting database migrations, and restoring from backups.\n\n"
        "Given a rollback request from the Manager with diagnostic context:\n"
        "1. Identify what needs to be rolled back and to which version/state.\n"
        "2. Assess rollback risk (data loss potential, dependency impacts, rollback "
        "duration).\n"
        "3. Describe the exact rollback steps (simulated for this exercise).\n"
        "4. Report the expected post-rollback state and verification steps.\n\n"
        "IMPORTANT: Rollbacks can cause data loss or break dependent services. Always "
        "assess cascading impacts before proceeding.\n\n"
        "Format your output as a rollback report with: Target Rollback, Risk Assessment, "
        "Execution Steps, Expected Post-Rollback State, and Verification Method."
    ),
    client=project,
)
```

### B4: Build the Magentic Workflow

Wire the agents into a Magentic orchestration where the Manager Agent creates and maintains the Task & Progress Ledger, dispatches specialists, and runs the Evaluate Goal Loop after each step.

#### Option 1 — MagenticBuilder (Recommended)

The `MagenticBuilder` creates a Manager-led workflow with a built-in ledger and evaluation loop. The Manager receives all agent outputs and decides what to do next.

**C#:**

```csharp
using Microsoft.AgentFramework.Orchestrations;

var workflow = new MagenticBuilder(
    participants: [diagnosticsAgent, infrastructureAgent, rollbackAgent, communicationAgent])
    .WithManagerInstructions("""
        You are an SRE Incident Manager coordinating a production incident response.
        You maintain a Task & Progress Ledger — a living plan that evolves as findings
        emerge. Your available agents are:

        - DiagnosticsAgent: Analyzes logs/metrics to identify root cause (READ-ONLY)
        - InfrastructureAgent: Scales resources, modifies infra (MAKES CHANGES)
        - RollbackAgent: Reverts deployments and configurations (MAKES CHANGES)
        - CommunicationAgent: Drafts stakeholder updates (READ-ONLY)

        Your workflow for each evaluation cycle:
        1. ASSESS: Review the current state of the incident and all agent reports.
        2. UPDATE LEDGER: Record what you've learned, what's changed, what remains.
        3. DECIDE: Choose the next agent to dispatch and what to ask them, OR declare
           the incident resolved.
        4. EXPLAIN: State your reasoning for the chosen action.

        Rules:
        - Always start with DiagnosticsAgent before dispatching tool agents.
        - Never dispatch InfrastructureAgent or RollbackAgent without a hypothesis.
        - Dispatch CommunicationAgent at least once during and once at resolution.
        - If mitigation fails, re-dispatch DiagnosticsAgent before retrying.
        - Resolve only when root cause is addressed AND service health is verified.
        """)
    .WithManagerClient(chatClient)
    .WithMaxIterations(8)
    .Build();
```

**Python:**

```python
from agent_framework.orchestrations import MagenticBuilder

workflow = (
    MagenticBuilder(
        participants=[diagnostics_agent, infrastructure_agent, rollback_agent, communication_agent])
    .with_manager_instructions(
        "You are an SRE Incident Manager coordinating a production incident response. "
        "You maintain a Task & Progress Ledger — a living plan that evolves as findings "
        "emerge. Your available agents are:\n\n"
        "- DiagnosticsAgent: Analyzes logs/metrics to identify root cause (READ-ONLY)\n"
        "- InfrastructureAgent: Scales resources, modifies infra (MAKES CHANGES)\n"
        "- RollbackAgent: Reverts deployments and configurations (MAKES CHANGES)\n"
        "- CommunicationAgent: Drafts stakeholder updates (READ-ONLY)\n\n"
        "Your workflow for each evaluation cycle:\n"
        "1. ASSESS: Review the current state of the incident and all agent reports.\n"
        "2. UPDATE LEDGER: Record what you've learned, what's changed, what remains.\n"
        "3. DECIDE: Choose the next agent to dispatch and what to ask them, OR declare "
        "the incident resolved.\n"
        "4. EXPLAIN: State your reasoning for the chosen action.\n\n"
        "Rules:\n"
        "- Always start with DiagnosticsAgent before dispatching tool agents.\n"
        "- Never dispatch InfrastructureAgent or RollbackAgent without a hypothesis.\n"
        "- Dispatch CommunicationAgent at least once during and once at resolution.\n"
        "- If mitigation fails, re-dispatch DiagnosticsAgent before retrying.\n"
        "- Resolve only when root cause is addressed AND service health is verified."
    )
    .with_manager_client(project)
    .with_max_iterations(8)
    .build()
)
```

#### Option 2 — WorkflowBuilder (For Learning)

The `WorkflowBuilder` lets you implement the Magentic pattern manually — you control the ledger, the evaluation loop, and the dispatch logic explicitly. This is more code but gives you full visibility into how the pattern works.

**C#:**

```csharp
using Microsoft.AgentFramework;

// The Manager is an agent that maintains the ledger and dispatches work
var manager = new AIAgent(
    name: "IncidentManager",
    instructions: """
        You are an SRE Incident Manager. Maintain a Task & Progress Ledger and
        decide which agent to dispatch next. Available agents: DiagnosticsAgent,
        InfrastructureAgent, RollbackAgent, CommunicationAgent.
        Respond with a JSON object: {"next_agent": "<name>", "task": "<description>",
        "ledger_update": "<updated ledger text>", "resolved": false}
        Set "resolved" to true when the incident is fully resolved.
        """,
    client: chatClient);

var builder = new WorkflowBuilder(manager);

// Manager can dispatch to any specialist
builder.AddConditionalEdge(manager, diagnosticsAgent);
builder.AddConditionalEdge(manager, infrastructureAgent);
builder.AddConditionalEdge(manager, rollbackAgent);
builder.AddConditionalEdge(manager, communicationAgent);

// All specialists report back to the Manager (the evaluate goal loop)
builder.AddEdge(diagnosticsAgent, manager);
builder.AddEdge(infrastructureAgent, manager);
builder.AddEdge(rollbackAgent, manager);
builder.AddEdge(communicationAgent, manager);

builder.SetMaxIterations(8);
builder.SetTerminationCondition(output => output.Contains("\"resolved\": true"));

var workflow = builder.Build();
```

**Python:**

```python
from agent_framework import WorkflowBuilder, AIAgent

# The Manager is an agent that maintains the ledger and dispatches work
manager = AIAgent(
    name="IncidentManager",
    instructions=(
        "You are an SRE Incident Manager. Maintain a Task & Progress Ledger and "
        "decide which agent to dispatch next. Available agents: DiagnosticsAgent, "
        "InfrastructureAgent, RollbackAgent, CommunicationAgent. "
        'Respond with a JSON object: {"next_agent": "<name>", "task": "<description>", '
        '"ledger_update": "<updated ledger text>", "resolved": false} '
        'Set "resolved" to true when the incident is fully resolved.'
    ),
    client=project,
)

builder = WorkflowBuilder(start_executor=manager)

# Manager can dispatch to any specialist
builder.add_conditional_edge(manager, diagnostics_agent)
builder.add_conditional_edge(manager, infrastructure_agent)
builder.add_conditional_edge(manager, rollback_agent)
builder.add_conditional_edge(manager, communication_agent)

# All specialists report back to the Manager (the evaluate goal loop)
builder.add_edge(diagnostics_agent, manager)
builder.add_edge(infrastructure_agent, manager)
builder.add_edge(rollback_agent, manager)
builder.add_edge(communication_agent, manager)

builder.set_max_iterations(8)
builder.set_termination_condition(lambda output: '"resolved": true' in output)

workflow = builder.build()
```

### B5: Run the Incident Response

Execute the workflow with the production incident alert.

#### C\#

```csharp
var alert = """
    INCIDENT ALERT — Severity 1

    Service: Payment Processing API
    Impact: 35% of payment transactions failing with HTTP 500 errors
    Started: 12 minutes ago (14:23 UTC)
    Environment: Production (US-East)

    Recent changes:
    - Deploy v2.47.3 (payment-service) completed at 14:15 UTC — added retry logic
      for downstream payment gateway timeouts
    - Scheduled database maintenance window completed at 13:00 UTC — index rebuild
      on transactions table

    Monitoring signals:
    - API latency p99 spiked from 200ms to 4,800ms at 14:20 UTC
    - Database connection pool utilization at 98% (normal: 40-60%)
    - Payment gateway response times normal (no upstream issue)
    - Memory usage on payment-service pods: 89% and climbing (normal: 45-55%)
    - No network errors between services

    Customer reports: "Payment page hangs for 30+ seconds then shows error"
    Revenue impact: ~$12,000/minute in failed transactions
    """;

Console.WriteLine("=== Incident Response Started ===\n");
int iteration = 0;

await foreach (var ev in workflow.RunStreamAsync(alert))
{
    if (ev.Source == "IncidentManager" || ev.Source == "Manager")
    {
        iteration++;
        Console.WriteLine($"\n{'='..50}");
        Console.WriteLine($"  MANAGER — Iteration {iteration}");
        Console.WriteLine($"{'='..50}");
    }
    else
    {
        Console.WriteLine($"\n--- [{ev.Source}] ---");
    }
    Console.WriteLine(ev.Content);
}

Console.WriteLine($"\n=== Incident Resolved in {iteration} iterations ===");
```

#### Python

```python
alert = (
    "INCIDENT ALERT — Severity 1\n\n"
    "Service: Payment Processing API\n"
    "Impact: 35% of payment transactions failing with HTTP 500 errors\n"
    "Started: 12 minutes ago (14:23 UTC)\n"
    "Environment: Production (US-East)\n\n"
    "Recent changes:\n"
    "- Deploy v2.47.3 (payment-service) completed at 14:15 UTC — added retry logic "
    "for downstream payment gateway timeouts\n"
    "- Scheduled database maintenance window completed at 13:00 UTC — index rebuild "
    "on transactions table\n\n"
    "Monitoring signals:\n"
    "- API latency p99 spiked from 200ms to 4,800ms at 14:20 UTC\n"
    "- Database connection pool utilization at 98% (normal: 40-60%)\n"
    "- Payment gateway response times normal (no upstream issue)\n"
    "- Memory usage on payment-service pods: 89% and climbing (normal: 45-55%)\n"
    "- No network errors between services\n\n"
    "Customer reports: 'Payment page hangs for 30+ seconds then shows error'\n"
    "Revenue impact: ~$12,000/minute in failed transactions"
)

print("=== Incident Response Started ===\n")
iteration = 0

async for event in workflow.run_stream(alert):
    if event.source in ("IncidentManager", "Manager"):
        iteration += 1
        print(f"\n{'=' * 50}")
        print(f"  MANAGER — Iteration {iteration}")
        print(f"{'=' * 50}")
    else:
        print(f"\n--- [{event.source}] ---")
    print(event.content)

print(f"\n=== Incident Resolved in {iteration} iterations ===")
```

### B6: Verify the Magentic Behavior

Check that the system demonstrates genuine adaptive planning, not just sequential dispatch:

* **Ledger creation** — the Manager produces an initial Task & Progress Ledger with a hypothesis and plan after seeing the alert
* **Diagnostics-first** — the Manager dispatches the Diagnostics Agent before any tool agents
* **Hypothesis evolution** — the initial hypothesis likely focuses on the database (connection pool at 98%), but the Diagnostics Agent should identify the deployment's retry logic as the root cause (creating connection pool exhaustion + memory leak)
* **Replanning** — when the diagnosis shifts from "database issue" to "deployment issue," the Manager changes its plan from infrastructure scaling to deployment rollback
* **Ordered mitigation** — the Manager dispatches Infrastructure Agent for immediate relief (scale pods) and Rollback Agent for root cause fix (revert v2.47.3), or sequences them based on urgency
* **Communication** — stakeholder updates appear at least during the incident and upon resolution
* **Resolution criteria** — the Manager does not declare resolved until both the root cause is addressed and service metrics are expected to normalize

### B7: Measure the Cost of Adaptive Planning

Instrument the workflow to quantify the trade-offs that make Magentic the most expensive pattern.

#### C\#

```csharp
using System.Diagnostics;

var stopwatch = Stopwatch.StartNew();
int managerCalls = 0;
int specialistCalls = 0;
int totalTokens = 0;

await foreach (var ev in workflow.RunStreamAsync(alert))
{
    int tokens = ev.TokenUsage?.TotalTokens ?? 0;
    totalTokens += tokens;

    if (ev.Source is "IncidentManager" or "Manager")
    {
        managerCalls++;
        Console.WriteLine($"[Manager iteration {managerCalls}] Tokens: {tokens}");
    }
    else
    {
        specialistCalls++;
        Console.WriteLine($"[{ev.Source}] Tokens: {tokens}");
    }
}

Console.WriteLine($"\n--- Cost Summary ---");
Console.WriteLine($"Manager evaluations: {managerCalls}");
Console.WriteLine($"Specialist calls: {specialistCalls}");
Console.WriteLine($"Total LLM calls: {managerCalls + specialistCalls}");
Console.WriteLine($"Total tokens: {totalTokens}");
Console.WriteLine($"Wall-clock time: {stopwatch.Elapsed:mm\\:ss\\.ff}");
```

#### Python

```python
import time

start = time.monotonic()
manager_calls = 0
specialist_calls = 0
total_tokens = 0

async for event in workflow.run_stream(alert):
    tokens = getattr(event, "token_usage", None)
    event_tokens = tokens.total_tokens if tokens else 0
    total_tokens += event_tokens

    if event.source in ("IncidentManager", "Manager"):
        manager_calls += 1
        print(f"[Manager iteration {manager_calls}] Tokens: {event_tokens}")
    else:
        specialist_calls += 1
        print(f"[{event.source}] Tokens: {event_tokens}")

total_time = time.monotonic() - start
print(f"\n--- Cost Summary ---")
print(f"Manager evaluations: {manager_calls}")
print(f"Specialist calls: {specialist_calls}")
print(f"Total LLM calls: {manager_calls + specialist_calls}")
print(f"Total tokens: {total_tokens}")
print(f"Wall-clock time: {total_time:.2f}s")
```

Compare these numbers to the other labs. A typical Magentic run includes 5-8 Manager evaluations plus 4-6 specialist calls — significantly more LLM invocations than Sequential (4 calls), Concurrent (5 calls), or Handoff (2-4 calls). This is the cost of adaptive planning.

---

## Stretch Goals

Once the basic incident response system is working, try one or more of these extensions:

### Force a Replan with a Failed Mitigation

Modify the Infrastructure Agent's instructions to simulate a failed scaling attempt — it reports that the resource quota is exhausted and scaling was unsuccessful. Observe how the Manager handles this: does it replan to prioritize the rollback instead? Does it re-dispatch the Diagnostics Agent to reassess? The ability to recover from dead ends and adapt is the core value proposition of Magentic. A sequential pipeline would have no mechanism to handle this failure.

### Vary the Manager Model Quality

Run the same incident through the workflow three times, using different models for the Manager Agent: a highly capable model (GPT-4o), a mid-tier model (GPT-4o-mini), and a smaller model. Compare the quality of the ledger, the efficiency of the dispatch decisions, and whether the Manager stalls or loops. The slides warn that "a weak model here will lead to poor plans and frequent stalls" — quantify what that means for your scenario.

### Add Security Boundaries for Tool Agents

The slides distinguish between Model + Knowledge agents (read-only) and Model + Tools agents (make changes). Add explicit approval gates before the Infrastructure Agent and Rollback Agent can execute: the Manager must confirm the action before the tool agent proceeds. This mimics a real-world security boundary where agents that modify production systems require additional authorization. Compare the workflow with and without approval gates — the added safety increases latency but prevents unintended changes.

### Compare All Five Patterns on the Same Problem

Build simplified versions of the incident response system using Sequential, Concurrent, Handoff, and Group Chat patterns alongside your Magentic implementation. Run the same alert through all five and compare: resolution quality, total cost (tokens), wall-clock time, and auditability. This exercise demonstrates the complexity spectrum from the slides — Magentic produces the richest audit trail and most adaptive response, but at the highest cost and lowest speed.
