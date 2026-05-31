---
title: "B2 — S3 Path Issues"
description: "Diagnose S3 path configuration problems in Lake Formation"
status: active
severity: MEDIUM
triggers:
  - "S3 path error"
  - "location not found"
  - "invalid S3 path"
  - "path mismatch"
owner: devops-agent
objective: "Identify and resolve S3 path issues in Lake Formation"
context: "Lake Formation uses S3 paths for data location registration and table storage. Path mismatches between Glue table locations and registered LF locations cause permission issues. S3 paths must be correctly formatted and accessible."
---

## Phase 1 — Triage

MUST:
- Check table S3 location: `aws glue get-table --database-name <db> --name <table> --query 'Table.StorageDescriptor.Location'`
- List registered locations: `aws lakeformation list-resources`
- Verify the table location falls under a registered path
- Check S3 path accessibility: `aws s3 ls <s3-path>`

SHOULD:
- Verify path format (s3://bucket/prefix/)
- Check for trailing slash consistency
- Verify partition paths are under the registered location
- Check for case sensitivity issues in paths

MAY:
- Verify S3 bucket region matches LF region
- Check for S3 access points affecting path resolution

## Phase 2 — Remediate

MUST:
- Ensure table S3 location falls under a registered LF location
- Use consistent path formatting
- Register the correct S3 path if not registered

SHOULD:
- Align Glue table locations with LF registered paths
- Use consistent path conventions across the data lake
- Document path naming standards

MAY:
- Automate path validation in data pipelines
- Create path governance policies

## Common Issues

- symptoms: "Table accessible via IAM but not via LF permissions"
  diagnosis: "Table S3 location not under a registered LF location."
  resolution: "Register the S3 path or move data under a registered path."

- symptoms: "Partition data not accessible"
  diagnosis: "Partition paths outside the registered location."
  resolution: "Ensure all partition paths fall under the registered S3 location."

## Output Format

```yaml
root_cause: "s3_path_issue — <specific_cause>"
evidence:
  - type: table_location
    content: "<Glue table S3 location>"
  - type: registered_paths
    content: "<LF registered locations>"
severity: MEDIUM
mitigation:
  immediate: "Align S3 paths with registered locations"
  long_term: "Standardize S3 path conventions and automate validation"
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
