---
title: "C2 — EC2 Instance Action Errors"
description: "Diagnose EC2 instance stop budget action errors"
status: active
severity: CRITICAL
triggers: ["ec2 stop action", "instance stop failed", "ec2 budget action", "stop instances action"]
owner: devops-agent
objective: "Resolve EC2 instance stop action errors"
context: "EC2 stop actions stop running instances when budget thresholds are breached. Stopped instances remain stopped until manually restarted. Ephemeral data is lost."
---

## Phase 1 — Triage

MUST:
- Check action configuration: `aws budgets describe-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Verify execution role has ec2:StopInstances permission: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names ec2:StopInstances`
- Check target instances: `aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --query 'Reservations[*].Instances[*].{Id:InstanceId,Type:InstanceType,State:State.Name}'`

SHOULD:
- Review action history for execution details: `aws budgets describe-budget-action-histories --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Check if instances have termination protection or stop protection

MAY:
- Verify instance tags match budget filter criteria
- Check for Auto Scaling groups that may restart stopped instances

## Phase 2 — Remediate

MUST:
- Grant ec2:StopInstances to the execution role
- Verify the action targets the correct instances (via tags or region)
- Ensure critical instances are excluded from the action scope

SHOULD:
- Set up approval workflow for EC2 stop actions
- Tag instances appropriately to control action scope
- Configure Auto Scaling groups to respect budget actions (suspend scaling)

MAY:
- Implement graceful shutdown scripts via SSM before stop
- Set up CloudWatch alarms for instance state changes

## Safety Ratings

```
safety_ratings:
  - "budgets describe-budget-action: GREEN — read-only action check"
  - "ec2 describe-instances: GREEN — read-only instance listing"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
  - "ec2 stop-instances: RED — stops running instances, causes downtime"
```

## Escalation Conditions

- EC2 stop action targets production instances
- Stopped instances have ephemeral data that was not backed up
- Auto Scaling groups conflict with stop actions
- Multiple instances stopped causing service outage

## Data Sensitivity

- Instance details: MEDIUM — reveals infrastructure configuration
- Action configuration: MEDIUM — reveals cost control strategy
- Instance tags: LOW — metadata
- Execution role: MEDIUM — reveals permission scope

## Prohibited Actions

- NEVER configure EC2 stop actions for production instances without approval workflows
- NEVER stop instances with ephemeral storage containing critical data
- NEVER disable stop protection to allow budget actions without review
- NEVER ignore stopped instances — they still incur EBS storage charges

## Phase 3 — Rollback

1. Restart stopped instances: `aws ec2 start-instances --instance-ids <instance-id>`
2. Verify applications are healthy after restart
3. Reset the budget action: `aws budgets execute-budget-action --account-id <id> --budget-name <name> --action-id <id> --execution-type RESET_BUDGET_ACTION`
4. Check for data loss on instances with ephemeral storage
5. Resume Auto Scaling groups if suspended: `aws autoscaling resume-processes --auto-scaling-group-name <name>`

## Common Issues

- symptoms: "EC2 stop action fails with UnauthorizedOperation"
  diagnosis: "Execution role missing ec2:StopInstances permission."
  resolution: "Add ec2:StopInstances to the execution role policy. Verify role trust policy includes budgets.amazonaws.com."

## Output Format

```yaml
root_cause: "ec2_instance_action — <specific_cause>"
evidence:
  - type: ec2_action_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix EC2 action and restart instances"
  long_term: "Implement approval workflows and instance tagging"
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
