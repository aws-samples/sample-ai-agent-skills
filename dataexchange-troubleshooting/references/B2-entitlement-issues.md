---
title: "B2 — Entitlement Issues"
description: "Diagnose Data Exchange entitlement and access issues"
status: active
severity: HIGH
triggers: ["not entitled", "access denied to data set", "entitlement expired", "cannot access subscribed data"]
owner: devops-agent
objective: "Resolve entitlement issues including expired subscriptions, revoked access, and permission errors"
context: "Entitlements grant subscribers access to data sets. They are tied to active subscriptions. Expired or cancelled subscriptions revoke entitlements."
---

## Phase 1 — Triage

MUST:
- List entitled data sets: `aws dataexchange list-data-sets --origin ENTITLED`
- Check specific entitlement: `aws dataexchange get-data-set --data-set-id <dataset-id>`
- Verify subscription is active (not expired or cancelled)

SHOULD:
- Check IAM permissions for dataexchange:GetDataSet and dataexchange:ListRevisions
- Verify the data set has finalized revisions available
- Check if entitlement was recently revoked by the provider

MAY:
- Review Marketplace subscription expiry dates
- Check for RAM resource share invitations (Lake Formation/Redshift assets)

## Phase 2 — Remediate

MUST:
- Renew expired subscriptions through Marketplace
- Fix IAM permissions for Data Exchange API access
- Accept pending RAM resource share invitations for Lake Formation/Redshift assets

SHOULD:
- Verify the correct AWS account is entitled (not a different account)
- Check that the data set region matches the subscription region

MAY:
- Contact provider if entitlement was unexpectedly revoked
- Set up subscription renewal reminders

## Safety Ratings

```
safety_ratings:
  - "dataexchange list-data-sets --origin ENTITLED: GREEN — read-only listing"
  - "dataexchange get-data-set: GREEN — read-only data set details"
  - "ram get-resource-share-invitations: GREEN — read-only RAM check"
  - "Marketplace subscription renewal: YELLOW — creates billing commitment"
```

## Escalation Conditions

- Entitlement revoked by provider without notice
- Subscription renewal fails due to billing issues
- RAM resource share invitation expired and cannot be re-sent
- Multiple data sets lose entitlement simultaneously

## Data Sensitivity

- `aws dataexchange list-data-sets`: LOW — returns data set metadata
- `aws dataexchange get-data-set`: LOW — returns data set configuration
- `subscription details`: MEDIUM — contains billing and access information
- `RAM resource share details`: MEDIUM — reveals cross-account sharing configuration

## Prohibited Actions

- NEVER assume entitlement exists without verifying subscription status
- NEVER attempt to access data sets from a different account than the entitled one
- NEVER share entitlement credentials or tokens with unauthorized users
- NEVER ignore entitlement expiry warnings — data access will be lost

## Phase 3 — Rollback

1. If entitlement was lost, verify subscription status and renew if expired
2. If RAM invitation was declined, request provider to re-send
3. If wrong account was entitled, cancel and re-subscribe from correct account
4. Verify data access is restored after entitlement fix
5. Set up monitoring for entitlement expiry

## Common Issues

- symptoms: "Cannot access data set — origin is not ENTITLED"
  diagnosis: "Subscription expired or was cancelled."
  resolution: "Renew subscription through AWS Marketplace."

## Output Format

```yaml
root_cause: "entitlement — <specific_cause>"
evidence:
  - type: entitlement_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore entitlement"
  long_term: "Monitor subscription expiry and set up auto-renewal"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
