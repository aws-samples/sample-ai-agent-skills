---
title: "A2 — Ledger Configuration Issues"
description: "Diagnose ledger configuration and access problems"
status: active
severity: MEDIUM
triggers: ["ledger error", "configuration issue", "deletion protection"]
owner: devops-agent
objective: "Identify and resolve ledger configuration issues"
---

## Phase 1 — Triage

MUST:
- Check ledger details: `aws qldb describe-ledger --name <name>`
- Verify deletion protection status
- Check encryption configuration
- Verify permissions mode (STANDARD or ALLOW_ALL)

SHOULD:
- Check if ledger is in ACTIVE state
- Verify KMS key is enabled
- Review CloudTrail for configuration changes

## Phase 2 — Remediate

MUST:
- Update permissions mode if needed: `aws qldb update-ledger --name <name> --permissions-mode STANDARD`
- Fix KMS key issues
- Enable/disable deletion protection as needed

## Output Format

```yaml
root_cause: "ledger_config — <specific_cause>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — configuration changes affect data access controls
- Service Disruption Risk: MEDIUM — misconfigurations can affect ledger availability
- Compliance Impact: HIGH — ledger configuration directly affects audit compliance

## Escalation Conditions
- Configuration changes affect ledger availability
- Deletion protection accidentally disabled
- Encryption configuration modified
- Permissions mode changed without understanding impact
- Configuration drift from approved baseline

## Data Sensitivity
- Ledger configuration parameters — INTERNAL
- Encryption settings — CONFIDENTIAL
- Permissions mode — CONFIDENTIAL
- Deletion protection status — INTERNAL
- Never expose ledger configuration details externally

## Prohibited Actions
- Do NOT disable deletion protection on production ledgers
- Do NOT change permissions mode without understanding impact
- Do NOT modify encryption settings without key management review
- Do NOT bypass configuration change management
- Do NOT modify configurations during active transactions
- Do NOT ignore configuration drift

## Phase 3 — Rollback
- Restore previous ledger configurations
- Re-enable deletion protection if disabled
- Revert permissions mode changes
- Restore encryption configurations
- Verify ledger health after configuration changes
- Document configuration changes and their impact

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
