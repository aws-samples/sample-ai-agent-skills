---
title: "A3 — Resource Discovery"
description: "Diagnose resource discovery issues"
status: active
severity: MEDIUM
triggers: ["missing resources", "resources not found", "incomplete discovery"]
owner: devops-agent
objective: "Resolve resource discovery gaps"
context: "Resilience Hub discovers resources from CFN, Resource Groups, AppRegistry, or Terraform. Missing resources lead to incomplete assessments."
---

## Phase 1 — Triage

MUST:
- List discovered resources: `aws resiliencehub list-app-version-resources --app-arn <arn> --app-version release`
- Compare with expected resources
- Verify resource sources include all relevant stacks/groups

SHOULD:
- Check if missing resources are in a supported service
- Verify Resource Group tag filters match expected resources

MAY:
- Check if resources were created after last import

## Phase 2 — Remediate

MUST:
- Add missing resource sources (CFN stacks, Resource Groups)
- Re-import resources after adding sources

SHOULD:
- Publish new app version after import
- Verify all critical resources are discovered

MAY:
- Use AppRegistry for comprehensive resource tracking

## Common Issues

- symptoms: "RDS instance not in assessment"
  diagnosis: "RDS instance not in any CFN stack or Resource Group."
  resolution: "Add RDS instance to a Resource Group or CFN stack, then re-import."

## Output Format

```yaml
root_cause: "resource_discovery — <specific_cause>"
evidence:
  - type: resource_list
    content: "<discovered vs expected resources>"
severity: MEDIUM
mitigation:
  immediate: "Add missing resource sources"
  long_term: "Ensure all resources are managed via CFN or Resource Groups"
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
