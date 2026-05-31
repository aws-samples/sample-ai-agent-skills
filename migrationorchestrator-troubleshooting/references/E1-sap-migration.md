---
title: "E1 — SAP Migration"
description: "Diagnose SAP migration workflow issues"
status: active
severity: HIGH
triggers: ["SAP migration failed", "SAP workflow error", "SAP on AWS"]
owner: devops-agent
objective: "Resolve SAP migration workflow issues"
context: "SAP migration workflows have SAP-specific steps requiring SAP credentials, system IDs, and SAP-aware configuration."
---

## Phase 1 — Triage

MUST:
- Check SAP workflow status: `aws migrationhuborchestrator get-workflow --id <id>`
- Verify SAP-specific parameters (SID, instance number, credentials)
- Check plugin health on SAP source server

SHOULD:
- Verify SAP system is accessible from plugin
- Check SAP-specific step errors

MAY:
- Review SAP system logs for migration-related errors

## Phase 2 — Remediate

MUST:
- Fix SAP-specific configuration (credentials, SID, connectivity)
- Ensure SAP system is in correct state for migration step

SHOULD:
- Follow SAP migration best practices
- Coordinate with SAP Basis team

MAY:
- Use AWS Launch Wizard for SAP for guided deployment

## Common Issues

- symptoms: "SAP system copy step fails"
  diagnosis: "SAP credentials invalid or system not in correct state."
  resolution: "Verify SAP credentials and system state. Ensure SAP is stopped for system copy."

## Output Format

```yaml
root_cause: "sap_migration — <specific_cause>"
evidence:
  - type: sap_step_error
    content: "<SAP-specific error details>"
severity: HIGH
mitigation:
  immediate: "Fix SAP configuration"
  long_term: "Document SAP migration prerequisites"
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
