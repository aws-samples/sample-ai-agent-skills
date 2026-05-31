---
title: "B2 — Grant Acceptance Issues"
description: "Diagnose license grant acceptance and activation issues"
status: active
severity: HIGH
triggers: ["grant acceptance failed", "cannot accept grant", "grant pending", "grant not active"]
owner: devops-agent
objective: "Resolve grant acceptance issues for received licenses"
context: "Grants must be accepted by the recipient before they provide license access. Acceptance requires proper IAM permissions in the recipient account."
---

## Phase 1 — Triage

MUST:
- List received grants: `aws license-manager list-received-grants --query 'Grants[*].{Arn:GrantArn,Name:GrantName,Status:Status}'`
- Check grant status and details: `aws license-manager get-grant --grant-arn <grant-arn>`
- Verify IAM permissions for acceptance: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names license-manager:AcceptGrant`

SHOULD:
- Verify the grant is in PENDING_ACCEPT status
- Check if the grant has expired or been revoked by the grantor

MAY:
- Contact the grantor to verify grant details
- Check if a previous version of the grant was already accepted

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for license-manager:AcceptGrant
- Accept the grant: `aws license-manager accept-grant --grant-arn <grant-arn>`
- Verify grant status changes to ACTIVE

SHOULD:
- Verify received license is accessible after acceptance: `aws license-manager list-received-licenses`
- Test license checkout: `aws license-manager checkout-license --product-sku <sku> --checkout-type PROVISIONAL --entitlements '[{"Name":"<name>","Unit":"Count","Value":"1"}]' --client-token <token>`

MAY:
- Set up notifications for new grant arrivals
- Document accepted grants for compliance tracking

## Safety Ratings

```
safety_ratings:
  - "license-manager list-received-grants: GREEN — read-only listing"
  - "license-manager get-grant: GREEN — read-only grant details"
  - "license-manager accept-grant: YELLOW — activates cross-account license sharing"
  - "license-manager checkout-license: YELLOW — consumes license entitlement"
```

## Escalation Conditions

- Grant acceptance fails with internal service errors
- Grant was revoked before acceptance could complete
- Accepted grant does not provide expected license access
- Multiple grants need acceptance across many accounts

## Data Sensitivity

- `aws license-manager list-received-grants`: MEDIUM — reveals sharing relationships
- Grant details: MEDIUM — contains grantor and license information
- License checkout: MEDIUM — reveals consumption patterns
- Cross-account relationships: MEDIUM — reveals organizational structure

## Prohibited Actions

- NEVER accept grants from unknown or untrusted accounts
- NEVER accept grants without verifying the license terms
- NEVER reject grants without confirming with the grantor
- NEVER share accepted license details with unauthorized parties

## Phase 3 — Rollback

1. If grant was accepted in error, contact grantor to revoke
2. Check in any licenses checked out from the grant: `aws license-manager check-in-license --license-consumption-token <token>`
3. Verify no resources are using the granted license before revocation
4. Document the acceptance/rejection for audit trail
5. Notify relevant teams of grant status changes

## Common Issues

- symptoms: "AcceptGrant returns InvalidGrantException"
  diagnosis: "Grant is not in PENDING_ACCEPT status or has been revoked."
  resolution: "Check grant status. If revoked, request a new grant from the grantor."

## Output Format

```yaml
root_cause: "grant_acceptance — <specific_cause>"
evidence:
  - type: acceptance_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix grant acceptance issue"
  long_term: "Automate grant acceptance workflows"
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
