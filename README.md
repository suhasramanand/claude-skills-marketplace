# Suhas Reddy — Claude Code Skills Marketplace

A personal marketplace of Claude Code plugins and skills for production agentic workflows, data platform engineering, streaming agents, SRE incident response, IaC, and Kubernetes.

## Install this marketplace

```bash
/plugin marketplace add suhasramanand-marketplace https://github.com/suhasramanand/claude-skills-marketplace
```

Or add it manually to `~/.claude/plugins/known_marketplaces.json`:

```json
{
  "suhasramanand-marketplace": {
    "source": {
      "source": "github",
      "repo": "suhasramanand/claude-skills-marketplace"
    }
  }
}
```

## Available Plugins

| Plugin | Category | Auto-activates when… |
|--------|----------|----------------------|
| [agentic-orchestration](./plugins/agentic-orchestration) | AI Agents | Setting up Claude Code, writing CLAUDE.md, designing sub-agents, hooks, MCP servers, or multi-agent systems |
| [dbt-data-platform](./plugins/dbt-data-platform) | Data Engineering | Writing dbt models, configuring dbt MCP, building self-healing pipelines, or text-to-SQL agents |
| [kafka-streaming-agents](./plugins/kafka-streaming-agents) | Data Engineering | Working with Kafka, Flink, Schema Registry, Confluent Streaming Agents, or event-driven orchestration |
| [sre-incident-response](./plugins/sre-incident-response) | SRE | Building incident-response agents, kagent CRDs, HolmesGPT/k8sgpt workflows, or on-call automation |
| [terraform-best-practices](./plugins/terraform-best-practices) | DevOps / IaC | Writing or reviewing `.tf` files, Terraform state, modules, or CI/CD pipelines |
| [kubernetes-best-practices](./plugins/kubernetes-best-practices) | DevOps / K8s | Writing or reviewing K8s manifests, Helm charts, RBAC, NetworkPolicy, or cluster config |

## Plugin Details

### agentic-orchestration
Production Claude Code agentic patterns drawn from Anthropic's internal engineering practice:
- **CLAUDE.md architecture**: what to include, what to cut, the Gotchas section
- **Sub-agent YAML frontmatter**: tool allowlists, delegation heuristics, context isolation
- **Hooks**: PostToolUse formatters (gofmt/ruff/terraform fmt), PreToolUse guards, HTTP audit hooks
- **MCP server config**: `.mcp.json` with env-var creds, scope rules, security considerations
- **Orchestrator-worker pattern**: planning, parallel sub-agents, condensed returns
- **Writer/Reviewer pattern**: fresh-context reviewer sub-agent, bias elimination
- **Git worktree parallelism**: parallel instances, `claude -p` batch loops

### dbt-data-platform
Production dbt and data agent patterns based on real systems (Hiflylabs self-healing dbt, LinkedIn SQL Bot):
- **dbt MCP setup**: local `uvx`, remote HTTP, PAT vs service token, read-only first
- **Model/test structure**: staging/intermediate/marts, `schema.yml`, `dbt-expectations`
- **Semantic Layer**: MetricFlow metrics, `meta.owner`, freshness SLAs
- **Self-healing pipeline agent**: webhook → containerize → sandbox → CODE/SOURCE classification → PR (never auto-merge)
- **Text-to-SQL architecture**: table re-ranker → field re-ranker → SQL planner → EXPLAIN validation
- **Slim CI**: `state:modified+`, column-level CI on dependents

### kafka-streaming-agents
Production Kafka + Flink streaming agent patterns (Confluent Streaming Agents, Grab Coban):
- **Confluent Streaming Agents**: `CREATE TOOL`, `CREATE AGENT`, `AI_RUN_AGENT` in Flink SQL
- **Schema Registry data contracts**: CEL/JSONata quality rules, migration rules, FORWARD/BACKWARD/FULL compatibility
- **LLM-recommended test rules**: anonymize → recommend → human-approve → register
- **Replayable orchestration**: Kafka topics as checkpoints, per-step replay, zero-trust agent credentials
- **Real-Time Context Engine**: Flink-materialized context store served over MCP

### sre-incident-response
Production SRE agent patterns (kagent CNCF, HolmesGPT, Pinterest Tricorder):
- **Tiered investigation**: k8sgpt fast scan → HolmesGPT ReAct loop → human gate → scoped remediation
- **kagent CRDs**: Agent/Tool/Session manifests, RBAC-scoped service accounts, human approval policy
- **Incident context assembly**: recent deploys, feature flags, config changes, topology, past post-mortems
- **OTel tracing**: per-tool-call spans, `gen_ai.*` semantic conventions, Prometheus SLO metrics
- **Blast-radius assessment**: service topology graph, criticality gating, dual-approval for HIGH/CRITICAL

## Plugin Structure

```
claude-skills-marketplace/
├── plugin.json                              # Marketplace registry
├── README.md
└── plugins/
    ├── agentic-orchestration/
    │   ├── .claude-plugin/plugin.json
    │   └── skills/agentic-orchestration/SKILL.md
    ├── dbt-data-platform/
    │   ├── .claude-plugin/plugin.json
    │   └── skills/dbt-data-platform/SKILL.md
    ├── kafka-streaming-agents/
    │   ├── .claude-plugin/plugin.json
    │   └── skills/kafka-streaming-agents/SKILL.md
    ├── sre-incident-response/
    │   ├── .claude-plugin/plugin.json
    │   └── skills/sre-incident-response/SKILL.md
    ├── terraform-best-practices/
    │   ├── .claude-plugin/plugin.json
    │   └── skills/terraform/SKILL.md
    └── kubernetes-best-practices/
        ├── .claude-plugin/plugin.json
        └── skills/kubernetes/SKILL.md
```

## Adding More Plugins

Each plugin lives under `plugins/<name>/` and follows the standard Claude Code plugin structure. Skills in `skills/<name>/SKILL.md` auto-activate based on their `description` trigger conditions — no slash command needed. Register new plugins in the root `plugin.json`.

## License

MIT
