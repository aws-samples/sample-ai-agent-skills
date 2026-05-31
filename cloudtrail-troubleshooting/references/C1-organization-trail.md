---
title: "C1 — Organization Trail"
description: "Diagnose organization trail configuration issues"
status: active
severity: HIGH
triggers:
  - "organization trail error"
  - "org trail not logging"
  - "member account events missing"
  - "central logging"
owner: devops-agent
objective: "Configure and troubleshoot organization-wide CloudTrail logging"
context: "Organization trails log events for all member accounts to a central S3 bucket. They must be created from the management account or delegated admin. Member accounts cannot modify organization trails."
---

## Phase 1 — Triage

MUST:
- Check organization trails: `aws cloudtrail describe-trails --query 'trailList[?IsOrganizationTrail==`true`]'`
- Check trail status: `aws cloudtrail get-trail-status --name <org-trail-name>`
- Verify Organizations integration: `aws organizations describe-organization`
- Check S3 bucket policy allows all member accounts

SHOULD:
- Verify trail is multi-region
- Check if member account events are being delivered
- Verify trusted access for CloudTrail: `aws organizations list-aws-service-access-for-organization --query 'EnabledServicePrincipals[?ServicePrincipal==`cloudtrail.amazonaws.com`]'`

MAY:
- Check if member accounts have their own trails (may duplicate)
- Review S3 bucket structure for member account log paths

## Phase 2 — Remediate

MUST:
- Create org trail from management account: `aws cloudtrail create-trail --name <name> --s3-bucket-name <bucket> --is-organization-trail --is-multi-region-trail`
- Enable trusted access: `aws organizations enable-aws-service-access --service-principal cloudtrail.amazonaws.com`
- Start logging: `aws cloudtrail start-logging --name <trail-name>`

SHOULD:
- Update S3 bucket policy for organization-wide access
- Enable log file validation
- Configure CloudWatch Logs integration

MAY:
- Set up delegated admin for CloudTrail
- Configure separate event selectors for org trail

## Common Issues

- symptoms: "Cannot create organization trail"
  diagnosis: "Not in management account or trusted access not enabled."
  resolution: "Run from management account. Enable CloudTrail trusted access."

- symptoms: "Member account events not in central bucket"
  diagnosis: "S3 bucket policy doesn't allow member account writes."
  resolution: "Update bucket policy to allow CloudTrail from organization accounts."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create organization trail | RED | Affects all member accounts; logs centrally |
| Enable trusted access | YELLOW | Grants CloudTrail cross-account permissions |
| Start logging on org trail | GREEN | Enables event recording |
| Update S3 bucket policy for org | YELLOW | Broad policy change affecting all member accounts |

## Escalation Conditions

- Organization trail creation or modification — escalate to management account administrator and security team
- S3 bucket policy changes for centralized logging — escalate to bucket owner and security team
- Member account events not being delivered — escalate to security operations
- Trusted access changes — escalate to Organizations administrator

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail describe-trails` (org trails) | HIGH | Exposes organization-wide trail configuration |
| `aws organizations describe-organization` | HIGH | Exposes organization structure |
| `aws organizations list-aws-service-access-for-organization` | HIGH | Exposes service integrations |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails, especially organization trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** modify organization trail configuration without management account approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created organization trail | Delete the trail (from management account): `aws cloudtrail delete-trail` |
| Enabled trusted access | Disable trusted access: `aws organizations disable-aws-service-access --service-principal cloudtrail.amazonaws.com` |
| Updated S3 bucket policy | Restore previous bucket policy |
| Started logging | Stop logging (use with extreme caution on org trails) |

## Output Format

```yaml
root_cause: "organization_trail — <specific_cause>"
evidence:
  - type: org_trail
    content: "<organization trail configuration>"
  - type: trusted_access
    content: "<CloudTrail trusted access status>"
severity: HIGH
mitigation:
  immediate: "Fix organization trail configuration"
  long_term: "Automate org trail setup with IaC"
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
