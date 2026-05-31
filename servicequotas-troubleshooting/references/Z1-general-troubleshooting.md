---
title: "Z1 — General Service Quotas Troubleshooting"
description: "Catch-all systematic investigation for Service Quotas issues"
status: active
severity: MEDIUM
triggers:
  - "quota issue"
  - "limit problem"
  - "service quota error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Service Quotas issues"
context: "General-purpose investigation framework for Service Quotas issues. Covers quota lookups, increase requests, utilization, and organization management. Use as starting point, then pivot to specific runbook."
---

## Phase 1 — Triage

MUST:
- Identify the service and quota in question
- Check current applied quota: `aws service-quotas get-service-quota --service-code <code> --quota-code <code>`
- Check if quota is adjustable
- Check recent increase requests: `aws service-quotas list-requested-service-quota-change-history --service-code <code>`

SHOULD:
- Check current utilization against quota
- Review if the issue is quota-related or capacity-related
- Check CloudWatch for usage metrics
- Review error messages for specific quota codes

MAY:
- Check organization quota templates
- Review quota management automation
- Check AWS Health Dashboard

## Phase 2 — Remediate

MUST:
- Pivot to specific runbook:
  - Quotas → A1 (lookup), A2 (increase), A3 (applied vs default)
  - Alarms → B1 (CloudWatch alarms), B2 (utilization)
  - Organization → C1 (org quotas), C2 (templates)
  - Service-Specific → D1 (EC2), D2 (Lambda), D3 (VPC)
  - Automation → E1 (automation), E2 (API)

SHOULD:
- Document the specific quota and current values
- Plan for future growth

MAY:
- Set up comprehensive quota monitoring
- Create quota management documentation

## Common Issues

- symptoms: "Hitting a limit but don't know which one"
  diagnosis: "Need to identify the specific quota from error message."
  resolution: "Check error message for quota code. Use list-service-quotas to find it."

- symptoms: "Quota increase not taking effect"
  diagnosis: "Request still pending or applied to wrong region."
  resolution: "Check request status. Verify region. Wait for approval."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: quota_value
    content: "<current quota>"
  - type: utilization
    content: "<current usage>"
severity: MEDIUM
general_analysis:
  quota_status: "within limits | approaching | exceeded"
  adjustable: "yes | no"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow identified specific runbook"
  long_term: "Implement proactive quota management"
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
