---
title: "D2 — Cutover Finalization"
description: "Diagnose cutover finalization issues"
status: active
severity: HIGH
triggers: ["finalize cutover", "cutover not completing", "finalization error"]
owner: devops-agent
objective: "Resolve cutover finalization issues"
context: "FinalizeCutover permanently stops replication. This is irreversible. Issues include premature finalization or finalization failures."
---

## Phase 1 — Triage

MUST:
- Check server state: `aws mgn describe-source-servers --filters 'sourceServerIDs=<id>' --query 'items[0].lifeCycle.state'`
- Verify cutover instance is running and validated
- Confirm application is working on cutover instance

SHOULD:
- Verify DNS/traffic has been redirected to new instance
- Check all dependent services are accessible

MAY:
- Run application-specific health checks

## Phase 2 — Remediate

MUST:
- Only finalize after thorough validation
- Finalize: `aws mgn finalize-cutover --source-server-id <id>`
- WARNING: This is irreversible — replication stops permanently

SHOULD:
- Document validation checklist before finalization
- Keep source server running for rollback period

MAY:
- Automate validation checks before finalization

## Common Issues

- symptoms: "Finalized too early — application has issues"
  diagnosis: "Insufficient validation before finalization."
  resolution: "Cannot undo finalization. Fix application issues on AWS instance. Re-install agent for re-migration if needed."

## Output Format

```yaml
root_cause: "cutover_finalization — <specific_cause>"
evidence:
  - type: finalization_status
    content: "<server state and validation results>"
severity: HIGH
mitigation:
  immediate: "Complete validation or fix finalization error"
  long_term: "Implement mandatory validation checklist before finalization"
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
