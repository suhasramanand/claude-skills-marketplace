---
name: kubernetes
description: This skill should be used when the user asks about Kubernetes, "write a manifest", "review my K8s", "kubectl", "Helm chart", "pod security", "RBAC", "NetworkPolicy", "Deployment", "StatefulSet", "HPA", "resource limits", "liveness probe", "readiness probe", "ingress", "service mesh", or discusses container orchestration. Applies production Kubernetes best practices when generating or reviewing manifests and cluster config.
version: 1.0.0
---

# Kubernetes Best Practices Skill

You are an expert Kubernetes engineer. Apply the following best practices whenever you write, review, or advise on Kubernetes manifests, Helm charts, or cluster configuration.

---

## 1. Workload Design

- Always set `resources.requests` and `resources.limits` on every container — missing limits causes noisy-neighbor problems and OOMKills.
- Use `Deployment` for stateless workloads, `StatefulSet` for stateful ones (databases, message queues). Never run a database as a `Deployment`.
- Set `replicas >= 2` for any production workload. Use a `PodDisruptionBudget` to guarantee availability during node drains.
- Spread replicas across nodes and zones with `topologySpreadConstraints` or `podAntiAffinity`.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
```

---

## 2. Pod Security (Pod Security Standards)

Use Pod Security Standards (PSS) — `PodSecurityPolicy` is removed since K8s 1.25. Apply the `restricted` profile in production.

Label namespaces to enforce the standard:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

Every container must have a hardened `securityContext`:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

- Never use `privileged: true` or `hostNetwork: true` in production.
- Mount only the volumes the container actually needs; use `emptyDir` for scratch space rather than `hostPath`.

---

## 3. RBAC — Least Privilege

- Create a dedicated `ServiceAccount` per workload. Never use the `default` service account.
- Set `automountServiceAccountToken: false` on the `ServiceAccount` and opt-in per pod when actually needed.
- Use `Role` + `RoleBinding` (namespace-scoped) over `ClusterRole` + `ClusterRoleBinding` wherever possible.
- Grant the minimum verbs needed (`get`, `list` not `*`). Audit with `kubectl auth can-i --list`.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-reader
  namespace: production
subjects:
  - kind: ServiceAccount
    name: my-app
roleRef:
  kind: Role
  name: my-app-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 4. Network Policies — Default Deny

- Start with a default-deny-all policy per namespace, then explicitly allow only required traffic.
- Restrict both ingress and egress; most workloads only need egress to a few services and DNS.

```yaml
# Default deny all ingress and egress in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
# Allow DNS egress (required for service discovery)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - ports:
        - protocol: UDP
          port: 53
```

- Use a service mesh (Istio, Cilium, Linkerd) for mTLS, traffic shaping, and L7 policies beyond what `NetworkPolicy` supports.

---

## 5. Health Probes

Every production container must define all three probes:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

- `livenessProbe` — restarts the container if the app is deadlocked.
- `readinessProbe` — removes the pod from Service endpoints if not ready; prevents traffic to warming pods.
- `startupProbe` — buys slow-starting apps time before liveness kicks in.
- Never make a liveness probe call downstream dependencies — it causes cascading restarts.

---

## 6. Resource Autoscaling

```yaml
# HPA — scale on CPU and custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

- Use **HPA** for stateless workloads, **VPA** (in "Off" or "Initial" mode) to right-size requests, and **Cluster Autoscaler** / **Karpenter** to add/remove nodes.
- Do not run HPA and VPA in "Auto" mode on the same deployment simultaneously — use VPA recommendations to set requests manually.

---

## 7. Secrets Management

- Never store plaintext secrets in manifests or ConfigMaps.
- Use **External Secrets Operator** to sync secrets from AWS Secrets Manager, GCP Secret Manager, or HashiCorp Vault into Kubernetes Secrets.
- Enable **encryption at rest** for the `etcd` secrets store (KMS provider).
- Rotate secrets without pod restarts by mounting them as volumes (K8s auto-refreshes volume-mounted secrets).

```yaml
# External Secrets Operator example
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: prod/db
        property: password
```

---

## 8. Image Security

- Use specific image digests (`image: nginx@sha256:…`) in production, not `:latest` or mutable tags.
- Scan images in CI with **Trivy**, **Grype**, or **Snyk** — block on CRITICAL/HIGH CVEs.
- Sign images with **Cosign** (Sigstore) and enforce signature verification with an admission controller (Kyverno, Connaisseur).
- Use minimal base images (`distroless`, `alpine`) to shrink attack surface.

```yaml
# Pin by digest, not tag
image: gcr.io/distroless/nodejs20-debian12@sha256:abc123...
```

---

## 9. Observability

- Export metrics with **Prometheus** (or the OpenTelemetry Collector). Every service must expose a `/metrics` endpoint.
- Use **structured JSON logs** — never mix log formats in a cluster.
- Implement distributed tracing with **OpenTelemetry** SDKs; export to Jaeger, Tempo, or a managed backend.
- Deploy **Falco** for runtime threat detection (eBPF mode in K8s 1.28+).
- Alert on: pod OOMKilled, CrashLoopBackOff, HPA at max replicas, PVC near capacity, certificate expiry.

---

## 10. Cluster Hardening

- Run CIS Kubernetes Benchmark with **kube-bench** and resolve all FAIL/WARN items.
- Disable anonymous auth on the API server (`--anonymous-auth=false`).
- Enable audit logging on the API server; ship logs to a SIEM.
- Upgrade clusters within 3 minor versions of the latest stable release. Use managed K8s (EKS, GKE, AKS) for automated patch management.
- Use **Kyverno** or **OPA/Gatekeeper** to enforce policy as code (e.g., require labels, block `latest` tags, enforce resource limits).

```yaml
# Kyverno policy: require resource limits on all pods
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-container-limits
      match:
        resources:
          kinds: ["Pod"]
      validate:
        message: "Resource limits are required."
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

---

## 11. Common Anti-Patterns to Avoid

| Anti-pattern | Correct approach |
|---|---|
| `image: myapp:latest` | Pin to digest or immutable tag |
| No `resources` set | Always set requests and limits |
| `privileged: true` | Drop all capabilities, run as non-root |
| Secrets in `ConfigMap` or env vars | External Secrets Operator + volume mounts |
| Single replica in production | `replicas >= 2` + PodDisruptionBudget |
| No `NetworkPolicy` | Default-deny + explicit allow rules |
| `default` ServiceAccount | Dedicated SA per workload, no auto-mount |
| No health probes | liveness + readiness + startup probes |
| `ClusterRoleBinding` for all | Prefer namespace-scoped `Role`/`RoleBinding` |
| Manual `kubectl apply` in prod | GitOps with ArgoCD or Flux |

---

## Quick Reference

```bash
# Validate manifests before applying
kubectl apply --dry-run=server -f manifest.yaml

# Check RBAC permissions
kubectl auth can-i --list --namespace=production --as=system:serviceaccount:production:my-app

# Run CIS benchmark
kube-bench run --targets node,master

# Scan image for CVEs
trivy image --severity HIGH,CRITICAL myapp:1.2.3

# Check resource usage
kubectl top pods -n production --sort-by=memory

# Audit pod security
kubectl label --dry-run=server ns production \
  pod-security.kubernetes.io/enforce=restricted
```
