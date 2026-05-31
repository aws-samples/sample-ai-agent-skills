---
title: "B2 — SMS Spending Limits"
description: "Diagnose SMS spending limit and cost management issues"
status: active
severity: MEDIUM
triggers:
  - "spending limit"
  - "SMS cost"
  - "monthly limit"
  - "SMS budget"
owner: devops-agent
objective: "Resolve SMS spending limit issues and optimize SMS costs"
context: "AWS accounts have a default SMS spending limit of $1.00/month. This limit applies to all SMS sent from the account across all topics and direct publish. When the limit is reached, all SMS sending stops until the next month or the limit is increased."
---

## Phase 1 — Triage

MUST:
- Check current spending limit: `aws sns get-sms-attributes --attributes MonthlySpendLimit`
- Check SMS delivery metrics for the month
- Verify if limit has been reached (SMS stops when limit is hit)
- Check CloudWatch for SMS spending metrics

SHOULD:
- Calculate current month's SMS spend
- Review SMS sending patterns and volume
- Check for unexpected SMS volume spikes

MAY:
- Analyze SMS costs by destination country
- Review SMS type distribution (Transactional vs Promotional)

## Phase 2 — Remediate

MUST:
- If limit reached: request spending limit increase through AWS Support
- Set appropriate limit based on expected volume
- Monitor spending to avoid unexpected cutoffs

SHOULD:
- Set up CloudWatch alarms for SMS spending approaching the limit
- Use Transactional type only for critical messages (higher cost)
- Use Promotional type for marketing (lower cost, may be filtered)

MAY:
- Implement SMS cost allocation by application or team
- Use Amazon Pinpoint for advanced SMS cost management

## Common Issues

- symptoms: "All SMS stopped mid-month"
  diagnosis: "Monthly spending limit reached."
  resolution: "Request limit increase. Set up spending alerts."

- symptoms: "SMS costs higher than expected"
  diagnosis: "Sending to international numbers or using Transactional type for all messages."
  resolution: "Use Promotional type where appropriate. Review destination countries."

## Output Format

```yaml
root_cause: "spending_limit — <specific_cause>"
evidence:
  - type: spending_limit
    content: "<current limit and usage>"
  - type: sms_metrics
    content: "<SMS volume and costs>"
severity: MEDIUM
mitigation:
  immediate: "Increase spending limit"
  long_term: "Implement spending monitoring and cost optimization"
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

- Message payloads may contain sensitive application data
- Subscription endpoints reveal integration architecture
- SMS messages may contain customer PII or authentication codes
- Classification: CONFIDENTIAL - message content and subscriber endpoint data

## Prohibited Actions

- NEVER delete SNS topics with active subscriptions in production
- NEVER modify filter policies on production subscriptions without testing
- NEVER change access policies that could disrupt cross-account message delivery
- NEVER purge dead-letter queues without analyzing failed messages

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
