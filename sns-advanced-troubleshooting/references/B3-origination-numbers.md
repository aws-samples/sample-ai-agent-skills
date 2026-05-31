---
title: "B3 — Origination Number Issues"
description: "Diagnose SMS origination number and sender ID problems"
status: active
severity: MEDIUM
triggers:
  - "origination number"
  - "sender ID"
  - "short code"
  - "long code"
  - "toll-free"
owner: devops-agent
objective: "Resolve SMS origination number configuration and delivery issues"
context: "Origination numbers (long codes, short codes, toll-free) and sender IDs affect SMS deliverability. Different countries have different requirements. Some countries require pre-registered sender IDs or specific number types."
---

## Phase 1 — Triage

MUST:
- List origination numbers: `aws sns list-origination-numbers`
- Check SMS attributes for default sender: `aws sns get-sms-attributes`
- Verify origination number status and capabilities
- Check destination country requirements for origination type

SHOULD:
- Verify number is active and not suspended
- Check for 10DLC registration requirements (US)
- Review sender ID registration status (where required)

MAY:
- Check carrier filtering for the origination number
- Review throughput limits for the number type

## Phase 2 — Remediate

MUST:
- If no origination number: purchase one through Amazon Pinpoint or SNS
- If 10DLC required (US): register brand and campaign
- If sender ID required: register in the destination country

SHOULD:
- Use appropriate number type for the use case (short code for high volume)
- Configure default origination number in SMS attributes
- Monitor origination number health and deliverability

MAY:
- Implement number rotation for high-volume sending
- Use Amazon Pinpoint for advanced origination management

## Common Issues

- symptoms: "SMS not delivered to US numbers"
  diagnosis: "10DLC registration required for A2P messaging to US numbers."
  resolution: "Register 10DLC brand and campaign through Amazon Pinpoint."

- symptoms: "Sender ID not showing on recipient's phone"
  diagnosis: "Destination country doesn't support sender IDs or registration required."
  resolution: "Check country-specific requirements. Use origination number instead."

## Output Format

```yaml
root_cause: "origination — <specific_cause>"
evidence:
  - type: origination_numbers
    content: "<number list and status>"
  - type: sms_attributes
    content: "<default sender configuration>"
severity: MEDIUM
mitigation:
  immediate: "Configure appropriate origination number"
  long_term: "Register 10DLC/sender IDs and monitor deliverability"
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
