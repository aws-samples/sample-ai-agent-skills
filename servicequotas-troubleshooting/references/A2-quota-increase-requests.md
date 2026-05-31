---
title: "A2 — Quota Increase Requests"
description: "Submit and troubleshoot quota increase requests"
status: active
severity: HIGH
triggers:
  - "increase quota"
  - "raise limit"
  - "request denied"
  - "quota increase"
owner: devops-agent
objective: "Successfully submit and track quota increase requests"
context: "Quota increases are requested through Service Quotas API or console. Some increases are automatic, others require manual review. Requests can be denied if the increase is too large or the account is new. Check request status regularly."
---

## Phase 1 — Triage

MUST:
- Verify quota is adjustable: `aws service-quotas get-service-quota --service-code <service-code> --quota-code <quota-code> --query 'Quota.Adjustable'`
- Check existing requests: `aws service-quotas list-requested-service-quota-change-history --service-code <service-code>`
- Get current applied value: `aws service-quotas get-service-quota --service-code <service-code> --quota-code <quota-code> --query 'Quota.Value'`
- Check request status: `aws service-quotas get-requested-service-quota-change --request-id <request-id>`

SHOULD:
- Verify the requested value is reasonable (not excessively large)
- Check if a previous request is still pending
- Review account history for the quota

MAY:
- Check if the service supports automatic increases
- Review AWS documentation for recommended limits

## Phase 2 — Remediate

MUST:
- Submit increase request: `aws service-quotas request-service-quota-increase --service-code <service-code> --quota-code <quota-code> --desired-value <value>`
- Monitor request status until approved or denied
- If denied, review the reason and adjust the request

SHOULD:
- Request reasonable increments (2-3x current, not 100x)
- Plan ahead — submit requests before hitting limits
- Document the business justification for large increases

MAY:
- Contact AWS Support for urgent or large increase requests
- Use quota templates for organization-wide defaults

## Common Issues

- symptoms: "Request denied"
  diagnosis: "Requested value too high or account too new."
  resolution: "Request a smaller increase. Build usage history. Contact support for justification."

- symptoms: "Request stuck in PENDING"
  diagnosis: "Requires manual review by AWS service team."
  resolution: "Wait 1-5 business days. Contact support if urgent."

- symptoms: "Cannot request increase — quota not adjustable"
  diagnosis: "This is a hard limit."
  resolution: "Architectural change needed (multi-region, multi-account, different approach)."

## Output Format

```yaml
root_cause: "quota_increase — <specific_cause>"
evidence:
  - type: current_quota
    content: "<current applied value>"
  - type: request_status
    content: "<increase request status>"
severity: HIGH
mitigation:
  immediate: "Submit or fix the increase request"
  long_term: "Plan quota increases ahead of growth"
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
