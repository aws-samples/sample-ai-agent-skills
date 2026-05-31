---
title: "D3 — Cutover Rollback"
description: "Diagnose cutover rollback scenarios"
status: active
severity: HIGH
triggers: ["rollback cutover", "revert migration", "cutover failed need rollback"]
owner: devops-agent
objective: "Execute cutover rollback when migration validation fails"
context: "Rollback before finalization means reverting traffic to source. After finalization, rollback requires re-migration. MGN does not have a built-in rollback button."
---

## Phase 1 — Triage

MUST:
- Check if cutover has been finalized: `aws mgn describe-source-servers --filters 'sourceServerIDs=<id>' --query 'items[0].lifeCycle.state'`
- If NOT finalized: replication is still active, rollback is straightforward
- If FINALIZED: replication stopped, rollback requires re-migration

SHOULD:
- Verify source server is still running and accessible
- Check DNS/traffic routing current state

MAY:
- Assess data changes made on cutover instance since cutover

## Phase 2 — Remediate

MUST:
- If NOT finalized: redirect DNS/traffic back to source server. Cutover instance can be terminated.
- If FINALIZED: source server must be kept running. Re-install agent for re-migration if needed.

SHOULD:
- Document what failed during validation
- Plan remediation before next cutover attempt

MAY:
- Use post-launch actions to automate validation for next attempt

## Common Issues

- symptoms: "Application broken on cutover instance, need to go back"
  diagnosis: "Check if cutover was finalized. If not, simply redirect traffic."
  resolution: "Redirect DNS to source. Terminate cutover instance. Fix issues. Retry."

## Output Format

```yaml
root_cause: "cutover_rollback — <specific_cause>"
evidence:
  - type: rollback_scenario
    content: "<finalization state and rollback approach>"
severity: HIGH
mitigation:
  immediate: "Execute rollback procedure"
  long_term: "Improve validation before cutover"
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
