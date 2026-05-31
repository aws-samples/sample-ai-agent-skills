# E1 — Mainframe Modernization CICS Transaction Issues (Micro Focus)

## Symptoms
- CICS transaction fails
- Transaction timeout
- CICS region not starting

## Triage Commands

```bash
aws m2 get-application --application-id <app-id>
aws m2 get-environment --environment-id <env-id>
# Check CloudWatch logs for CICS output
aws logs get-log-events --log-group-name /aws/m2/<env-id> --log-stream-name <stream>
```

## Common Causes
1. **CICS resource definition missing** — CSD entries not configured
2. **Program not found** — Transaction references missing program
3. **Map not found** — BMS map not included in application
4. **Database connectivity** — CICS can't reach database
5. **Region configuration** — CICS region parameters incorrect

## Resolution
- Verify CICS resource definitions (CSD) are included in application
- Ensure all programs referenced by transactions are in load library
- Include BMS maps in application artifacts
- Configure database connectivity in environment (security groups, endpoints)
- Check CICS region logs in CloudWatch for specific error codes
- Note: CICS is Micro Focus engine only


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
