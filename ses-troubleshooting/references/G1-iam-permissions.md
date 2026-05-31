---
title: "G1 — IAM Permissions"
description: "Diagnose IAM permission issues for SES operations"
status: active
severity: HIGH
triggers:
  - "access denied SES"
  - "not authorized to perform ses"
  - "IAM permission error"
  - "SES authorization"
owner: devops-agent
objective: "Resolve IAM permission issues for SES API operations"
context: "SES operations require specific IAM permissions. Common actions include ses:SendEmail, ses:SendRawEmail, ses:SendTemplatedEmail, ses:SendBulkTemplatedEmail. Resource-level permissions can restrict which identities a user can send from."
---

## Phase 1 — Triage

MUST:
- Check IAM policy for SES permissions: `aws iam get-user-policy --user-name <user> --policy-name <policy>` or `aws iam list-attached-user-policies --user-name <user>`
- Verify the specific SES action being denied
- Check for SES sending authorization policies: `aws sesv2 get-email-identity-policies --email-identity <identity>`
- Verify no SCP or permission boundary is blocking SES access

SHOULD:
- Use IAM Policy Simulator to test permissions: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names ses:SendEmail`
- Check for condition keys restricting SES actions (ses:FromAddress, ses:Recipients)
- Review CloudTrail for authorization failure details

MAY:
- Check if cross-account sending authorization is needed
- Verify Lambda execution role if sending from Lambda

## Phase 2 — Remediate

MUST:
- Add required SES permissions to IAM policy
- Ensure ses:SendEmail and ses:SendRawEmail are granted for the sending identity
- Fix resource ARN in policy if restricting to specific identities

SHOULD:
- Use least-privilege: restrict to specific identities and actions
- Add condition keys to limit From addresses: `ses:FromAddress`

MAY:
- Set up cross-account sending authorization for shared identities
- Create SES-specific IAM roles for applications

## Common Issues

- symptoms: "AccessDeniedException on ses:SendEmail"
  diagnosis: "IAM policy missing ses:SendEmail permission."
  resolution: "Add ses:SendEmail and ses:SendRawEmail to the IAM policy."

- symptoms: "Can send from one address but not another"
  diagnosis: "IAM policy restricts SES actions to specific identity ARNs."
  resolution: "Add the additional identity ARN to the policy resource."

- symptoms: "Lambda cannot send email"
  diagnosis: "Lambda execution role missing SES permissions."
  resolution: "Add ses:SendEmail to the Lambda execution role, not the function policy."

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: iam_policy
    content: "<relevant IAM policy statements>"
  - type: error_details
    content: "<specific authorization failure>"
severity: HIGH
mitigation:
  immediate: "Fix IAM policy to grant required SES permissions"
  long_term: "Implement least-privilege SES IAM policies"
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

- Email content may contain customer PII and business communications
- Recipient lists contain personal email addresses
- Bounce and complaint data reveals delivery issues with specific addresses
- Classification: HIGHLY CONFIDENTIAL - email content and recipient PII

## Prohibited Actions

- NEVER send test emails to real customer addresses
- NEVER modify production DKIM/SPF/DMARC records without email team approval
- NEVER disable bounce or complaint handling in production
- NEVER remove addresses from the suppression list without investigating the original cause

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
  - "NEVER suggest disabling DKIM/SPF/DMARC"
  - "NEVER suggest removing sending limits"
  - "NEVER suggest disabling bounce/complaint handling"
