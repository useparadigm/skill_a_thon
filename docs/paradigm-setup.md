# Paradigm MCP Setup

These skills use [Paradigm](https://useparadigm.app) via the Model Context Protocol (MCP) to get deterministic code analysis: call graphs, propagation maps, remote call detection, and database access patterns.

## 1. Add the MCP Server

Add this to your Claude Code MCP configuration (`.mcp.json` in your project root or global settings):

```json
{
  "mcpServers": {
    "paradigm": {
      "type": "http",
      "url": "https://api.useparadigm.app/mcp"
    }
  }
}
```

## 2. Authenticate

Sign up at [app.useparadigm.app](https://app.useparadigm.app). When you first use a Paradigm MCP tool, you'll be prompted to authenticate through your browser.

## 3. Index Your Repositories

Before you can analyze code, Paradigm needs to index the repository:

```
> Use the paradigm tool to index my repo: acme-corp/order-service
```

This calls `index_repo(repo="acme-corp/order-service")`. Indexing takes 1-5 minutes depending on repo size.

Check progress:

```
> Check the indexing status for job_id abc123
```

## 4. Verify It's Working

```
> Use the paradigm status tool
```

You should see your user info and a list of indexed repositories.

## Tools Used by These Skills

| Tool | What It Does | Used By |
|------|-------------|---------|
| `analyze_task(repo, description)` | Returns propagation map, call graph, cross-file dependencies | `/blast-radius`, `/impact-report`, `/deep-check` |
| `get_method_body(repo, method_id)` | Returns full source code of a method | `/blast-radius`, `/impact-report`, `/deep-check` |
| `status()` | Checks connection and lists indexed repos | Setup verification |
| `index_repo(repo, branch)` | Indexes a GitHub repo for analysis | Setup |
| `index_status(job_id)` | Checks indexing progress | Setup |

## What Paradigm Analyzes

When you index a repository, Paradigm builds a code graph that includes:

- **Call graphs** — who calls whom, across files and classes
- **Override chains** — interface/abstract method implementations
- **Remote call detection** — HTTP, gRPC, Kafka, RabbitMQ, Celery, SNS/SQS calls
- **Database access patterns** — ORM queries, raw SQL, table read/write classification
- **Cross-file dependencies** — which files depend on each other through call chains
- **Semantic search** — find code by natural language description

This gives the skills a deterministic foundation. Instead of guessing what code might be connected, Paradigm shows what IS connected.

## Without Paradigm

The skills work without Paradigm — they'll analyze the diff using Claude's reasoning alone. You'll see a note in the output: "Paradigm MCP unavailable — analysis based on diff inspection only."

The difference: Claude can reason about patterns it sees in the diff. Paradigm deterministically maps connections it has indexed across the entire codebase, including repos you're not currently looking at.

## Troubleshooting

**"No matching methods found"** — Make sure the repository is indexed. Run `status()` to check.

**Indexing stuck** — Check `index_status(job_id)`. If it's been more than 10 minutes, the repo may be very large. For repos >100k lines, indexing can take up to 15 minutes.

**Private repos** — When you index a private repo, Paradigm will prompt you to grant GitHub access. Follow the OAuth flow in your browser.
