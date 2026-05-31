---
title: "E2 — Configuration Drift Detection"
description: "Diagnose State Manager drift detection and remediation issues"
status: active
severity: MEDIUM
triggers:
  - "configuration drift"
  - "drift detection"
  - "state not maintained"
  - "configuration changed"
owner: devops-agent
objective: "Identify configuration drift and ensure State Manager maintains desired state"
context: "State Manager maintains desired configuration by running associations on schedule. Drift occurs when configuration changes between association runs. Detection relies on association compliance status and comparison of actual vs desired state."
---

## Phase 1 — Triage

MUST:
- Check association compliance: `aws ssm list-compliance-items --resource-ids <instance-id> --resource-types ManagedInstance --filters Key=ComplianceType,Values=Association`
- List association execution history: `aws ssm describe-association-executions --association-id <assoc-id> --max-results 10`
- Check last successful execution output
- Verify association schedule frequency

SHOULD:
- Compare current instance state with desired state
- Check for external changes (manual modifications, other automation)
- Review association parameters for completeness

MAY:
- Use AWS Config for continuous drift detection
- Check CloudTrail for manual changes between association runs

## Phase 2 — Remediate

MUST:
- Run the association immediately to remediate drift: `aws ssm start-associations-once --association-ids <assoc-id>`
- Verify the association document covers all configuration aspects
- Ensure the schedule frequency is appropriate for the drift rate

SHOULD:
- Increase association frequency if drift occurs frequently
- Implement change detection mechanisms
- Document expected configuration state

MAY:
- Use AWS Config rules for real-time drift detection
- Implement EventBridge rules to trigger remediation on changes

## Common Issues

- symptoms: "Configuration reverts between association runs"
  diagnosis: "External process or user modifying configuration."
  resolution: "Identify the source of changes. Increase association frequency."

- symptoms: "Association reports compliant but configuration is wrong"
  diagnosis: "Association document does not check all configuration aspects."
  resolution: "Update the document to cover all desired configuration items."

- symptoms: "Drift detected but association cannot remediate"
  diagnosis: "Association document lacks remediation steps or permissions."
  resolution: "Update document with remediation actions. Verify IAM permissions."

## Output Format

```yaml
root_cause: "drift_detection — <specific_cause>"
evidence:
  - type: compliance_status
    content: "<association compliance details>"
  - type: execution_history
    content: "<recent execution results>"
severity: MEDIUM
mitigation:
  immediate: "Run association to remediate current drift"
  long_term: "Implement continuous drift detection and automated remediation"
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
