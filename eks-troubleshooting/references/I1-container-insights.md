---
title: "I1 — Container Insights Issues"
description: "Diagnose and resolve CloudWatch Container Insights setup and data collection problems"
status: active
severity: MEDIUM
triggers:
  - "Container Insights"
  - "CloudWatch"
  - "metrics missing"
  - "no metrics"
  - "monitoring"
owner: devops-agent
objective: "Identify why Container Insights is not collecting metrics/logs and fix the setup"
context: "CloudWatch Container Insights provides metrics and logs for EKS clusters, nodes, pods, and containers. It requires the CloudWatch Agent (or ADOT collector) and Fluent Bit deployed as DaemonSets. Data goes to CloudWatch Logs and CloudWatch Metrics. For Fargate, use the built-in Fluent Bit log router."
---

## Phase 1 — Triage

MUST:
- Check if CloudWatch Agent is deployed: `kubectl get pods -n amazon-cloudwatch -l name=cloudwatch-agent`
- Check if Fluent Bit is deployed: `kubectl get pods -n amazon-cloudwatch -l k8s-app=fluent-bit`
- Check CloudWatch Agent logs: `kubectl logs -n amazon-cloudwatch -l name=cloudwatch-agent --tail=50`
- Check Fluent Bit logs: `kubectl logs -n amazon-cloudwatch -l k8s-app=fluent-bit --tail=50`
- Verify the CloudWatch log groups exist: `aws logs describe-log-groups --log-group-name-prefix /aws/containerinsights/<cluster>`

SHOULD:
- Check IAM permissions for the CloudWatch Agent (needs CloudWatchAgentServerPolicy)
- Verify the agent's ConfigMap: `kubectl get configmap cwagentconfig -n amazon-cloudwatch -o yaml`
- Check if metrics are appearing in CloudWatch: `aws cloudwatch list-metrics --namespace ContainerInsights --dimensions Name=ClusterName,Value=<cluster>`
- Verify the ADOT collector if using OpenTelemetry instead of CloudWatch Agent

MAY:
- Check if enhanced observability is enabled on the EKS add-on
- Verify Fluent Bit configuration: `kubectl get configmap fluent-bit-config -n amazon-cloudwatch -o yaml`
- Check for Container Insights EKS add-on: `aws eks describe-addon --cluster-name <cluster> --addon-name amazon-cloudwatch-observability`

## Phase 2 — Remediate

MUST:
- If agents not deployed: install Container Insights using the EKS add-on or manual deployment
- If IAM permissions missing: attach CloudWatchAgentServerPolicy to the node role or use IRSA
- If agents are crashing: check logs for specific errors (permissions, configuration, resource limits)
- If log groups missing: agents will create them automatically once permissions are fixed

SHOULD:
- Use the EKS managed add-on `amazon-cloudwatch-observability` for simplified setup
- Configure log retention on Container Insights log groups to control costs
- Set appropriate resource limits for the agent and Fluent Bit pods

MAY:
- Use ADOT (AWS Distro for OpenTelemetry) for more flexible telemetry collection
- Configure custom metrics collection
- Set up Container Insights dashboards and alarms

## Common Issues

- symptoms: "No metrics in CloudWatch Container Insights dashboard"
  diagnosis: "CloudWatch Agent not deployed or not sending metrics."
  resolution: "Deploy the CloudWatch Agent. Check IAM permissions and agent logs."

- symptoms: "CloudWatch Agent running but 'AccessDeniedException' in logs"
  diagnosis: "Node IAM role or IRSA role lacks CloudWatch permissions."
  resolution: "Attach CloudWatchAgentServerPolicy to the appropriate IAM role."

- symptoms: "Metrics appear but logs are missing"
  diagnosis: "Fluent Bit not deployed or misconfigured."
  resolution: "Deploy Fluent Bit. Check Fluent Bit configuration and logs."

- symptoms: "Container Insights works on EC2 nodes but not Fargate"
  diagnosis: "Fargate doesn't support DaemonSets. Need Fargate-specific log configuration."
  resolution: "Configure Fargate logging via the built-in Fluent Bit sidecar (ConfigMap in aws-observability namespace)."

## Output Format

```yaml
root_cause: "Container Insights — <specific_cause>"
evidence:
  - type: agent_status
    content: "<CloudWatch Agent and Fluent Bit pod status>"
  - type: agent_logs
    content: "<relevant agent log entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix agent deployment, IAM permissions, or configuration"
  long_term: "Use EKS managed observability add-on, set up log retention and alerting"
```

## Safety Ratings
- GREEN: read-only (`kubectl get pods -n amazon-cloudwatch`, `kubectl logs`, `aws logs describe-log-groups`, `aws cloudwatch list-metrics`)
- YELLOW: state-changing recoverable (`aws eks create-addon` for observability, `kubectl apply` CloudWatch Agent/Fluent Bit DaemonSets, update IAM role for CloudWatch permissions, `kubectl scale` agent replicas)
- RED: destructive/irreversible (`aws logs delete-log-group` removing historical metrics/logs, deleting CloudWatch Agent DaemonSet losing all metric collection)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Change affects node group scaling in production"
- "Fix involves deploying DaemonSets to all production nodes"
- "Remediation requires modifying node IAM role permissions in production"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current CloudWatch Agent ConfigMap before modification"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify metrics and logs appearing in CloudWatch after change"
- Revert: "Restore agent ConfigMap from backup if metric collection breaks"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Node group configuration reveals instance types and scaling"
- MEDIUM: "CloudWatch logs may contain application-level sensitive data"
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
