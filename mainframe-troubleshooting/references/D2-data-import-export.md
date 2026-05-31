# D2 — Mainframe Modernization Data Import/Export Issues

## Symptoms
- Data import task fails
- Imported data corrupted
- Export task errors

## Triage Commands

```bash
aws m2 list-data-set-import-history --application-id <app-id> --max-results 10
aws s3 ls s3://<data-bucket>/<path>/
```

## Common Causes
1. **S3 access denied** — M2 can't read from/write to S3
2. **File format wrong** — Data not in expected format
3. **Character encoding** — EBCDIC to ASCII conversion issues
4. **Large file** — Import timeout for very large data sets
5. **Missing definition** — Data set definition not provided

## Resolution
- Verify S3 bucket permissions for M2 service
- Use correct format for data set type (binary for VSAM, text for sequential)
- Handle EBCDIC to ASCII conversion during export from mainframe
- For large data sets, consider splitting into smaller files
- Provide complete data set definition (LRECL, BLKSIZE, RECFM, DSORG)


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

- **NEVER** terminate a running mainframe environment without confirming all batch jobs have completed
- **NEVER** modify CICS transaction definitions on a production runtime without a tested rollback plan
- **NEVER** delete or overwrite production data sets without verified backups
- **NEVER** alter JCL configurations for active batch processing workflows without scheduling a maintenance window
- **NEVER** force-stop a Blu Age or Micro Focus runtime with in-flight transactions

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
