# Your PR looks clean. Tests pass. You just broke 3 services.

47 lines changed. A field renamed in the order model. API updated. Migration added. Tests green. Linter happy.

Ship it, right?

Here's what no tool showed you: that field flows through Kafka to the billing service. It's in the HTTP payload to the payment service. Three other services query that column directly. **None of them are in your repo. None of them are in your tests. None of them are in your AI code reviewer's context window.**

This is the problem nobody is solving.

## The invisible architecture

Every tool we have is excellent within a single repo:

- Your **LSP** resolves types and finds references — in one language, one project
- Your **linter** catches style and type issues — in the files it can see
- Your **AI code reviewer** reasons about the diff — within its context window
- Your **service mesh** shows the dependency — after it breaks in production

The connections between systems — shared database tables, implicit API consumers, cross-language message schemas, undocumented Kafka topics — are invisible until 2am.

And AI agents are making this worse. They ship code faster across more files per PR, with zero awareness of downstream impact on services they can't see.

Amazon learned this the hard way. In March 2026, they started a [90-day code safety reset](https://eweek.com/news/amazon-90-day-code-safety-reset/) across 335 Tier-1 systems after AI-assisted changes caused "high blast radius" incidents — including a 6-hour retail outage. The internal briefing called it "a trend of incidents" and noted that "best practices and safeguards are not yet fully established" for AI-assisted changes.

## What this does

Three [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/slash-commands) that turn Paradigm's cross-system code analysis into actionable code review. One pipeline: **map the blast radius, assess severity, tell you exactly what to do.**

| Skill | What it does |
|-------|-------------|
| `/paradigm` | Get started with Paradigm — understand what it sees, how to use it, setup |
| `/blast-radius` | Full pipeline — propagation map + severity assessment + action plan |
| `/impact-report` | Quick scan — just the blast radius map, no analysis |
| `/deep-check` | Verify one finding — read the actual consumer/producer code |

**Paradigm** maps your entire codebase: call graphs, override chains, remote calls (HTTP, gRPC, Kafka, RabbitMQ), database access patterns, cross-file dependencies. These skills turn that map into code review that catches cross-system breakage before you merge.

## Quick start

```bash
# Clone the repo
git clone https://github.com/useparadigm/skill_a_thon.git
cd skill_a_thon

# Open in Claude Code — the skills are auto-discovered
claude

# Run the full pipeline on the example PR
/blast-radius Review this PR: [paste diff or point to examples/sample-pr-diff.md]
```

The skills appear as slash commands automatically — Claude Code discovers anything in `.claude/commands/`. Zero install.

**With Paradigm MCP** (recommended): The skills call `analyze_task` and `get_method_body` for deterministic, graph-based analysis. See [Paradigm setup guide](docs/paradigm-setup.md).

**Without Paradigm**: The skills still work — Claude analyzes the diff directly using pattern matching. You'll see a note that Paradigm would provide deterministic analysis.

## Example walkthrough

Here's what happens when you run `/blast-radius` on a real-looking PR. The example: an e-commerce `order-service` renames `amount` → `total_amount`, updates the Kafka event, and changes the API response.

**Input:** [examples/sample-pr-diff.md](examples/sample-pr-diff.md) — a clean, 47-line PR. Tests pass. Linter happy.

### Section A: Blast Radius Map

Paradigm's `analyze_task` returns the propagation map. The skill detects cross-system connections:

- **Kafka:** `_publish_order_event` sends to topic `order.events` — billing-service consumes it
- **HTTP:** `_notify_payment_service` POSTs to payment-service — payload field renamed
- **Database:** `orders` table column renamed — analytics and reporting read it directly

### Section B: Severity Assessment

Each finding gets a verdict with reasoning:

```
BREAKING | Kafka topic `order.events` | Schema field renamed: amount → total_amount
  Why: Consumer in billing-service deserializes `amount` directly — no schema
       registry, no fallback. data["amount"] → KeyError on every message.
  Blast radius: Billing pipeline stops processing orders. Revenue impact.
       No DLQ configured — messages silently committed. Data loss.
```

Not just a label — the *mechanism* of failure and the *production consequence*.

### Section C: Action Plan

Concrete, prioritized, specific to this PR:

```
1. [BLOCK MERGE] Update billing-service consumer before merging
   - File: billing/consumers/order_handler.py:47
   - Change: data["amount"] → data.get("total_amount", data.get("amount"))
   - Or: add backward-compatible alias in producer

2. [BLOCK MERGE] Verify payment-service request validation
   - Run /deep-check on acme-corp/payment-service
   - If strict validation: update payment-service first

3. [BEFORE MERGE] Coordinate DB migration with downstream readers
   - Analytics, reporting, ETL all query `orders.amount` directly
   - Consider two-phase migration: add column → migrate consumers → drop old
```

Not "be careful with this change" — **"do this, then merge."**

Full example output: [examples/blast-radius-report.md](examples/blast-radius-report.md)

## How it works

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│   PR Diff    │────>│  Paradigm MCP    │────>│   Claude Reasoning  │
│  (input)     │     │  analyze_task()  │     │   (the skills)      │
└─────────────┘     │  get_method_body()│     │                     │
                    └──────────────────┘     └─────────────────────┘
                            │                         │
                    ┌───────▼──────────┐     ┌───────▼─────────────┐
                    │  Propagation Map │     │  Section A: Map      │
                    │  Call graph       │     │  Section B: Severity │
                    │  Remote calls     │     │  Section C: Actions  │
                    │  DB access        │     │                     │
                    └──────────────────┘     └─────────────────────┘
                      deterministic              reasoning
                      (what's connected)         (so what? now what?)
```

**Paradigm** provides the deterministic analysis: which methods are connected, through what mechanism (calls, Kafka, HTTP, shared DB), with what confidence. This is computed from the actual code graph — not guessed from patterns.

**Claude** provides the reasoning: given these connections, what breaks? How severe is it? What specific actions should the team take? This is where the domain knowledge and contextual judgment happen.

The combination is the point. Paradigm alone gives you a graph. Claude alone gives you guesses. Together: **actionable cross-system code review.**

## About Paradigm

[Paradigm](https://useparadigm.app) is a code analysis platform that maps your entire codebase — call graphs, cross-system connections, database access, message queues — so your tools can see what's really connected.

- **Web:** [useparadigm.app](https://useparadigm.app)
- **GitHub:** [github.com/useparadigm](https://github.com/useparadigm)

---

Built for [Skill-a-thon HackNight](https://lu.ma/hackbarna) by [HackBarna](https://hackbarna.com) — Barcelona, March 19, 2026.
