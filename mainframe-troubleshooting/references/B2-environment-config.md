# B2 — Mainframe Modernization Environment Configuration Issues

## Symptoms
- Environment performance poor
- Environment connectivity issues
- Configuration update failures

## Triage Commands

```bash
aws m2 get-environment --environment-id <env-id>
aws cloudwatch get-metric-statistics --namespace AWS/M2 --metric-name CPUUtilization --dimensions Name=EnvironmentId,Value=<env-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/M2 --metric-name MemoryUtilization --dimensions Name=EnvironmentId,Value=<env-id> --start-time <start> --end-time <end> --period 300 --statistics Average
```

## Common Causes
1. **Under-provisioned** — Instance type too small for workload
2. **Storage insufficient** — EFS storage limits
3. **Network latency** — Database in different AZ/region
4. **Security group rules** — Blocking required connectivity
5. **High availability** — Single-AZ deployment for production

## Resolution
- Monitor CPU/memory metrics and scale instance type if needed
- Ensure EFS throughput mode matches workload (bursting vs provisioned)
- Place environment in same AZ as database for low latency
- Configure security groups for database, EFS, and external connectivity
- Use multi-AZ deployment for production workloads


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
