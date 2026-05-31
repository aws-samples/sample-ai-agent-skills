---
title: "A3 — Applied vs Default Quotas"
description: "Understand the difference between applied and default quota values"
status: active
severity: LOW
triggers:
  - "applied quota"
  - "default quota"
  - "quota mismatch"
  - "different limit values"
owner: devops-agent
objective: "Clarify the difference between applied and default quotas"
context: "Applied quota is the current effective limit for the account. Default quota is what new accounts receive. They differ when increases have been granted. Always use applied quota for capacity planning."
---

## Phase 1 — Triage

MUST:
- Get applied quota: `aws service-quotas get-service-quota --service-code <service-code> --quota-code <quota-code>`
- Get default quota: `aws service-quotas get-aws-default-service-quota --service-code <service-code> --quota-code <quota-code>`
- Compare values
- Check increase history: `aws service-quotas list-requested-service-quota-change-history --service-code <service-code> --query 'RequestedQuotas[?QuotaCode==`<quota-code>`]'`

SHOULD:
- Document both values for reference
- Check if the applied value meets current needs
- Verify the applied value in the correct region

MAY:
- Compare applied quotas across multiple accounts
- Check if organization templates set different defaults

## Phase 2 — Remediate

MUST:
- Use applied quota (not default) for all capacity planning
- Request increase if applied quota is insufficient

SHOULD:
- Document quota values and increase history
- Set up monitoring based on applied quota values

MAY:
- Create a quota comparison report across accounts
- Use organization templates to standardize new account quotas

## Common Issues

- symptoms: "Documentation says limit is X but I can use more"
  diagnosis: "Previous quota increase raised the applied value above default."
  resolution: "Use get-service-quota for the current applied value."

- symptoms: "New account has lower limits than expected"
  diagnosis: "New accounts start with default quotas."
  resolution: "Request increases for the new account or use organization templates."

## Output Format

```yaml
root_cause: "applied_vs_default — <specific_cause>"
evidence:
  - type: applied_value
    content: "<current applied quota>"
  - type: default_value
    content: "<default quota>"
  - type: increase_history
    content: "<previous increase requests>"
severity: LOW
mitigation:
  immediate: "Use applied quota for planning"
  long_term: "Document and monitor quota values"
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
