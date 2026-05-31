---
title: "C2 — Step Failures"
description: "Diagnose and recover from step failures"
status: active
severity: HIGH
triggers: ["step failed cannot retry", "step recovery", "workflow blocked by step"]
owner: devops-agent
objective: "Recover from step failures"
context: "Failed steps block subsequent steps. Recovery options include retry, skip, or creating a new workflow."
---

## Phase 1 — Triage

MUST:
- Identify failed step and error details
- Determine if step is retryable
- Assess impact on subsequent steps

SHOULD:
- Check if the step made partial changes
- Verify resources are in consistent state

MAY:
- Check if step can be skipped

## Phase 2 — Remediate

MUST:
- Fix underlying issue and retry step if possible
- If not retryable: assess if workflow can continue

SHOULD:
- Document failure and resolution
- Clean up partial changes if needed

MAY:
- Create new workflow if current one is unrecoverable

## Common Issues

- symptoms: "Step failed and retry also fails"
  diagnosis: "Underlying issue not resolved or resource in bad state."
  resolution: "Fix resource state manually, then retry step."

## Output Format

```yaml
root_cause: "step_failure — <specific_cause>"
evidence:
  - type: failure_details
    content: "<step failure and recovery options>"
severity: HIGH
mitigation:
  immediate: "Recover from step failure"
  long_term: "Improve pre-flight validation"
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

- **NEVER** skip mandatory workflow steps in SAP or SQL Server migration templates without documented justification
- **NEVER** delete or modify an in-progress workflow without confirming the current step state and rollback options
- **NEVER** remove migration plugins from servers with active workflow step execution
- **NEVER** alter template parameters on running workflows without creating a new workflow instance
- **NEVER** bypass pre-migration validation steps even under time pressure

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
