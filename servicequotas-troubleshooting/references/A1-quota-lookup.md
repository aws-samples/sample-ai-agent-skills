---
title: "A1 — Quota Lookup"
description: "Look up current service quota values"
status: active
severity: LOW
triggers:
  - "what is my quota"
  - "check limit"
  - "current quota"
  - "service limit"
owner: devops-agent
objective: "Find the current applied quota value for a specific service and resource"
context: "Service Quotas provides a centralized way to view and manage quotas across AWS services. The applied quota may differ from the default if increases were previously granted. Not all services are available through Service Quotas API."
---

## Phase 1 — Triage

MUST:
- List available services: `aws service-quotas list-services --query 'Services[?ServiceCode==`<service-code>`]'`
- Get current applied quota: `aws service-quotas get-service-quota --service-code <service-code> --quota-code <quota-code>`
- Get default quota: `aws service-quotas get-aws-default-service-quota --service-code <service-code> --quota-code <quota-code>`
- List all quotas for a service: `aws service-quotas list-service-quotas --service-code <service-code>`

SHOULD:
- Check if the quota is adjustable (Adjustable field)
- Check if a usage metric exists (UsageMetric field)
- Compare applied vs default values

MAY:
- Check quota in multiple regions if needed
- Look up quota code in Service Quotas documentation

## Phase 2 — Remediate

MUST:
- Use the applied quota value (not default) for capacity planning
- Note if the quota is adjustable for future increase requests

SHOULD:
- Document current quotas for the account
- Set up monitoring for quotas approaching limits

MAY:
- Create a quota inventory across all regions
- Automate quota checks with scripts

## Common Issues

- symptoms: "Service not found in Service Quotas"
  diagnosis: "Service not yet integrated with Service Quotas."
  resolution: "Use the service's own API or console to check limits."

- symptoms: "Quota value seems wrong"
  diagnosis: "Looking at default instead of applied quota."
  resolution: "Use get-service-quota for current applied value."

- symptoms: "Don't know the quota code"
  diagnosis: "Need to find the specific quota code."
  resolution: "Use list-service-quotas to list all quotas for the service."

## Output Format

```yaml
root_cause: "quota_lookup — <specific_query>"
evidence:
  - type: applied_quota
    content: "<current applied quota value>"
  - type: default_quota
    content: "<default quota value>"
  - type: adjustable
    content: "<whether quota is adjustable>"
severity: LOW
mitigation:
  immediate: "Use the applied quota value for planning"
  long_term: "Set up quota monitoring and alerting"
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
