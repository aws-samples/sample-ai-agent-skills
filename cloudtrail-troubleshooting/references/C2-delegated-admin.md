---
title: "C2 — Delegated Admin"
description: "Diagnose CloudTrail delegated administrator issues"
status: active
severity: MEDIUM
triggers:
  - "delegated admin CloudTrail"
  - "cannot manage org trail"
  - "admin permissions"
owner: devops-agent
objective: "Configure CloudTrail delegated administrator for Organizations"
context: "CloudTrail supports delegated administrators who can manage organization trails and CloudTrail Lake event data stores on behalf of the management account."
---

## Phase 1 — Triage

MUST:
- Check delegated admins: `aws organizations list-delegated-administrators --service-principal cloudtrail.amazonaws.com`
- Verify caller account role
- Check CloudTrail trusted access status

SHOULD:
- Verify delegated admin permissions
- Check if org trails exist under delegated admin

MAY:
- Review CloudTrail for admin-related events
- Check IAM policies on delegated admin account

## Phase 2 — Remediate

MUST:
- Register delegated admin (from management account): `aws organizations register-delegated-administrator --account-id <id> --service-principal cloudtrail.amazonaws.com`
- Enable trusted access if not already enabled

SHOULD:
- Verify delegated admin can manage org trails
- Document delegated admin configuration

MAY:
- Set up CloudTrail Lake under delegated admin
- Configure cross-account access for log analysis

## Common Issues

- symptoms: "Cannot register delegated admin"
  diagnosis: "Not running from management account."
  resolution: "Run register-delegated-administrator from management account."

- symptoms: "Delegated admin cannot create org trail"
  diagnosis: "Trusted access not enabled or insufficient permissions."
  resolution: "Enable trusted access. Verify delegated admin IAM permissions."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Register delegated admin | YELLOW | Grants account organization-wide CloudTrail management |
| Enable trusted access | YELLOW | Grants CloudTrail cross-account permissions |
| Verify delegated admin permissions | GREEN | Read-only verification |

## Escalation Conditions

- Delegated admin registration or changes — escalate to management account administrator
- Trusted access changes — escalate to Organizations administrator
- Delegated admin account compromise suspected — escalate immediately to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws organizations list-delegated-administrators` | HIGH | Exposes delegated admin account IDs |
| `aws cloudtrail describe-trails` | HIGH | Exposes trail configuration |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** register a delegated admin without management account approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Registered delegated admin | Deregister: `aws organizations deregister-delegated-administrator --account-id <id> --service-principal cloudtrail.amazonaws.com` |
| Enabled trusted access | Disable: `aws organizations disable-aws-service-access --service-principal cloudtrail.amazonaws.com` |

## Output Format

```yaml
root_cause: "delegated_admin — <specific_cause>"
evidence:
  - type: admin_status
    content: "<delegated admin configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix delegated admin registration"
  long_term: "Document admin responsibilities"
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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
