---
name: appmesh-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS App Mesh problems by
  analyzing mesh configurations, virtual nodes, virtual services, virtual
  routers, Envoy proxy injection, proxy health, proxy configuration, route
  configuration, traffic shifting, mTLS, certificate management, access
  logging, tracing, EKS integration, and controller issues using structured
  runbooks. Activate when: mesh creation or configuration failures, virtual
  node/service/router issues, Envoy proxy injection failures, proxy health
  check failures, proxy configuration errors, route configuration problems,
  traffic shifting not working, mTLS setup issues, certificate management
  errors, access logging problems, tracing configuration issues, EKS
  integration failures, App Mesh controller issues, or the user says
  something is wrong with App Mesh without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with App Mesh, ECS, EKS, EC2, IAM,
  ACM-PCA, CloudWatch, X-Ray, CloudTrail, and optionally Envoy admin
  API access for proxy diagnostics.
---

# AWS App Mesh Diagnostics

## When to use

Any AWS App Mesh investigation where the console alone is insufficient — mesh configuration analysis, virtual node/service/router debugging, Envoy proxy troubleshooting, routing issues, TLS/mTLS configuration, observability setup, or EKS integration problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws appmesh list-meshes
aws appmesh describe-mesh --mesh-name <mesh-name>
aws appmesh list-virtual-nodes --mesh-name <mesh-name>
aws appmesh list-virtual-services --mesh-name <mesh-name>
aws appmesh list-virtual-routers --mesh-name <mesh-name>
```

### Step 2 — Domain deep dive

```
aws appmesh describe-virtual-node --mesh-name <mesh-name> --virtual-node-name <node-name>
aws appmesh describe-virtual-service --mesh-name <mesh-name> --virtual-service-name <service-name>
aws appmesh describe-virtual-router --mesh-name <mesh-name> --virtual-router-name <router-name>
aws appmesh describe-route --mesh-name <mesh-name> --virtual-router-name <router-name> --route-name <route-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appmesh.amazonaws.com --max-results 20
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].name}' | tr ' ' '\n' | grep envoy
kubectl logs <pod-name> -c envoy -n <namespace> --tail=100
curl -s http://localhost:9901/clusters | head -50
curl -s http://localhost:9901/config_dump | python3 -m json.tool | head -100
```

Read `references/guardrails.md` before concluding on any App Mesh issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `appmesh list-meshes` | List all meshes |
| `appmesh describe-mesh` | Mesh details and status |
| `appmesh describe-virtual-node` | Virtual node configuration |
| `appmesh describe-virtual-service` | Virtual service provider details |
| `appmesh describe-virtual-router` | Router and listener config |
| `appmesh describe-route` | Route rules and weights |
| `appmesh list-virtual-gateways` | Virtual gateway configurations |
| `kubectl get pods` | Check Envoy sidecar injection |
| `kubectl logs -c envoy` | Envoy proxy logs |
| `Envoy admin API (:9901)` | Proxy clusters, config dump, stats |
| `cloudtrail lookup-events` | Recent App Mesh API events |
| `acm-pca list-certificate-authorities` | Private CA for mTLS |

## Gotchas: AWS App Mesh

- App Mesh is a control plane — Envoy is the data plane. App Mesh pushes configuration to Envoy sidecars. If Envoy isn't running or can't connect to the App Mesh control plane, traffic routing won't work regardless of mesh configuration.
- Virtual nodes represent actual compute (ECS tasks, EKS pods, EC2 instances). A virtual node must have a service discovery mechanism (DNS or Cloud Map) to be reachable.
- Virtual services are the abstraction layer — clients connect to virtual services, which route to virtual nodes via virtual routers. Misconfigured virtual service providers cause routing failures.
- Virtual routers define routing rules — routes match on path, headers, method, etc. Route priority matters: more specific routes should have higher priority.
- Envoy sidecar must be injected into every service in the mesh — missing sidecars mean traffic bypasses the mesh entirely. On EKS, the App Mesh controller handles injection; on ECS, the task definition must include the Envoy container.
- mTLS requires ACM Private CA certificates — App Mesh mTLS uses certificates from ACM-PCA, not public ACM certificates. Certificate rotation is automatic but CA configuration must be correct.
- Traffic shifting uses weighted routes — canary and blue/green deployments use route weights. Weights must sum correctly and both target virtual nodes must be healthy.
- Envoy admin API (port 9901) is essential for debugging — config_dump, clusters, and stats endpoints provide real-time proxy state. This is the most reliable source of truth for proxy configuration.
- Health checks in App Mesh are separate from ECS/EKS health checks — App Mesh health checks determine if a virtual node is healthy for routing. Container health checks determine if the container is running.
- Retry policies and timeouts are configured per-route — default timeouts may be too short for slow backends. Retry policies should use appropriate retry conditions.
- Cross-account mesh sharing requires RAM — meshes can be shared across accounts using AWS Resource Access Manager.
- App Mesh controller for EKS manages CRDs — the controller translates Kubernetes custom resources into App Mesh API calls. Controller issues affect all mesh operations on EKS.

## Anti-hallucination rules

1. Always cite specific mesh configurations, Envoy logs, or API responses as evidence. Never guess at mesh settings.
2. App Mesh is the control plane, Envoy is the data plane. Never conflate the two.
3. mTLS requires ACM-PCA certificates, NOT public ACM certificates. Never suggest using public certificates for mTLS.
4. Envoy sidecar must be present for mesh routing. Never assume traffic is routed through the mesh without verifying sidecar injection.
5. Virtual service providers must be correctly configured. Never assume a virtual service automatically routes to a virtual node.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Mesh | A1-A2 | Mesh creation, configuration |
| B — Virtual | B1-B3 | Virtual node, virtual service, virtual router |
| C — Envoy | C1-C3 | Proxy injection, proxy health, proxy config |
| D — Routing | D1-D2 | Route configuration, traffic shifting |
| E — TLS | E1-E2 | mTLS, certificate management |
| F — Observability | F1-F2 | Access logging, tracing |
| G — EKS | G1-G2 | EKS integration, controller issues |
| Z — Catch-All | Z1 | General troubleshooting |
