---
title: "B1 — Queue Issues"
description: "Diagnose queue creation, configuration, and job scheduling issues"
status: active
severity: HIGH
triggers: ["queue error", "queue not processing", "jobs stuck in queue", "queue creation failed"]
owner: devops-agent
objective: "Resolve queue configuration and job scheduling issues"
context: "Queues organize and schedule jobs for execution on associated fleets. Queues have IAM roles, storage profiles, and fleet associations that control job execution."
---

## Phase 1 — Triage

MUST:
- List queues: `aws deadline list-queues --farm-id <farm-id> --query 'queues[*].{QueueId:queueId,DisplayName:displayName,Status:status}'`
- Get queue details: `aws deadline get-queue --farm-id <farm-id> --queue-id <queue-id>`
- Check queue-fleet associations: `aws deadline list-queue-fleet-associations --farm-id <farm-id> --queue-id <queue-id>`

SHOULD:
- Verify queue IAM role has required permissions
- Check queue status is IDLE or SCHEDULING (not in error state)
- List pending jobs: `aws deadline list-jobs --farm-id <farm-id> --queue-id <queue-id> --query 'jobs[?lifecycleStatus==`PENDING`]'`

MAY:
- Review queue environment configuration
- Check storage profile associations

## Phase 2 — Remediate

MUST:
- Fix queue IAM role permissions for S3 access (job attachments) and CloudWatch Logs
- Create queue-fleet association if missing: `aws deadline create-queue-fleet-association --farm-id <farm-id> --queue-id <queue-id> --fleet-id <fleet-id>`
- Ensure queue status is active

SHOULD:
- Configure appropriate job run-as user settings
- Set up queue-level environment variables

MAY:
- Configure priority scheduling between queues
- Set up queue-level budget associations

## Safety Ratings

```
safety_ratings:
  - "list-queues: GREEN — read-only queue enumeration"
  - "get-queue: GREEN — read-only queue details"
  - "create-queue: YELLOW — creates job scheduling resource"
  - "update-queue: YELLOW — changes queue configuration affecting future jobs"
  - "delete-queue: RED — removes queue and blocks job submission"
```

## Escalation Conditions

- Queue IAM role requires permissions changes managed by security team
- Jobs stuck in PENDING across all queues indicating systemic issue
- Queue configuration changes needed during active render production
- Queue-fleet association changes affect job routing for multiple teams

## Data Sensitivity

- Queue configuration: MEDIUM — reveals job scheduling architecture
- Queue IAM role: MEDIUM — execution role with S3 and service access
- Job list: MEDIUM — reveals render pipeline workload
- Environment variables: HIGH — may contain application secrets or paths

## Prohibited Actions

- NEVER delete a queue with pending or running jobs
- NEVER modify queue IAM role to have overly broad S3 permissions
- NEVER remove queue-fleet associations during active job execution
- NEVER change queue run-as user settings without understanding security implications

## Phase 3 — Rollback

1. If queue configuration was changed, revert: `aws deadline update-queue --farm-id <id> --queue-id <id>`
2. If queue-fleet association was removed, recreate it
3. If queue was deleted, recreate with same configuration and associations
4. Verify pending jobs resume processing after fix

## Common Issues

- symptoms: "Jobs submitted to queue stay in PENDING indefinitely"
  diagnosis: "No queue-fleet association or associated fleet has no available workers."
  resolution: "Create queue-fleet association. Verify fleet has workers and is in active status."

- symptoms: "Queue status shows SCHEDULING_BLOCKED"
  diagnosis: "Queue IAM role lacks required permissions or budget is exhausted."
  resolution: "Fix queue IAM role permissions. Check budget status and increase if needed."

## Output Format

```yaml
root_cause: "queue_issues — <specific_cause>"
evidence:
  - type: queue_detail
    content: "<queue configuration and association status>"
severity: HIGH
mitigation:
  immediate: "Fix queue configuration or associations"
  long_term: "Implement queue monitoring and alerting"
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
