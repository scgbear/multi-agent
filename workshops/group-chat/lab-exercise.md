# Group Chat Orchestration — Hands-On Lab

## Lab Overview

Build a **city parks proposal review system** using the group chat orchestration pattern. Three specialized agents debate a development proposal through a shared conversation thread, moderated by an LLM-based Chat Manager that controls turn order:

```text
                             ┌──→ [Environmental Agent]
Request → [Chat Manager] ←──┼──→ [Community Agent]      ←→ Accumulating Chat Thread → Consensus Result
                             └──→ [Budget Agent]
```

You will choose one of two implementation tracks:

| Track | Technology | Language |
|-------|-----------|----------|
| **Track A** | Azure AI Foundry Workflow Agents | No code (portal + YAML) |
| **Track B** | Microsoft Agent Framework | C# or Python |

Both tracks build the same three-agent group chat with LLM-moderated turn selection. Pick the track that matches your team's skills and goals.

## Prerequisites

* An Azure subscription with access to Azure AI Foundry
* A deployed chat model (GPT-4o or equivalent) in your Foundry project
* **Track A only:** Access to the Azure AI Foundry portal
* **Track B only:** .NET 9+ SDK or Python 3.11+, and an IDE of your choice

## Scenario

Your city council reviews park development proposals through a committee process that takes weeks — environmental consultants, community advocates, and fiscal analysts submit written reports that rarely reference each other's concerns. Decisions are slow and stakeholders feel unheard. You are building a collaborative review system where three agents debate the proposal in real time, responding to and refining each other's positions:

| Agent | Perspective | Focus |
|-------|------------|-------|
| **Environmental Agent** | Ecological impact | Habitat disruption, tree canopy, stormwater, wildlife corridors, mitigation strategies |
| **Community Agent** | Resident needs | Green space access, recreation, walkability, equity, neighborhood engagement |
| **Budget Agent** | Fiscal responsibility | Cost estimates, funding sources, maintenance projections, ROI, phased implementation |

The key insight: unlike Sequential or Concurrent, these agents need to see and respond to each other's reasoning. The Environmental Agent's habitat concerns may change the Budget Agent's cost estimates. The Community Agent's equity argument may shift priorities for both. This back-and-forth refinement is what makes Group Chat the right pattern.

---

## Track A — Azure AI Foundry Workflow Agents

This track uses the Foundry portal's visual workflow designer to build the group chat without writing code.

### A1: Create the Three Participant Agents

In the Azure AI Foundry portal, create three **prompt agents** — one for each committee perspective. Each agent uses the same deployed model but has distinct instructions and a clear stakeholder voice.

**Environmental Agent**

* Name: `environmental-agent`
* Instructions:

  ```text
  You are an environmental consultant reviewing a city park development proposal.
  Your role is to evaluate ecological impact including: habitat disruption to
  existing wildlife, tree canopy preservation, stormwater management implications,
  wildlife corridor connectivity, and soil/water quality effects. You advocate for
  environmental protection but are willing to negotiate when effective mitigation
  strategies are proposed. When other agents raise concerns, respond constructively
  — propose mitigation approaches that address their needs while protecting
  ecological value. Keep responses focused and under 200 words.
  ```

**Community Agent**

* Name: `community-agent`
* Instructions:

  ```text
  You are a community advocate reviewing a city park development proposal. Your
  role is to represent resident needs including: green space accessibility for all
  neighborhoods, recreational facilities for diverse age groups, walkability and
  transit connections, equity considerations for underserved communities, and
  neighborhood engagement in the design process. You champion community benefit
  but acknowledge budget and environmental constraints when raised by other agents.
  When other agents raise concerns, respond constructively — find ways to preserve
  community value while respecting their constraints. Keep responses focused and
  under 200 words.
  ```

**Budget Agent**

* Name: `budget-agent`
* Instructions:

  ```text
  You are a fiscal analyst reviewing a city park development proposal. Your role
  is to evaluate financial feasibility including: construction cost estimates,
  available funding sources (municipal bonds, grants, public-private partnerships),
  long-term maintenance projections, return on investment through property value
  and tourism impact, and phased implementation options to manage cash flow. You
  advocate for fiscal responsibility but recognize that some investments have
  long-term returns that justify upfront costs. When other agents raise concerns,
  respond constructively — propose funding strategies or phasing that addresses
  their priorities within fiscal constraints. Keep responses focused and under
  200 words.
  ```

### A2: Create the Workflow Agent with Group Chat

1. In the Foundry portal, create a new **Workflow Agent**.
2. Select the **Group Chat** template.
3. Add the three agents as participants: `environmental-agent`, `community-agent`, and `budget-agent`.
4. Configure the Chat Manager with the following instructions:

   ```text
   You are a city council meeting moderator. Your job is to manage a structured
   discussion about a park development proposal between three committee members:
   Environmental, Community, and Budget. Follow these rules:
   1. Let each agent speak at least once before any agent speaks a second time.
   2. After the first round, select the next speaker based on who has the most
      relevant response to what was just said.
   3. When agents begin repeating positions or the discussion is converging,
      ask each agent for a final summary position.
   4. End the discussion after 3-4 rounds or when consensus emerges.
   Output the name of the next agent to speak, or "DONE" to end the discussion.
   ```

5. Set the maximum turns to **12** (roughly 4 rounds of 3 agents).
6. Configure a termination condition: the Chat Manager outputs "DONE" or the max turn limit is reached.

### A3: Test the Group Chat

Send this proposal to your workflow agent:

```text
Review the following city park development proposal:

Riverside Park Redevelopment — Phase 1
Location: 12-acre site along Cedar Creek, adjacent to the Millbrook and Oakdale
neighborhoods. The site includes 3 acres of mature riparian forest and a
decommissioned water treatment facility.

Proposed features: Community playground, splash pad, multi-use sports courts,
1.2-mile walking trail along the creek, native plant restoration zone, community
garden plots, and a small events pavilion.

Estimated budget: $2.4M construction, $180K annual maintenance.
Funding: $1.5M municipal bond allocation, seeking $900K in state green
infrastructure grants.

Timeline: 18-month construction, phased opening.

The council needs a recommendation that balances environmental preservation,
community benefit, and fiscal responsibility.
```

**Verify that:**

* The Environmental Agent raises specific concerns about the riparian forest and creek habitat
* The Community Agent advocates for features that serve both Millbrook and Oakdale neighborhoods
* The Budget Agent evaluates the $2.4M cost and the funding gap if the grant falls through
* Agents respond to each other — not just repeating their initial positions
* The conversation converges toward a recommendation rather than looping indefinitely
* The Chat Manager moderates effectively, selecting relevant speakers and ending the discussion

### A4: Inspect the Conversation Thread

In the Foundry portal, open the run trace for your workflow execution. Read through the full conversation thread and observe:

* **Turn order** — did the Chat Manager select speakers based on relevance, or was it round-robin?
* **Cross-references** — do agents reference specific points from other agents' messages?
* **Convergence** — did the discussion narrow toward a recommendation, or did it spiral?
* **Token cost** — note the total tokens consumed across all turns. Compare this to what a single-agent analysis would cost. This is the overhead trade-off of group chat.

---

## Track B — Microsoft Agent Framework

This track builds the same three-agent group chat in code using the Agent Framework's `GroupChatBuilder` or `SelectorGroupChat`.

Choose your language: **C#** or **Python**.

### B1: Project Setup

#### C\#

```bash
dotnet new console -n ParkProposalReview
cd ParkProposalReview
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Microsoft.AgentFramework --prerelease
dotnet add package Microsoft.AgentFramework.Orchestrations --prerelease
```

#### Python

```bash
mkdir park-proposal-review && cd park-proposal-review
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

### B3: Define the Three Participant Agents

Each agent represents a stakeholder perspective. Their instructions should produce focused, concise responses that engage with the conversation thread.

#### C\#

```csharp
using Microsoft.AgentFramework;

var environmentalAgent = new AIAgent(
    name: "EnvironmentalAgent",
    instructions: """
        You are an environmental consultant reviewing a city park development proposal.
        Your role is to evaluate ecological impact including: habitat disruption to
        existing wildlife, tree canopy preservation, stormwater management implications,
        wildlife corridor connectivity, and soil/water quality effects. You advocate for
        environmental protection but are willing to negotiate when effective mitigation
        strategies are proposed. When other agents raise concerns, respond constructively —
        propose mitigation approaches that address their needs while protecting ecological
        value. Keep responses focused and under 200 words.
        """,
    client: chatClient);

var communityAgent = new AIAgent(
    name: "CommunityAgent",
    instructions: """
        You are a community advocate reviewing a city park development proposal. Your
        role is to represent resident needs including: green space accessibility for all
        neighborhoods, recreational facilities for diverse age groups, walkability and
        transit connections, equity considerations for underserved communities, and
        neighborhood engagement in the design process. You champion community benefit
        but acknowledge budget and environmental constraints when raised by other agents.
        When other agents raise concerns, respond constructively — find ways to preserve
        community value while respecting their constraints. Keep responses focused and
        under 200 words.
        """,
    client: chatClient);

var budgetAgent = new AIAgent(
    name: "BudgetAgent",
    instructions: """
        You are a fiscal analyst reviewing a city park development proposal. Your role
        is to evaluate financial feasibility including: construction cost estimates,
        available funding sources (municipal bonds, grants, public-private partnerships),
        long-term maintenance projections, return on investment through property value
        and tourism impact, and phased implementation options to manage cash flow. You
        advocate for fiscal responsibility but recognize that some investments have
        long-term returns that justify upfront costs. When other agents raise concerns,
        respond constructively — propose funding strategies or phasing that addresses
        their priorities within fiscal constraints. Keep responses focused and under
        200 words.
        """,
    client: chatClient);
```

#### Python

```python
from agent_framework import AIAgent

environmental_agent = AIAgent(
    name="EnvironmentalAgent",
    instructions=(
        "You are an environmental consultant reviewing a city park development proposal. "
        "Your role is to evaluate ecological impact including: habitat disruption to "
        "existing wildlife, tree canopy preservation, stormwater management implications, "
        "wildlife corridor connectivity, and soil/water quality effects. You advocate for "
        "environmental protection but are willing to negotiate when effective mitigation "
        "strategies are proposed. When other agents raise concerns, respond constructively — "
        "propose mitigation approaches that address their needs while protecting ecological "
        "value. Keep responses focused and under 200 words."
    ),
    client=project,
)

community_agent = AIAgent(
    name="CommunityAgent",
    instructions=(
        "You are a community advocate reviewing a city park development proposal. Your "
        "role is to represent resident needs including: green space accessibility for all "
        "neighborhoods, recreational facilities for diverse age groups, walkability and "
        "transit connections, equity considerations for underserved communities, and "
        "neighborhood engagement in the design process. You champion community benefit "
        "but acknowledge budget and environmental constraints when raised by other agents. "
        "When other agents raise concerns, respond constructively — find ways to preserve "
        "community value while respecting their constraints. Keep responses focused and "
        "under 200 words."
    ),
    client=project,
)

budget_agent = AIAgent(
    name="BudgetAgent",
    instructions=(
        "You are a fiscal analyst reviewing a city park development proposal. Your role "
        "is to evaluate financial feasibility including: construction cost estimates, "
        "available funding sources (municipal bonds, grants, public-private partnerships), "
        "long-term maintenance projections, return on investment through property value "
        "and tourism impact, and phased implementation options to manage cash flow. You "
        "advocate for fiscal responsibility but recognize that some investments have "
        "long-term returns that justify upfront costs. When other agents raise concerns, "
        "respond constructively — propose funding strategies or phasing that addresses "
        "their priorities within fiscal constraints. Keep responses focused and under "
        "200 words."
    ),
    client=project,
)
```

### B4: Build the Group Chat

Wire the three agents into a group chat with an LLM-moderated Chat Manager that selects the next speaker based on conversation context.

#### Option 1 — GroupChatBuilder with LLM Selection (Recommended)

The `GroupChatBuilder` creates a group chat where a Chat Manager LLM decides which agent speaks next. This is the most flexible approach — the manager reads the conversation and picks the most relevant next speaker.

**C#:**

```csharp
using Microsoft.AgentFramework.Orchestrations;

var workflow = new GroupChatBuilder(
    participants: [environmentalAgent, communityAgent, budgetAgent])
    .WithSelectionStrategy(new LLMSelectionStrategy(
        client: chatClient,
        instructions: """
            You are a city council meeting moderator managing a discussion between
            EnvironmentalAgent, CommunityAgent, and BudgetAgent. Select the next
            speaker based on who has the most relevant response to the latest message.
            Ensure each agent speaks at least once before any agent speaks again.
            When positions are converging or repeating, output "DONE" to end.
            Respond with only the agent name or "DONE".
            """))
    .WithMaxTurns(12)
    .WithTerminationCondition(msg => msg.Content.Contains("DONE"))
    .Build();
```

**Python:**

```python
from agent_framework.orchestrations import GroupChatBuilder, LLMSelectionStrategy

workflow = (
    GroupChatBuilder(
        participants=[environmental_agent, community_agent, budget_agent])
    .with_selection_strategy(LLMSelectionStrategy(
        client=project,
        instructions=(
            "You are a city council meeting moderator managing a discussion between "
            "EnvironmentalAgent, CommunityAgent, and BudgetAgent. Select the next "
            "speaker based on who has the most relevant response to the latest message. "
            "Ensure each agent speaks at least once before any agent speaks again. "
            "When positions are converging or repeating, output 'DONE' to end. "
            "Respond with only the agent name or 'DONE'."
        )))
    .with_max_turns(12)
    .with_termination_condition(lambda msg: "DONE" in msg.content)
    .build()
)
```

#### Option 2 — Round-Robin for Comparison

A simpler alternative uses fixed turn order instead of LLM selection. This is less dynamic but predictable and cheaper (no LLM call per turn for speaker selection).

**C#:**

```csharp
using Microsoft.AgentFramework.Orchestrations;

var workflow = new GroupChatBuilder(
    participants: [environmentalAgent, communityAgent, budgetAgent])
    .WithSelectionStrategy(new RoundRobinSelectionStrategy())
    .WithMaxTurns(12)
    .Build();
```

**Python:**

```python
from agent_framework.orchestrations import GroupChatBuilder, RoundRobinSelectionStrategy

workflow = (
    GroupChatBuilder(
        participants=[environmental_agent, community_agent, budget_agent])
    .with_selection_strategy(RoundRobinSelectionStrategy())
    .with_max_turns(12)
    .build()
)
```

### B5: Run the Group Chat

Execute the workflow with the park development proposal.

#### C\#

```csharp
var proposal = """
    Review the following city park development proposal:

    Riverside Park Redevelopment — Phase 1
    Location: 12-acre site along Cedar Creek, adjacent to the Millbrook and Oakdale
    neighborhoods. The site includes 3 acres of mature riparian forest and a
    decommissioned water treatment facility.

    Proposed features: Community playground, splash pad, multi-use sports courts,
    1.2-mile walking trail along the creek, native plant restoration zone, community
    garden plots, and a small events pavilion.

    Estimated budget: $2.4M construction, $180K annual maintenance.
    Funding: $1.5M municipal bond allocation, seeking $900K in state green
    infrastructure grants.

    Timeline: 18-month construction, phased opening.

    The council needs a recommendation that balances environmental preservation,
    community benefit, and fiscal responsibility.
    """;

await foreach (var ev in workflow.RunStreamAsync(proposal))
{
    Console.WriteLine($"[Turn {ev.Turn}] [{ev.Source}] {ev.Content}");
    Console.WriteLine("---");
}
```

#### Python

```python
proposal = (
    "Review the following city park development proposal:\n\n"
    "Riverside Park Redevelopment — Phase 1\n"
    "Location: 12-acre site along Cedar Creek, adjacent to the Millbrook and Oakdale "
    "neighborhoods. The site includes 3 acres of mature riparian forest and a "
    "decommissioned water treatment facility.\n\n"
    "Proposed features: Community playground, splash pad, multi-use sports courts, "
    "1.2-mile walking trail along the creek, native plant restoration zone, community "
    "garden plots, and a small events pavilion.\n\n"
    "Estimated budget: $2.4M construction, $180K annual maintenance.\n"
    "Funding: $1.5M municipal bond allocation, seeking $900K in state green "
    "infrastructure grants.\n\n"
    "Timeline: 18-month construction, phased opening.\n\n"
    "The council needs a recommendation that balances environmental preservation, "
    "community benefit, and fiscal responsibility."
)

async for event in workflow.run_stream(proposal):
    print(f"[Turn {event.turn}] [{event.source}] {event.content}")
    print("---")
```

### B6: Verify the Output

Read through the full conversation thread and check for evidence of genuine collaboration:

* **Cross-referencing** — agents mention specific points from other agents' messages, not just restating their own position
* **Position evolution** — at least one agent adjusts their stance based on another agent's argument (e.g., Budget concedes ROI for environmental mitigation, Environmental proposes a compromise on trail routing)
* **Convergence** — the discussion moves toward a shared recommendation rather than three parallel monologues
* **Moderation quality** — the Chat Manager selected relevant speakers (if using LLM selection) and the conversation ended at an appropriate point

### B7: Track the Conversation Dynamics

Add instrumentation to understand the group chat's behavior.

#### C\#

```csharp
using System.Diagnostics;

var stopwatch = Stopwatch.StartNew();
var turnLog = new List<(int Turn, string Agent, int TokenCount, TimeSpan Elapsed)>();
int turnCount = 0;

await foreach (var ev in workflow.RunStreamAsync(proposal))
{
    turnCount++;
    turnLog.Add((turnCount, ev.Source, ev.TokenUsage?.TotalTokens ?? 0, stopwatch.Elapsed));
    Console.WriteLine($"[Turn {turnCount}] [{ev.Source}] {ev.Content}");
    Console.WriteLine("---");
}

Console.WriteLine($"\nTotal turns: {turnCount}");
Console.WriteLine($"Total time: {stopwatch.Elapsed:mm\\:ss\\.ff}");
Console.WriteLine($"Total tokens: {turnLog.Sum(t => t.TokenCount)}");
Console.WriteLine("\nSpeaker distribution:");
foreach (var group in turnLog.GroupBy(t => t.Agent))
{
    Console.WriteLine($"  {group.Key}: {group.Count()} turns, {group.Sum(t => t.TokenCount)} tokens");
}
```

#### Python

```python
import time
from collections import defaultdict

start = time.monotonic()
turn_count = 0
speaker_stats = defaultdict(lambda: {"turns": 0, "tokens": 0})

async for event in workflow.run_stream(proposal):
    turn_count += 1
    elapsed = time.monotonic() - start
    tokens = getattr(event, "token_usage", None)
    total_tokens = tokens.total_tokens if tokens else 0
    speaker_stats[event.source]["turns"] += 1
    speaker_stats[event.source]["tokens"] += total_tokens
    print(f"[Turn {turn_count}] [{event.source}] {event.content}")
    print("---")

total_time = time.monotonic() - start
print(f"\nTotal turns: {turn_count}")
print(f"Total time: {total_time:.2f}s")
print("\nSpeaker distribution:")
for agent, stats in speaker_stats.items():
    print(f"  {agent}: {stats['turns']} turns, {stats['tokens']} tokens")
```

Review the speaker distribution. In a well-moderated group chat, all three agents should have roughly equal turns, with mild variation based on relevance. If one agent dominates, the Chat Manager instructions may need tuning.

---

## Stretch Goals

Once the basic group chat is working, try one or more of these extensions:

### Implement a Maker-Checker Loop

Replace the three-agent roundtable with a two-agent Maker-Checker pattern — one of the most practical applications of group chat. Create a `ProposalDrafter` agent that writes a park development plan and a `ComplianceReviewer` agent that evaluates it against city regulations. The drafter revises based on the reviewer's feedback, and they iterate until the reviewer approves. Set a maximum of 6 turns to prevent infinite loops. This demonstrates iterative refinement as a focused variant of group chat.

### Compare LLM Selection vs Round-Robin

If you used the LLM selection strategy in B4, build a second version with round-robin and run both on the same proposal. Compare:

* **Conversation quality** — does LLM selection produce more coherent discussion, or does round-robin work well enough for three agents?
* **Token cost** — LLM selection requires an extra model call per turn to pick the speaker. How much overhead does this add?
* **Convergence speed** — does one approach reach consensus faster?

This exercise illustrates the Chat Manager design decision highlighted in the slides — it's the most important choice in group chat orchestration.

### Add a Human-in-the-Loop Participant

Group Chat naturally supports human participation. Add a step where, after the first round of agent discussion, the system pauses and prompts a human participant (you) to weigh in with a council member's perspective. Observe how the agents incorporate your input in subsequent turns. This demonstrates the "natural HITL support" benefit from the slides — a human joins the conversation as a peer, not through a separate approval gate.

### Test the Three-Agent Limit

Add a fourth agent — a `TransportationAgent` that evaluates traffic impact and transit connections. Then add a fifth — a `HistoricalPreservationAgent` for the decommissioned water treatment facility. Observe how conversation quality degrades as agent count increases. The slides warn that control becomes difficult with more than about three agents. Quantify this: measure turns-to-consensus and conversation coherence for 3, 4, and 5 agents on the same proposal.
