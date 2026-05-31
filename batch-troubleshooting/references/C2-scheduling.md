---
title: "C2 — Job Scheduling Issues"
description: "Diagnose job scheduling and fair-share policy problems"
status: active
severity: MEDIUM
triggers:
  - "scheduling unfair"
  - "fair share"
  - "job priority"
  - "scheduling policy"
  - "jobs not scheduled"
owner: devops-agent
objective: "Identify and resolve job scheduling issues"
context: "Scheduling issues include: fair-share policy misconfigured, share identifiers not set, scheduling decay wrong, or priority-based scheduling not working as expected."
---

## Phase 1 — Triage

MUST:
- Check scheduling policy: `aws batch describe-scheduling-policies --arns <policy-arn>`
- Check queue scheduling policy association: `aws batch describe-job-queues --job-queues <queue-name> --query 'jobQueues[*].schedulingPolicyArn'`
- Verify share identifiers in job submissions
- Check job queue priorities

SHOULD:
- Review fair-share weights and decay settings
- Check compute share distribution across users/groups
- Verify scheduling policy is attached to the correct queue

MAY:
- Analyze scheduling patterns over time
- Compare expected vs actual resource distribution

## Phase 2 — Remediate

MUST:
- Fix scheduling policy configuration
- Set share identifiers in job submissions
- Ensure scheduling policy is attached to queue

SHOULD:
- Create scheduling policy: `aws batch create-scheduling-policy --name <name> --fairshare-policy '{"shareDecaySeconds":3600,"computeReservation":10,"shareDistribution":[{"shareIdentifier":"team-a","weightFactor":1},{"shareIdentifier":"team-b","weightFactor":2}]}'`
- Submit jobs with share identifier: `--share-identifier team-a`

MAY:
- Implement per-team scheduling policies
- Monitor scheduling fairness metrics

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix scheduling policy configuration | YELLOW | Changes resource distribution — affects all users |
| Set share identifiers in job submissions | GREEN | Job submission parameter — non-destructive |
| Create scheduling policy | GREEN | New resource creation — non-destructive |
| Attach scheduling policy to queue | YELLOW | Changes scheduling behavior for all jobs in queue |
| Implement per-team scheduling policies | YELLOW | Multi-tenant governance change — review with stakeholders |

## Escalation Conditions

- Compute environment runs production batch jobs
- Scheduling changes affect resource allocation for multiple teams
- Fair-share policy misconfiguration causes resource starvation
- Priority changes affect SLA-critical job processing

## Data Sensitivity

- **Classification: MEDIUM**
- Scheduling policies reveal multi-tenant resource governance
- Share distribution reveals team-level resource allocation
- Compute reservation settings expose baseline capacity guarantees
- Share identifiers reveal organizational structure

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest removing scheduling policies without stakeholder approval
- **NEVER** suggest setting weight factors that starve any team of resources
- **NEVER** suggest bypassing fair-share scheduling for individual jobs

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Created scheduling policy | Delete scheduling policy if not needed |
| Attached policy to queue | Detach scheduling policy from queue |
| Changed share distribution weights | Revert weight factors to previous values |
| Changed compute reservation | Revert compute reservation to previous value |

## Output Format

```yaml
root_cause: "scheduling_issue — <specific_cause>"
evidence:
  - type: scheduling_policy
    content: "<policy configuration>"
  - type: share_distribution
    content: "<resource distribution>"
severity: MEDIUM
mitigation:
  immediate: "Fix scheduling policy or share identifiers"
  long_term: "Implement fair-share scheduling with monitoring"
```

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
  - "NEVER suggest overly broad compute environment permissions"
  - "NEVER suggest disabling job queue scheduling"
  - "NEVER suggest public subnet placement for compute environments"
