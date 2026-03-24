# Handoff Orchestration — Hands-On Lab

## Lab Overview

Build a **telecom customer support system** using the handoff orchestration pattern. A Triage Agent classifies incoming requests and delegates to the right specialist. Each specialist can handle the issue, transfer to another specialist if the problem shifts domain, or escalate to a human operator. Only one agent is active at any given time:

```text
                                 ┌──→ [Technical Support] ──┐
Request → [Triage Agent] ───────┼──→ [Financial Services]   ├──→ [Human Employee]
                                 ├──→ [Account Access] ─────┘
                                 └──→ [Human Employee]
```

You will choose one of two implementation tracks:

| Track | Technology | Language |
|-------|-----------|----------|
| **Track A** | Azure AI Foundry Workflow Agents | No code (portal + YAML) |
| **Track B** | Microsoft Agent Framework | C# or Python |

Both tracks build the same four-agent handoff system with human escalation. Pick the track that matches your team's skills and goals.

## Prerequisites

* An Azure subscription with access to Azure AI Foundry
* A deployed chat model (GPT-4o or equivalent) in your Foundry project
* **Track A only:** Access to the Azure AI Foundry portal
* **Track B only:** .NET 9+ SDK or Python 3.11+, and an IDE of your choice

## Scenario

Your telecom company's support line routes all calls through a single queue, leading to long hold times and repeated transfers as agents manually figure out who should handle each issue. You are building an intelligent routing system where a Triage Agent classifies the problem and delegates to the right specialist — and specialists can re-route as the issue unfolds:

| Agent | Domain | Handles | Can Transfer To |
|-------|--------|---------|----------------|
| **Triage Agent** | Classification | Classifies issue type from customer description | Technical, Financial, Account, or Human |
| **Technical Support** | Connectivity & devices | Network outages, device configuration, signal issues, service disruptions | Account Access, Human |
| **Financial Services** | Billing & payments | Bill disputes, payment plans, refunds, pricing inquiries | Account Access, Human |
| **Account Access** | Account management | Password resets, locked accounts, permissions, identity verification | Human |
| **Human Employee** | Escalation | Policy exceptions, unresolved issues, compliance-sensitive decisions | (terminal) |

The key insight: the routing path is discovered at runtime. A customer calling about a billing error might get routed to Financial Services, which discovers the root cause is a duplicate account, triggering a handoff to Account Access. This dynamic delegation is what makes Handoff the right pattern — you cannot predict the full path at design time.

---

## Track A — Azure AI Foundry Workflow Agents

This track uses the Foundry portal's visual workflow designer to build the handoff system without writing code.

### A1: Create the Four Specialist Agents

In the Azure AI Foundry portal, create four **prompt agents**. Each agent has domain-specific instructions and knows which other agents it can hand off to.

**Triage Agent**

* Name: `triage-agent`
* Instructions:

  ```text
  You are a telecom customer support triage specialist. Given a customer's
  description of their issue, classify it into one of these categories and
  transfer to the appropriate agent:

  - TECHNICAL: Network outages, slow speeds, device issues, service disruptions
    → Transfer to technical-support
  - FINANCIAL: Billing disputes, unexpected charges, payment plans, refund requests
    → Transfer to financial-services
  - ACCOUNT: Password resets, locked accounts, permission changes, identity issues
    → Transfer to account-access
  - ESCALATION: Abusive language, legal threats, requests for a manager
    → Transfer to human-employee

  Ask one clarifying question if the category is ambiguous, but do not attempt
  to solve the issue yourself. Always transfer after classification.
  ```

**Technical Support Agent**

* Name: `technical-support`
* Instructions:

  ```text
  You are a telecom technical support specialist. You handle network connectivity
  issues, device configuration, signal problems, and service disruptions. Given
  the customer's issue (routed from triage or another agent):

  1. Diagnose the technical problem using the information provided.
  2. Provide troubleshooting steps or a resolution.
  3. If the issue is actually account-related (locked account, permissions), transfer
     to account-access with a summary of what you've found.
  4. If you cannot resolve the issue after troubleshooting, escalate to
     human-employee with your diagnostic findings.

  Always include a brief summary of your diagnosis when transferring.
  ```

**Financial Services Agent**

* Name: `financial-services`
* Instructions:

  ```text
  You are a telecom financial services specialist. You handle billing disputes,
  unexpected charges, payment arrangements, and refund requests. Given the
  customer's issue (routed from triage or another agent):

  1. Review the billing concern and provide an explanation or resolution.
  2. For refunds under $50, approve directly and confirm.
  3. For refunds over $50, explain the finding and escalate to human-employee
     for approval.
  4. If the billing issue stems from a duplicate account or unauthorized access,
     transfer to account-access with your findings.

  Always include a brief summary of your analysis when transferring.
  ```

**Account Access Agent**

* Name: `account-access`
* Instructions:

  ```text
  You are a telecom account access specialist. You handle password resets,
  locked accounts, permission changes, and identity verification. Given the
  customer's issue (routed from triage or another agent):

  1. Verify the customer's identity using the information provided.
  2. Resolve account access issues (unlock, reset, update permissions).
  3. If the issue involves a policy exception (e.g., account owned by a
     deceased family member, corporate account without authorization letter),
     escalate to human-employee with your verification findings.

  Always include a brief summary of your findings when transferring.
  ```

### A2: Create the Workflow Agent with Handoff Routing

1. In the Foundry portal, create a new **Workflow Agent**.
2. Select the **Handoff** (or Routing/Triage) template.
3. Set `triage-agent` as the entry point.
4. Configure the handoff connections:
   * `triage-agent` can transfer to: `technical-support`, `financial-services`, `account-access`, `human-employee`
   * `technical-support` can transfer to: `account-access`, `human-employee`
   * `financial-services` can transfer to: `account-access`, `human-employee`
   * `account-access` can transfer to: `human-employee`
   * `human-employee` is a terminal node (no outgoing transfers)
5. Set a **maximum handoff limit of 4** to prevent infinite loops.
6. Configure the human escalation node to output a structured escalation summary rather than expecting live human input.

### A3: Test with Multiple Scenarios

Run these three scenarios to exercise different routing paths through the system:

**Scenario 1 — Straightforward routing:**

```text
Customer: My internet has been dropping every 30 minutes for the past two days.
I've already restarted my router three times. Account number: TLC-4892.
```

Expected path: Triage → Technical Support → Resolution

**Scenario 2 — Cross-domain handoff:**

```text
Customer: I was charged $247 on my last bill for international calls, but I
never made any international calls. I'm worried someone else has access to my
account. Account number: TLC-7731.
```

Expected path: Triage → Financial Services → (discovers possible unauthorized access) → Account Access → Resolution

**Scenario 3 — Human escalation:**

```text
Customer: My mother passed away last month and I need to transfer her account
to my name. Her account number is TLC-2205. I have the death certificate and
I'm listed as next of kin.
```

Expected path: Triage → Account Access → (policy exception) → Human Employee

**Verify that:**

* The Triage Agent correctly classifies each scenario without attempting to resolve it
* Specialists provide domain-appropriate responses before transferring
* Cross-domain handoffs include a summary of findings from the transferring agent
* The human escalation path activates for policy exceptions
* No scenario triggers more than 4 handoffs

### A4: Inspect the Routing Traces

In the Foundry portal, open the run traces for all three scenarios. For each trace:

* Map the actual routing path and compare it to the expected path
* Note whether any handoff included a context summary from the transferring agent
* Check that the human escalation scenario correctly terminated at the human node
* Compare total latency across the three scenarios — more handoffs mean more time, since only one agent is active at a time. This is the key trade-off vs concurrent execution.

---

## Track B — Microsoft Agent Framework

This track builds the same handoff system in code using the Agent Framework's Swarm pattern, where agents decide when to transfer via tool-based handoff functions.

Choose your language: **C#** or **Python**.

### B1: Project Setup

#### C\#

```bash
dotnet new console -n TelecomSupport
cd TelecomSupport
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Microsoft.AgentFramework --prerelease
dotnet add package Microsoft.AgentFramework.Orchestrations --prerelease
```

#### Python

```bash
mkdir telecom-support && cd telecom-support
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

### B3: Define the Agents with Handoff Targets

Each agent gets domain-specific instructions and a list of agents it can hand off to. The handoff mechanism is tool-based — agents call a `transfer_to` function to route the conversation.

#### C\#

```csharp
using Microsoft.AgentFramework;

var humanEmployee = new AIAgent(
    name: "HumanEmployee",
    instructions: """
        You represent the human escalation endpoint. Produce a structured escalation
        summary including: the original customer issue, the routing path taken,
        findings from each agent that handled the case, and the specific reason
        escalation was triggered. Format as a clear handoff brief for a human operator.
        """,
    client: chatClient);

var accountAccess = new AIAgent(
    name: "AccountAccess",
    instructions: """
        You are a telecom account access specialist. You handle password resets,
        locked accounts, permission changes, and identity verification. Given the
        customer's issue:
        1. Verify the customer's identity using the information provided.
        2. Resolve account access issues (unlock, reset, update permissions).
        3. If the issue involves a policy exception (e.g., account owned by a
           deceased family member, corporate account without authorization letter),
           transfer to HumanEmployee with your verification findings.
        Always include a brief summary of your findings when transferring.
        """,
    client: chatClient);
accountAccess.AddHandoffTargets(humanEmployee);

var technicalSupport = new AIAgent(
    name: "TechnicalSupport",
    instructions: """
        You are a telecom technical support specialist. You handle network connectivity
        issues, device configuration, signal problems, and service disruptions. Given
        the customer's issue:
        1. Diagnose the technical problem using the information provided.
        2. Provide troubleshooting steps or a resolution.
        3. If the issue is actually account-related (locked account, permissions),
           transfer to AccountAccess with a summary of what you've found.
        4. If you cannot resolve the issue after troubleshooting, escalate to
           HumanEmployee with your diagnostic findings.
        Always include a brief summary of your diagnosis when transferring.
        """,
    client: chatClient);
technicalSupport.AddHandoffTargets(accountAccess, humanEmployee);

var financialServices = new AIAgent(
    name: "FinancialServices",
    instructions: """
        You are a telecom financial services specialist. You handle billing disputes,
        unexpected charges, payment arrangements, and refund requests. Given the
        customer's issue:
        1. Review the billing concern and provide an explanation or resolution.
        2. For refunds under $50, approve directly and confirm.
        3. For refunds over $50, explain the finding and escalate to HumanEmployee
           for approval.
        4. If the billing issue stems from a duplicate account or unauthorized access,
           transfer to AccountAccess with your findings.
        Always include a brief summary of your analysis when transferring.
        """,
    client: chatClient);
financialServices.AddHandoffTargets(accountAccess, humanEmployee);

var triageAgent = new AIAgent(
    name: "TriageAgent",
    instructions: """
        You are a telecom customer support triage specialist. Classify the customer's
        issue and transfer to the appropriate agent:
        - TECHNICAL (network, devices, connectivity) → TechnicalSupport
        - FINANCIAL (billing, charges, refunds) → FinancialServices
        - ACCOUNT (passwords, locked accounts, permissions) → AccountAccess
        - ESCALATION (policy exceptions, manager requests) → HumanEmployee
        Ask one clarifying question if the category is ambiguous, but do not attempt
        to solve the issue yourself. Always transfer after classification.
        """,
    client: chatClient);
triageAgent.AddHandoffTargets(technicalSupport, financialServices, accountAccess, humanEmployee);
```

#### Python

```python
from agent_framework import AIAgent

human_employee = AIAgent(
    name="HumanEmployee",
    instructions=(
        "You represent the human escalation endpoint. Produce a structured escalation "
        "summary including: the original customer issue, the routing path taken, "
        "findings from each agent that handled the case, and the specific reason "
        "escalation was triggered. Format as a clear handoff brief for a human operator."
    ),
    client=project,
)

account_access = AIAgent(
    name="AccountAccess",
    instructions=(
        "You are a telecom account access specialist. You handle password resets, "
        "locked accounts, permission changes, and identity verification. Given the "
        "customer's issue: "
        "1. Verify the customer's identity using the information provided. "
        "2. Resolve account access issues (unlock, reset, update permissions). "
        "3. If the issue involves a policy exception (e.g., account owned by a "
        "deceased family member, corporate account without authorization letter), "
        "transfer to HumanEmployee with your verification findings. "
        "Always include a brief summary of your findings when transferring."
    ),
    client=project,
)
account_access.add_handoff_targets(human_employee)

technical_support = AIAgent(
    name="TechnicalSupport",
    instructions=(
        "You are a telecom technical support specialist. You handle network connectivity "
        "issues, device configuration, signal problems, and service disruptions. Given "
        "the customer's issue: "
        "1. Diagnose the technical problem using the information provided. "
        "2. Provide troubleshooting steps or a resolution. "
        "3. If the issue is actually account-related (locked account, permissions), "
        "transfer to AccountAccess with a summary of what you've found. "
        "4. If you cannot resolve the issue after troubleshooting, escalate to "
        "HumanEmployee with your diagnostic findings. "
        "Always include a brief summary of your diagnosis when transferring."
    ),
    client=project,
)
technical_support.add_handoff_targets(account_access, human_employee)

financial_services = AIAgent(
    name="FinancialServices",
    instructions=(
        "You are a telecom financial services specialist. You handle billing disputes, "
        "unexpected charges, payment arrangements, and refund requests. Given the "
        "customer's issue: "
        "1. Review the billing concern and provide an explanation or resolution. "
        "2. For refunds under $50, approve directly and confirm. "
        "3. For refunds over $50, explain the finding and escalate to HumanEmployee "
        "for approval. "
        "4. If the billing issue stems from a duplicate account or unauthorized access, "
        "transfer to AccountAccess with your findings. "
        "Always include a brief summary of your analysis when transferring."
    ),
    client=project,
)
financial_services.add_handoff_targets(account_access, human_employee)

triage_agent = AIAgent(
    name="TriageAgent",
    instructions=(
        "You are a telecom customer support triage specialist. Classify the customer's "
        "issue and transfer to the appropriate agent: "
        "- TECHNICAL (network, devices, connectivity) → TechnicalSupport "
        "- FINANCIAL (billing, charges, refunds) → FinancialServices "
        "- ACCOUNT (passwords, locked accounts, permissions) → AccountAccess "
        "- ESCALATION (policy exceptions, manager requests) → HumanEmployee "
        "Ask one clarifying question if the category is ambiguous, but do not attempt "
        "to solve the issue yourself. Always transfer after classification."
    ),
    client=project,
)
triage_agent.add_handoff_targets(technical_support, financial_services, account_access, human_employee)
```

### B4: Build the Handoff Workflow

Wire the agents into a Swarm-based handoff system. The Swarm pattern lets agents decide when and where to transfer using tool calls — the framework handles context passing between agents.

#### Option 1 — HandoffBuilder (Recommended)

The `HandoffBuilder` creates a Swarm where the entry agent receives the request and agents use their registered handoff targets to route dynamically.

**C#:**

```csharp
using Microsoft.AgentFramework.Orchestrations;

var workflow = new HandoffBuilder(
    entryAgent: triageAgent,
    participants: [triageAgent, technicalSupport, financialServices, accountAccess, humanEmployee])
    .WithMaxHandoffs(4)
    .Build();
```

**Python:**

```python
from agent_framework.orchestrations import HandoffBuilder

workflow = (
    HandoffBuilder(
        entry_agent=triage_agent,
        participants=[triage_agent, technical_support, financial_services, account_access, human_employee])
    .with_max_handoffs(4)
    .build()
)
```

#### Option 2 — WorkflowBuilder (For Learning)

The `WorkflowBuilder` gives you explicit control over the routing edges. This helps you visualize the handoff graph and understand how the Swarm pattern works under the hood.

**C#:**

```csharp
using Microsoft.AgentFramework;

var builder = new WorkflowBuilder(triageAgent);

// Triage can route to any specialist or escalate directly
builder.AddConditionalEdge(triageAgent, technicalSupport);
builder.AddConditionalEdge(triageAgent, financialServices);
builder.AddConditionalEdge(triageAgent, accountAccess);
builder.AddConditionalEdge(triageAgent, humanEmployee);

// Specialists can cross-route or escalate
builder.AddConditionalEdge(technicalSupport, accountAccess);
builder.AddConditionalEdge(technicalSupport, humanEmployee);
builder.AddConditionalEdge(financialServices, accountAccess);
builder.AddConditionalEdge(financialServices, humanEmployee);
builder.AddConditionalEdge(accountAccess, humanEmployee);

builder.SetMaxHandoffs(4);

var workflow = builder.Build();
```

**Python:**

```python
from agent_framework import WorkflowBuilder

builder = WorkflowBuilder(start_executor=triage_agent)

# Triage can route to any specialist or escalate directly
builder.add_conditional_edge(triage_agent, technical_support)
builder.add_conditional_edge(triage_agent, financial_services)
builder.add_conditional_edge(triage_agent, account_access)
builder.add_conditional_edge(triage_agent, human_employee)

# Specialists can cross-route or escalate
builder.add_conditional_edge(technical_support, account_access)
builder.add_conditional_edge(technical_support, human_employee)
builder.add_conditional_edge(financial_services, account_access)
builder.add_conditional_edge(financial_services, human_employee)
builder.add_conditional_edge(account_access, human_employee)

builder.set_max_handoffs(4)

workflow = builder.build()
```

### B5: Run the Three Scenarios

Execute the workflow with each test scenario and observe the routing decisions.

#### C\#

```csharp
var scenarios = new Dictionary<string, string>
{
    ["Straightforward"] = """
        Customer: My internet has been dropping every 30 minutes for the past
        two days. I've already restarted my router three times.
        Account number: TLC-4892.
        """,

    ["Cross-domain"] = """
        Customer: I was charged $247 on my last bill for international calls,
        but I never made any international calls. I'm worried someone else has
        access to my account. Account number: TLC-7731.
        """,

    ["Human escalation"] = """
        Customer: My mother passed away last month and I need to transfer her
        account to my name. Her account number is TLC-2205. I have the death
        certificate and I'm listed as next of kin.
        """
};

foreach (var (name, request) in scenarios)
{
    Console.WriteLine($"\n=== Scenario: {name} ===\n");
    var path = new List<string>();

    await foreach (var ev in workflow.RunStreamAsync(request))
    {
        path.Add(ev.Source);
        Console.WriteLine($"[{ev.Source}] {ev.Content}");
        if (ev.HandoffTo is not null)
            Console.WriteLine($"  → Handing off to: {ev.HandoffTo}");
        Console.WriteLine("---");
    }

    Console.WriteLine($"\nRouting path: {string.Join(" → ", path)}");
}
```

#### Python

```python
scenarios = {
    "Straightforward": (
        "Customer: My internet has been dropping every 30 minutes for the past "
        "two days. I've already restarted my router three times. "
        "Account number: TLC-4892."
    ),

    "Cross-domain": (
        "Customer: I was charged $247 on my last bill for international calls, "
        "but I never made any international calls. I'm worried someone else has "
        "access to my account. Account number: TLC-7731."
    ),

    "Human escalation": (
        "Customer: My mother passed away last month and I need to transfer her "
        "account to my name. Her account number is TLC-2205. I have the death "
        "certificate and I'm listed as next of kin."
    ),
}

for name, request in scenarios.items():
    print(f"\n=== Scenario: {name} ===\n")
    path = []

    async for event in workflow.run_stream(request):
        path.append(event.source)
        print(f"[{event.source}] {event.content}")
        if hasattr(event, "handoff_to") and event.handoff_to:
            print(f"  → Handing off to: {event.handoff_to}")
        print("---")

    print(f"\nRouting path: {' → '.join(path)}")
```

### B6: Verify the Routing Behavior

For each scenario, check the routing path and agent behavior:

**Scenario 1 — Straightforward routing:**

* Triage correctly identifies a technical issue and transfers to Technical Support
* Technical Support diagnoses the connectivity problem and provides a resolution
* No unnecessary handoffs — the issue is resolved in 2 steps (Triage → Technical)

**Scenario 2 — Cross-domain handoff:**

* Triage identifies a billing issue and transfers to Financial Services
* Financial Services discovers possible unauthorized account access and transfers to Account Access with findings
* Account Access resolves the security concern
* The cross-domain handoff includes a context summary so Account Access does not re-ask questions

**Scenario 3 — Human escalation:**

* Triage or Account Access recognizes the policy exception (deceased account holder)
* The human escalation node produces a structured brief with the full routing history
* The escalation reason is clearly stated (policy exception requiring human judgment)

### B7: Add Loop Detection

The slides call out infinite handoff loops as the biggest risk. Add explicit safeguards to your workflow.

#### C\#

```csharp
using Microsoft.AgentFramework.Orchestrations;

var workflow = new HandoffBuilder(
    entryAgent: triageAgent,
    participants: [triageAgent, technicalSupport, financialServices, accountAccess, humanEmployee])
    .WithMaxHandoffs(4)
    .WithLoopDetection(maxVisitsPerAgent: 2)
    .WithFallback(humanEmployee)  // If max handoffs exceeded, escalate to human
    .Build();
```

#### Python

```python
from agent_framework.orchestrations import HandoffBuilder

workflow = (
    HandoffBuilder(
        entry_agent=triage_agent,
        participants=[triage_agent, technical_support, financial_services, account_access, human_employee])
    .with_max_handoffs(4)
    .with_loop_detection(max_visits_per_agent=2)
    .with_fallback(human_employee)  # If max handoffs exceeded, escalate to human
    .build()
)
```

Test the loop detection with an adversarial scenario designed to trigger circular routing:

```text
Customer: I keep getting bounced between departments. My bill is wrong because
my service was interrupted, and my service was interrupted because my account
was locked, and my account was locked because of the billing dispute. Nobody
can help me. Account number: TLC-9103.
```

This scenario touches all three domains (billing, technical, account) and could trigger circular routing. Verify that the loop detection catches it and falls back to human escalation.

---

## Stretch Goals

Once the basic handoff system is working, try one or more of these extensions:

### Add Context Preservation Across Handoffs

By default, each agent receives the full conversation history. Modify the workflow to instead pass a structured handoff summary — a condensed context packet with the customer's issue, the routing path so far, and key findings from each agent. This reduces token usage (shorter context windows) and prevents later agents from being confused by earlier agents' reasoning. Compare the output quality and token cost of both approaches.

### Implement a Confidence-Based Triage

Replace the simple keyword-based classification in the Triage Agent with a confidence score. If the Triage Agent is more than 80% confident in the category, route directly. If confidence is between 50-80%, ask one clarifying question before routing. If below 50%, escalate to the human employee. This addresses the "if you can identify the appropriate agent from initial input" guidance from the slides — sometimes the answer is "not confidently enough."

### Compare Handoff vs Sequential

Build the same four-agent system as a sequential pipeline (Triage → Technical → Financial → Account) and run the same three scenarios through both. Compare:

* **Efficiency** — how many agents actually process the request in Handoff vs all four in Sequential?
* **Quality** — does the handoff version produce better results by routing to the right specialist first?
* **Latency** — fewer active agents in Handoff means fewer LLM calls, but unpredictable path length

This exercise demonstrates why the slides recommend Handoff over Sequential when the right expertise emerges during processing rather than being known upfront.

### Add Real Human-in-the-Loop

Replace the `HumanEmployee` agent with an actual human interaction point. When the workflow reaches the human escalation node, pause execution and prompt the user for input via the console (Track B) or a manual approval step (Track A). Resume the workflow after the human responds. This transforms the human node from a mock endpoint into a genuine escalation gate — the "natural fit for human-in-the-loop" benefit described in the slides.
