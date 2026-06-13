---
name: dbt-data-platform
description: This skill should be used when the user asks about dbt, "dbt model", "dbt test", "dbt MCP", "dbt lineage", "dbt Semantic Layer", "dbt run", "dbt build", "text-to-SQL", "self-healing pipeline", "data agent", "Lakehouse", "analytics engineering", "ELT", "data transformation", "data contracts", "dbt Cloud", "dbt Core", "sources.yml", "schema.yml", or discusses data modeling, pipeline failures, or data platform tooling.
version: 1.0.0
---

# dbt Data Platform & Agent Skill

You are an expert data platform engineer specializing in dbt, agentic data workflows, and Lakehouse architectures. Apply the following patterns whenever you write, review, or advise on dbt models, data agents, or platform tooling.

---

## 1. Project Structure

```
dbt_project/
├── dbt_project.yml
├── profiles.yml              # never commit — use env vars
├── models/
│   ├── staging/              # raw → typed, renamed, no business logic
│   │   └── stg_<source>__<entity>.sql
│   ├── intermediate/         # joins, pivots (optional layer)
│   │   └── int_<entity>__<verb>.sql
│   └── marts/                # business-facing, metric-ready
│       ├── core/
│       └── finance/
├── tests/
├── macros/
├── seeds/
└── analyses/
```

Rules:
- Staging models are 1:1 with source tables — rename columns, cast types, nothing else
- Marts are the contract with downstream consumers; change them like a public API
- Use `{{ ref() }}` everywhere — never hardcode schema.table paths
- Keep model files under 100 lines; extract complex logic into macros

---

## 2. dbt MCP Server Setup

The `dbt-labs/dbt-mcp` server exposes discovery, lineage, Semantic Layer queries, and CLI commands to Claude Code.

```bash
# Local setup (Personal Access Token required, not service token)
claude mcp add dbt -- uvx dbt-mcp \
  --dbt-host "$DBT_CLOUD_HOST" \
  --token "$DBT_PERSONAL_ACCESS_TOKEN" \
  --environment-id "$DBT_ENVIRONMENT_ID"
```

Commit `.mcp.json` with env-var references — never hardcode tokens:

```json
{
  "mcpServers": {
    "dbt": {
      "command": "uvx",
      "args": ["dbt-mcp"],
      "env": {
        "DBT_TOKEN": "${DBT_PERSONAL_ACCESS_TOKEN}",
        "DBT_HOST": "${DBT_CLOUD_HOST}",
        "DBT_ENVIRONMENT_ID": "${DBT_ENVIRONMENT_ID}"
      }
    }
  }
}
```

Start with read-only operations (`get_lineage`, `get_models`, `text_to_sql`). Gate `execute_sql` behind a sandbox environment, never prod. `text_to_sql` consumes dbt Wizard credits — track usage.

Available tools: `get_models`, `get_sources`, `get_lineage`, `get_column_lineage` (Fusion engine), `get_metrics`, `execute_sql`, `text_to_sql`, `run_dbt_command`.

---

## 3. Model & Test Best Practices

Every model needs tests in its `schema.yml`:

```yaml
models:
  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers')
              field: customer_id
      - name: status
        tests:
          - accepted_values:
              values: ['placed', 'shipped', 'completed', 'returned']
```

Go beyond built-ins for production data quality:
- Use `dbt-expectations` for statistical tests (row counts, distributions, regex)
- Add `store_failures: true` so failed test rows are queryable
- Tag slow tests (`+tags: ['slow']`) and exclude from `--select` in CI fast-path
- Use `warn_if: ">= 1"` and `error_if: ">= 100"` for graceful degradation

---

## 4. Semantic Layer & Metrics

Define metrics in `metrics/` using MetricFlow syntax — this is what `text_to_sql` and dbt Agents query:

```yaml
metrics:
  - name: order_revenue
    label: Order Revenue
    type: simple
    type_params:
      measure:
        name: revenue
    filter: |
      {{ Dimension('order__status') }} = 'completed'
    meta:
      owner: data-platform
      sla_freshness_hours: 4
```

Rules:
- One semantic model per mart model — don't reuse semantic models across business domains
- Encode grain in the measure name (`daily_active_users` not `dau`)
- Add `meta.owner` and `meta.sla_freshness_hours` to every metric — agents need these for RCA

---

## 5. Self-Healing Pipeline Agent (Hiflylabs Pattern)

Wire dbt Cloud webhooks to an autonomous triage-and-fix agent:

```
dbt Cloud job failure
  → webhook → fast receiver (classify: CODE vs SOURCE)
  → containerized job:
      1. Clone repo + download run artifacts (manifest.json, run_results.json)
      2. Pull KB (Notion/Confluence markdown) — precedence over model "creativity"
      3. Clone prod data → sandbox schema (BigQuery roles: prod read-only)
      4. Headless Claude Code + dbt MCP → diagnose in sandbox
  → CODE path:
      5. Fix model in sandbox, verify with dbt build --select <model>+
      6. Run column-level CI on dependents
      7. Open PR (model diff + lineage impact + test results) — NEVER auto-merge
  → SOURCE path:
      5. Document root cause
      6. Draft non-technical incident summary
      7. Create Jira/Linear ticket
```

Critical guardrails:
- Prod is read-only via IAM/BigQuery roles — the agent can never write to it
- KB takes precedence over model rewrites — fix upstream, not downstream
- PR opens for human review; auto-merge is never acceptable
- Fix only the failing model's DAG slice — don't touch unrelated models

---

## 6. Text-to-SQL Agent Architecture (LinkedIn SQL Bot Pattern)

Multi-agent pipeline for governed natural-language queries:

```
User query (Slack / internal tool)
  → Table Re-ranker Agent
      RAG over DataHub/dbt metadata + query logs + certified examples
      Top 7 of 20 candidate tables → pass forward
  → Field Re-ranker Agent
      Column prune: trim to fit token limit (LLM selects relevant columns)
  → SQL Planner Agent
      Step-by-step planning with chain-of-thought
      Output: SQL + explanation
  → Self-Validation Agent
      EXPLAIN-based correctness/perf check (no data egress)
      Statistical result checks: row counts, null rates, value distributions
  → Response + "Fix with AI" button
```

The key insight: accuracy lives in the **curated context layer**, not the model. A semantic model + DataHub lineage + certified query examples beats a larger model with raw schema dumps.

DoorDash-style deterministic guardrails (run outside the model):
```python
# Never expose data to the model for validation — use EXPLAIN
def validate_query(sql: str, engine: str) -> ValidationResult:
    explain = run_explain(sql, engine)          # structural correctness
    check_plan_cost(explain)                    # performance gate
    check_statistical_results(run_query(sql))  # row count, null rate drift
    return ValidationResult(passed=True)
```

---

## 7. CI/CD with dbt

Slim CI — only build and test the changed DAG slice:

```yaml
# .github/workflows/dbt-ci.yml
- name: dbt build changed models
  run: |
    dbt build \
      --select state:modified+ \
      --defer \
      --state ./prod-artifacts \
      --target ci
- name: Column-level CI on dependents
  run: |
    dbt test \
      --select state:modified+1 \
      --store-failures
```

Always:
- Download `manifest.json` from the prod environment before running slim CI
- Run `dbt compile` and check for Jinja errors before `dbt build`
- Post model diffs + lineage impact to the PR as a comment
- Gate merge on all tests green; treat `warn` threshold breaches as errors for marts

---

## 8. Common Anti-Patterns

| Anti-pattern | Correct approach |
|---|---|
| Hardcode `schema.table` in SQL | Always use `{{ ref() }}` / `{{ source() }}` |
| No tests on mart models | Every mart column needs `unique` + `not_null` at minimum |
| Agent writes directly to prod | Sandbox schema only; prod is read-only via roles |
| Auto-merge agent PRs | Human review gate — always |
| Semantic model reuse across domains | One semantic model per mart; owned by one team |
| `execute_sql` against prod | Sandbox or read-replica only |
| LLM validates results with data | Use EXPLAIN + statistical checks — no data egress to model |

---

## Quick Reference

```bash
# Run only changed models + their downstream dependents
dbt build --select state:modified+ --defer --state ./prod-artifacts

# Check lineage for a model
dbt ls --select +my_model+ --output json

# Compile without running (syntax check)
dbt compile --select my_model

# Run dbt MCP text-to-SQL
# (via Claude Code with dbt MCP wired)
"What was the total revenue by region last quarter?"

# Validate a source is fresh
dbt source freshness --select source:my_source

# Generate docs and open
dbt docs generate && dbt docs serve
```
