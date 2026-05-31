---
title: "E3 — EKS ADOT Add-on Issues"
description: "Diagnose and resolve EKS ADOT managed add-on issues for AMP integration"
status: active
severity: HIGH
triggers:
  - "EKS add-on"
  - "ADOT add-on"
  - "adot-addon"
  - "managed add-on"
  - "aws-otel"
  - "EKS ADOT"
owner: devops-agent
objective: "Identify and fix EKS ADOT managed add-on issues affecting AMP metrics collection"
context: "The ADOT Collector is available as an EKS managed add-on (adot). It simplifies deployment and lifecycle management of the collector on EKS clusters. The add-on requires proper IAM configuration (IRSA or Pod Identity) and a collector configuration via ConfigMap or custom resource."
---

## Phase 1 — Triage

MUST:
- Check add-on status: `aws eks describe-addon --cluster-name <cluster> --addon-name adot --query 'addon.{status:status,version:addonVersion,health:health}'`
- Check ADOT collector pods: `kubectl get pods -n opentelemetry-operator-system`
- Check collector custom resources: `kubectl get opentelemetrycollectors -A`
- Check collector pod logs: `kubectl logs -n <namespace> -l app.kubernetes.io/name=collector --tail=100`

SHOULD:
- Verify add-on version compatibility with cluster version
- Check if the ADOT operator is running: `kubectl get pods -n opentelemetry-operator-system -l app.kubernetes.io/name=opentelemetry-operator`
- Verify IRSA or Pod Identity is configured for the collector service account
- Check for add-on configuration conflicts: `aws eks describe-addon --cluster-name <cluster> --addon-name adot --query 'addon.configurationConflicts'`

MAY:
- Check add-on update history: `aws eks list-updates --cluster-name <cluster> --addon-name adot`
- Verify the OpenTelemetryCollector CR has correct AMP configuration
- Check if cert-manager is installed (required by ADOT operator)

## Phase 2 — Remediate

MUST:
- If add-on not installed: `aws eks create-addon --cluster-name <cluster> --addon-name adot --addon-version <version> --service-account-role-arn <role-arn>`
- If add-on degraded: check health issues and resolve (usually IAM or config)
- If operator not running: check operator pod logs for startup errors
- If collector CR missing: create OpenTelemetryCollector custom resource with AMP config

SHOULD:
- Ensure cert-manager is installed before the ADOT add-on
- Configure IRSA for the collector with aps:RemoteWrite permission
- Use the add-on's configuration values for customization instead of manual edits
- Keep the add-on version updated for security and compatibility

MAY:
- Use advanced configuration for the collector via add-on configuration values
- Set up multiple collector instances for different workloads
- Configure collector autoscaling for variable workloads

## Common Issues

- symptoms: "Add-on status shows DEGRADED"
  diagnosis: "Add-on health check failing — usually IAM, cert-manager, or resource issues."
  resolution: "Check addon health issues in describe output. Verify cert-manager is installed. Check IRSA configuration."

- symptoms: "OpenTelemetry operator pod is CrashLoopBackOff"
  diagnosis: "cert-manager not installed or webhook certificates not ready."
  resolution: "Install cert-manager: kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml. Wait for cert-manager pods to be ready."

- symptoms: "Collector CR created but no collector pods appear"
  diagnosis: "Operator is not processing the CR — check operator logs."
  resolution: "Verify operator is running. Check operator logs for CR validation errors. Ensure CR spec is valid."

- symptoms: "Add-on installation fails with 'ConfigurationConflict'"
  diagnosis: "Existing resources conflict with the add-on's managed resources."
  resolution: "Remove conflicting resources or use --resolve-conflicts OVERWRITE flag."

## Safety Ratings

```
safety_ratings:
  - "describe-addon: GREEN — read-only add-on status check"
  - "kubectl get pods: GREEN — read-only pod status check"
  - "create-addon: YELLOW — installs managed add-on on EKS cluster"
  - "install cert-manager: YELLOW — installs cluster-wide certificate management"
```

## Escalation Conditions

- ADOT add-on DEGRADED status affecting production metrics collection
- cert-manager installation required but conflicts with existing cluster resources
- Add-on configuration conflicts require --resolve-conflicts OVERWRITE
- Add-on version incompatible with EKS cluster version

## Data Sensitivity

- Add-on status and health: LOW — operational metadata
- Operator and collector logs: MEDIUM — may contain configuration details
- IRSA configuration: MEDIUM — reveals service account trust relationships

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER use --resolve-conflicts OVERWRITE without understanding what will be overwritten
- NEVER delete the ADOT add-on on a production cluster without a replacement collection strategy

## Phase 3 — Rollback

1. If the add-on was installed incorrectly, delete it: `aws eks delete-addon --cluster-name <cluster> --addon-name adot`
2. If cert-manager was installed and caused issues, remove it: `kubectl delete -f cert-manager.yaml`
3. If collector CR was created incorrectly, delete it: `kubectl delete opentelemetrycollector <name> -n <namespace>`
4. Verify existing workloads are unaffected by the add-on removal
5. Restore previous metrics collection method if the add-on is rolled back

## Output Format

```yaml
root_cause: "EKS ADOT add-on — <specific_cause>"
evidence:
  - type: addon_status
    content: "<add-on status and health>"
  - type: operator_logs
    content: "<operator or collector pod logs>"
severity: HIGH
mitigation:
  immediate: "Fix add-on configuration, IAM, or dependency issues"
  long_term: "Automate add-on lifecycle management, monitor add-on health"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
