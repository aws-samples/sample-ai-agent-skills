---
title: "E3 — Capacity Provider Issues"
description: "Diagnose ECS capacity provider scaling and placement failures"
status: active
severity: HIGH
triggers:
  - "No capacity available"
  - "Capacity provider scaling"
  - "Auto Scaling group not scaling"
owner: devops-agent
objective: "Fix capacity provider configuration and restore task placement"
context: "Capacity providers link ECS to Auto Scaling groups (EC2) or Fargate. Managed scaling adjusts ASG capacity based on ECS demand. Target capacity percentage controls how aggressively to scale. Issues arise when scaling is too slow or misconfigured."
---

## Phase 1 — Triage

MUST:
- Check capacity providers: `aws ecs describe-capacity-providers --capacity-providers <names>`
- Verify managed scaling is enabled and target capacity percentage
- Check ASG current/desired/max capacity: `aws autoscaling describe-auto-scaling-groups`
- Check if ASG max capacity is reached

SHOULD:
- Check ECS CapacityProviderReservation metric in CloudWatch
- Verify the ASG launch template has the correct instance type and AMI
- Check if instances are joining the ECS cluster after launch

MAY:
- Check ASG scaling activities for errors
- Verify instance profile has ECS permissions

## Common Issues

- symptoms: "Tasks pending, ASG at max capacity"
  diagnosis: "ASG max capacity reached. Can't launch more instances."
  resolution: "Increase ASG max capacity or use Fargate as a fallback capacity provider."

- symptoms: "ASG scales up but instances don't join cluster"
  diagnosis: "ECS agent not starting, wrong cluster name in user data, or IAM role missing."
  resolution: "Check instance user data for correct cluster name, verify IAM role."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
```

## Escalation Conditions

- Service handles production traffic or is business-critical
- Fix requires modifying the task definition for a running service
- Resolution involves changing capacity provider or cluster configuration
- Multiple services affected suggesting cluster-level or account-level issue
- Deployment rollback has been triggered by the circuit breaker

## Data Sensitivity

HIGH - ECS task definitions may contain secrets ARNs, environment variables with credentials, and sensitive configuration. Container logs may capture application data including PII. Service Connect and service discovery configurations may expose internal architecture.

## Prohibited Actions

- NEVER suggest stopping all tasks in a service to fix a deployment issue - this causes a full outage
- NEVER suggest removing or modifying the task execution role while tasks are running
- NEVER recommend deleting a service without first scaling desired count to 0 and draining connections
- NEVER suggest modifying a running task definition - create a new revision instead
- NEVER recommend disabling the deployment circuit breaker on a production service

## Phase 3 - Rollback

1. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
2. If IAM roles were modified, revert the task definition to the previous revision with correct role ARNs
3. If capacity provider strategy was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --capacity-provider-strategy <original>`
## Output Format

```yaml
root_cause: "capacity_provider — <detail>"
evidence:
  - type: capacity_provider
    content: "<scaling state and ASG configuration>"
severity: HIGH
mitigation:
  immediate: "Increase ASG max or fix instance configuration"
  long_term: "Use Fargate Spot as fallback, set appropriate target capacity"
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
