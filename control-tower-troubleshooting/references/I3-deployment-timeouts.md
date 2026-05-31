---
title: "I3 — StackSet Deployment Timeouts"
description: "Diagnose StackSet operations that take too long or time out"
status: active
severity: MEDIUM
triggers:
  - "StackSet operation timeout"
  - "Deployment taking too long"
  - "StackSet operation stuck"
owner: devops-agent
objective: "Resolve slow or stuck StackSet deployments"
context: "CT StackSet operations deploy to all accounts in a registered OU across all governed regions. Large organizations with many accounts and regions can experience slow deployments. Operations have concurrency limits and can be affected by throttling."
---

## Phase 1 — Triage

MUST:
- Check StackSet operation status: `aws cloudformation describe-stack-set-operation --stack-set-name <name> --operation-id <id>`
- Check individual stack instance statuses for the operation
- Identify how many accounts/regions are being targeted
- Check for throttling errors in stack instance failures

SHOULD:
- Check if other StackSet operations are running concurrently
- Verify the operation concurrency settings
- Check if specific accounts are causing delays

MAY:
- Check CloudFormation service quotas
- Review the StackSet template for slow-creating resources

## Phase 2 — Remediate

MUST:
- Wait for the operation to complete if it's still progressing
- If truly stuck: cancel the operation and retry
- Fix any failed stack instances before retrying

SHOULD:
- Increase max concurrent accounts/percentage for faster deployment
- Stagger deployments for very large organizations
- Monitor operation progress

MAY:
- Request CloudFormation service quota increases
- Optimize StackSet templates for faster deployment

## Common Issues

- symptoms: "StackSet operation running for hours"
  diagnosis: "Large number of accounts/regions with default concurrency."
  resolution: "This may be normal. Check progress. Increase concurrency for future operations."

- symptoms: "Operation stuck with no progress"
  diagnosis: "A stack instance is stuck waiting for a resource."
  resolution: "Identify the stuck instance, cancel the operation, fix the issue, and retry."

## Safety Ratings

```
safety_ratings:
  - "describe-stack-set-operation: GREEN — read-only operation status"
  - "list-stack-instances: GREEN — read-only instance listing"
  - "cancel stuck StackSet operation: YELLOW — recoverable, operation can be retried"
  - "increase max concurrent accounts: YELLOW — recoverable, setting can be changed back"
  - "delete stuck stack instances: YELLOW — recoverable, CT will recreate on re-register"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- StackSet operation stuck with no progress for extended period
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production
- CloudFormation service quota limits reached

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: StackSet operation details (account IDs, regions, progress)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If a StackSet operation was cancelled, check which accounts were updated and which were not before retrying
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Document the StackSet operation ID and progress before cancelling
- If concurrency settings were increased, monitor for throttling and revert if needed

## Output Format

```yaml
root_cause: "deployment_timeout — <specific cause>"
evidence:
  - type: stackset_operation
    content: "<operation status and progress>"
severity: MEDIUM
mitigation:
  immediate: "Wait for completion or cancel and retry"
  long_term: "Optimize concurrency settings and monitor deployments"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"
