---
title: "E1 — EKS Integration"
description: "Diagnose CloudWatch Application Signals EKS integration issues"
status: active
severity: HIGH
triggers: ["EKS addon error", "CloudWatch Observability addon", "EKS instrumentation", "pod annotation"]
owner: devops-agent
objective: "Resolve EKS-specific Application Signals integration issues"
context: "EKS integration uses the CloudWatch Observability addon (amazon-cloudwatch-observability) to deploy agents and enable auto-instrumentation."
---

## Phase 1 — Triage

MUST:
- Check addon status: `aws eks describe-addon --cluster-name <cluster> --addon-name amazon-cloudwatch-observability --query '{Status:status,AddonVersion:addonVersion}'`
- Verify addon pods are running: `kubectl get pods -n amazon-cloudwatch -l app.kubernetes.io/name=amazon-cloudwatch-observability`
- Check pod annotations for auto-instrumentation: `kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.metadata.annotations}'`

SHOULD:
- Verify addon version supports Application Signals
- Check DaemonSet status: `kubectl get daemonset -n amazon-cloudwatch`
- Review addon pod logs for errors

MAY:
- Check EKS cluster IAM OIDC provider configuration
- Verify service account IAM role association

## Phase 2 — Remediate

MUST:
- Install or update the CloudWatch Observability addon to a compatible version
- Annotate workload pods for auto-instrumentation: `kubectl annotate pod <pod> instrumentation.opentelemetry.io/inject-java="true"`
- Restart annotated pods to apply instrumentation

SHOULD:
- Use namespace-level annotations for broad instrumentation
- Verify addon IAM role has CloudWatchAgentServerPolicy

MAY:
- Configure addon advanced settings for custom ADOT configuration
- Use Helm for addon management if more control is needed

## Common Issues

- symptoms: "Addon status shows DEGRADED or CREATE_FAILED"
  diagnosis: "Addon IAM role lacks required permissions or cluster OIDC provider is not configured."
  resolution: "Create OIDC provider: aws eks associate-identity-provider-config. Attach CloudWatchAgentServerPolicy to addon role."

- symptoms: "Pods annotated but no instrumentation applied"
  diagnosis: "Pods were not restarted after annotation or annotation key is incorrect."
  resolution: "Restart pods: kubectl rollout restart deployment <name>. Verify annotation key matches the language."

## Safety Ratings

```
safety_ratings:
  - "eks describe-addon: GREEN — read-only addon status"
  - "kubectl get pods: GREEN — read-only pod listing"
  - "kubectl annotate: YELLOW — modifies pod metadata, triggers restart"
  - "eks create-addon: YELLOW — deploys addon to cluster"
```

## Escalation Conditions

- Addon installation fails due to cluster version incompatibility
- Addon causes node resource exhaustion (CPU/memory)
- Auto-instrumentation causes application startup failures
- Addon conflicts with existing monitoring tools on the cluster

## Data Sensitivity

- `aws eks describe-addon`: LOW — returns addon metadata
- `kubectl get pods`: LOW — returns pod names and status
- Pod annotations: LOW — contain instrumentation configuration
- Addon pod logs: MEDIUM — may contain cluster and application details

## Prohibited Actions

- NEVER delete the CloudWatch Observability addon without understanding impact on monitoring
- NEVER annotate all pods simultaneously — use canary approach
- NEVER use addon versions not tested with the cluster's Kubernetes version
- NEVER modify addon DaemonSet directly — use addon configuration

## Phase 3 — Rollback

1. Remove instrumentation annotations: `kubectl annotate pod <pod> instrumentation.opentelemetry.io/inject-java-`
2. Restart pods to remove instrumentation: `kubectl rollout restart deployment <name>`
3. If addon must be removed: `aws eks delete-addon --cluster-name <cluster> --addon-name amazon-cloudwatch-observability`
4. Verify application pods return to normal operation
5. Document the EKS integration issue and addon version

## Output Format

```yaml
root_cause: "eks_integration — <specific_cause>"
evidence:
  - type: addon_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix addon or annotation configuration"
  long_term: "Automate addon management via EKS blueprints"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
