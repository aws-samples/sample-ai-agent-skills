---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for Deadline Cloud issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Deadline Cloud error", "Deadline Cloud not working", "general Deadline Cloud issue", "unknown render farm problem"]
owner: devops-agent
objective: "Systematically diagnose and resolve general Deadline Cloud issues"
context: "Use this runbook when the issue does not clearly match a specific runbook category. Follow the systematic investigation workflow to identify the root cause domain."
---

## Phase 1 — Triage

MUST:
- List farms: `aws deadline list-farms`
- List fleets and check status: `aws deadline list-fleets --farm-id <farm-id>`
- List queues and check status: `aws deadline list-queues --farm-id <farm-id>`

SHOULD:
- Check CloudTrail for recent Deadline Cloud errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=deadline.amazonaws.com --max-results 20`
- Verify IAM permissions for the calling principal
- Check AWS Health Dashboard for Deadline Cloud service events

MAY:
- Review recent job statuses for patterns
- Check worker agent logs on fleet instances
- Test API connectivity to Deadline Cloud endpoints

## Phase 2 — Remediate

MUST:
- Identify the specific domain (farm, fleet, queue, job, worker, storage, budget, license) and route to appropriate runbook
- Fix IAM permission issues if API calls are failing
- Verify service availability in the region

SHOULD:
- Document the issue pattern for future reference
- Check if the issue affects multiple farms or is isolated

MAY:
- Open AWS Support case if issue appears service-side
- Review recent configuration changes that may have caused the issue

## Safety Ratings

```
safety_ratings:
  - "list-farms: GREEN — read-only farm enumeration"
  - "list-fleets: GREEN — read-only fleet enumeration"
  - "list-queues: GREEN — read-only queue enumeration"
  - "cloudtrail lookup-events: GREEN — read-only audit log"
  - "Any remediation action: varies — refer to specific runbook"
```

## Escalation Conditions

- Issue affects all Deadline Cloud resources in the account
- Problem persists after following specific runbook remediation
- Issue appears to be an AWS service-side problem
- Multiple unrelated Deadline Cloud features failing simultaneously

## Data Sensitivity

- Farm/fleet/queue metadata: LOW — resource identifiers and status
- CloudTrail events: MEDIUM — reveals IAM principals and API actions
- Job details: MEDIUM — reveals render pipeline workload
- Worker logs: MEDIUM — contain session and error details

## Prohibited Actions

- NEVER delete farms, fleets, or queues as a troubleshooting step
- NEVER terminate workers with running sessions during investigation
- NEVER modify production configurations without a rollback plan
- NEVER assume a service-side issue without ruling out configuration problems

## Phase 3 — Rollback

1. If any resources were modified during troubleshooting, revert to pre-investigation state
2. Document all changes made during investigation for audit trail
3. If wrong runbook was followed, undo all remediation steps
4. Verify all farms, fleets, and queues are in expected state after investigation

## Common Issues

- symptoms: "All Deadline Cloud API calls return AccessDeniedException"
  diagnosis: "IAM permissions missing or SCP blocking Deadline Cloud actions."
  resolution: "Verify IAM policy includes deadline:* permissions. Check SCPs for deny rules."

- symptoms: "Jobs submitted but nothing happens"
  diagnosis: "Queue has no fleet association or fleet has no workers."
  resolution: "Check queue-fleet associations. Verify fleet status and worker count."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<triage findings and domain identification>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement comprehensive Deadline Cloud monitoring"
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
