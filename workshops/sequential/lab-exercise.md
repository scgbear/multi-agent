# Sequential Orchestration — Hands-On Lab

## Lab Overview

Build a **contract generation pipeline** using the sequential orchestration pattern. Four specialized agents process a contract request in a linear chain, each enriching the artifact produced by the previous stage:

```text
Input → [Template Selection] → [Clause Customization] → [Regulatory Compliance] → [Risk Assessment] → Result
```

You will choose one of two implementation tracks:

| Track | Technology | Language |
|-------|-----------|----------|
| **Track A** | Azure AI Foundry Workflow Agents | No code (portal + YAML) |
| **Track B** | Microsoft Agent Framework | C# or Python |

Both tracks build the same four-agent sequential pipeline. Pick the track that matches your team's skills and goals.

## Prerequisites

* An Azure subscription with access to Azure AI Foundry
* A deployed chat model (GPT-4o or equivalent) in your Foundry project
* **Track A only:** Access to the Azure AI Foundry portal
* **Track B only:** .NET 9+ SDK or Python 3.11+, and an IDE of your choice

## Scenario

Your legal department manually assembles contracts, a process that takes hours and often misses regulatory updates. You are building an automated pipeline where each stage is handled by a specialized agent:

| Stage | Agent | Responsibility |
|-------|-------|----------------|
| 1 | **Template Selector** | Picks the right contract template based on deal type, jurisdiction, and counterparty |
| 2 | **Clause Customizer** | Tailors clauses to the specific deal terms — pricing, SLAs, liability caps |
| 3 | **Compliance Checker** | Reviews the draft against current regulatory requirements and flags violations |
| 4 | **Risk Assessor** | Evaluates overall contract exposure and produces a risk summary with recommendations |

---

## Track A — Azure AI Foundry Workflow Agents

This track uses the Foundry portal's visual workflow designer to build the sequential pipeline without writing code.

### A1: Create the Four Prompt Agents

In the Azure AI Foundry portal, create four **prompt agents** — one for each pipeline stage. Each agent uses the same deployed model but has distinct instructions.

**Template Selector Agent**

* Name: `template-selector`
* Instructions:

  ```text
  You are a contract template selection specialist. Given a contract request
  containing deal type, jurisdiction, and counterparty information, select and
  return the most appropriate contract template. Output a structured contract
  draft with placeholder sections for clauses, compliance, and risk.
  ```

**Clause Customizer Agent**

* Name: `clause-customizer`
* Instructions:

  ```text
  You are a contract clause customization specialist. Given a contract draft
  with placeholder sections, tailor the clauses to the specific deal terms
  including pricing, SLAs, liability caps, and indemnification. Preserve the
  overall document structure and pass the enriched draft forward.
  ```

**Compliance Checker Agent**

* Name: `compliance-checker`
* Instructions:

  ```text
  You are a regulatory compliance reviewer. Given a customized contract draft,
  review it against current regulatory requirements for the specified
  jurisdiction. Flag any violations or missing required clauses. Add a
  compliance status section to the document and pass the updated draft forward.
  ```

**Risk Assessor Agent**

* Name: `risk-assessor`
* Instructions:

  ```text
  You are a contract risk assessment specialist. Given a compliance-reviewed
  contract draft, evaluate overall contract exposure including financial risk,
  liability gaps, and unfavorable terms. Produce a final risk summary with
  severity ratings and concrete recommendations appended to the contract.
  ```

### A2: Create the Workflow Agent

1. In the Foundry portal, create a new **Workflow Agent**.
2. Select the **Sequential** template.
3. Add the four agents in order: `template-selector` → `clause-customizer` → `compliance-checker` → `risk-assessor`.
4. Configure each step to pass its output as the input to the next agent.

### A3: Test the Pipeline

Send this request to your workflow agent:

```text
Generate a contract for a 3-year enterprise SaaS agreement with Contoso Ltd
in the state of California. Deal value: $2.4M annually. Requirements include
99.9% uptime SLA, data residency in US-West, HIPAA compliance for healthcare
data, and a $5M liability cap.
```

**Verify that:**

* The Template Selector picks an enterprise SaaS template and produces a structured draft
* The Clause Customizer fills in pricing, SLA, liability, and indemnification clauses
* The Compliance Checker flags HIPAA and California-specific requirements
* The Risk Assessor produces a severity-rated risk summary with recommendations

### A4: Inspect the Pipeline Trace

In the Foundry portal, open the run trace for your workflow execution. Note how each agent's output feeds directly into the next agent's input — this is the sequential pattern in action. Observe token usage per stage and total pipeline latency.

---

## Track B — Microsoft Agent Framework

This track builds the same four-agent pipeline in code using the Agent Framework's `SequentialBuilder` or `WorkflowBuilder`.

Choose your language: **C#** or **Python**.

### B1: Project Setup

#### C\#

```bash
dotnet new console -n ContractPipeline
cd ContractPipeline
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Microsoft.AgentFramework --prerelease
dotnet add package Microsoft.AgentFramework.Orchestrations --prerelease
```

#### Python

```bash
mkdir contract-pipeline && cd contract-pipeline
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

### B3: Define the Four Agents

Each agent gets specific instructions matching its role in the pipeline. Define all four with the same model but distinct system prompts.

#### C\#

```csharp
using Microsoft.AgentFramework;

var templateSelector = new AIAgent(
    name: "TemplateSelector",
    instructions: """
        You are a contract template selection specialist. Given a contract request
        containing deal type, jurisdiction, and counterparty information, select and
        return the most appropriate contract template. Output a structured contract
        draft with placeholder sections for clauses, compliance, and risk.
        """,
    client: chatClient);

var clauseCustomizer = new AIAgent(
    name: "ClauseCustomizer",
    instructions: """
        You are a contract clause customization specialist. Given a contract draft
        with placeholder sections, tailor the clauses to the specific deal terms
        including pricing, SLAs, liability caps, and indemnification. Preserve the
        overall document structure and pass the enriched draft forward.
        """,
    client: chatClient);

var complianceChecker = new AIAgent(
    name: "ComplianceChecker",
    instructions: """
        You are a regulatory compliance reviewer. Given a customized contract draft,
        review it against current regulatory requirements for the specified
        jurisdiction. Flag any violations or missing required clauses. Add a
        compliance status section to the document and pass the updated draft forward.
        """,
    client: chatClient);

var riskAssessor = new AIAgent(
    name: "RiskAssessor",
    instructions: """
        You are a contract risk assessment specialist. Given a compliance-reviewed
        contract draft, evaluate overall contract exposure including financial risk,
        liability gaps, and unfavorable terms. Produce a final risk summary with
        severity ratings and concrete recommendations appended to the contract.
        """,
    client: chatClient);
```

#### Python

```python
from agent_framework import AIAgent

template_selector = AIAgent(
    name="TemplateSelector",
    instructions=(
        "You are a contract template selection specialist. Given a contract request "
        "containing deal type, jurisdiction, and counterparty information, select and "
        "return the most appropriate contract template. Output a structured contract "
        "draft with placeholder sections for clauses, compliance, and risk."
    ),
    client=project,
)

clause_customizer = AIAgent(
    name="ClauseCustomizer",
    instructions=(
        "You are a contract clause customization specialist. Given a contract draft "
        "with placeholder sections, tailor the clauses to the specific deal terms "
        "including pricing, SLAs, liability caps, and indemnification. Preserve the "
        "overall document structure and pass the enriched draft forward."
    ),
    client=project,
)

compliance_checker = AIAgent(
    name="ComplianceChecker",
    instructions=(
        "You are a regulatory compliance reviewer. Given a customized contract draft, "
        "review it against current regulatory requirements for the specified "
        "jurisdiction. Flag any violations or missing required clauses. Add a "
        "compliance status section to the document and pass the updated draft forward."
    ),
    client=project,
)

risk_assessor = AIAgent(
    name="RiskAssessor",
    instructions=(
        "You are a contract risk assessment specialist. Given a compliance-reviewed "
        "contract draft, evaluate overall contract exposure including financial risk, "
        "liability gaps, and unfavorable terms. Produce a final risk summary with "
        "severity ratings and concrete recommendations appended to the contract."
    ),
    client=project,
)
```

### B4: Build the Sequential Workflow

Wire the four agents into a sequential pipeline.

#### Option 1 — SequentialBuilder (Recommended)

The `SequentialBuilder` is the simplest way to create a sequential pipeline. It chains agents in the order you provide them.

**C#:**

```csharp
using Microsoft.AgentFramework.Orchestrations;

var workflow = new SequentialBuilder(
    participants: [templateSelector, clauseCustomizer, complianceChecker, riskAssessor]
).Build();
```

**Python:**

```python
from agent_framework.orchestrations import SequentialBuilder

workflow = SequentialBuilder(
    participants=[template_selector, clause_customizer, compliance_checker, risk_assessor]
).build()
```

#### Option 2 — WorkflowBuilder (For Learning)

The `WorkflowBuilder` gives you explicit control over edges, which helps you understand how sequential orchestration works under the hood.

**C#:**

```csharp
using Microsoft.AgentFramework;

var builder = new WorkflowBuilder(templateSelector);
builder.AddEdge(templateSelector, clauseCustomizer);
builder.AddEdge(clauseCustomizer, complianceChecker);
builder.AddEdge(complianceChecker, riskAssessor);
var workflow = builder.Build();
```

**Python:**

```python
from agent_framework import WorkflowBuilder

workflow = (
    WorkflowBuilder(start_executor=template_selector)
    .add_edge(template_selector, clause_customizer)
    .add_edge(clause_customizer, compliance_checker)
    .add_edge(compliance_checker, risk_assessor)
    .build()
)
```

### B5: Run the Pipeline

Execute the workflow with the same contract request.

#### C\#

```csharp
var request = """
    Generate a contract for a 3-year enterprise SaaS agreement with Contoso Ltd
    in the state of California. Deal value: $2.4M annually. Requirements include
    99.9% uptime SLA, data residency in US-West, HIPAA compliance for healthcare
    data, and a $5M liability cap.
    """;

await foreach (var ev in workflow.RunStreamAsync(request))
{
    Console.WriteLine($"[{ev.Source}] {ev.Content}");
}
```

#### Python

```python
request = (
    "Generate a contract for a 3-year enterprise SaaS agreement with Contoso Ltd "
    "in the state of California. Deal value: $2.4M annually. Requirements include "
    "99.9% uptime SLA, data residency in US-West, HIPAA compliance for healthcare "
    "data, and a $5M liability cap."
)

async for event in workflow.run_stream(request):
    print(f"[{event.source}] {event.content}")
```

### B6: Verify the Output

Check that the pipeline produced a contract document that shows clear evidence of each stage:

* **Template Selection** — correct template type chosen, structured with placeholder sections
* **Clause Customization** — deal-specific terms filled in (pricing, SLA, liability cap)
* **Compliance Review** — HIPAA and California requirements flagged, compliance section added
* **Risk Assessment** — risk summary with severity ratings and recommendations appended

---

## Stretch Goals

Once the basic pipeline is working, try one or more of these extensions:

### Add Observability

Instrument your pipeline so you can see each agent's input/output and token usage. In Track B, enable OpenTelemetry tracing. In Track A, use the Foundry portal's built-in run trace view.

### Swap Models Per Stage

Not every agent needs the most capable model. Try assigning a smaller, faster model (e.g., GPT-4o-mini) to the Template Selector while keeping GPT-4o for the Risk Assessor. Compare quality and latency.

### Add Output Validation Between Stages

Insert validation logic between agents that checks the output format or content before passing it to the next stage. For example, verify that the Clause Customizer's output contains all required clause headings before sending it to the Compliance Checker. This addresses the "single point of failure cascades" trade-off from the slide.

### Graduate from Foundry to Code (Track A Participants)

Use the Foundry portal's **Generate Code** button to export your workflow as Agent Framework code. Compare the generated code with the Track B implementation.

---

## Discussion Points

After completing the lab, consider these questions for group discussion:

* Where did the pipeline add the most value compared to a single-agent approach?
* Which stage would benefit most from a more capable (or less capable) model?
* What happens if the Compliance Checker finds a blocking violation — should the pipeline backtrack to the Clause Customizer? (Hint: that would require a different pattern.)
* How would you add a human approval gate before the Risk Assessor finalizes the contract?
* At what point would you consider breaking this into separate services communicating via A2A rather than running all agents in-process?
