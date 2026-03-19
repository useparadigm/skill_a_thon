# Blast Radius — Full Cross-System Impact Analysis

You are performing a **cross-system blast radius analysis** for a code change. This is a three-part deliverable: a blast radius map, severity assessment, and action plan. All three sections are equally important.

## Input

The user will provide one of:
- A PR diff (inline or as a file path)
- A task description + repository name in `owner/name` format

If the input is a diff, extract: which methods changed, what fields were renamed/added/removed, what endpoints were modified, what schemas changed.

## Step 1: Get the Propagation Map from Paradigm

Call the Paradigm MCP tool `analyze_task` with:
- `repo`: the repository in `owner/name` format
- `description`: a summary of the changes (extract from the diff or use the user's description)

This returns a **propagation map** showing:
- Matched methods (the code being changed)
- Change propagation priorities (P1: override children, P2: direct callers, P3: cross-file callers, P4: transitive callers)
- Cross-file dependencies
- Override chains

If the MCP tool is not available, analyze the diff directly using your own reasoning. Note in the output: "Paradigm MCP unavailable — analysis based on diff inspection only. For deterministic cross-system analysis, configure Paradigm (see docs/paradigm-setup.md)."

## Step 2: Get Method Bodies

For each method in the propagation map (especially P1-P2 priority), call `get_method_body` with:
- `repo`: the repository in `owner/name` format
- `method_id`: the method ID from the propagation map (shown as `id: \`...\``)

Read the actual implementation to understand what each method does.

## Step 3: Detect Cross-System Patterns

Examine each method body for patterns that indicate cross-system connections:

**Remote Calls (HTTP/gRPC):**
- `requests.get()`, `requests.post()`, `httpx.post()`, `aiohttp`
- gRPC stubs: `stub.MethodName()`, `channel.unary_unary()`
- `fetch()`, `axios`, REST client calls

**Message Queues:**
- `KafkaProducer.send()`, `producer.produce()`
- `pika.basic_publish()`, RabbitMQ channels
- `celery.send_task()`, `.delay()`, `.apply_async()`
- SNS/SQS: `sns.publish()`, `sqs.send_message()`

**Database Access:**
- ORM: `.objects.filter()`, `.objects.create()`, `session.query()`, `session.add()`, `.save()`, `.delete()`
- Raw SQL: `SELECT`, `INSERT`, `UPDATE`, `DELETE` statements
- SQLAlchemy: `engine.execute()`, `text()`

**WebSocket / Streaming:**
- `websocket.send()`, `ws.emit()`, SSE endpoints

**Shared State:**
- Redis: `redis.get()`, `redis.set()`, `cache.get()`
- S3/GCS: `s3.put_object()`, `storage.blob()`

For each detection, record: the method, the pattern type, the target (topic name, URL route, table name), and your confidence level.

## Step 4: Generate Output

Structure your response as **three equally prominent sections**. Spend MORE tokens on Sections B and C than Section A — the map is evidence, but severity and actions are the value.

---

### Section A: Blast Radius Map

Present the raw findings:

**Direct Changes**
List every method modified in the PR with file path and line numbers.

**Propagation Map**
Show the P1-P4 propagation from Paradigm's analysis:
- P1 — Override children (must co-edit)
- P2 — Direct callers (likely affected)
- P3 — Cross-file callers (check impact)
- P4 — Transitive callers (review)

**Remote Calls Detected**
| Method | Sink Type | Target (route/topic) | Confidence |
|--------|-----------|---------------------|------------|
| ... | http/mq/grpc | ... | high/medium |

**Database Access**
| Method | Table | Read/Write | Source (ORM/raw SQL) |
|--------|-------|------------|---------------------|
| ... | ... | ... | ... |

**Cross-System Impact Summary**
List every external system that could be affected and how (Kafka consumer, HTTP client, shared DB reader).

---

### Section B: Severity Assessment

**THIS IS CRITICAL. Not a summary — a verdict with reasoning for each finding.**

For each cross-system connection found, classify:

**BREAKING** (red) — Will cause failures in production:
- Schema field renamed/removed that consumers deserialize directly
- API endpoint removed or response shape changed without versioning
- DB column renamed/dropped that other services query
- Message format changed without backward compatibility

**CAUTION** (yellow) — May cause issues, needs verification:
- New nullable column added (existing queries may not expect it)
- Behavioral change in a method called by other services
- New required field added to API request (backward compatible for responses, not requests)
- Performance change in a hot path

**SAFE** (green) — No cross-system impact:
- Internal refactors with unchanged interfaces
- Additive changes (new optional fields, new endpoints)
- Test/documentation changes

Format each finding like this:

```
SEVERITY | Connection Type + Target | What Changed
  Why: Explain the mechanism — HOW does this break the consumer?
  Evidence: What specific code/schema/contract is affected?
  Blast radius: What happens in production if this ships as-is?
```

Example:
```
BREAKING | Kafka topic `order.events` | Schema field renamed: amount → total_amount
  Why: Consumer in billing-service deserializes `amount` directly (no schema registry, no fallback).
  Evidence: billing/consumers/order_handler.py:47 — data["amount"]
  Blast radius: Billing pipeline stops processing orders. Revenue impact.
```

---

### Section C: Action Plan

**THE ACTUAL DELIVERABLE. Concrete, prioritized actions specific to this PR.**

Not generic advice. File paths, line numbers, code changes.

Format:

```
1. [BLOCK MERGE] <what to do>
   - File: <exact path:line>
   - Change: <specific code modification>
   - Alternative: <if applicable>

2. [BEFORE MERGE] <what to do>
   - ...

3. [AFTER MERGE] <what to monitor>
   - ...
```

Priority levels:
- **BLOCK MERGE** — Must be done before this PR can merge. Usually: update consumers first, or add backward compatibility.
- **BEFORE MERGE** — Should be done but can be a separate PR merged first.
- **AFTER MERGE** — Monitor or follow-up tasks.

**Suggest Deep Checks:**
For each BREAKING or CAUTION finding, suggest using `/deep-check` to verify:
```
Run `/deep-check` to verify:
- billing-service consumer for `order.events` topic schema change
- payment-service client for `/api/orders` response schema
```

---

## Formatting Notes

- Use tables for structured data, code blocks for examples
- Bold the severity labels and use consistent formatting
- If the propagation map is large, summarize lower-priority items (P3-P4)
- Always end with the deep check suggestions
