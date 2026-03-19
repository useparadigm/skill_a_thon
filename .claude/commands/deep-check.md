# Deep Check — Verify a Cross-System Finding

You are **verifying a specific cross-system concern** by examining the actual consumer/producer code. Use this after `/blast-radius` identifies a BREAKING or CAUTION finding that needs confirmation.

## Input

The user will provide:
1. **The concern**: what cross-system connection to verify (e.g., "billing-service consumer for Kafka topic `order.events` schema change")
2. **The consumer/producer repo**: repository name in `owner/name` format
3. **Context from blast-radius**: what changed in the upstream PR (e.g., field renamed from `amount` to `total_amount`)

## Step 1: Analyze the Consumer/Producer Repo

Call the Paradigm MCP tool `analyze_task` with:
- `repo`: the consumer/producer repository in `owner/name` format
- `description`: describe the relevant function to find (e.g., "Kafka consumer for order.events topic" or "HTTP client that calls /api/orders endpoint")

If the MCP tool is not available, ask the user to point you to the relevant file/function in the consumer repo.

## Step 2: Read the Actual Code

Call `get_method_body` for each relevant method found. Focus on:
- The consumer/handler function that processes the data
- Any deserialization logic
- Schema validation (if any)
- Fallback handling

## Step 3: Verify the Finding

Check specifically:

**For Kafka/MQ schema changes:**
- Does the consumer parse the changed field directly? (`data["field_name"]`, `event.field_name`)
- Is there schema validation? (Avro, Protobuf, JSON Schema, Pydantic)
- Are there fallbacks? (`data.get("new_name", data.get("old_name"))`)
- Is there a dead letter queue for failed messages?

**For HTTP API changes:**
- Does the client have strict schema validation? (Pydantic, dataclass, TypedDict)
- Are response fields accessed directly or through a flexible parser?
- Is there retry logic? Error handling for unexpected schemas?
- Is the API versioned? Does the client specify a version?

**For DB schema changes:**
- Is the consumer using ORM or raw SQL?
- ORM: will the model auto-adapt or does it have explicit column definitions?
- Raw SQL: does it `SELECT *` or specific columns?
- Are there migrations? Will they run before/after the upstream migration?

**For shared state (Redis/cache):**
- Does the consumer assume a specific data format?
- Is there TTL handling? Race conditions during deployment?

## Step 4: Classify

Based on your analysis, classify the finding:

### Confirmed
The concern is real. The consumer WILL break.
- Show the exact code that breaks and explain why
- Provide the specific fix needed

### False Positive
The concern does not apply.
- Show the code that handles the change safely (fallbacks, flexible parsing, schema evolution)
- Explain why it's safe

### Needs Manual Review
Cannot determine from code analysis alone.
- Explain what's ambiguous
- List specific questions for the team
- Suggest tests to run

## Output Format

```
## Deep Check: [brief description of concern]

**Verdict: CONFIRMED / FALSE POSITIVE / NEEDS MANUAL REVIEW**

### Consumer Code Analysis
- Repository: owner/name
- File: path/to/consumer.py:line
- Function: function_name

[Show the relevant code snippet]

### Finding
[Explain what you found — does it parse the changed field? Schema validation? Fallbacks?]

### Recommended Fix
[Specific code change with before/after, or explanation of why no fix needed]

### Suggested PR Comment
[A ready-to-paste comment for the PR that surfaced this concern]

### Migration Strategy (if applicable)
1. [Step-by-step deployment order]
2. [Who needs to merge what first]
3. [Verification steps]
```
