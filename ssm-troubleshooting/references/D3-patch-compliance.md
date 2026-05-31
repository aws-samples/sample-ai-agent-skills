---
title: "D3 — Patch Compliance"
description: "Diagnose patch compliance reporting and gap issues"
status: active
severity: MEDIUM
triggers:
  - "patch compliance"
  - "non-compliant instances"
  - "missing patches"
  - "compliance report"
owner: devops-agent
objective: "Identify patch compliance gaps and ensure accurate compliance reporting"
context: "Patch compliance is tracked per instance after scan or install operations. Compliance data shows installed, missing, failed, and not-applicable patches. Compliance is per patch group and can be aggregated across the fleet."
---

## Phase 1 — Triage

MUST:
- Check patch group compliance: `aws ssm describe-patch-group-state --patch-group <group-name>`
- Check instance patch state: `aws ssm describe-instance-patch-states --instance-ids <instance-id>`
- List non-compliant patches: `aws ssm describe-instance-patches --instance-id <instance-id> --filters Key=State,Values=Missing`
- Verify last scan time in patch state output

SHOULD:
- Check compliance summary across all instances: `aws ssm list-compliance-summaries --filters Key=ComplianceType,Values=Patch`
- Verify patch baseline is correctly assigned to the patch group
- Check if scan was run recently

MAY:
- Export compliance data to S3 via Resource Data Sync: `aws ssm list-resource-data-sync`
- Compare compliance across patch groups
- Check for patches that failed to install: `aws ssm describe-instance-patches --instance-id <instance-id> --filters Key=State,Values=Failed`

## Phase 2 — Remediate

MUST:
- Run a fresh scan to get current compliance data
- Address missing critical and security patches
- Investigate and fix failed patch installations

SHOULD:
- Schedule regular scan operations via maintenance windows
- Set up compliance notifications via EventBridge
- Document patch exception justifications

MAY:
- Use AWS Config rules for continuous compliance monitoring
- Implement automated remediation for non-compliant instances

## Common Issues

- symptoms: "Compliance data is stale or empty"
  diagnosis: "No recent scan operation has been run."
  resolution: "Run AWS-RunPatchBaseline with Scan operation to refresh compliance data."

- symptoms: "Instance shows non-compliant but patches are installed"
  diagnosis: "Reboot required after patch installation but not performed."
  resolution: "Reboot the instance to complete patch installation."

- symptoms: "Compliance shows patches as NotApplicable"
  diagnosis: "Patches are for a different OS version or architecture."
  resolution: "Verify baseline rules match the instance OS version."

## Output Format

```yaml
root_cause: "patch_compliance — <specific_cause>"
evidence:
  - type: patch_group_state
    content: "<patch group compliance summary>"
  - type: instance_patches
    content: "<missing or failed patches>"
severity: MEDIUM
mitigation:
  immediate: "Run scan and address missing patches"
  long_term: "Implement continuous compliance monitoring"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Parameter Store may contain secrets, API keys, and credentials
- Command output may contain system configuration details
- Session logs may capture sensitive terminal activity
- Classification: HIGHLY CONFIDENTIAL - secrets management and system access data

## Prohibited Actions

- NEVER execute untested commands via Run Command on production instances
- NEVER modify patch baselines during active patching windows
- NEVER delete SSM parameters containing production secrets without rotation
- NEVER deregister managed instances without confirming no active sessions

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
