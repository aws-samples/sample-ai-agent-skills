---
title: "Z1 — General SNS Advanced Troubleshooting (Catch-All)"
description: "Fallback SOP for SNS issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown SNS issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check topic attributes: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- List subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Check delivery metrics: `aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfMessagesPublished --dimensions Name=TopicName,Value=<topic-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Check failure metrics: `aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfNotificationsFailed --dimensions Name=TopicName,Value=<topic-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Check CloudTrail for recent SNS events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=sns.amazonaws.com --max-results 10`

SHOULD:
- Check delivery status logs if enabled
- Verify subscription confirmation status
- Check SMS attributes if SMS-related
- Check platform applications if push-related

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Message delivery failures → Delivery (A1-A3)
- SMS issues → SMS (B1-B3)
- Mobile push issues → Mobile Push (C1-C2)
- Filter policy issues → Filtering (D1-D2)
- FIFO or ordering issues → FIFO (E1-E2)
- Cross-account or policy issues → Cross-Account (F1-F2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all topic and subscription configurations
- Review delivery status logs for specific errors
- Verify IAM permissions and topic policies
- Check for service-level events on AWS Health Dashboard

SHOULD:
- Test with a simple publish/subscribe to isolate the issue
- Compare with known-good topic configuration
- Check for recent configuration changes

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<delivery|sms|push|filtering|fifo|cross_account|unknown>"
investigation_path: "topic attributes → subscriptions → metrics → <domain_classification>"
evidence:
  - type: topic_config
    content: "<topic configuration>"
  - type: delivery_metrics
    content: "<published/failed counts>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
