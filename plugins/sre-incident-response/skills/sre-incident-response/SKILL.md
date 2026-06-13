---
name: sre-incident-response
description: This skill should be used when the user asks about "incident response", "SRE agent", "on-call", "alert triage", "RCA", "root cause analysis", "MTTR", "HolmesGPT", "k8sgpt", "kagent", "CrashLoopBackOff", "OOMKilled", "PagerDuty", "runbook", "post-mortem", "blast radius", "change feed", "observability agent", "kubectl-ai", "incident automation", "auto-remediation", or discusses automating investigation workflows for production systems.
version: 1.0.0
---

# SRE Incident Response Agents Skill

You are an expert in production SRE agentic workflows. Apply the following patterns whenever you design, implement, or advise on incident investigation agents, automated triage, or observability tooling.

---

## 1. Tiered Investigation Architecture

The production-proven pattern: lightweight scanner first, investigation agent second, human gate before any remediation.

```
Tier 1 — Fast Scanner (k8sgpt, seconds)
  CrashLoopBackOff / ImagePullBackOff / Pending pods
  Outputs: structured finding + severity
  Action: auto-post to Slack; no write access

Tier 2 — Investigation Agent (HolmesGPT / custom ReAct, minutes)
  Full telemetry access: Prometheus, Grafana, Datadog, K8s API
  ReAct loop: observe → hypothesize → gather more evidence → conclude
  Outputs: ranked hypotheses + supporting evidence + confidence scores
  Action: post RCA draft to incident channel; draft runbook steps

Tier 3 — Human Review Gate
  SRE reviews RCA draft
  Approves one of: [acknowledge-and-monitor | apply-runbook | open-fix-PR | escalate]

Tier 4 — Remediation (if approved)
  Scoped, reversible actions only: restart deployment, scale up HPA, rollback image
  All actions logged to audit topic; changes tracked in git
```

Never skip the human gate for Tier 4 on services with blast radius > 1 team. The asymmetry: a false-positive restart that takes down payments costs far more than a delayed page.

---

## 2. kagent CRD Structure (CNCF Sandbox)

kagent is the Kubernetes-native agent runtime (accepted CNCF Sandbox May 2025). Agents, tools, and sessions are CRDs — GitOps-managed, RBAC-scoped, OTel-traced.

```yaml
# Agent CRD
apiVersion: kagent.dev/v1alpha1
kind: Agent
metadata:
  name: incident-investigator
  namespace: sre-agents
spec:
  description: >
    Investigates production alerts by querying Prometheus, Kubernetes API,
    and Grafana. Produces ranked hypotheses with evidence. Read-only.
  systemPrompt: |
    You are a senior SRE investigating a production incident. Your goal is to
    identify the root cause. You have read-only access to metrics, logs, and
    K8s resources. Always check: recent deployments, config changes, resource
    exhaustion, upstream dependencies. Rank hypotheses by confidence (0–1).
  modelConfig:
    apiKeySecretRef:
      name: anthropic-api-key
    model: claude-opus-4
  tools:
    - name: prometheus-query
    - name: kubernetes-get
    - name: grafana-dashboard
    - name: runbook-search
  memory:
    enabled: true
    vectorStore: pgvector
  humanApprovalPolicy:
    required: true
    approvalTool: slack-approval
---
# Tool CRD (read-only K8s access)
apiVersion: kagent.dev/v1alpha1
kind: Tool
metadata:
  name: kubernetes-get
  namespace: sre-agents
spec:
  type: McpServer
  mcpServer:
    toolName: get_resource
    serverRef:
      name: kubernetes-mcp
---
# RBAC for the agent's service account
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sre-agent-readonly
rules:
  - apiGroups: [""]
    resources: ["pods", "nodes", "events", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch"]
```

---

## 3. Incident Context Assembly

Feed the agent out-of-band context it cannot discover from metrics alone:

```python
def assemble_incident_context(alert: Alert) -> IncidentContext:
    return IncidentContext(
        alert=alert,
        # Recent changes — the most common root cause
        recent_deploys=get_deploys_last_2h(alert.service),
        feature_flags=get_flag_changes_last_2h(alert.service),
        config_changes=get_config_changes_last_2h(alert.service),
        # Topology
        upstream_deps=get_service_dependencies(alert.service),
        downstream_consumers=get_downstream_consumers(alert.service),
        # Historical signal
        past_incidents=search_post_mortems(alert.title, limit=3),
        runbook=fetch_runbook(alert.runbook_url),
        # On-call context
        oncall_engineer=get_current_oncall(alert.team),
        escalation_policy=get_escalation_policy(alert.team),
    )
```

Pinterest Tricorder pattern: generate pre-filtered dashboard links with time-range + service context baked in, rather than handing the raw metrics to the agent. Agents reason better over filtered signal than raw time-series.

```python
def generate_dashboard_links(alert: Alert, window: TimeRange) -> list[str]:
    return [
        f"https://grafana/d/{alert.service}-overview?from={window.start}&to={window.end}",
        f"https://grafana/d/error-rate?var-service={alert.service}&from={window.start}",
    ]
```

---

## 4. ReAct Investigation Loop

Structure the investigation as a bounded ReAct loop with hard iteration ceiling:

```python
MAX_ITERATIONS = 8

def investigate(context: IncidentContext) -> RCA:
    observations = []
    hypotheses = []

    for i in range(MAX_ITERATIONS):
        action = agent.decide_next_action(context, observations, hypotheses)

        if action.type == "QUERY":
            result = execute_tool(action.tool, action.params)  # read-only
            observations.append(Observation(tool=action.tool, result=result))

        elif action.type == "HYPOTHESIZE":
            hypotheses.append(Hypothesis(
                description=action.description,
                confidence=action.confidence,  # 0.0–1.0
                supporting_evidence=action.evidence,
                suggested_next_steps=action.next_steps,
            ))

        elif action.type == "CONCLUDE":
            return RCA(
                primary_hypothesis=max(hypotheses, key=lambda h: h.confidence),
                all_hypotheses=hypotheses,
                evidence=observations,
                suggested_actions=action.remediation_options,
            )

    return RCA(status="MAX_ITERATIONS_REACHED", hypotheses=hypotheses)
```

Parallel hypothesis testing (Cleric pattern): spawn sub-agents for independent hypotheses simultaneously rather than sequentially. Each sub-agent queries its own slice of telemetry and returns a confidence-weighted finding.

---

## 5. OTel Tracing for Agent Workflows

Every agent action must emit a span. This is non-negotiable for post-incident review.

```python
from opentelemetry import trace
tracer = trace.get_tracer("sre-agent")

def execute_tool(tool_name: str, params: dict) -> str:
    with tracer.start_as_current_span(f"agent.tool.{tool_name}") as span:
        span.set_attribute("agent.tool.name", tool_name)
        span.set_attribute("agent.tool.params", json.dumps(params))
        span.set_attribute("incident.id", current_incident_id())
        span.set_attribute("agent.iteration", current_iteration())
        result = tool_registry[tool_name].execute(params)
        span.set_attribute("agent.tool.result_length", len(result))
        return result
```

Required trace attributes per kagent / OTel semantic conventions:
- `gen_ai.system` = `"anthropic"`
- `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`
- `incident.id`, `incident.severity`, `incident.service`
- `agent.iteration`, `agent.tool.name`, `agent.hypothesis.confidence`

Export to Prometheus for SLO tracking:
```yaml
# Alert: agent investigation taking too long
- alert: IncidentAgentSLOBreach
  expr: histogram_quantile(0.95, rate(agent_investigation_duration_seconds_bucket[5m])) > 300
  labels:
    severity: warning
```

---

## 6. Blast-Radius Assessment

Before any remediation action, assess blast radius using the service topology graph:

```python
def assess_blast_radius(service: str, action: RemediationAction) -> BlastRadius:
    graph = load_service_graph()  # Neo4j or in-cluster topology
    affected = graph.downstream_services(service, max_hops=3)

    return BlastRadius(
        directly_affected=[service],
        indirectly_affected=affected,
        criticality=max(s.business_criticality for s in affected),
        shared_dependencies=[s for s in affected if s.is_shared_infra],
        estimated_user_impact=sum(s.daily_active_users for s in affected),
    )
```

Gate by criticality:
- `LOW`: agent can apply automatically after human ACK
- `MEDIUM`: requires explicit engineer approval via Slack/PagerDuty
- `HIGH`/`CRITICAL`: requires dual approval (engineer + on-call lead)

---

## 7. Runbook Integration

Agents are only as good as their runbooks. Structure runbooks for machine readability:

```markdown
# Runbook: High Error Rate — payments-service

## Trigger conditions
- error_rate_5m > 0.05 for payments-service
- p99_latency > 2000ms

## Automated checks (safe to run without approval)
1. `kubectl get pods -n payments -l app=payments-service`
2. `kubectl logs -n payments -l app=payments-service --since=10m | grep ERROR`
3. Query Prometheus: `rate(http_requests_total{status=~"5..",service="payments"}[5m])`

## Common causes + signals
| Cause | Signal | Remediation |
|-------|--------|-------------|
| DB connection exhaustion | `db_pool_size > 95%` | Restart oldest pod first |
| Downstream timeout (fraud-service) | `upstream_latency{target="fraud"} > 1s` | Circuit-break fraud calls |
| Memory leak after deploy | `container_memory_usage` monotonically increasing since deploy | Rollback image |

## Escalation
- If not resolved in 15m: page payments-team lead
- If DB involved: page DBA on-call
```

Feed the full runbook into the agent's context at investigation start. "Runbook-free" agents (Cleric's claim) may work for commodity patterns but fail on business-specific failure modes.

---

## 8. Common Anti-Patterns

| Anti-pattern | Correct approach |
|---|---|
| Auto-remediation without human gate | Every write action requires explicit approval |
| Agent with broad K8s RBAC | Read-only by default; write RBAC added per action type |
| No iteration ceiling | Hard `MAX_ITERATIONS` ceiling + convergence check |
| Missing OTel spans | Every tool call emits a span; required for post-incident review |
| Agent ignores change feed | Always check deploys/flags/config changes in the investigation window |
| One giant investigation agent | Tiered: fast scanner → investigation → human gate → remediation |
| Static dashboard links | Pre-filter with service+time context before passing to agent |

---

## Quick Reference

```bash
# k8sgpt quick scan
k8sgpt analyze --explain --backend anthropic

# HolmesGPT investigate a specific pod
kubectl holmes investigate pod my-pod -n production

# kagent: list active agent sessions
kubectl get agentsessions -n sre-agents

# kagent: check agent tool call history
kubectl describe agentsession <session-id> -n sre-agents

# View OTel spans for an incident
# (Jaeger UI — filter by incident.id tag)
curl "http://jaeger:16686/api/traces?service=sre-agent&tags={\"incident.id\":\"INC-1234\"}"

# Check blast radius before restarting a service
kubectl get endpoints -n production | grep payments
kubectl get hpa -n production | grep payments
```
