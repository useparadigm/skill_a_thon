# Paradigm — Understand Your Code Across the Entire System

You have access to **Paradigm** via MCP — a code analysis platform that gives you (and your AI agents) a complete picture of your codebase architecture: call graphs, cross-file dependencies, remote calls, database access, and how it all connects.

The core problem: your tools only see one file, one repo, one language at a time. Paradigm sees the whole system — every call chain, every service boundary, every shared database table. It lets you understand the real architecture of your code, and it gives AI agents the same understanding so they stop making changes that look safe locally but break things across service boundaries.

## When to use this

- "Where is this method called from?" — across files, classes, repos
- "What happens if I change this function?" — propagation map shows everything affected
- "What does this service talk to?" — remote calls (HTTP, Kafka, gRPC), DB access
- "I need to work on [feature/bug/task]" — find all relevant code before you start

## Tools

### `analyze_task(repo, description)`

The main tool. Give it a repo and describe what you're working on — it returns a map of everything relevant.

**Arguments:**
- `repo` — GitHub repository in `owner/name` format
- `description` — what you're trying to do, the bug you're fixing, the feature you're building. Natural language. You can include method names, class names, or code references in backticks for more precise matching.

**What it returns:**

1. **Change Propagation Map** — the most valuable part. For each method that matches your task:
   - **P1 — Override children:** subclass implementations that MUST be co-edited
   - **P2 — Direct callers:** methods that call this one directly (likely affected)
   - **P3 — Cross-file callers:** callers from other files (check impact)
   - **P4 — Transitive callers:** indirect callers (review)

2. **Matched methods** — methods matching your task description, ranked by relevance. Each shows:
   - File path and line number
   - Method ID (use with `get_method_body`)
   - Who calls it, what it calls
   - Override relationships

3. **Cross-file dependencies** — which files depend on each other through call chains

4. **Class inheritance** — override maps, so you know if changing a base method affects subclasses

**Example call:**
```
analyze_task(repo="acme-corp/order-service", description="Kafka producer that publishes order events")
```

**Example call with code references:**
```
analyze_task(repo="acme-corp/order-service", description="Rename `amount` field in `OrderService.create_order`")
```

### `get_method_body(repo, method_id)`

After `analyze_task` gives you the map, use this to read the actual code of any method.

**Arguments:**
- `repo` — same `owner/name` format
- `method_id` — the ID from `analyze_task` output (shown as `id: \`...\``)

Use this to read the implementation of methods in the propagation map — especially P1 and P2 priority targets that are most affected by your change.

### `index_repo(repo, branch)`

Index a new repository. Call this once per repo — after that, `analyze_task` works on it.

### `status()`

Check connection and see which repos are indexed.

## How to use the output

When `analyze_task` returns results, here's the workflow:

1. **Read the propagation map first** — it tells you everything that's connected to the code you're about to touch. P1 items must be co-edited. P2 items are likely affected.

2. **Get method bodies for P1-P2 items** — call `get_method_body` to read the actual implementation of the most affected methods.

3. **Look for cross-system patterns** in the method bodies:
   - HTTP/gRPC calls (`requests.post`, `httpx`, gRPC stubs) — means another service depends on this
   - Message queues (`KafkaProducer.send`, `pika.basic_publish`, `celery.send_task`) — async consumers depend on the message schema
   - Database access (ORM queries, raw SQL) — other services may read the same tables
   - These are the connections your LSP can't see

4. **Use the map to plan your change** — now you know every file and method that needs updating, not just the ones in front of you.

## How it works (briefly)

Paradigm indexes your repository and builds a code graph in Neo4j:
- Parses every method, class, and file
- Maps all call relationships (who calls whom)
- Detects override chains (interface → implementation)
- Identifies remote calls (HTTP, gRPC, Kafka, MQ) with confidence scoring
- Maps database access patterns (ORM and raw SQL, read vs write, which tables)
- Creates embeddings for semantic search

When you call `analyze_task`, it searches this graph — keyword matching + semantic search — scores results by relevance and connectivity, then computes the propagation map from the top matches. The output is deterministic: based on the actual code graph, not guesses.

You can also explore the full architecture visually at [useparadigm.app](https://useparadigm.app) — browse call graphs, module structures, and cross-system connections in the web UI. Same underlying graph, visual interface for when you want to see the big picture yourself rather than query it through an agent.

## Setup

Add to your MCP configuration (`.mcp.json` in project root):

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

Sign up at [useparadigm.app](https://useparadigm.app), then index your repo:

```
> Use paradigm to index my repo: your-org/your-repo
```

After indexing (1-5 min), `analyze_task` is ready.

More details: [docs/paradigm-setup.md](../docs/paradigm-setup.md)
