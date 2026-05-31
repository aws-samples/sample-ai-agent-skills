---
title: "C2 — Entitlement Issues"
description: "Diagnose MediaConnect entitlement sharing and subscription problems"
status: active
severity: HIGH
triggers: ["entitlement not working", "cannot subscribe", "entitlement error", "cross-account sharing", "GrantEntitlement failed"]
owner: devops-agent
objective: "Resolve entitlement creation, sharing, and subscription issues"
context: "Entitlements share flow content between AWS accounts. The granter creates an entitlement, the subscriber creates a flow using the entitlement ARN as source."
---

## Phase 1 — Triage

MUST:
- List entitlements on the flow: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Entitlements[*].{Name:Name,EntitlementArn:EntitlementArn,Subscribers:Subscribers,Status:EntitlementStatus}'`
- List all entitlements in the account: `aws mediaconnect list-entitlements`
- Verify the subscriber account ID is correct in the entitlement

SHOULD:
- Check if the entitlement is ENABLED
- Verify the subscriber has created a flow using the entitlement ARN
- Check data transfer subscriber fee percentage

MAY:
- Review CloudTrail for GrantFlowEntitlements errors
- Verify the subscriber account exists and is active

## Phase 2 — Remediate

MUST:
- Grant entitlement if missing: `aws mediaconnect grant-flow-entitlements --flow-arn <flow-arn> --entitlements Subscribers=<subscriber-account-id>,Description="<description>",DataTransferSubscriberFeePercent=0`
- Enable entitlement if disabled: `aws mediaconnect update-flow-entitlement --flow-arn <flow-arn> --entitlement-arn <entitlement-arn> --entitlement-status ENABLED`
- Verify subscriber account ID is correct (12-digit AWS account number)

SHOULD:
- Set appropriate data transfer subscriber fee percentage
- Add encryption to the entitlement if content requires protection

MAY:
- Notify the subscriber account of the entitlement ARN
- Configure entitlement with encryption: `aws mediaconnect update-flow-entitlement --flow-arn <flow-arn> --entitlement-arn <entitlement-arn> --encryption Algorithm=aes256,RoleArn=<role-arn>,SecretArn=<secret-arn>`

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow inspection"
  - "list-entitlements: GREEN — read-only entitlement enumeration"
  - "grant-flow-entitlements: YELLOW — shares content with another account"
  - "revoke-flow-entitlement: RED — immediately stops content sharing with subscriber"
```

## Escalation Conditions

- Subscriber cannot create a flow from the entitlement despite correct ARN
- Entitlement shows ENABLED but subscriber flow receives no content
- Cross-account encryption key sharing fails
- Data transfer costs are unexpectedly high due to entitlement fee configuration

## Data Sensitivity

- `aws mediaconnect describe-flow`: MEDIUM — contains subscriber account IDs and encryption details
- `aws mediaconnect list-entitlements`: MEDIUM — reveals cross-account sharing relationships
- `aws mediaconnect grant-flow-entitlements`: HIGH — grants content access to another account
- `aws mediaconnect revoke-flow-entitlement`: HIGH — revokes content access from another account

## Prohibited Actions

- NEVER grant entitlements to unknown or unverified AWS account IDs
- NEVER revoke a production entitlement without notifying the subscriber
- NEVER set DataTransferSubscriberFeePercent without understanding billing implications
- NEVER share unencrypted entitlements for content requiring DRM or content protection

## Phase 3 — Rollback

1. If entitlement was granted to wrong account, revoke it: `aws mediaconnect revoke-flow-entitlement --flow-arn <flow-arn> --entitlement-arn <entitlement-arn>`
2. If encryption was misconfigured, update: `aws mediaconnect update-flow-entitlement --flow-arn <flow-arn> --entitlement-arn <entitlement-arn> --encryption Algorithm=aes256,RoleArn=<role-arn>,SecretArn=<secret-arn>`
3. If subscriber fee was set incorrectly, update: `aws mediaconnect update-flow-entitlement --flow-arn <flow-arn> --entitlement-arn <entitlement-arn> --data-transfer-subscriber-fee-percent <correct-percent>`
4. Notify the subscriber of any entitlement changes
5. Document the entitlement change and update the content sharing agreement

## Common Issues

- symptoms: "Subscriber cannot create flow from entitlement ARN"
  diagnosis: "Subscriber account ID does not match the entitlement, or entitlement is DISABLED."
  resolution: "Verify subscriber account ID in entitlement. Ensure entitlement status is ENABLED."

- symptoms: "Entitlement granted but subscriber flow shows no source bitrate"
  diagnosis: "Granter flow is not ACTIVE, or source has no content."
  resolution: "Verify granter flow is ACTIVE with healthy source bitrate. Subscriber flow inherits source health."

- symptoms: "GrantFlowEntitlements returns LimitExceededException"
  diagnosis: "Maximum entitlements per flow reached."
  resolution: "Check entitlement count. Remove unused entitlements or request quota increase."

## Output Format

```yaml
root_cause: "entitlement — <specific_cause>"
evidence:
  - type: entitlement_config
    content: "<entitlement details and subscriber info>"
severity: HIGH
mitigation:
  immediate: "Fix entitlement configuration"
  long_term: "Document entitlement sharing agreements and automate via IaC"
```

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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
