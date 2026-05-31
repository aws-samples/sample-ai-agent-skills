---
title: "B1 — ECS Agent Issues"
description: "Diagnose Bottlerocket ECS agent connectivity and task placement failures"
status: active
severity: HIGH
triggers: ["ecs agent disconnected", "container instance not registered", "task placement failed", "ecs agent not running"]
owner: devops-agent
objective: "Restore ECS agent connectivity and task placement on Bottlerocket instances"
context: "Bottlerocket ECS variant runs the ECS agent as a system container. Agent configuration is through the settings API, not ecs.config."
---

## Phase 1 — Triage

MUST:
- Check container instance status: `aws ecs describe-container-instances --cluster <cluster> --container-instances <arn> --query 'containerInstances[0].{Status:status,AgentConnected:agentConnected,RunningTasks:runningTasksCount}'`
- List container instances in cluster: `aws ecs list-container-instances --cluster <cluster> --status ACTIVE`
- Verify ECS settings: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="apiclient get settings.ecs"`

SHOULD:
- Check instance IAM role has ECS permissions: `aws iam get-instance-profile --instance-profile-name <profile-name>`
- Verify the instance is using an ECS variant AMI (not EKS variant)

MAY:
- Check ECS agent logs via logdog: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="logdog"`

## Phase 2 — Remediate

MUST:
- Ensure ECS cluster name is set in settings: `apiclient set ecs.cluster=<cluster-name>`
- Verify IAM instance profile includes `AmazonEC2ContainerServiceforEC2Role` or equivalent
- Confirm network connectivity to ECS endpoints (ecs, ecs-agent, ecs-telemetry)

SHOULD:
- For VPC without internet, create VPC endpoints for ECS services
- Verify security group allows outbound HTTPS (443) to ECS endpoints

MAY:
- Restart the ECS agent by rebooting: `apiclient reboot`

## Safety Ratings

```
safety_ratings:
  - "ecs describe-container-instances: GREEN — read-only status check"
  - "apiclient get settings.ecs: GREEN — read-only settings query"
  - "apiclient set ecs.cluster: YELLOW — changes ECS cluster assignment"
  - "apiclient reboot: RED — causes instance reboot and task disruption"
```

## Escalation Conditions

- ECS agent remains disconnected after verifying IAM, networking, and settings
- Container instance registers but tasks fail to start with agent errors
- Multiple instances in the cluster simultaneously lose agent connectivity
- Agent version incompatibility with the ECS cluster version

## Data Sensitivity

- `aws ecs describe-container-instances`: MEDIUM — reveals cluster topology and task counts
- `aws iam get-instance-profile`: MEDIUM — reveals IAM role assignments
- `apiclient get settings.ecs`: LOW — returns ECS configuration settings
- ECS agent logs: MEDIUM — may contain task definition details and environment variables

## Prohibited Actions

- NEVER suggest editing /etc/ecs/ecs.config — Bottlerocket uses the settings API
- NEVER deregister a container instance with running tasks without draining first
- NEVER change the ECS cluster assignment on a production instance without draining
- NEVER suggest using Docker commands — Bottlerocket uses containerd

## Phase 3 — Rollback

1. If cluster assignment was changed, revert: `apiclient set ecs.cluster=<original-cluster>`
2. If instance was rebooted and tasks were disrupted, verify task placement recovered: `aws ecs list-tasks --cluster <cluster> --container-instance <arn>`
3. If container instance is in DRAINING state, re-activate: `aws ecs update-container-instances-state --cluster <cluster> --container-instances <arn> --status ACTIVE`
4. If IAM role was modified, revert to previous policy version
5. Document the root cause and update launch template to prevent recurrence

## Common Issues

- symptoms: "Container instance shows agentConnected: false"
  diagnosis: "Network connectivity to ECS endpoints is blocked or IAM permissions are insufficient."
  resolution: "Verify security group allows outbound 443, check VPC endpoints, and verify IAM instance profile."

- symptoms: "Instance uses EKS AMI variant but is assigned to ECS cluster"
  diagnosis: "Wrong Bottlerocket AMI variant. EKS variant does not include the ECS agent."
  resolution: "Terminate instance and relaunch with the correct ECS variant AMI."

## Output Format

```yaml
root_cause: "ecs_agent — <specific_cause>"
evidence:
  - type: agent_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore agent connectivity"
  long_term: "Validate AMI variant and networking in launch template"
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
