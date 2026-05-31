---
title: "F2 — SOP Execution"
description: "Diagnose SOP execution issues"
status: active
severity: HIGH
triggers: ["SOP failed", "runbook execution error", "SSM document failed"]
owner: devops-agent
objective: "Resolve SOP execution failures"
context: "Generated SOPs are SSM documents that need customization. Execution failures are typically due to uncustomized parameters, IAM permissions, or resource access."
---

## Phase 1 — Triage

MUST:
- Check SSM document execution: `aws ssm list-command-invocations --filters "Key=DocumentName,Values=<doc-name>"`
- Review execution output for errors
- Verify SSM document parameters were customized

SHOULD:
- Check IAM role used for execution has required permissions
- Verify target resources exist and are accessible

MAY:
- Review SSM document content for placeholder values

## Phase 2 — Remediate

MUST:
- Customize all placeholder values in the SOP
- Ensure execution role has permissions for all actions in the SOP
- Test in non-production first

SHOULD:
- Add error handling to SOPs
- Document customization requirements

MAY:
- Create approval workflows for SOP execution

## Common Issues

- symptoms: "SOP fails with parameter validation error"
  diagnosis: "Placeholder parameters not replaced with actual values."
  resolution: "Review and customize all parameters in the SSM document."

## Output Format

```yaml
root_cause: "sop_execution — <specific_cause>"
evidence:
  - type: execution_results
    content: "<SSM execution status and errors>"
severity: HIGH
mitigation:
  immediate: "Fix SOP parameters and retry"
  long_term: "Maintain customized SOPs with regular testing"
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

- Application architecture details reveal infrastructure topology
- Assessment results may expose security weaknesses
- RTO/RPO configurations reveal business continuity priorities
- Classification: INTERNAL - contains infrastructure architecture details

## Prohibited Actions

- NEVER delete resiliency policies attached to production applications
- NEVER modify RTO/RPO targets without stakeholder approval
- NEVER dismiss assessment recommendations without documented justification
- NEVER remove application resources from resilience assessments without review

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
