---
name: kafka-streaming-agents
description: This skill should be used when the user asks about Kafka, "Kafka consumer", "Kafka producer", "Kafka topic", "Flink", "streaming agents", "Schema Registry", "data contracts", "Confluent", "Confluent Cloud", "event-driven agents", "Kafka-backed orchestration", "real-time context", "streaming data quality", "schema evolution", "KSQL", "Kafka Connect", "Flink SQL", or discusses building agents on top of event streams.
version: 1.0.0
---

# Kafka Streaming Agents Skill

You are an expert in production Kafka and Flink streaming architectures, with a focus on agentic workflows built on event streams. Apply the following patterns whenever you design, implement, or advise on streaming agents, data contracts, or real-time platform tooling.

---

## 1. Why Kafka for Agent Orchestration

Kafka's immutable log gives agentic systems properties that in-memory message passing cannot:

- **Replayability**: replay agent inputs from any offset to reproduce failures
- **Auditability**: every inter-agent message is durably logged
- **Decoupling**: agents scale independently; the log is the contract
- **Backpressure**: consumer lag is a native health signal

The architectural principle: **stochastic agents inside, replayable/auditable control flow outside**. Route agent inputs/outputs through Kafka topics, not direct RPC calls.

```
[Trigger Event]
  → kafka: agent.input.v1 (partitioned by entity_id)
  → Agent Worker (LLM + tools)
  → kafka: agent.output.v1 (result artifact)
  → kafka: agent.audit.v1 (full trace + tool calls)
  → Downstream Consumer (human review / next agent)
```

---

## 2. Confluent Streaming Agents (Flink SQL)

Confluent GA'd Streaming Agents in Confluent Cloud for Apache Flink. Define agents and tools entirely in Flink SQL:

```sql
-- Register an MCP tool
CREATE TOOL search_runbook (
  query STRING COMMENT 'Search query for the runbook KB'
) RETURNS STRING
WITH (
  'connector' = 'mcp',
  'mcp.server.url' = 'https://internal-kb.corp/mcp',
  'mcp.tool.name'  = 'search'
);

-- Register an external data tool
CREATE TOOL get_table_schema (
  table_name STRING
) RETURNS STRING
WITH (
  'connector' = 'http',
  'url' = 'https://datahub.corp/api/schema/${table_name}'
);

-- Define the agent
CREATE AGENT data_quality_agent (
  INPUT  failure_event STRING,
  OUTPUT action_plan   STRING
) WITH (
  'model'         = 'claude-opus-4',
  'system.prompt' = 'You are a data quality triage agent. Classify failures as CODE or SOURCE. For CODE: propose a fix. For SOURCE: draft an incident summary.',
  'tools'         = 'search_runbook, get_table_schema',
  'max.iterations' = '5'
);

-- Wire agent into a streaming pipeline
SELECT
  event_id,
  AI_RUN_AGENT('data_quality_agent', failure_json) AS action_plan
FROM pipeline_failures
WHERE severity IN ('ERROR', 'CRITICAL');
```

For drafter-critic reflection:

```sql
CREATE AGENT critic_agent (
  INPUT  draft_fix STRING,
  OUTPUT review    STRING
) WITH (
  'model'         = 'claude-sonnet-4',
  'system.prompt' = 'Review the proposed fix for correctness, blast radius, and compliance with data contracts.'
);
```

---

## 3. Schema Registry Data Contracts

Encode quality rules and migration policies directly in the schema, not in application code:

```json
{
  "subject": "orders-value",
  "schemaType": "JSON",
  "schema": "...",
  "metadata": {
    "properties": {
      "owner": "data-platform",
      "sla.freshness.minutes": "15",
      "pii.fields": "customer_email,customer_name"
    }
  },
  "ruleSet": {
    "domainRules": [
      {
        "name": "validate_order_amount",
        "kind": "CONDITION",
        "type": "CEL",
        "mode": "WRITE",
        "expr": "message.amount > 0 && message.amount < 1000000",
        "onFailure": "DLQ"
      },
      {
        "name": "redact_pii",
        "kind": "TRANSFORM",
        "type": "CEL",
        "mode": "READ",
        "expr": "message.customer_email.redact()"
      }
    ],
    "migrationRules": [
      {
        "name": "rename_user_id",
        "kind": "TRANSFORM",
        "type": "JSONATA",
        "mode": "UPGRADE",
        "expr": "$ ~> |$|{'customer_id': user_id}|"
      }
    ]
  }
}
```

Schema evolution policy:
- `FORWARD`: new schema can read old data — safe for producers to upgrade first
- `BACKWARD`: old schema can read new data — safe for consumers to upgrade first
- `FULL`: both — the safest, required for critical topics
- Never break `FULL` compatibility on topics consumed by agents — agent tool call schemas are contracts

---

## 4. LLM-Recommended Test Rules (Grab Coban Pattern)

Use an LLM to recommend field-level semantic test rules from schema + anonymized samples:

```python
def recommend_quality_rules(schema: dict, sample_rows: list[dict]) -> list[Rule]:
    # Anonymize PII before sending to LLM (never send raw PII)
    anonymized = anonymize(sample_rows, schema["metadata"]["pii.fields"])

    prompt = f"""
    Schema: {json.dumps(schema, indent=2)}
    Sample rows (anonymized): {json.dumps(anonymized[:10], indent=2)}

    Recommend data quality rules as JSONata/CEL expressions.
    For each rule: field, expression, failure_action (WARN|DLQ|REJECT), rationale.
    Focus on: nullability, cardinality, value ranges, referential integrity.
    """

    rules = llm.complete(prompt)
    return [Rule.parse(r) for r in rules]

# Register approved rules into Schema Registry
def register_rules(subject: str, rules: list[Rule], approved_by: str):
    for rule in rules:
        registry.add_rule(subject, rule, metadata={"approved_by": approved_by})
```

Critical: **human approval before registering any rule** — a bad REJECT rule silently drops messages.

---

## 5. Replayable Orchestration Pattern

Structure multi-step agent workflows as a sequence of Kafka topics, not a single long-running agent:

```
Step 1: pipeline_failure_events (raw trigger)
  → Agent A: triage_agent
  → Step 2: triage_results (CODE | SOURCE | UNKNOWN + evidence)

Step 2 CODE path:
  → Agent B: fix_agent (sandbox only, read-only prod)
  → Step 3: fix_proposals (patch diff + test results)
  → Human review gate → approve_topic | reject_topic

Step 2 SOURCE path:
  → Agent C: incident_summary_agent
  → Step 3: incident_reports → Slack / PagerDuty
```

Each Kafka topic is a checkpoint. On failure: replay from the last successful topic. This gives you exactly-once semantics at the workflow level even when agent calls are not idempotent.

Consumer group naming: `agent.<pipeline>.<step>` — gives you per-step lag metrics in Kafka monitoring dashboards.

---

## 6. Real-Time Context Engine

Serve governed, fresh context to external agents (Claude Code, LangChain, Bedrock) over MCP:

```python
# Flink SQL: materialize fresh context into a queryable store
CREATE TABLE agent_context_store (
  entity_type STRING,
  entity_id   STRING,
  context_json STRING,   -- freshly aggregated from upstream topics
  updated_at  TIMESTAMP(3),
  PRIMARY KEY (entity_type, entity_id) NOT ENFORCED
) WITH (
  'connector' = 'upsert-kafka',
  'topic'     = 'agent.context.v1'
);

# Expose via MCP server
@mcp_tool("get_entity_context")
def get_entity_context(entity_type: str, entity_id: str) -> dict:
    return context_store.get(entity_type, entity_id)
```

Freshness SLA: define `max_staleness_seconds` per entity type and emit to Prometheus. Agents must check `context.updated_at` before acting on stale data.

---

## 7. Zero-Trust Agent Configuration

- Agents get **no shared credentials** — each agent container has its own scoped service account
- MCP tool calls are logged to the audit topic with full request/response (redact PII)
- Producer/consumer ACLs: agents get write to their output topic only, read from their input topic only
- DLQ every agent output topic — malformed or out-of-contract messages must not silently drop

```yaml
# Kafka ACL for triage-agent service account
- resource_type: TOPIC
  resource_name: agent.triage.output.v1
  operation: WRITE
  principal: User:triage-agent
- resource_type: TOPIC
  resource_name: pipeline_failure_events
  operation: READ
  principal: User:triage-agent
```

---

## 8. Common Anti-Patterns

| Anti-pattern | Correct approach |
|---|---|
| Agent state in memory across steps | Kafka topics as durable checkpoints |
| Shared credentials across agents | Per-agent scoped service accounts + ACLs |
| PII in LLM prompts | Anonymize before sending; redact in Schema Registry rules |
| LLM chooses schema migration strategy | Human approves; LLM recommends only |
| Auto-REJECT rules from LLM | Human gate before registering REJECT rules |
| Single giant agent topic | One topic per logical step; enables per-step replay |
| Kafka Connect without schema | Schema Registry enforced on all production connectors |

---

## Quick Reference

```bash
# Check consumer lag for an agent pipeline step
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --describe --group agent.triage.triage-agent

# Replay from a specific offset (rerun an agent step)
kafka-console-consumer.sh --bootstrap-server broker:9092 \
  --topic pipeline_failure_events \
  --offset 142857 --partition 0

# List Schema Registry subjects
curl -s http://schema-registry:8081/subjects | jq .

# Test a CEL rule locally before registering
cel-eval 'message.amount > 0 && message.amount < 1000000' \
  --input '{"amount": 150.00}'

# Check topic config (retention, replication)
kafka-configs.sh --bootstrap-server broker:9092 \
  --describe --entity-type topics --entity-name pipeline_failure_events
```
