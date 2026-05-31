---
title: "G2 — Delegated Administrator Issues"
description: "Diagnose AWS Config delegated administrator setup and permission issues"
status: active
severity: MEDIUM
triggers:
  - "delegated administrator"
  - "delegated admin config"
  - "delegate admin failed"
  - "delegated admin permissions"
  - "config delegation"
owner: devops-agent
objective: "Identify and resolve delegated administrator configuration issues for AWS Config multi-account management"
context: "A delegated administrator account can manage organization Config rules and conformance packs on behalf of the management account. Setup requires Organizations access and proper registration. Issues arise from incorrect registration, missing permissions, or Organizations configuration."
---

## Phase 1 — Triage

MUST:
- Check current delegated admins: `aws organizations list-delegated-administrators --service-principal config-multiaccountsetup.amazonaws.com`
- Verify Organizations trusted access: `aws organizations list-aws-service-access-for-organization`
- Check if the account is a member of the organization: `aws organizations describe-account --account-id <account-id>`
- Verify the caller has Organizations permissions: `aws organizations describe-organization`

SHOULD:
- Check CloudTrail for delegation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RegisterDelegatedAdministrator --max-results 10`
- Verify the delegated admin account has Config properly configured
- Check for SCPs that might block Config operations in the delegated admin account

MAY:
- List all delegated administrators across services: `aws organizations list-delegated-administrators`
- Verify the delegated admin can list organization rules: `aws configservice describe-organization-config-rules`

## Phase 2 — Remediate

MUST:
- Register the delegated admin from the management account: `aws organizations register-delegated-administrator --account-id <account-id> --service-principal config-multiaccountsetup.amazonaws.com`
- Enable trusted access for Config: `aws organizations enable-aws-service-access --service-principal config-multiaccountsetup.amazonaws.com`
- Verify the registration was successful

SHOULD:
- Ensure the delegated admin account has Config recorder running
- Test organization rule creation from the delegated admin account

MAY:
- Document the delegated admin setup for the team
- Set up monitoring for delegated admin status changes

## Common Issues

- symptoms: "Cannot register delegated administrator — AccessDeniedException"
  diagnosis: "The caller is not the management account or lacks Organizations permissions."
  resolution: "Run the registration command from the management account with organizations:RegisterDelegatedAdministrator permission."

- symptoms: "Delegated admin cannot create organization rules"
  diagnosis: "Trusted access for Config is not enabled, or the account is not properly registered."
  resolution: "Enable trusted access and verify the account is registered as delegated admin for config-multiaccountsetup.amazonaws.com."

- symptoms: "Delegated admin registered but cannot see member account data"
  diagnosis: "The delegated admin needs an aggregator configured to collect data from member accounts."
  resolution: "Create an organization aggregator in the delegated admin account."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Register delegated admin | YELLOW | Organization change; reversible |
| Enable trusted access | YELLOW | Organization change; reversible |

## Escalation Conditions

- Recorder configuration changes
- Delegated admin designation changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-delegated-administrators` | MEDIUM | Admin account IDs |
| `list-aws-service-access-for-organization` | LOW | Service access status |
| `describe-organization` | MEDIUM | Organization details |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Admin registration | Deregister via `deregister-delegated-administrator` |
| Trusted access enablement | Disable via `disable-aws-service-access` |

## Output Format

```yaml
root_cause: "delegated_administrator — <specific_cause>"
evidence:
  - type: delegated_admins
    content: "<list-delegated-administrators output>"
  - type: trusted_access
    content: "<service access configuration>"
  - type: organizations
    content: "<organization details>"
severity: MEDIUM
mitigation:
  immediate: "Register the delegated admin and enable trusted access"
  long_term: "Document the delegation setup and monitor for changes"
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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
