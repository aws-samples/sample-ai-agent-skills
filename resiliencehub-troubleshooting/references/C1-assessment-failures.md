---
title: "C1 — Assessment Failures"
description: "Diagnose assessment execution failures"
status: active
severity: HIGH
triggers: ["assessment failed", "assessment error", "cannot run assessment"]
owner: devops-agent
objective: "Resolve assessment execution failures"
context: "Assessments evaluate application architecture against resiliency policy. Failures can be due to permissions, resource access, or service issues."
---

## Phase 1 — Triage

MUST:
- Check assessment status: `aws resiliencehub list-app-assessments --app-arn <arn> --query 'assessmentSummaries[0]'`
- Get assessment details: `aws resiliencehub describe-app-assessment --assessment-arn <arn>`
- Verify app has published version with resources

SHOULD:
- Check IAM permissions for reading assessed resources
- Verify all resource sources are accessible
- Check CloudTrail for assessment errors

MAY:
- Check if assessment is still in progress (can take minutes)

## Phase 2 — Remediate

MUST:
- Ensure IAM role can read all resources in the application
- Publish app version before assessment if using draft
- Retry assessment after fixing issues

SHOULD:
- Verify resource sources haven't changed since import

MAY:
- Run assessment from console for better error visibility

## Common Issues

- symptoms: "Assessment status FAILED with no details"
  diagnosis: "IAM permissions insufficient to read application resources."
  resolution: "Add read permissions for all resource types in the application."

## Output Format

```yaml
root_cause: "assessment_failure — <specific_cause>"
evidence:
  - type: assessment_status
    content: "<assessment status and error>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or resource access"
  long_term: "Validate permissions before running assessments"
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
