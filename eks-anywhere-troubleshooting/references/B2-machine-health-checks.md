---
title: "B2 — Machine Health Checks"
description: "Diagnose machine health check failures and automatic remediation"
status: active
severity: HIGH
triggers: ["machine unhealthy", "MachineHealthCheck triggered", "node auto-remediation", "machine replaced unexpectedly"]
owner: devops-agent
objective: "Resolve machine health check issues including false positives and remediation loops"
context: "MachineHealthChecks (MHC) monitor node conditions and automatically replace unhealthy machines. Misconfigured MHCs can cause unnecessary node replacements or fail to detect real issues."
---

## Phase 1 — Triage

MUST:
- List MachineHealthChecks: `kubectl get machinehealthchecks -A --kubeconfig <mgmt-kubeconfig>`
- Check MHC status and targets: `kubectl get machinehealthchecks -A -o yaml --kubeconfig <mgmt-kubeconfig>`
- Review machine conditions: `kubectl get machines -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[*].type}{"\t"}{.status.conditions[*].status}{"\n"}{end}'`

SHOULD:
- Check node conditions on the workload cluster: `kubectl get nodes -o wide --kubeconfig <workload-kubeconfig>`
- Review recent machine replacement events: `kubectl get events -A --field-selector reason=RemediationRestricted --kubeconfig <mgmt-kubeconfig>`

MAY:
- Check kubelet logs on unhealthy nodes for root cause
- Review node resource utilization (CPU, memory, disk pressure)

## Phase 2 — Remediate

MUST:
- Fix underlying node health issues (kubelet crashes, disk pressure, network unreachable)
- Adjust MHC timeout values if false positives are occurring
- Ensure maxUnhealthy threshold prevents cascading replacements

SHOULD:
- Verify replacement machines provision successfully after MHC triggers remediation
- Check that workloads are properly drained before machine deletion

MAY:
- Temporarily pause MHC by annotating: `kubectl annotate mhc <name> cluster.x-k8s.io/paused=""`
- Adjust node conditions monitored by MHC

## Safety Ratings

```
safety_ratings:
  - "kubectl get machinehealthchecks: GREEN — read-only status check"
  - "kubectl annotate mhc paused: YELLOW — disables automatic remediation"
  - "kubectl get events: GREEN — read-only event retrieval"
  - "MHC timeout adjustment: YELLOW — affects remediation sensitivity"
```

## Escalation Conditions

- MHC triggers continuous remediation loop (machines replaced repeatedly)
- More than maxUnhealthy machines are unhealthy simultaneously
- Control plane machines are being replaced causing cluster instability
- MHC is paused and manual intervention is required for unhealthy nodes

## Data Sensitivity

- `kubectl get machinehealthchecks`: LOW — returns MHC configuration
- `kubectl get machines conditions`: LOW — returns health status
- `kubelet logs`: MEDIUM — may contain workload and node configuration details
- `kubectl get events`: LOW — returns cluster events

## Prohibited Actions

- NEVER disable MachineHealthChecks permanently without alternative monitoring
- NEVER set maxUnhealthy to 100% — this allows all machines to be replaced simultaneously
- NEVER manually delete machines that MHC is actively remediating — let MHC complete
- NEVER ignore repeated MHC triggers — they indicate a systemic infrastructure issue

## Phase 3 — Rollback

1. If MHC was paused, resume it: `kubectl annotate mhc <name> cluster.x-k8s.io/paused-`
2. If MHC configuration was changed, revert to previous values from Git or backup
3. If machines were incorrectly replaced, verify new machines are healthy and workloads are running
4. Restore original maxUnhealthy and timeout values
5. Document the MHC issue and any configuration changes made

## Common Issues

- symptoms: "Machines being replaced in a loop — new machines also become unhealthy"
  diagnosis: "Underlying infrastructure issue affecting all machines (network, DNS, storage)."
  resolution: "Pause MHC, fix the infrastructure issue, then resume MHC."

## Output Format

```yaml
root_cause: "machine_health_check — <specific_cause>"
evidence:
  - type: mhc_status
    content: "<MHC details and machine conditions>"
severity: HIGH
mitigation:
  immediate: "Fix node health or adjust MHC"
  long_term: "Tune MHC thresholds based on environment"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-cluster"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get configmap aws-auth"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get pods"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest system:masters group for troubleshooting"
  - "NEVER suggest disabling RBAC or Pod Security Standards"
  - "NEVER suggest running pods as root to fix permissions"
