---
title: "E1 — State Manager Association Failures"
description: "Diagnose State Manager association execution failures"
status: active
severity: MEDIUM
triggers:
  - "association failed"
  - "State Manager error"
  - "association not applying"
  - "association status Failed"
owner: devops-agent
objective: "Identify why State Manager associations are failing and fix execution"
context: "State Manager associations apply SSM documents to instances on a schedule or at launch. Failures can be due to document errors, targeting issues, IAM permissions, or instance-level execution problems. Associations track status per instance."
---

## Phase 1 — Triage

MUST:
- List associations: `aws ssm list-associations --query 'Associations[*].{Id:AssociationId,Name:Name,Status:Overview.Status,Targets:Targets}'`
- Describe failed association: `aws ssm describe-association --association-id <assoc-id>`
- Check association execution history: `aws ssm describe-association-executions --association-id <assoc-id> --filters Key=Status,Value=Failed`
- Get execution details per instance: `aws ssm describe-association-execution-targets --association-id <assoc-id> --execution-id <exec-id>`

SHOULD:
- Verify the SSM document is valid: `aws ssm describe-document --name <document-name>`
- Check association parameters and schedule
- Verify target instances are managed

MAY:
- Check CloudTrail for association events
- Review association compliance: `aws ssm list-compliance-items --resource-ids <instance-id> --resource-types ManagedInstance --filters Key=ComplianceType,Values=Association`

## Phase 2 — Remediate

MUST:
- Fix the document or parameters causing the failure
- Ensure target instances are managed and connected
- Update the association after fixing the issue

SHOULD:
- Test the document manually via Run Command before creating an association
- Set appropriate schedule expressions
- Configure association compliance severity

MAY:
- Use association rate control for large fleets
- Implement association monitoring with EventBridge

## Common Issues

- symptoms: "Association status shows Failed for all instances"
  diagnosis: "Document error or invalid parameters."
  resolution: "Test the document manually. Fix parameters and update association."

- symptoms: "Association applies to some instances but not others"
  diagnosis: "Some instances not managed or targeting criteria mismatch."
  resolution: "Verify all target instances are managed and match targeting."

- symptoms: "Association runs but drift reoccurs"
  diagnosis: "External changes override association configuration between runs."
  resolution: "Increase association frequency or address the source of drift."

## Output Format

```yaml
root_cause: "association_failure — <specific_cause>"
evidence:
  - type: association_details
    content: "<describe-association output>"
  - type: execution_history
    content: "<failed execution details>"
severity: MEDIUM
mitigation:
  immediate: "Fix document or parameters and retry"
  long_term: "Implement association testing and monitoring"
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
