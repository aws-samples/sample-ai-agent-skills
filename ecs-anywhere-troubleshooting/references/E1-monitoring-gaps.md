---
title: "E1 — Monitoring Gaps"
description: "Diagnose monitoring and metrics gaps for ECS Anywhere external instances"
status: active
severity: MEDIUM
triggers: ["no metrics", "monitoring not working", "CloudWatch empty", "instance metrics missing"]
owner: devops-agent
objective: "Resolve monitoring gaps for ECS Anywhere external instances"
context: "ECS Anywhere external instances report metrics differently from EC2. CloudWatch Agent may need manual installation. Container Insights requires additional setup."
---

## Phase 1 — Triage

MUST:
- Check ECS cluster metrics: `aws cloudwatch list-metrics --namespace AWS/ECS --dimensions Name=ClusterName,Value=<cluster-name>`
- Check if Container Insights is enabled: `aws ecs describe-clusters --clusters <cluster-name> --include SETTINGS --query 'clusters[*].settings'`
- Verify ECS Agent is reporting metrics: `aws ecs describe-container-instances --cluster <cluster-name> --container-instances <instance-arn> --query 'containerInstances[*].{Agent:agentConnected,Resources:registeredResources}'`

SHOULD:
- Check if CloudWatch Agent is installed on external instances
- Verify CloudWatch Agent can reach CloudWatch endpoints
- Check for custom metrics from the application

MAY:
- Check CloudWatch Agent logs on the instance
- Verify IAM permissions for CloudWatch PutMetricData
- Test metric publishing from the instance

## Phase 2 — Remediate

MUST:
- Enable Container Insights on the cluster: `aws ecs update-cluster-settings --cluster <cluster-name> --settings name=containerInsights,value=enabled`
- Install CloudWatch Agent on external instances for host-level metrics
- Ensure ECS Agent is connected and reporting

SHOULD:
- Configure CloudWatch Agent to collect OS-level metrics (CPU, memory, disk, network)
- Set up CloudWatch dashboards for ECS Anywhere instances
- Create alarms for critical metrics (agent disconnection, high CPU/memory)

MAY:
- Implement custom metrics for application-specific monitoring
- Set up Prometheus/Grafana for additional monitoring
- Configure CloudWatch anomaly detection for key metrics

## Common Issues

- symptoms: "No host-level metrics (CPU, memory, disk) for external instances"
  diagnosis: "CloudWatch Agent is not installed on external instances. ECS only reports task-level metrics."
  resolution: "Install and configure CloudWatch Agent on each external instance."

- symptoms: "Container Insights shows no data for external instances"
  diagnosis: "Container Insights is not enabled on the cluster, or CloudWatch Agent is not configured for Container Insights."
  resolution: "Enable Container Insights on the cluster. Install CloudWatch Agent with ECS Container Insights configuration."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch list-metrics: GREEN — read-only"
  - "ecs describe-clusters: GREEN — read-only"
  - "ecs update-cluster-settings: YELLOW — changes cluster configuration"
  - "ssm send-command: RED — installs software on remote instances"
```

## Escalation Conditions

- CloudWatch Agent cannot be installed due to OS compatibility issues
- Monitoring requires metrics not available from ECS or CloudWatch Agent
- Network restrictions prevent metric delivery to CloudWatch
- Monitoring costs exceed budget for large numbers of external instances

## Data Sensitivity

- `aws cloudwatch list-metrics`: LOW — metric names and dimensions only
- `aws ecs describe-clusters`: LOW — cluster settings
- CloudWatch Agent configuration: MEDIUM — reveals monitored resources
- Host-level metrics: LOW — performance data only

## Prohibited Actions

- NEVER disable Container Insights on production clusters without approval
- NEVER install monitoring agents that conflict with existing security tools
- NEVER collect metrics at intervals shorter than 1 minute without cost analysis
- NEVER expose monitoring endpoints publicly

## Phase 3 — Rollback

1. If Container Insights was enabled and causing issues, disable it
2. If CloudWatch Agent was installed and causing problems, stop or uninstall it
3. If custom metrics were configured incorrectly, remove the configuration
4. Verify monitoring returns to previous state after rollback
5. Document the monitoring architecture and requirements

## Output Format

```yaml
root_cause: "monitoring — <specific_cause>"
evidence:
  - type: monitoring_gap
    content: "<missing metrics and configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Install monitoring agents and enable Container Insights"
  long_term: "Implement comprehensive monitoring with dashboards and alerts"
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
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"
