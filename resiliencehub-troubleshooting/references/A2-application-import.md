---
title: "A2 — Application Import"
description: "Diagnose resource import failures"
status: active
severity: HIGH
triggers: ["import failed", "resources not imported", "import status error"]
owner: devops-agent
objective: "Resolve resource import issues"
context: "Importing resources from CFN stacks, Resource Groups, or Terraform state files into a Resilience Hub application."
---

## Phase 1 — Triage

MUST:
- Check import status: `aws resiliencehub describe-draft-app-version-resources-import-status --app-arn <arn>`
- Verify source (CFN stack/Resource Group) exists and is accessible
- Check IAM permissions for reading source resources

SHOULD:
- Verify CFN stack is in a stable state (CREATE_COMPLETE, UPDATE_COMPLETE)
- Check Resource Group query returns expected resources

MAY:
- For Terraform: verify S3 state file is accessible and valid format

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to read source resources
- Ensure source is in a valid state

SHOULD:
- Retry import after fixing issues: `aws resiliencehub import-resources-to-draft-app-version --app-arn <arn>`

MAY:
- Add resources manually if import source is problematic

## Common Issues

- symptoms: "Import shows 0 resources discovered"
  diagnosis: "CFN stack has no resources or Resource Group query returns empty."
  resolution: "Verify source contains resources. Check Resource Group tag filters."

## Output Format

```yaml
root_cause: "application_import — <specific_cause>"
evidence:
  - type: import_status
    content: "<import status and resource count>"
severity: HIGH
mitigation:
  immediate: "Fix import source or permissions"
  long_term: "Validate resource sources before import"
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
