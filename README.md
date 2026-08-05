# AI Marketing Strategy Manager

A multi-agent marketing assistant built with the **OpenAI Agents SDK**. A single triage agent (the "Marketing Manager") routes requests to specialist agents that handle market research, competitor analysis, campaign planning, content calendars, performance analytics, and budget optimisation — with a human-in-the-loop approval step for any spending decision above a configurable threshold.

## Why

Marketing teams juggle competitor watching, trend spotting, campaign planning, content drafting, and analytics review across scattered tools. This project automates that workflow end-to-end while keeping a human in control of anything that touches real budget.

**Objectives**
- Automate competitor and market-trend research
- Turn research into a structured, budgeted campaign plan
- Generate a content calendar aligned to the campaign
- Summarise campaign performance from analytics data
- Recommend concrete optimisations, with human approval required above a budget threshold
- Retain session memory so the team doesn't repeat context every turn

## Architecture

```
                         ┌───────────────────────────┐
                         │   Marketing Manager        │   <- Triage / entry point
                         │   (handoff router)         │
                         └─────────────┬──────────────┘
        ┌───────────────┬──────────────┼───────────────┬───────────────┐
        ▼                ▼             ▼                ▼               ▼
 Market Research   Competitor     Campaign Planner  Content       Optimisation
     Agent         Analysis Agent      Agent       Strategist       Advisor
                                                       Agent      (uses the 3 agents
                                                                   below AS TOOLS)
                                                                        │
                                            ┌───────────────────────────┼───────────────┐
                                            ▼                           ▼               ▼
                                     Market Research(tool)     Competitor(tool)   Analytics Agent
```

The **Marketing Manager** is the only agent the user talks to. It classifies the request and hands off to exactly one specialist via the SDK's `handoffs` mechanism. The **Optimisation Advisor** is itself a mini-manager: it calls the Market Research, Competitor Analysis, and Analytics agents *as tools* (`agent.as_tool(...)`) rather than handing off, so it can synthesise all three into one recommendation.

| Agent | Role |
|---|---|
| **Marketing Manager** (triage) | Understands the request and hands off to the right specialist |
| **Market Research Agent** | Looks up industry trends, growth rate, and hot channels |
| **Competitor Analysis Agent** | Looks up a named competitor's positioning, pricing, and channels |
| **Campaign Planner Agent** | Picks channels, splits the budget, and returns a structured `CampaignPlan` |
| **Content Strategist Agent** | Turns a campaign into a structured `ContentCalendar` |
| **Analytics Agent** | Reads campaign performance data and returns a structured `PerformanceReport` |
| **Optimisation Advisor** | Combines research + competitor + analytics into one `OptimizationRecommendation`, flagging when human approval is required |

## Features

- **Handoff-based routing** — the manager delegates the whole turn to a specialist agent
- **Agents-as-tools** — the Optimisation Advisor composes three other agents without a handoff
- **Structured outputs** — `CampaignPlan`, `ContentCalendar`, `PerformanceReport`, and `OptimizationRecommendation` are all Pydantic models, so downstream code gets typed, validated data instead of freeform text
- **Function tools** — `get_market_trends`, `get_competitor_profile`, `calculate_campaign_budget`, `get_campaign_performance`, `get_current_date`, `log_event`
- **Input guardrail** — a small classifier agent checks every incoming request is marketing-related and trips a tripwire (blocking the run) on off-topic input
- **Human-in-the-loop approval** — any proposed budget change beyond the brand's `approval_threshold` is flagged, and the run pauses for a yes/no confirmation before proceeding
- **Session memory** — `SQLiteSession` persists conversation history so later turns can refer back to earlier ones (e.g. "which campaign did we just plan?")
- **Brand context** — a `BrandContext` dataclass (brand name, industry, budget, audience, approval threshold) is threaded through every agent via the SDK's typed `context`

## Tech stack

- Python 3
- [`openai-agents`](https://github.com/openai/openai-agents-python) SDK
- `pydantic` for structured outputs
- Model: `gpt-4o-mini`
- Notebook environment: Google Colab (uses `google.colab.userdata` for API key storage, with a `getpass` fallback for other environments)

## Repo contents

```
AI_Marketing_Strategy_Manager.ipynb   # full implementation + walkthrough
```

## Setup

1. **Install dependencies**
   ```bash
   pip install -U openai-agents pydantic
   ```

2. **Set your OpenAI API key**
   ```bash
   export OPENAI_API_KEY="sk-..."
   ```
   (In Colab, the notebook first tries `google.colab.userdata`, then falls back to a `getpass` prompt.)

3. **Run the notebook** top to bottom. It will:
   - Define the mock data sources (`MARKET_TRENDS_DB`, `COMPETITOR_DB`, `CAMPAIGN_PERFORMANCE_DB`)
   - Build the function tools and Pydantic output schemas
   - Construct the six agents and wire up handoffs / agents-as-tools
   - Attach the relevance guardrail to the Marketing Manager
   - Open a `SQLiteSession` and walk through a demo conversation

## Usage

The notebook exposes a single async entry point:

```python
result = await run_manager("What are the current market trends for fitness apps?")
```

Every call goes through the Marketing Manager, which routes it to the right specialist and returns `result.final_output` (plain text or a structured Pydantic object, depending on the agent).

Example demo turns included in the notebook:

```python
await run_manager("What are the current market trends for fitness apps?")
await run_manager("How does FlexFit compare to us on pricing and channels?")
await run_manager("Plan a campaign called Autumn Reset using TikTok, Instagram and YouTube.")
await run_manager("Build a content calendar for the Autumn Reset campaign on TikTok and Instagram...")
await run_manager("How did the spring-launch campaign perform?")
await run_manager("Based on our market position, FlexFit, and the spring-launch results, what should we do next with our budget?")
await run_manager("Remind me which campaign we just planned and which channels it uses.")
await run_manager("Can you write me a poem about the ocean?")  # blocked by the relevance guardrail
```

When the Optimisation Advisor proposes a budget change above the configured `approval_threshold`, the notebook pauses and asks for a human `yes`/`no` before treating the change as applied:

```python
recommendation = opt_result.final_output
if recommendation.human_approval_required:
    request_human_approval(recommendation)
```

## Configuration

Brand settings live in a single `BrandContext` dataclass, shared across every agent:

```python
brand_context = BrandContext(
    brand_name="Nimbus Fitness",
    industry="fitness apps",
    monthly_budget=12000,
    target_audience="urban professionals aged 25-40",
    approval_threshold=5000.0,
)
```

Swap in your own brand, industry, budget, and audience to repurpose the whole system. The mock `MARKET_TRENDS_DB`, `COMPETITOR_DB`, and `CAMPAIGN_PERFORMANCE_DB` dictionaries can be replaced with real API calls (e.g. to a CRM, ad platform, or analytics warehouse) without changing any agent logic.

## Notes

- All data sources in this notebook (`MARKET_TRENDS_DB`, `COMPETITOR_DB`, `CAMPAIGN_PERFORMANCE_DB`) are **mocked** for demonstration purposes — swap them for real integrations before production use.
- The relevance guardrail and human-approval step are meant to be extended, not bypassed, if you adapt this for a real budget.

## License

Add a license of your choice (e.g. MIT) here.
