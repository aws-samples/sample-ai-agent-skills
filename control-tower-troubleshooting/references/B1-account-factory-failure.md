---
title: "B1 — Account Factory Provisioning Failure"
description: "Diagnose failures when provisioning accounts through Account Factory"
status: active
severity: HIGH
triggers:
  - "Account provisioning failed"
  - "Service Catalog provisioned product ERROR"
  - "UNDER_CHANGE stuck"
  - "Account Factory error"
owner: devops-agent
objective: "Identify why account provisioning failed and complete the provisioning"
context: "Account Factory uses Service Catalog to provision accounts. The flow is: Service Catalog → CloudFormation → Organizations CreateAccount → baseline StackSets. Failures can occur at any stage."
---

## Phase 1 — Triage

MUST:
- Check provisioned product status: `aws servicecatalog describe-provisioned-product --id <id>`
- Check provisioned product record: `aws servicecatalog describe-record --id <record-id>`
- Check CloudFormation stack events in management account for the provisioning stack
- Check Organizations for the account creation status: `aws organizations describe-create-account-status --create-account-request-id <id>`

SHOULD:
- Verify the target OU exists and is registered with Control Tower
- Check if the email address is unique and not already in use
- Verify the SSO user email is valid and accessible
- Check for concurrent provisioning operations (CT has concurrency limits)

MAY:
- Check CloudTrail for CreateManagedAccount or CreateAccount events
- Review Service Catalog portfolio and product version

## Phase 2 — Remediate

MUST:
- If email conflict: use a unique email address
- If OU not registered: register the OU first
- If stuck UNDER_CHANGE: wait for timeout, then check for underlying failures
- If CloudFormation failed: check stack events for the specific error

SHOULD:
- Terminate the failed provisioned product before retrying
- Verify Account Factory product version matches landing zone version
- Check if the account was partially created in Organizations

MAY:
- If account was created but baseline failed: enroll the account instead of re-provisioning
- Contact AWS Support for stuck provisioning operations

## Common Issues

- symptoms: "Provisioned product stuck in UNDER_CHANGE for hours"
  diagnosis: "CloudFormation stack is waiting for a resource that won't complete."
  resolution: "Check CF stack events. Common causes: email verification pending, SCP blocking, or StackSet timeout."

- symptoms: "ERROR state with 'email already exists'"
  diagnosis: "The account email is already associated with an AWS account."
  resolution: "Use a unique email address. Terminate the failed product and retry."

- symptoms: "Account created but not governed"
  diagnosis: "Account was created in Organizations but baseline StackSets failed."
  resolution: "Enroll the account into Control Tower or re-register the OU."

## Safety Ratings

```
safety_ratings:
  - "describe-provisioned-product: GREEN — read-only product status"
  - "describe-record: GREEN — read-only provisioning record"
  - "describe-create-account-status: GREEN — read-only account creation status"
  - "list-stacks: GREEN — read-only stack listing"
  - "terminate failed provisioned product: YELLOW — recoverable, cleans up failed provisioning"
  - "enroll partially created account: YELLOW — recoverable, applies CT baselines"
  - "re-provision account: RED — creates new AWS account with governance, irreversible account creation"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Account was partially created in Organizations but baseline failed
- Provisioning stuck in UNDER_CHANGE for extended period with no clear cause
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: Organization account list (all account IDs and emails)
- HIGH: SSO user email and account root email (PII)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Service Catalog provisioned product details

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If account provisioning fails partway, terminate the failed provisioned product before retrying — do NOT leave orphaned products
- If an account was created in Organizations but not governed, enroll it rather than attempting to delete and recreate
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Document the provisioned product ID and account details before terminating failed products

## Output Format

```yaml
root_cause: "account_factory_failure — <specific cause>"
evidence:
  - type: service_catalog_record
    content: "<provisioned product status and error>"
  - type: cloudformation_events
    content: "<stack failure details>"
severity: HIGH
mitigation:
  immediate: "Fix the blocking condition and retry provisioning"
  long_term: "Validate inputs (email, OU, SSO) before provisioning"
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
