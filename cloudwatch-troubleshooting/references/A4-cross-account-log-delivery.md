---
title: "A4 — Cross-Account Log Delivery"
description: "Diagnose cross-account log delivery failures"
status: active
severity: HIGH
triggers:
  - "cross-account logs"
  - "log delivery other account"
  - "destination policy"
  - "cross-account subscription"
owner: devops-agent
objective: "Identify why logs are not being delivered from a source account to a destination in another account"
context: "Cross-account log delivery uses subscription filters with a destination in another account. The destination account must create a destination with a resource policy allowing the source account. Alternatively, CloudWatch Observability Access Manager (OAM) can share logs across accounts."
---

## Phase 1 — Triage

MUST:
- Check subscription filter in source account: `aws logs describe-subscription-filters --log-group-name <name>`
- Verify destination exists in destination account: `aws logs describe-destinations --destination-name-prefix <name>`
- Check destination access policy allows the source account
- Verify the destination's target (Kinesis stream) is active

SHOULD:
- Check OAM links if using Observability Access Manager: `aws oam list-links`
- Verify IAM roles in both accounts have correct trust policies
- Check CloudTrail in both accounts for API errors

MAY:
- Test with a same-account subscription filter first to isolate cross-account issues
- Check network connectivity if using VPC endpoints

## Phase 2 — Remediate

MUST:
- Create or fix the destination access policy to allow the source account
- Ensure the destination's Kinesis stream IAM role is correct
- Verify the subscription filter references the correct destination ARN

SHOULD:
- Use OAM for simpler cross-account log sharing setup
- Validate the Kinesis stream in the destination account is not throttled

MAY:
- Set up monitoring on the destination for delivery metrics
- Document the cross-account architecture for operational clarity

## Common Issues

- symptoms: "InvalidParameterException when creating cross-account subscription filter"
  diagnosis: "Destination access policy does not allow the source account."
  resolution: "Update the destination access policy to include the source account ID."

- symptoms: "Subscription filter created but no logs in destination account"
  diagnosis: "Destination Kinesis stream IAM role cannot write to the stream."
  resolution: "Fix the IAM role trust policy and permissions for the Kinesis stream."

- symptoms: "OAM link created but logs not visible in monitoring account"
  diagnosis: "OAM link does not include CloudWatch Logs as a shared resource type."
  resolution: "Update the OAM link to include AWS::Logs::LogGroup in resource types."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix destination access policy | YELLOW | Cross-account policy changes |
| Fix Kinesis stream IAM role | YELLOW | IAM changes in destination account |
| Create OAM link | GREEN | Establishes cross-account sharing |
| Update OAM link resource types | GREEN | Additive configuration |

## Escalation Conditions

- Cross-account log delivery broken for centralized security monitoring — escalate to security operations
- Destination account policy changes required — escalate to destination account owner
- OAM configuration changes — escalate to monitoring account administrator
- Cross-account IAM role changes — escalate to IAM administrators in both accounts

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws logs describe-subscription-filters` | MEDIUM | Exposes filter and destination configuration |
| `aws logs describe-destinations` | HIGH | Exposes cross-account destination policies |
| `aws oam list-links` | MEDIUM | Exposes cross-account sharing configuration |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** modify cross-account policies without coordinating with both account owners

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Fixed destination access policy | Restore previous destination policy |
| Fixed Kinesis stream IAM role | Revert IAM role changes |
| Created OAM link | Delete link: `aws oam delete-link --identifier <link-arn>` |
| Updated OAM link resource types | Revert resource types |

## Output Format

```yaml
root_cause: "cross_account_log_delivery — <specific_cause>"
evidence:
  - type: subscription_filter
    content: "<source account filter config>"
  - type: destination_policy
    content: "<destination access policy>"
severity: HIGH
mitigation:
  immediate: "Fix destination policy, IAM roles, or OAM configuration"
  long_term: "Use OAM for centralized cross-account observability"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
