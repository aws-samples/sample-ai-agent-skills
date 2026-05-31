---
title: "E2 — SAP-Specific Issues"
description: "Diagnose SAP-specific migration issues"
status: active
severity: HIGH
triggers: ["SAP database", "SAP application server", "HANA migration"]
owner: devops-agent
objective: "Resolve SAP-specific migration issues"
context: "SAP migrations involve database migration (HANA, Oracle, SQL Server), application server migration, and SAP-specific post-migration steps."
---

## Phase 1 — Triage

MUST:
- Identify which SAP component is failing (database, app server, central services)
- Check SAP system health on source
- Verify target infrastructure is properly sized for SAP

SHOULD:
- Check SAP kernel and database versions compatibility
- Verify SAP license for target system

MAY:
- Review SAP migration guide for the specific scenario

## Phase 2 — Remediate

MUST:
- Fix SAP component-specific issue
- Ensure target meets SAP sizing requirements

SHOULD:
- Follow SAP migration checklist
- Validate SAP system after each migration step

MAY:
- Engage AWS SAP specialists for complex issues

## Common Issues

- symptoms: "HANA database migration step fails"
  diagnosis: "Target instance type doesn't meet HANA memory requirements."
  resolution: "Use HANA-certified instance type with sufficient memory."

## Output Format

```yaml
root_cause: "sap_specific — <specific_cause>"
evidence:
  - type: sap_details
    content: "<SAP component error details>"
severity: HIGH
mitigation:
  immediate: "Fix SAP-specific issue"
  long_term: "Follow SAP on AWS best practices"
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
