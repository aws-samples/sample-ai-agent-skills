---
title: "Z1 — General CloudTrail Troubleshooting"
description: "Catch-all systematic investigation for CloudTrail issues"
status: active
severity: MEDIUM
triggers:
  - "CloudTrail issue"
  - "CloudTrail problem"
  - "logging not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate CloudTrail issues"
context: "General-purpose investigation framework for CloudTrail issues. Covers trail status, event delivery, configuration, and integrations. Use as starting point, then pivot to specific runbook."
---

## Phase 1 — Triage

MUST:
- List all trails: `aws cloudtrail describe-trails`
- Check trail status: `aws cloudtrail get-trail-status --name <trail-name>`
- Check event selectors: `aws cloudtrail get-event-selectors --trail-name <trail-name>`
- Search recent events: `aws cloudtrail lookup-events --max-results 10`

SHOULD:
- Check for delivery errors in trail status
- Verify S3 bucket accessibility
- Check CloudTrail Lake data stores: `aws cloudtrail list-event-data-stores`
- Review CloudTrail for recent configuration changes

MAY:
- Check Insights configuration
- Review CloudWatch Logs integration
- Check AWS Health Dashboard

## Phase 2 — Remediate

MUST:
- Pivot to specific runbook:
  - Trail → A1 (creation), A2 (delivery), A3 (S3 bucket)
  - Events → B1 (missing), B2 (data events), B3 (management)
  - Organization → C1 (org trail), C2 (delegated admin)
  - Lake → D1 (queries), D2 (data store)
  - Insights → E1 (events), E2 (anomaly detection)
  - Integration → F1 (CloudWatch Logs), F2 (EventBridge)
  - Security → G1 (validation), G2 (encryption)
  - Cost → H1 (data events), H2 (Lake pricing)

SHOULD:
- Check for multiple concurrent issues
- Review recent changes

MAY:
- Set up comprehensive monitoring
- Document findings

## Common Issues

- symptoms: "CloudTrail not working — unclear what's wrong"
  diagnosis: "Follow systematic triage."
  resolution: "Check trail status → delivery → event selectors → integrations."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Pivot to specific runbook | GREEN | Investigation only, no state changes |
| Check trail status and configuration | GREEN | Read-only operations |
| Review recent changes | GREEN | Read-only analysis |

## Escalation Conditions

- CloudTrail not logging in any trail — escalate immediately to security team
- Multiple concurrent issues affecting audit logging — escalate to incident management
- Issue cannot be classified after systematic triage — escalate to AWS Support
- Suspected unauthorized trail modification — escalate to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail describe-trails` | HIGH | Exposes all trail configurations |
| `aws cloudtrail get-trail-status` | HIGH | Exposes trail operational status |
| `aws cloudtrail get-event-selectors` | HIGH | Exposes event logging configuration |
| `aws cloudtrail lookup-events` | HIGH | Exposes API call history |
| `aws cloudtrail list-event-data-stores` | HIGH | Exposes Lake configuration |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** make state-changing remediations during general troubleshooting without first classifying the issue

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| General triage is read-only | No rollback needed for Phase 1 triage |
| Pivot to specific runbook | Follow rollback steps in the specific runbook |

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: trail_status
    content: "<trail status>"
  - type: delivery
    content: "<delivery status>"
severity: MEDIUM
general_analysis:
  trails: "active | stopped | error"
  delivery: "working | failed"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow identified specific runbook"
  long_term: "Implement comprehensive monitoring"
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
