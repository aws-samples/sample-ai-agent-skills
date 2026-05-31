---
title: "D1 — Subscription Filter Policy Issues"
description: "Diagnose subscription filter policy configuration and matching problems"
status: active
severity: MEDIUM
triggers:
  - "filter policy"
  - "message filtering"
  - "subscription filter"
  - "messages not received"
owner: devops-agent
objective: "Resolve subscription filter policy configuration and matching issues"
context: "Filter policies allow subscriptions to receive only messages matching specific criteria. By default, policies match against message attributes (not body). FilterPolicyScope must be set to MessageBody for body-based filtering."
---

## Phase 1 — Triage

MUST:
- Check subscription filter policy: `aws sns get-subscription-attributes --subscription-arn <sub-arn> --query 'Attributes.FilterPolicy'`
- Check filter policy scope: `aws sns get-subscription-attributes --subscription-arn <sub-arn> --query 'Attributes.FilterPolicyScope'`
- Verify message attributes match the filter policy
- Check NumberOfNotificationsFilteredOut metric: `aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfNotificationsFilteredOut --dimensions Name=TopicName,Value=<topic-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`

SHOULD:
- Test filter policy with a sample message
- Verify attribute names and values are exact matches (case-sensitive)
- Check for filter policy syntax errors

MAY:
- Review filter policy complexity (nested conditions)
- Check for filter policy size limits (256 KB)

## Phase 2 — Remediate

MUST:
- If scope wrong: set FilterPolicyScope to MessageBody for body filtering: `aws sns set-subscription-attributes --subscription-arn <sub-arn> --attribute-name FilterPolicyScope --attribute-value MessageBody`
- If policy syntax wrong: fix JSON syntax in the filter policy
- If attributes not matching: verify publisher sends correct attribute names and types

SHOULD:
- Use exact match for simple filtering, prefix/numeric for complex
- Test filter policies with sample messages before production
- Document filter policy logic for each subscription

MAY:
- Implement filter policy versioning
- Use CloudWatch metrics to monitor filter effectiveness

## Common Issues

- symptoms: "Subscriber not receiving messages despite matching content"
  diagnosis: "Filter policy matching message attributes but publisher sending body content."
  resolution: "Set FilterPolicyScope to MessageBody, or add message attributes to published messages."

- symptoms: "Filter policy set but all messages delivered"
  diagnosis: "Filter policy is empty {} or matches all messages."
  resolution: "Verify filter policy has specific conditions. Empty policy matches everything."

## Output Format

```yaml
root_cause: "filter_policy — <specific_cause>"
evidence:
  - type: filter_policy
    content: "<subscription filter policy>"
  - type: filter_scope
    content: "<FilterPolicyScope setting>"
severity: MEDIUM
mitigation:
  immediate: "Fix filter policy or scope configuration"
  long_term: "Implement filter policy testing and documentation"
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
