---
title: "F2 — Topic Access Policy Issues"
description: "Diagnose SNS topic access policy configuration problems"
status: active
severity: HIGH
triggers:
  - "access policy"
  - "topic policy"
  - "publish denied"
  - "subscribe denied"
owner: devops-agent
objective: "Resolve SNS topic access policy configuration and permission issues"
context: "The topic access policy (resource policy) controls who can publish to, subscribe to, and manage the topic. It works alongside IAM policies. Both must allow the action for it to succeed. Overly restrictive policies block legitimate access."
---

## Phase 1 — Triage

MUST:
- Get topic policy: `aws sns get-topic-attributes --topic-arn <topic-arn> --query 'Attributes.Policy'`
- Parse and review the policy JSON
- Identify which principal and action is being denied
- Check CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=sns.amazonaws.com --max-results 10`

SHOULD:
- Verify the policy allows required principals (accounts, services, roles)
- Check for explicit Deny statements that may override Allow
- Verify condition keys are correct (aws:SourceArn, aws:SourceAccount)

MAY:
- Use IAM Policy Simulator to test access
- Compare with a known-good topic policy

## Phase 2 — Remediate

MUST:
- If publish denied: add sns:Publish permission for the publishing principal
- If subscribe denied: add sns:Subscribe permission for the subscribing principal
- If service integration: add the service principal (e.g., s3.amazonaws.com, events.amazonaws.com)

SHOULD:
- Use condition keys to restrict access (aws:SourceArn for service integrations)
- Follow least-privilege principle
- Test policy changes before applying to production topics

MAY:
- Implement policy versioning and change management
- Use AWS Organizations SCPs for additional controls

## Common Issues

- symptoms: "S3 event notification to SNS fails with AccessDenied"
  diagnosis: "Topic policy missing sns:Publish for s3.amazonaws.com with SourceArn condition."
  resolution: "Add sns:Publish for Principal: s3.amazonaws.com with aws:SourceArn condition."

- symptoms: "EventBridge cannot publish to SNS topic"
  diagnosis: "Topic policy missing sns:Publish for events.amazonaws.com."
  resolution: "Add sns:Publish for Principal: events.amazonaws.com."

## Output Format

```yaml
root_cause: "access_policy — <specific_cause>"
evidence:
  - type: topic_policy
    content: "<current topic policy>"
  - type: denied_action
    content: "<specific denied principal and action>"
severity: HIGH
mitigation:
  immediate: "Fix topic access policy"
  long_term: "Implement policy management and least-privilege"
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
