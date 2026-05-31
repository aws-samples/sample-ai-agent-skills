---
title: "E2 — Service Quotas API"
description: "Troubleshoot Service Quotas API usage issues"
status: active
severity: LOW
triggers:
  - "API error"
  - "service not found"
  - "quota code unknown"
  - "API throttling"
owner: devops-agent
objective: "Resolve Service Quotas API usage issues"
context: "The Service Quotas API provides programmatic access to quota management. Issues include unknown service/quota codes, API throttling, permission errors, and services not yet integrated with the API."
---

## Phase 1 — Triage

MUST:
- Verify service is supported: `aws service-quotas list-services --query 'Services[?ServiceCode==`<code>`]'`
- Check quota code: `aws service-quotas list-service-quotas --service-code <code> --query 'Quotas[*].{Name:QuotaName,Code:QuotaCode}'`
- Verify IAM permissions for Service Quotas API
- Check for API throttling errors

SHOULD:
- Review error message details
- Check API call rate against throttling limits
- Verify region is correct for the API call

MAY:
- Check CloudTrail for API call failures
- Review SDK/CLI version for compatibility

## Phase 2 — Remediate

MUST:
- Use correct service code and quota code
- Grant required IAM permissions (servicequotas:*)
- Implement retry logic for throttling

SHOULD:
- Cache quota values to reduce API calls
- Use list-services to discover correct service codes
- Implement exponential backoff for retries

MAY:
- Use AWS SDK pagination for large result sets
- Create a local quota code reference

## Common Issues

- symptoms: "NoSuchResourceException"
  diagnosis: "Service code or quota code is incorrect."
  resolution: "Use list-services and list-service-quotas to find correct codes."

- symptoms: "Service not in list-services"
  diagnosis: "Service not yet integrated with Service Quotas."
  resolution: "Use the service's own API or AWS Support for quota management."

- symptoms: "TooManyRequestsException"
  diagnosis: "API call rate exceeded throttling limit."
  resolution: "Implement exponential backoff. Cache results. Reduce call frequency."

## Output Format

```yaml
root_cause: "api_issue — <specific_cause>"
evidence:
  - type: error_message
    content: "<API error details>"
  - type: service_code
    content: "<service and quota codes>"
severity: LOW
mitigation:
  immediate: "Fix API call parameters or permissions"
  long_term: "Implement proper error handling and caching"
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
