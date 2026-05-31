---
title: "F2 — App Mesh Tracing Issues"
description: "Diagnose distributed tracing configuration problems"
status: active
severity: LOW
triggers:
  - "tracing not working"
  - "X-Ray traces missing"
  - "no traces"
  - "tracing configuration"
owner: devops-agent
objective: "Identify and resolve distributed tracing configuration issues in App Mesh"
context: "App Mesh integrates with AWS X-Ray and other tracing backends via Envoy. Tracing provides request-level visibility across services. Configuration requires enabling tracing on the mesh and deploying X-Ray daemon or ADOT collector."
---

## Phase 1 — Triage

MUST:
- Check Envoy tracing config in config dump: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/config_dump | python3 -c "import sys,json; d=json.load(sys.stdin); [print(json.dumps(c,indent=2)) for c in d.get('configs',[]) if 'tracing' in str(c).lower()]"`
- Verify X-Ray daemon or ADOT collector is running: `kubectl get pods -n <namespace> | grep xray`
- Check X-Ray traces: `aws xray get-trace-summaries --start-time <start> --end-time <end>`
- Verify Envoy is generating trace headers

SHOULD:
- Check X-Ray daemon logs for errors
- Verify IAM permissions for X-Ray: `xray:PutTraceSegments`, `xray:PutTelemetryRecords`
- Check sampling rate configuration

MAY:
- Verify trace context propagation between services
- Check Envoy stats for tracing metrics

## Phase 2 — Remediate

MUST:
- Deploy X-Ray daemon or ADOT collector as sidecar or daemonset
- Grant X-Ray permissions to the task/pod IAM role
- Configure Envoy tracing via App Mesh virtual node or mesh-level settings

SHOULD:
- Set appropriate sampling rate (not 100% in production)
- Configure trace context propagation headers
- Verify end-to-end trace visibility across services

MAY:
- Use ADOT collector for multi-backend tracing (X-Ray + Jaeger)
- Implement custom trace annotations

## Common Issues

- symptoms: "No traces appearing in X-Ray console"
  diagnosis: "X-Ray daemon not deployed or IAM permissions missing."
  resolution: "Deploy X-Ray daemon and grant xray:PutTraceSegments permission."

- symptoms: "Traces show only one service, not the full chain"
  diagnosis: "Trace context headers not propagated between services."
  resolution: "Ensure applications propagate X-Amzn-Trace-Id header."

- symptoms: "Traces appear but with gaps"
  diagnosis: "Some services don't have Envoy sidecar or tracing not configured."
  resolution: "Ensure all services in the mesh have Envoy with tracing enabled."

## Output Format

```yaml
root_cause: "tracing_issue — <specific_cause>"
evidence:
  - type: tracing_config
    content: "<Envoy tracing configuration>"
  - type: xray_daemon
    content: "<X-Ray daemon status>"
  - type: traces
    content: "<trace summary>"
severity: LOW
mitigation:
  immediate: "Deploy tracing infrastructure and configure permissions"
  long_term: "Implement comprehensive observability across the mesh"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Deploy X-Ray daemon or ADOT collector | YELLOW | New sidecar/daemonset deployment |
| Grant X-Ray permissions | YELLOW | IAM policy change |
| Configure Envoy tracing | GREEN | Observability configuration |
| Set appropriate sampling rate | GREEN | Sampling configuration |
| Configure trace context propagation | GREEN | Header configuration |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Trace data**: Request paths, latencies, service dependencies — INTERNAL
- **X-Ray daemon config**: Tracing infrastructure details — INTERNAL
- **IAM permissions**: X-Ray access roles — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER set sampling rate to 100% in production (cost and performance impact)
- NEVER deploy tracing infrastructure without resource limits

## Phase 3 — Rollback

1. Remove X-Ray daemon or ADOT collector deployment
2. Revert Envoy tracing configuration on virtual nodes
3. Remove X-Ray IAM permissions if no longer needed
4. Restart affected pods to remove tracing sidecars
5. Verify application performance is not impacted after removal

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
