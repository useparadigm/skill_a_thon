# Impact Report — Blast Radius Map Only

You are generating a **blast radius map** for a code change. This is the lightweight version — raw findings without severity analysis or action plan. For the full pipeline, use `/blast-radius`.

## Input

The user will provide one of:
- A PR diff (inline or as a file path)
- A task description + repository name in `owner/name` format

## Step 1: Get the Propagation Map

Call the Paradigm MCP tool `analyze_task` with:
- `repo`: the repository in `owner/name` format
- `description`: a summary of the changes

If the MCP tool is not available, analyze the diff directly. Note: "Paradigm MCP unavailable — analysis based on diff inspection only."

## Step 2: Get Method Bodies

For each method in the propagation map (focus on P1-P2), call `get_method_body` to read the actual implementation.

## Step 3: Detect Cross-System Patterns

Scan method bodies for:

- **HTTP/gRPC calls**: `requests.get/post`, `httpx`, gRPC stubs, `fetch`, `axios`
- **Message queues**: `KafkaProducer`, `pika.basic_publish`, `celery.send_task`, SNS/SQS
- **Database access**: ORM ops (`.objects.filter`, `session.query`, `.save()`), raw SQL
- **WebSocket/streaming**: `websocket.send()`, SSE
- **Shared state**: Redis, S3/GCS, shared caches

## Output Format

### Direct Changes
List methods modified in the PR with file paths.

### Propagation Map
Show P1-P4 propagation:
- P1 — Override children (must co-edit)
- P2 — Direct callers (likely affected)
- P3 — Cross-file callers (check impact)
- P4 — Transitive callers (review)

### Remote Calls Detected
| Method | Sink Type | Target (route/topic) | Confidence |
|--------|-----------|---------------------|------------|

### Database Access
| Method | Table | Read/Write | Source |
|--------|-------|------------|--------|

### Cross-System Impact Summary
List every external system affected and how.

---

**Next step:** Run `/blast-radius` with this same input for severity analysis and action plan.
