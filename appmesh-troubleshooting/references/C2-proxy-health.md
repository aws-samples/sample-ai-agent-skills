---
title: "C2 — Envoy Proxy Health Issues"
description: "Diagnose Envoy proxy health and connectivity problems"
status: active
severity: HIGH
triggers:
  - "envoy unhealthy"
  - "proxy not ready"
  - "envoy crash"
  - "proxy connection refused"
  - "envoy restart"
owner: devops-agent
objective: "Identify why Envoy proxy is unhealthy and restore proxy functionality"
context: "Envoy proxy health depends on successful connection to the App Mesh control plane (EMS), correct configuration delivery, and sufficient resources. Unhealthy proxies cause traffic routing failures."
---

## Phase 1 — Triage

MUST:
- Check Envoy container status: `kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[?(@.name=="envoy")]}'`
- Check Envoy logs: `kubectl logs <pod-name> -c envoy -n <namespace> --tail=100`
- Check Envoy server info: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/server_info`
- Check Envoy cluster health: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/clusters | grep health_flags`
- Check Envoy ready status: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/ready`

SHOULD:
- Check Envoy stats for errors: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/stats | grep -E "(cx_connect_fail|upstream_rq_5xx)"`
- Verify Envoy can reach App Mesh control plane endpoint
- Check Envoy resource usage (CPU, memory)

MAY:
- Check Envoy hot restart count
- Review Envoy access logs for error patterns

## Phase 2 — Remediate

MUST:
- Fix Envoy container resource limits if OOM killed
- Ensure network connectivity to App Mesh control plane
- Verify Envoy image version is compatible with App Mesh

SHOULD:
- Increase Envoy memory limits for high-traffic services
- Configure Envoy readiness probe correctly
- Monitor Envoy restart count

MAY:
- Enable Envoy debug logging temporarily for investigation
- Check for Envoy configuration conflicts

## Common Issues

- symptoms: "Envoy container in CrashLoopBackOff"
  diagnosis: "Envoy cannot connect to App Mesh control plane or configuration is invalid."
  resolution: "Check Envoy logs for connection errors. Verify mesh name and virtual node annotations."

- symptoms: "Envoy ready endpoint returns not ready"
  diagnosis: "Envoy hasn't received configuration from App Mesh yet."
  resolution: "Check connectivity to App Mesh EMS endpoint. Verify IAM permissions."

- symptoms: "Envoy OOMKilled"
  diagnosis: "Envoy memory limit too low for the traffic volume."
  resolution: "Increase Envoy container memory limits."

## Output Format

```yaml
root_cause: "proxy_health_issue — <specific_cause>"
evidence:
  - type: envoy_status
    content: "<container status and ready state>"
  - type: envoy_logs
    content: "<error messages from Envoy>"
  - type: envoy_stats
    content: "<connection and error statistics>"
severity: HIGH
mitigation:
  immediate: "Fix resource limits, connectivity, or configuration"
  long_term: "Implement Envoy health monitoring and alerting"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix Envoy container resource limits | YELLOW | Container resource change |
| Ensure network connectivity to control plane | GREEN | Verification |
| Verify Envoy image version compatibility | GREEN | Version check |
| Increase Envoy memory limits | YELLOW | Resource allocation change |
| Configure Envoy readiness probe | GREEN | Health check configuration |
| Enable Envoy debug logging | YELLOW | Increases log volume |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Envoy stats**: Connection and error statistics — INTERNAL
- **Envoy logs**: Proxy error messages and debug info — INTERNAL
- **Container status**: Resource usage and restart counts — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER leave debug logging enabled in production long-term
- NEVER reduce Envoy resource limits below recommended minimums

## Phase 3 — Rollback

1. Revert Envoy container resource limits to previous values
2. Disable debug logging: `kubectl exec <pod> -c envoy -- curl -s -X POST http://localhost:9901/logging?level=info`
3. Restart affected pods to restore previous Envoy configuration
4. Verify Envoy health: `kubectl exec <pod> -c envoy -- curl -s http://localhost:9901/ready`
5. Monitor proxy health metrics for 15 minutes

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
