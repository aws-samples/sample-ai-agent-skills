---
title: "C1 — Organization Quotas"
description: "Manage quotas across an AWS Organization"
status: active
severity: MEDIUM
triggers:
  - "organization quotas"
  - "multi-account limits"
  - "org-wide quotas"
  - "account quotas"
owner: devops-agent
objective: "Manage and standardize quotas across organization member accounts"
context: "Organization quota management involves setting templates for new accounts and managing individual account quotas. Templates only affect new accounts. Existing accounts require individual increase requests."
---

## Phase 1 — Triage

MUST:
- Check template association: `aws service-quotas get-association-for-service-quota-template`
- List template items: `aws service-quotas list-service-quota-increase-requests-in-template`
- Check member account quotas (from each account or with cross-account role)
- Verify Organizations integration

SHOULD:
- Compare quotas across member accounts
- Identify accounts with non-standard quotas
- Check if templates are associated with the organization

MAY:
- Review quota increase history across accounts
- Check for accounts hitting limits

## Phase 2 — Remediate

MUST:
- Associate template with organization: `aws service-quotas associate-service-quota-template`
- Add quotas to template: `aws service-quotas put-service-quota-increase-request-into-template --service-code <code> --quota-code <code> --desired-value <value> --aws-region <region>`
- Request individual increases for existing accounts

SHOULD:
- Standardize critical quotas across all accounts
- Document organization quota standards
- Set up monitoring for all member accounts

MAY:
- Automate quota management with Lambda and Organizations events
- Create quota compliance reports

## Common Issues

- symptoms: "New account has wrong quotas"
  diagnosis: "Template not associated or missing quota entries."
  resolution: "Associate template. Add required quotas to template."

- symptoms: "Template not applying to existing accounts"
  diagnosis: "Templates only affect new accounts."
  resolution: "Submit individual increase requests for existing accounts."

## Output Format

```yaml
root_cause: "organization_quotas — <specific_cause>"
evidence:
  - type: template_status
    content: "<template association and entries>"
  - type: account_quotas
    content: "<quota comparison across accounts>"
severity: MEDIUM
mitigation:
  immediate: "Fix template or submit individual requests"
  long_term: "Standardize and automate quota management"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Quota utilization reveals infrastructure scale and capacity
- Service usage patterns may indicate business activity levels
- Organization quota templates reveal multi-account structure
- Classification: INTERNAL - operational capacity data

## Prohibited Actions

- NEVER reduce service quotas below current utilization levels
- NEVER delete CloudWatch alarms monitoring quota utilization
- NEVER modify organization-level quota templates without member account review
- NEVER cancel pending quota increase requests without requester approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
