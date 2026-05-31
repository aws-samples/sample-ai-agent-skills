---
title: "F1 — Cross-Account Subscription Issues"
description: "Diagnose cross-account SNS subscription and delivery problems"
status: active
severity: HIGH
triggers:
  - "cross-account"
  - "another account"
  - "subscription denied"
  - "cross-account subscribe"
owner: devops-agent
objective: "Resolve cross-account subscription creation and message delivery issues"
context: "Cross-account subscriptions require the topic policy to allow the subscribing account. The subscription must be confirmed by the endpoint owner. Both the topic policy and IAM permissions must be correctly configured."
---

## Phase 1 — Triage

MUST:
- Check topic policy: `aws sns get-topic-attributes --topic-arn <topic-arn> --query 'Attributes.Policy'`
- Verify the policy allows sns:Subscribe from the other account
- Check subscription status: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Look for PendingConfirmation subscriptions

SHOULD:
- Verify the subscribing account's IAM permissions
- Check if the endpoint (SQS, Lambda) has a resource policy allowing SNS
- Verify subscription confirmation was completed

MAY:
- Check CloudTrail for Subscribe API calls
- Review cross-account IAM role configuration

## Phase 2 — Remediate

MUST:
- If topic policy missing: add sns:Subscribe permission for the subscribing account
- If subscription pending: confirm from the endpoint owner account
- If SQS cross-account: add SNS publish permission to the SQS queue policy

SHOULD:
- Use specific account IDs in the topic policy (not *)
- Implement least-privilege topic policies
- Document cross-account subscription architecture

MAY:
- Use AWS Organizations for simplified cross-account access
- Implement automated subscription management

## Common Issues

- symptoms: "Subscribe call returns AccessDenied"
  diagnosis: "Topic policy does not allow sns:Subscribe from the subscribing account."
  resolution: "Add the subscribing account to the topic policy with sns:Subscribe permission."

- symptoms: "Subscription created but messages not delivered"
  diagnosis: "Subscription in PendingConfirmation state — not yet confirmed."
  resolution: "Confirm the subscription from the endpoint owner account."

## Output Format

```yaml
root_cause: "cross_account — <specific_cause>"
evidence:
  - type: topic_policy
    content: "<topic access policy>"
  - type: subscription_status
    content: "<subscription confirmation state>"
severity: HIGH
mitigation:
  immediate: "Fix topic policy and confirm subscription"
  long_term: "Implement cross-account access management"
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
