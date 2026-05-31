# AWS App Mesh Diagnostics Skill

Agent skill for investigating and troubleshooting AWS App Mesh problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS App Mesh when the console alone isn't enough — mesh configuration analysis, virtual node/service/router debugging, Envoy proxy troubleshooting, routing issues, TLS/mTLS configuration, observability setup, and EKS integration problems.

### Activate When

- Mesh creation or configuration failures
- Virtual node connectivity or discovery issues
- Virtual service routing not working
- Virtual router or route configuration problems
- Envoy sidecar injection failures
- Envoy proxy health check failures
- Envoy proxy configuration errors
- Route matching or priority issues
- Traffic shifting / canary deployment problems
- mTLS setup or handshake failures
- Certificate management or rotation issues
- Access logging not working
- Distributed tracing configuration problems
- EKS App Mesh integration failures
- App Mesh controller issues on EKS

---

## Skill Structure

```
appmesh-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-mesh-creation.md
    ├── A2-mesh-configuration.md
    ├── B1-virtual-node.md
    ├── B2-virtual-service.md
    ├── B3-virtual-router.md
    ├── C1-proxy-injection.md
    ├── C2-proxy-health.md
    ├── C3-proxy-config.md
    ├── D1-route-configuration.md
    ├── D2-traffic-shifting.md
    ├── E1-mtls.md
    ├── E2-certificate-management.md
    ├── F1-access-logging.md
    ├── F2-tracing.md
    ├── G1-eks-integration.md
    ├── G2-controller-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Mesh** | A1–A2 | Mesh creation, configuration |
| **B — Virtual** | B1–B3 | Virtual node, virtual service, virtual router |
| **C — Envoy** | C1–C3 | Proxy injection, proxy health, proxy config |
| **D — Routing** | D1–D2 | Route configuration, traffic shifting |
| **E — TLS** | E1–E2 | mTLS, certificate management |
| **F — Observability** | F1–F2 | Access logging, tracing |
| **G — EKS** | G1–G2 | EKS integration, controller issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering control plane vs data plane distinction, virtual node service discovery requirements, virtual service provider configuration, Envoy sidecar injection necessity, mTLS certificate requirements, route priority ordering, health check separation, retry/timeout configuration, cross-account mesh sharing, Envoy admin API usage, App Mesh controller CRD management, and traffic shifting weight requirements.

---

## Investigation Workflow

1. **Triage** — List meshes, virtual nodes/services/routers, check mesh status
2. **Deep Dive** — Describe specific resources, check route configurations, verify providers
3. **Detailed** — Check Envoy logs, admin API, CloudTrail events, EKS controller status

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `appmesh:*`, `ecs:DescribeTaskDefinition`, `eks:DescribeCluster`, `iam:GetRole`, `acm-pca:ListCertificateAuthorities`, `cloudtrail:LookupEvents`, `xray:GetTraceSummaries`
- kubectl access for EKS-based meshes
- Envoy admin API access (port 9901) for proxy diagnostics

---

## Usage Examples

```
# List all meshes
aws appmesh list-meshes

# Describe a virtual node
aws appmesh describe-virtual-node --mesh-name my-mesh --virtual-node-name my-node

# Check Envoy sidecar in EKS
kubectl get pods -n my-namespace -o jsonpath='{.items[*].spec.containers[*].name}'

# Check Envoy proxy config
kubectl exec <pod> -c envoy -- curl -s http://localhost:9901/config_dump

# Check route configuration
aws appmesh describe-route --mesh-name my-mesh --virtual-router-name my-router --route-name my-route
```

---

## License

MIT-0
