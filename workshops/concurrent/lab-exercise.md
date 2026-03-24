# Concurrent Orchestration — Hands-On Lab

## Lab Overview

Build a **stock analysis system** using the concurrent orchestration pattern. Four specialized agents analyze the same stock in parallel, each providing an independent perspective. An aggregator combines their outputs into a unified recommendation:

```text
                        ┌→ [Fundamental Analysis] → Result 1 ─┐
Request → [Initiator]   ├→ [Technical Analysis]   → Result 2 ─┼→ [Aggregated Recommendation]
                        ├→ [Sentiment Analysis]   → Result 3 ─┤
                        └→ [ESG Analysis]         → Result 4 ─┘
```

You will choose one of two implementation tracks:

| Track | Technology | Language |
|-------|-----------|----------|
| **Track A** | Azure AI Foundry Workflow Agents | No code (portal + YAML) |
| **Track B** | Microsoft Agent Framework | C# or Python |

Both tracks build the same four-agent concurrent pipeline with aggregation. Pick the track that matches your team's skills and goals.

## Prerequisites

* An Azure subscription with access to Azure AI Foundry
* A deployed chat model (GPT-4o or equivalent) in your Foundry project
* **Track A only:** Access to the Azure AI Foundry portal
* **Track B only:** .NET 9+ SDK or Python 3.11+, and an IDE of your choice

## Scenario

Your investment advisory team currently produces stock reports by having analysts work one at a time — fundamental analysis first, then technical, then sentiment, then ESG. The serial process takes hours and creates a bottleneck. You are building a system where all four analyses run simultaneously and their results are merged into a combined recommendation:

| Agent | Specialization | Output |
|-------|---------------|--------|
| **Fundamental Analyst** | Financials, valuation metrics, earnings, balance sheet health | Valuation assessment with target price range |
| **Technical Analyst** | Price patterns, moving averages, volume indicators, support/resistance levels | Trend analysis with entry/exit signals |
| **Sentiment Analyst** | News coverage, social media sentiment, analyst ratings, insider activity | Sentiment score with key narrative drivers |
| **ESG Analyst** | Environmental impact, social responsibility, governance quality, regulatory risk | ESG rating with material risk factors |

The key insight: none of these agents need input from the others. They all analyze the same stock independently, making this an ideal fit for concurrent orchestration.

---

## Track A — Azure AI Foundry Workflow Agents

This track uses the Foundry portal's visual workflow designer to build the concurrent pipeline without writing code.

### A1: Create the Four Prompt Agents

In the Azure AI Foundry portal, create four **prompt agents** — one for each analysis specialization. Each agent uses the same deployed model but has distinct instructions.

**Fundamental Analyst Agent**

* Name: `fundamental-analyst`
* Instructions:

  ```text
  You are a fundamental stock analyst. Given a stock ticker and company context,
  produce a fundamental analysis covering: revenue and earnings trends, balance
  sheet health, valuation metrics (P/E, P/B, EV/EBITDA), competitive positioning,
  and growth catalysts. Conclude with a valuation assessment and 12-month target
  price range. Format your output as a structured report with clear section headings.
  ```

**Technical Analyst Agent**

* Name: `technical-analyst`
* Instructions:

  ```text
  You are a technical stock analyst. Given a stock ticker and recent price history
  context, produce a technical analysis covering: trend direction (short/medium/
  long-term), key support and resistance levels, moving average signals (50-day,
  200-day), volume analysis, and momentum indicators (RSI, MACD). Conclude with
  entry/exit signals and a technical outlook. Format your output as a structured
  report with clear section headings.
  ```

**Sentiment Analyst Agent**

* Name: `sentiment-analyst`
* Instructions:

  ```text
  You are a market sentiment analyst. Given a stock ticker and company context,
  produce a sentiment analysis covering: recent news coverage and tone, social
  media sentiment trends, sell-side analyst consensus and recent rating changes,
  institutional ownership shifts, and insider trading activity. Conclude with an
  overall sentiment score (bullish/neutral/bearish) and the key narrative drivers.
  Format your output as a structured report with clear section headings.
  ```

**ESG Analyst Agent**

* Name: `esg-analyst`
* Instructions:

  ```text
  You are an ESG (Environmental, Social, Governance) analyst. Given a stock ticker
  and company context, produce an ESG assessment covering: environmental impact and
  carbon commitments, social responsibility initiatives and labor practices,
  governance structure and board independence, regulatory exposure and compliance
  track record. Conclude with an overall ESG rating and material risk factors that
  could affect investment returns. Format your output as a structured report with
  clear section headings.
  ```

### A2: Create the Aggregator Agent

Create a fifth **prompt agent** that serves as the aggregator — it takes the four independent analyses and synthesizes them into a combined recommendation.

* Name: `recommendation-aggregator`
* Instructions:

  ```text
  You are a senior investment strategist. You will receive four independent analysis
  reports for the same stock: fundamental, technical, sentiment, and ESG. Synthesize
  these into a unified investment recommendation that includes:
  1. An overall rating (Strong Buy / Buy / Hold / Sell / Strong Sell)
  2. A confidence score based on how much the four analyses agree or conflict
  3. Key supporting points from each analysis
  4. Material conflicts between the analyses and how to interpret them
  5. A recommended position size and time horizon
  Weight the analyses appropriately — if fundamental and technical disagree, explain
  the divergence rather than averaging blindly.
  ```

### A3: Create the Workflow Agent

1. In the Foundry portal, create a new **Workflow Agent**.
2. Select the **Parallel** (or Fan-out/Fan-in) template.
3. Add the four analyst agents as concurrent steps: `fundamental-analyst`, `technical-analyst`, `sentiment-analyst`, and `esg-analyst`.
4. Add `recommendation-aggregator` as the fan-in step that receives all four outputs.
5. Configure the fan-in step to concatenate all agent outputs into a single input for the aggregator.

### A4: Test the Pipeline

Send this request to your workflow agent:

```text
Analyze MSFT (Microsoft Corporation) for a potential investment. The client is a
long-term institutional investor with a 3-5 year horizon, moderate risk tolerance,
and strong ESG requirements. Current position: none. Budget: $5M allocation.
Recent context: stock trading near all-time highs, AI investment cycle ongoing,
regulatory scrutiny in EU markets.
```

**Verify that:**

* All four analyst agents produced independent reports (check that no agent references another agent's analysis)
* The Fundamental Analyst covers valuation metrics and a target price range
* The Technical Analyst identifies trend direction and key price levels
* The Sentiment Analyst provides a sentiment score with narrative drivers
* The ESG Analyst flags material risk factors relevant to institutional requirements
* The Aggregator synthesizes all four into a coherent recommendation with a confidence score

### A5: Inspect the Pipeline Trace

In the Foundry portal, open the run trace for your workflow execution. Observe that the four analyst agents ran concurrently — their start times should overlap rather than being sequential. Compare the wall-clock time of the concurrent execution to the sum of individual agent durations. This difference is the latency benefit of the concurrent pattern.

---

## Track B — Microsoft Agent Framework

This track builds the same four-agent concurrent pipeline in code using the Agent Framework's `ConcurrentBuilder` or `WorkflowBuilder`.

Choose your language: **C#** or **Python**.

### B1: Project Setup

#### C\#

```bash
dotnet new console -n StockAnalysis
cd StockAnalysis
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Microsoft.AgentFramework --prerelease
dotnet add package Microsoft.AgentFramework.Orchestrations --prerelease
```

#### Python

```bash
mkdir stock-analysis && cd stock-analysis
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

### B3: Define the Five Agents

Create four analyst agents and one aggregator agent. Each analyst gets instructions matching its specialization. The aggregator synthesizes the parallel outputs.

#### C\#

```csharp
using Microsoft.AgentFramework;

var fundamentalAnalyst = new AIAgent(
    name: "FundamentalAnalyst",
    instructions: """
        You are a fundamental stock analyst. Given a stock ticker and company context,
        produce a fundamental analysis covering: revenue and earnings trends, balance
        sheet health, valuation metrics (P/E, P/B, EV/EBITDA), competitive positioning,
        and growth catalysts. Conclude with a valuation assessment and 12-month target
        price range. Format your output as a structured report with clear section headings.
        """,
    client: chatClient);

var technicalAnalyst = new AIAgent(
    name: "TechnicalAnalyst",
    instructions: """
        You are a technical stock analyst. Given a stock ticker and recent price history
        context, produce a technical analysis covering: trend direction (short/medium/
        long-term), key support and resistance levels, moving average signals (50-day,
        200-day), volume analysis, and momentum indicators (RSI, MACD). Conclude with
        entry/exit signals and a technical outlook. Format your output as a structured
        report with clear section headings.
        """,
    client: chatClient);

var sentimentAnalyst = new AIAgent(
    name: "SentimentAnalyst",
    instructions: """
        You are a market sentiment analyst. Given a stock ticker and company context,
        produce a sentiment analysis covering: recent news coverage and tone, social
        media sentiment trends, sell-side analyst consensus and recent rating changes,
        institutional ownership shifts, and insider trading activity. Conclude with an
        overall sentiment score (bullish/neutral/bearish) and the key narrative drivers.
        Format your output as a structured report with clear section headings.
        """,
    client: chatClient);

var esgAnalyst = new AIAgent(
    name: "ESGAnalyst",
    instructions: """
        You are an ESG (Environmental, Social, Governance) analyst. Given a stock ticker
        and company context, produce an ESG assessment covering: environmental impact and
        carbon commitments, social responsibility initiatives and labor practices,
        governance structure and board independence, regulatory exposure and compliance
        track record. Conclude with an overall ESG rating and material risk factors that
        could affect investment returns. Format your output as a structured report with
        clear section headings.
        """,
    client: chatClient);

var aggregator = new AIAgent(
    name: "RecommendationAggregator",
    instructions: """
        You are a senior investment strategist. You will receive four independent analysis
        reports for the same stock: fundamental, technical, sentiment, and ESG. Synthesize
        these into a unified investment recommendation that includes:
        1. An overall rating (Strong Buy / Buy / Hold / Sell / Strong Sell)
        2. A confidence score based on how much the four analyses agree or conflict
        3. Key supporting points from each analysis
        4. Material conflicts between the analyses and how to interpret them
        5. A recommended position size and time horizon
        Weight the analyses appropriately — if fundamental and technical disagree, explain
        the divergence rather than averaging blindly.
        """,
    client: chatClient);
```

#### Python

```python
from agent_framework import AIAgent

fundamental_analyst = AIAgent(
    name="FundamentalAnalyst",
    instructions=(
        "You are a fundamental stock analyst. Given a stock ticker and company context, "
        "produce a fundamental analysis covering: revenue and earnings trends, balance "
        "sheet health, valuation metrics (P/E, P/B, EV/EBITDA), competitive positioning, "
        "and growth catalysts. Conclude with a valuation assessment and 12-month target "
        "price range. Format your output as a structured report with clear section headings."
    ),
    client=project,
)

technical_analyst = AIAgent(
    name="TechnicalAnalyst",
    instructions=(
        "You are a technical stock analyst. Given a stock ticker and recent price history "
        "context, produce a technical analysis covering: trend direction (short/medium/"
        "long-term), key support and resistance levels, moving average signals (50-day, "
        "200-day), volume analysis, and momentum indicators (RSI, MACD). Conclude with "
        "entry/exit signals and a technical outlook. Format your output as a structured "
        "report with clear section headings."
    ),
    client=project,
)

sentiment_analyst = AIAgent(
    name="SentimentAnalyst",
    instructions=(
        "You are a market sentiment analyst. Given a stock ticker and company context, "
        "produce a sentiment analysis covering: recent news coverage and tone, social "
        "media sentiment trends, sell-side analyst consensus and recent rating changes, "
        "institutional ownership shifts, and insider trading activity. Conclude with an "
        "overall sentiment score (bullish/neutral/bearish) and the key narrative drivers. "
        "Format your output as a structured report with clear section headings."
    ),
    client=project,
)

esg_analyst = AIAgent(
    name="ESGAnalyst",
    instructions=(
        "You are an ESG (Environmental, Social, Governance) analyst. Given a stock ticker "
        "and company context, produce an ESG assessment covering: environmental impact and "
        "carbon commitments, social responsibility initiatives and labor practices, "
        "governance structure and board independence, regulatory exposure and compliance "
        "track record. Conclude with an overall ESG rating and material risk factors that "
        "could affect investment returns. Format your output as a structured report with "
        "clear section headings."
    ),
    client=project,
)

aggregator = AIAgent(
    name="RecommendationAggregator",
    instructions=(
        "You are a senior investment strategist. You will receive four independent analysis "
        "reports for the same stock: fundamental, technical, sentiment, and ESG. Synthesize "
        "these into a unified investment recommendation that includes: "
        "1. An overall rating (Strong Buy / Buy / Hold / Sell / Strong Sell) "
        "2. A confidence score based on how much the four analyses agree or conflict "
        "3. Key supporting points from each analysis "
        "4. Material conflicts between the analyses and how to interpret them "
        "5. A recommended position size and time horizon "
        "Weight the analyses appropriately — if fundamental and technical disagree, explain "
        "the divergence rather than averaging blindly."
    ),
    client=project,
)
```

### B4: Build the Concurrent Workflow

Wire the four analyst agents into a fan-out/fan-in pattern with the aggregator as the convergence point.

#### Option 1 — ConcurrentBuilder (Recommended)

The `ConcurrentBuilder` is the simplest way to create a concurrent pipeline. It fans out to all participants simultaneously and feeds their combined outputs to an aggregator.

**C#:**

```csharp
using Microsoft.AgentFramework.Orchestrations;

var workflow = new ConcurrentBuilder(
    participants: [fundamentalAnalyst, technicalAnalyst, sentimentAnalyst, esgAnalyst],
    aggregator: aggregator
).Build();
```

**Python:**

```python
from agent_framework.orchestrations import ConcurrentBuilder

workflow = ConcurrentBuilder(
    participants=[fundamental_analyst, technical_analyst, sentiment_analyst, esg_analyst],
    aggregator=aggregator,
).build()
```

#### Option 2 — WorkflowBuilder (For Learning)

The `WorkflowBuilder` gives you explicit control over the fan-out and fan-in edges. This helps you understand how concurrent orchestration works under the hood — the initiator sends the same input to all agents, and a barrier synchronizes their outputs before the aggregator runs.

**C#:**

```csharp
using Microsoft.AgentFramework;

var builder = new WorkflowBuilder(fundamentalAnalyst);

// Fan-out: initiator sends to all four analysts in parallel
builder.AddEdge(fundamentalAnalyst, aggregator);
builder.AddEdge(technicalAnalyst, aggregator);
builder.AddEdge(sentimentAnalyst, aggregator);
builder.AddEdge(esgAnalyst, aggregator);

// All four agents receive the same initial input
builder.AddFanOut(fundamentalAnalyst, technicalAnalyst, sentimentAnalyst, esgAnalyst);

// Barrier: aggregator waits for all four to complete
builder.AddBarrier(aggregator,
    waitFor: [fundamentalAnalyst, technicalAnalyst, sentimentAnalyst, esgAnalyst]);

var workflow = builder.Build();
```

**Python:**

```python
from agent_framework import WorkflowBuilder

builder = WorkflowBuilder(start_executor=fundamental_analyst)

# Fan-out: all four analysts receive the same input
builder.add_fan_out(fundamental_analyst, technical_analyst, sentiment_analyst, esg_analyst)

# Fan-in: all four feed into the aggregator
builder.add_edge(fundamental_analyst, aggregator)
builder.add_edge(technical_analyst, aggregator)
builder.add_edge(sentiment_analyst, aggregator)
builder.add_edge(esg_analyst, aggregator)

# Barrier: aggregator waits for all four to complete
builder.add_barrier(aggregator,
    wait_for=[fundamental_analyst, technical_analyst, sentiment_analyst, esg_analyst])

workflow = builder.build()
```

### B5: Run the Pipeline

Execute the workflow with a stock analysis request.

#### C\#

```csharp
var request = """
    Analyze MSFT (Microsoft Corporation) for a potential investment. The client is a
    long-term institutional investor with a 3-5 year horizon, moderate risk tolerance,
    and strong ESG requirements. Current position: none. Budget: $5M allocation.
    Recent context: stock trading near all-time highs, AI investment cycle ongoing,
    regulatory scrutiny in EU markets.
    """;

await foreach (var ev in workflow.RunStreamAsync(request))
{
    Console.WriteLine($"[{ev.Source}] {ev.Content}");
}
```

#### Python

```python
request = (
    "Analyze MSFT (Microsoft Corporation) for a potential investment. The client is a "
    "long-term institutional investor with a 3-5 year horizon, moderate risk tolerance, "
    "and strong ESG requirements. Current position: none. Budget: $5M allocation. "
    "Recent context: stock trading near all-time highs, AI investment cycle ongoing, "
    "regulatory scrutiny in EU markets."
)

async for event in workflow.run_stream(request):
    print(f"[{event.source}] {event.content}")
```

### B6: Verify the Output

Check that the pipeline produced a combined recommendation showing clear evidence of concurrent execution:

* **Independence** — each analyst report stands alone and does not reference another agent's analysis
* **Fundamental Analysis** — includes valuation metrics and a target price range
* **Technical Analysis** — identifies trend direction, support/resistance, and entry/exit signals
* **Sentiment Analysis** — provides a sentiment score with key narrative drivers
* **ESG Analysis** — flags material risk factors and provides an ESG rating
* **Aggregation** — the final recommendation synthesizes all four with a confidence score, handles any conflicts between analyses, and provides a position recommendation

### B7: Measure the Concurrency Benefit

Add timing instrumentation to quantify the latency advantage of concurrent execution.

#### C\#

```csharp
using System.Diagnostics;

var stopwatch = Stopwatch.StartNew();
var events = new List<(string Source, TimeSpan Timestamp)>();

await foreach (var ev in workflow.RunStreamAsync(request))
{
    events.Add((ev.Source, stopwatch.Elapsed));
    Console.WriteLine($"[{stopwatch.Elapsed:mm\\:ss\\.ff}] [{ev.Source}] {ev.Content[..Math.Min(80, ev.Content.Length)]}...");
}

Console.WriteLine($"\nTotal wall-clock time: {stopwatch.Elapsed:mm\\:ss\\.ff}");
```

#### Python

```python
import time

start = time.monotonic()

async for event in workflow.run_stream(request):
    elapsed = time.monotonic() - start
    preview = event.content[:80] if event.content else ""
    print(f"[{elapsed:.2f}s] [{event.source}] {preview}...")

total = time.monotonic() - start
print(f"\nTotal wall-clock time: {total:.2f}s")
```

Compare the wall-clock time to the sum of individual agent durations shown in the stream output. If four agents each take ~10 seconds sequentially (40 seconds total), the concurrent pipeline should complete in roughly ~10-12 seconds.

---

## Stretch Goals

Once the basic pipeline is working, try one or more of these extensions:

### Add Conflict Resolution Logic

The aggregator currently handles conflicts in its prompt, but you can make this explicit. Add logic that detects when analysts disagree (e.g., Fundamental says "Buy" but Technical says "Sell") and routes the conflicting analyses through a dedicated tiebreaker agent or applies a weighted voting scheme. This is the most important operational concern with concurrent orchestration.

### Swap Models Per Agent

Not every analyst needs the most capable model. Try assigning a smaller, faster model (e.g., GPT-4o-mini) to the Sentiment Analyst while keeping GPT-4o for the Fundamental Analyst. Compare the quality of each analysis and the overall recommendation. This addresses the "resource-intensive" trade-off from the slides — you can reduce cost without sacrificing quality where it matters most.

### Add a Fifth Analyst

Add a Macroeconomic Analyst agent that evaluates interest rate environment, sector rotation trends, and geopolitical risk. Observe how the concurrent pattern scales — adding a fifth parallel agent should add zero additional latency (assuming resource availability) while providing a richer recommendation. This is a key advantage over sequential: adding agents scales in breadth without scaling in time.

### Compare Sequential vs Concurrent

Build the same four-agent pipeline sequentially (use the `SequentialBuilder` or chain the agents in order). Run both versions on the same request and compare:

* **Latency** — how much faster is the concurrent version?
* **Output quality** — does the sequential version produce different results because later agents can see earlier analyses?
* **Cost** — token usage should be similar, but actual compute cost may differ

This exercise demonstrates the core trade-off: sequential allows agents to build on each other's reasoning, while concurrent provides independence and speed.
