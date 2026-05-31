---
title: "B1 — Data Location Registration"
description: "Diagnose data location registration issues in Lake Formation"
status: active
severity: HIGH
triggers:
  - "register resource failed"
  - "data location not registered"
  - "RegisterResource error"
  - "location registration"
owner: devops-agent
objective: "Identify and resolve data location registration issues"
context: "Lake Formation requires S3 locations to be registered before it can manage fine-grained permissions. Registration associates an IAM role with the S3 location. The role must have S3 access. Unregistered locations use IAM-only access control."
---

## Phase 1 — Triage

MUST:
- List registered resources: `aws lakeformation list-resources`
- Check if the S3 location exists: `aws s3 ls <s3-path>`
- Verify the registration role has S3 access: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject s3:PutObject --resource-arns <s3-arn>`
- Check CloudTrail for registration events

SHOULD:
- Verify the role trust policy allows LF to assume it
- Check if a parent path is already registered
- Verify the S3 bucket policy allows the registration role
- Check for KMS key permissions if bucket is encrypted

MAY:
- Verify cross-account S3 access if bucket is in another account
- Check for S3 bucket policies blocking access

## Phase 2 — Remediate

MUST:
- Register the S3 location: `aws lakeformation register-resource --resource-arn <s3-arn> --role-arn <role-arn>`
- Ensure the role has S3 read/write access to the location
- Verify the role trust policy includes lakeformation.amazonaws.com

SHOULD:
- Use a dedicated LF service role for registration
- Register at the appropriate path level (not too broad, not too narrow)
- Test access after registration

MAY:
- Automate location registration for new data sources
- Set up monitoring for registration status

## Common Issues

- symptoms: "RegisterResource fails with access denied"
  diagnosis: "Registration role lacks S3 permissions or trust policy."
  resolution: "Add S3 permissions to the role. Add lakeformation.amazonaws.com to trust policy."

- symptoms: "Location registered but LF permissions not enforced"
  diagnosis: "IAMAllowedPrincipals grants still in place."
  resolution: "Revoke IAMAllowedPrincipals grants on the database/table."

## Output Format

```yaml
root_cause: "data_location_registration — <specific_cause>"
evidence:
  - type: registered_resources
    content: "<registered locations>"
  - type: role_permissions
    content: "<registration role S3 access>"
severity: HIGH
mitigation:
  immediate: "Fix registration role permissions and register location"
  long_term: "Standardize data location registration process"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** revoke Lake Formation permissions on production databases without explicit customer approval
- **NEVER** modify IAMAllowedPrincipals settings without understanding the full permission inheritance chain
- **NEVER** deregister S3 data locations that are actively referenced by Glue tables or crawlers
- **NEVER** disable hybrid access mode on databases with active IAM-based consumers
- **NEVER** alter cross-account resource shares (RAM) without coordinating with all participating accounts

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
