---
title: "A1 — Landing Zone Setup Failure"
description: "Diagnose failures during initial Control Tower landing zone setup"
status: active
severity: CRITICAL
triggers:
  - "Landing zone setup failed"
  - "CreateLandingZone failed"
  - "Control Tower setup error"
owner: devops-agent
objective: "Identify why landing zone setup failed and complete the setup successfully"
context: "Landing zone setup creates the foundational multi-account structure: Security OU, Sandbox OU, Audit account, Log Archive account, CloudTrail org trail, Config aggregation, and baseline SCPs. Failures are typically due to pre-existing resources, permission issues, or Organizations configuration."
---

## Phase 1 — Triage

MUST:
- Check landing zone operation status: `aws controltower list-landing-zone-operations --filter statuses=FAILED`
- Check if Organizations is already set up: `aws organizations describe-organization`
- Verify the management account has no SCPs that block CT operations
- Check CloudFormation stack events for AWSControlTower* stacks: `aws cloudformation list-stacks --stack-status-filter CREATE_FAILED ROLLBACK_COMPLETE`
- Verify the account has not previously had Control Tower set up (residual resources)

SHOULD:
- Check if required service-linked roles exist or can be created
- Verify AWS Config is not already enabled with conflicting configuration
- Check if CloudTrail already has an organization trail
- Verify the email addresses for Audit and Log Archive accounts are unique and accessible

MAY:
- Check AWS Health Dashboard for service issues
- Review CloudTrail for the CreateLandingZone API call and error details

## Phase 2 — Remediate

MUST:
- If pre-existing Config recorder: delete it or let CT manage it
- If pre-existing org trail: evaluate whether to keep it or let CT create one
- If residual CT resources from previous setup: clean up AWSControlTower* stacks, roles, and SCPs
- If email conflict: use unique email addresses for Audit and Log Archive accounts

SHOULD:
- Ensure the management account has AdministratorAccess or equivalent permissions
- Verify all governed regions are enabled in the account
- Remove any SCPs that might block CT service actions

MAY:
- Contact AWS Support if setup fails repeatedly with no clear error

## Common Issues

- symptoms: "Setup fails with Config recorder already exists"
  diagnosis: "AWS Config is already enabled in the management account with a recorder."
  resolution: "Delete the existing Config recorder or delivery channel, then retry setup."

- symptoms: "Setup fails with email already in use"
  diagnosis: "The email address for Audit or Log Archive account is already associated with an AWS account."
  resolution: "Use a different unique email address. Consider email aliases (admin+audit@example.com)."

- symptoms: "Setup fails with insufficient permissions"
  diagnosis: "The IAM principal running setup lacks required permissions."
  resolution: "Use a principal with AdministratorAccess in the management account."

- symptoms: "Setup fails with Organizations not configured"
  diagnosis: "AWS Organizations is not enabled or is in consolidated billing mode only."
  resolution: "Enable all features in Organizations before setting up Control Tower."

## Safety Ratings

```
safety_ratings:
  - "list-landing-zone-operations: GREEN — read-only operation listing"
  - "describe-organization: GREEN — read-only org metadata"
  - "list-stacks: GREEN — read-only stack listing"
  - "delete Config recorder: YELLOW — recoverable, CT will recreate on setup"
  - "clean up residual CT resources: RED — deleting AWSControlTower stacks/roles/SCPs can break governance if misidentified"
  - "CreateLandingZone (retry setup): RED — creates foundational multi-account structure, SCPs, and org trail"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Drift affects multiple OUs or accounts
- Residual resources from a previous CT setup cannot be safely identified
- Setup fails repeatedly with no clear error after prerequisite checks

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: CloudFormation stack details (resource names, configurations)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If landing zone setup fails partway, do NOT attempt to manually complete the setup — retry via the CT console or API after fixing prerequisites
- If residual resources were cleaned up incorrectly, restore from CloudFormation stack outputs or CloudTrail event history
- If Organizations configuration was changed to support setup, document the original state before making changes
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- "If OU re-registration fails, document the error and do NOT retry without investigating"

## Output Format

```yaml
root_cause: "landing_zone_setup_failure — <specific cause>"
evidence:
  - type: controltower_operation
    content: "<operation status and error details>"
  - type: cloudformation_events
    content: "<stack failure events>"
severity: CRITICAL
mitigation:
  immediate: "Resolve the blocking condition and retry setup"
  long_term: "Document prerequisites checklist for future landing zone deployments"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"
