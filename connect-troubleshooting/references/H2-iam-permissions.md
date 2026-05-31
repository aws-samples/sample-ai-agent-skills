---
title: "H2 — IAM Permission Issues"
description: "Diagnose IAM permission issues affecting Connect operations"
status: active
severity: HIGH
triggers:
  - "Access denied"
  - "IAM permission error"
  - "Cannot manage Connect"
  - "API authorization failed"
owner: devops-agent
objective: "Fix IAM permissions for Connect administration and operations"
context: "Connect requires IAM permissions for: instance management, contact flow operations, user management, metrics access, and integration configuration. The Connect service-linked role handles internal operations. Custom IAM policies control admin and API access."
---

## Phase 1 — Triage

MUST:
- Identify the specific API call that's failing
- Check the IAM policy attached to the user/role making the call
- Verify the Connect service-linked role exists
- Check for SCP restrictions if in an Organization

SHOULD:
- Use IAM Access Analyzer to evaluate effective permissions
- Check if resource-level permissions are required (instance ARN, flow ARN)
- Verify the IAM policy includes the correct Connect actions

MAY:
- Check CloudTrail for the denied API call details
- Review the IAM policy for overly restrictive conditions

## Phase 2 — Remediate

MUST:
- Add the required Connect IAM actions to the policy
- Include the correct resource ARNs (or use * for all resources)
- Verify the service-linked role exists and is not modified

SHOULD:
- Use least-privilege principles when granting permissions
- Test the permissions after changes
- Document the required permissions for each role

MAY:
- Implement IAM permission boundaries for Connect admins
- Set up permission auditing

## Common Issues

- symptoms: "Cannot create or modify contact flows via API"
  diagnosis: "IAM policy missing connect:CreateContactFlow or connect:UpdateContactFlow."
  resolution: "Add the required Connect actions to the IAM policy."

- symptoms: "Cannot access Connect metrics"
  diagnosis: "IAM policy missing connect:GetCurrentMetricData or connect:GetMetricDataV2."
  resolution: "Add metrics-related Connect actions to the policy."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
```

## Escalation Conditions

- Contact center is live with active calls or chats in progress
- Issue affects agent ability to receive or handle contacts
- Fix requires modifying a contact flow associated with a phone number
- Resolution involves changes to hours of operation during business hours
- Issue impacts call recording, CTR generation, or compliance requirements

## Data Sensitivity

HIGH - Contact Trace Records (CTRs) contain customer phone numbers, call timestamps, and interaction details. Call recordings contain customer voice data. Chat transcripts may contain PII, financial data, or healthcare information. Agent data includes employee performance metrics.

## Prohibited Actions

- NEVER suggest deleting a contact flow that is associated with a phone number - this breaks inbound routing
- NEVER suggest modifying production hours of operation during active business hours
- NEVER recommend removing security profiles from agents while they are logged in
- NEVER suggest disabling call recording on queues that have compliance requirements
- NEVER recommend deleting or modifying a published contact flow without first testing in a non-production flow

## Phase 3 - Rollback

1. If a contact flow was modified, publish the previous saved version of the flow from the Connect console
2. If Lambda or Lex integration was changed, restore the original function/bot association in the contact flow
3. If agent routing or security profiles were modified, restore original profile assignments
4. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "iam_permissions — <missing action>"
evidence:
  - type: iam_policy
    content: "<policy details and denied action>"
severity: HIGH
mitigation:
  immediate: "Add required permissions"
  long_term: "Implement least-privilege IAM policies and auditing"
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
