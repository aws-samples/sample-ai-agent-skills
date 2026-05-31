---
title: "D2 — Attribute-Based Filtering Issues"
description: "Diagnose message attribute configuration and filtering problems"
status: active
severity: MEDIUM
triggers:
  - "message attributes"
  - "attribute filtering"
  - "attribute type"
  - "string array"
owner: devops-agent
objective: "Resolve message attribute configuration and filter matching issues"
context: "Message attributes are key-value pairs attached to SNS messages. Filter policies match against these attributes. Attributes have types (String, Number, String.Array). Type mismatches cause filter failures."
---

## Phase 1 — Triage

MUST:
- Check subscription filter policy: `aws sns get-subscription-attributes --subscription-arn <sub-arn> --query 'Attributes.FilterPolicy'`
- Verify message attributes being published match filter expectations
- Check attribute data types (String, Number, String.Array)
- Verify FilterPolicyScope is MessageAttributes (default)

SHOULD:
- Test publishing with explicit message attributes
- Check for case sensitivity in attribute names and values
- Verify numeric comparisons use Number type attributes

MAY:
- Review filter policy operators (exact, prefix, anything-but, numeric, exists)
- Check for attribute size limits

## Phase 2 — Remediate

MUST:
- If type mismatch: ensure published attribute type matches filter expectation
- If name mismatch: verify attribute names are identical (case-sensitive)
- If numeric filter on String: change attribute type to Number

SHOULD:
- Use consistent attribute naming conventions
- Document attribute schema for publishers and subscribers
- Test attribute-based filtering end-to-end

MAY:
- Implement attribute validation in the publishing application
- Use String.Array for multi-value attributes

## Common Issues

- symptoms: "Numeric filter not matching"
  diagnosis: "Attribute published as String type but filter uses numeric comparison."
  resolution: "Publish attribute with DataType: Number for numeric filtering."

- symptoms: "Filter matches some messages but not others"
  diagnosis: "Inconsistent attribute names or values from different publishers."
  resolution: "Standardize attribute names and values across all publishers."

## Output Format

```yaml
root_cause: "attribute_filtering — <specific_cause>"
evidence:
  - type: filter_policy
    content: "<filter policy>"
  - type: message_attributes
    content: "<published attributes>"
severity: MEDIUM
mitigation:
  immediate: "Fix attribute types or filter policy"
  long_term: "Implement attribute schema documentation and validation"
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
