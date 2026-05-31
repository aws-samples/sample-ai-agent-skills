---
title: "E2 — Application Testing"
description: "Diagnose application-level issues on migrated instances"
status: active
severity: MEDIUM
triggers: ["app not working after migration", "performance degraded", "connectivity issues"]
owner: devops-agent
objective: "Resolve application-level post-migration issues"
context: "Applications may need configuration changes for AWS environment — new IPs, different DNS, security group rules, IAM roles instead of local credentials."
---

## Phase 1 — Triage

MUST:
- Verify application process is running on the instance
- Check application logs for errors
- Test application endpoints from within VPC

SHOULD:
- Verify database connectivity from migrated instance
- Check security group rules allow application ports
- Verify dependent services are accessible

MAY:
- Run application-specific health checks
- Compare response times with source baseline

## Phase 2 — Remediate

MUST:
- Update hardcoded IPs/hostnames in application config
- Ensure security groups allow required traffic

SHOULD:
- Update connection strings for AWS-hosted dependencies
- Configure IAM roles for AWS service access

MAY:
- Set up application monitoring in CloudWatch

## Common Issues

- symptoms: "App starts but cannot connect to database"
  diagnosis: "Database connection string points to on-premises IP."
  resolution: "Update connection string to AWS database endpoint."

## Output Format

```yaml
root_cause: "application_testing — <specific_cause>"
evidence:
  - type: app_health
    content: "<application status and error logs>"
severity: MEDIUM
mitigation:
  immediate: "Fix application configuration"
  long_term: "Document all config changes needed for migration"
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

- **NEVER** finalize a cutover without confirmed application validation and customer sign-off
- **NEVER** disconnect replication on source servers before cutover completion is verified
- **NEVER** delete source server records that have active replication or pending cutover operations
- **NEVER** modify launch templates for production migrations without testing in a staging environment first
- **NEVER** skip post-migration validation steps even if the launch appears successful

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
