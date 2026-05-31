---
title: "I2 — Logging Issues"
description: "Diagnose and resolve EKS control plane and application logging problems"
status: active
severity: MEDIUM
triggers:
  - "logging"
  - "logs missing"
  - "control plane logs"
  - "audit logs"
  - "application logs"
owner: devops-agent
objective: "Identify why logs are not being collected and fix the logging configuration"
context: "EKS has two logging layers: (1) Control plane logging — 5 log types sent to CloudWatch by the managed control plane, (2) Application/node logging — collected by agents (Fluent Bit, CloudWatch Agent) running on worker nodes. Control plane logging must be explicitly enabled. Application logging requires deploying log collection agents."
---

## Phase 1 — Triage

MUST:
- Check control plane logging status: `aws eks describe-cluster --name <cluster> --query 'cluster.logging.clusterLogging'`
- Verify which log types are enabled (api, audit, authenticator, controllerManager, scheduler)
- Check if the CloudWatch log group exists: `aws logs describe-log-groups --log-group-name-prefix /aws/eks/<cluster>/cluster`
- For application logs: check if Fluent Bit or other log collector is deployed
- Check log group retention settings: `aws logs describe-log-groups --log-group-name-prefix /aws/eks/<cluster>`

SHOULD:
- Test log retrieval: `aws logs filter-log-events --log-group-name /aws/eks/<cluster>/cluster --log-stream-name-prefix kube-apiserver --limit 5`
- Check Fluent Bit pod status and logs for application log collection
- Verify IAM permissions for log delivery (both control plane and application)
- Check for log group subscription filters that might affect delivery

MAY:
- Check CloudWatch Logs Insights for querying: `aws logs start-query --log-group-name /aws/eks/<cluster>/cluster --query-string 'fields @timestamp, @message | sort @timestamp desc | limit 20'`
- Verify log format and parsing configuration in Fluent Bit
- Check for log volume and associated CloudWatch costs

## Phase 2 — Remediate

MUST:
- If control plane logging disabled: enable it: `aws eks update-cluster-config --name <cluster> --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'`
- If application logs missing: deploy Fluent Bit or CloudWatch Agent
- If log group missing: it will be created automatically when logging is enabled
- If IAM permissions issue: fix the node role or IRSA for log collection agents

SHOULD:
- Enable at least `api` and `audit` log types for production clusters
- Set log retention policies to control costs (default is never expire)
- Use structured logging (JSON) in applications for easier querying

MAY:
- Set up CloudWatch Logs Insights saved queries for common investigations
- Configure log-based alarms for critical events
- Use S3 export for long-term log archival

## Common Issues

- symptoms: "No control plane logs in CloudWatch"
  diagnosis: "Control plane logging is not enabled (disabled by default)."
  resolution: "Enable control plane logging via update-cluster-config."

- symptoms: "Control plane logs enabled but log group is empty"
  diagnosis: "Logging was just enabled — it takes a few minutes for logs to appear."
  resolution: "Wait 5-10 minutes. If still empty, check the cluster status."

- symptoms: "Application logs not appearing in CloudWatch"
  diagnosis: "Fluent Bit or log collection agent not deployed or misconfigured."
  resolution: "Deploy Fluent Bit. Check its configuration and IAM permissions."

- symptoms: "CloudWatch Logs costs are very high"
  diagnosis: "Verbose logging (especially audit logs) generates large volumes."
  resolution: "Set log retention policies. Consider disabling verbose log types or filtering."

## Output Format

```yaml
root_cause: "Logging issue — <specific_cause>"
evidence:
  - type: logging_config
    content: "<cluster logging configuration>"
  - type: log_groups
    content: "<CloudWatch log group status>"
severity: MEDIUM
mitigation:
  immediate: "Enable logging or fix log collection agent"
  long_term: "Implement log retention policies, set up log-based alerting"
```

## Safety Ratings
- GREEN: read-only (`aws eks describe-cluster --query 'cluster.logging'`, `aws logs describe-log-groups`, `aws logs filter-log-events`, `kubectl get pods`)
- YELLOW: state-changing recoverable (`aws eks update-cluster-config` to enable/disable logging, deploy Fluent Bit for log collection, set log retention policies, update IAM permissions)
- RED: destructive/irreversible (`aws logs delete-log-group` removing historical audit/control plane logs, disabling audit logging in production)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Change affects node group scaling in production"
- "Fix involves enabling or disabling control plane logging in production"
- "Remediation requires deploying log collection agents to all production nodes"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current cluster logging configuration before modification"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify logs appearing in CloudWatch after change"
- Revert: "Restore previous logging configuration if log collection breaks"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- HIGH: "Audit logs reveal all API server requests including who accessed what resources"
- MEDIUM: "Node group configuration reveals instance types and scaling"
- LOW: "Pod status and events"

## Prohibited Actions
- NEVER suggest adding `system:masters` group for troubleshooting access
- NEVER suggest disabling Pod Security Standards/Admission
- NEVER suggest running containers as root to fix permission issues
- NEVER suggest `kubectl delete namespace` in production without confirmation
- NEVER suggest modifying kube-system resources without backup

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

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
