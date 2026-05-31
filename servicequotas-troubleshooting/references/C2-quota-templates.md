---
title: "C2 — Quota Templates"
description: "Configure and troubleshoot organization quota templates"
status: active
severity: LOW
triggers:
  - "quota template"
  - "default quotas new accounts"
  - "template not working"
  - "template configuration"
owner: devops-agent
objective: "Configure quota templates for consistent new account provisioning"
context: "Quota templates set default quota values for new accounts joining the organization. They must be associated with the organization and contain specific quota entries. Templates do not affect existing accounts."
---

## Phase 1 — Triage

MUST:
- Check template association: `aws service-quotas get-association-for-service-quota-template`
- List template entries: `aws service-quotas list-service-quota-increase-requests-in-template`
- Verify Organizations is enabled

SHOULD:
- Check if template entries cover required quotas
- Verify desired values are appropriate
- Check if new accounts are receiving template values

MAY:
- Compare template values with actual new account quotas
- Review template change history

## Phase 2 — Remediate

MUST:
- Associate template: `aws service-quotas associate-service-quota-template`
- Add entries: `aws service-quotas put-service-quota-increase-request-into-template --service-code <code> --quota-code <code> --desired-value <value> --aws-region <region>`

SHOULD:
- Add entries for all critical quotas
- Test by creating a new account and verifying quotas
- Document template configuration

MAY:
- Remove unnecessary entries: `aws service-quotas delete-service-quota-increase-request-from-template --service-code <code> --quota-code <code> --aws-region <region>`
- Disassociate template if no longer needed

## Common Issues

- symptoms: "Template not applying to new accounts"
  diagnosis: "Template not associated with organization."
  resolution: "Run associate-service-quota-template."

- symptoms: "Some quotas not in template"
  diagnosis: "Quota entries not added for all required quotas."
  resolution: "Add entries for each required quota and region."

## Output Format

```yaml
root_cause: "quota_template — <specific_cause>"
evidence:
  - type: template_association
    content: "<association status>"
  - type: template_entries
    content: "<template quota entries>"
severity: LOW
mitigation:
  immediate: "Fix template association and entries"
  long_term: "Maintain template as part of account provisioning"
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
