---
title: "Z1 — General Troubleshooting"
description: "Catch-all troubleshooting for Lookout for Vision issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Lookout for Vision error", "unknown issue", "general problem", "not sure what's wrong"]
owner: devops-agent
objective: "Systematically diagnose and resolve unclassified Lookout for Vision issues"
context: "Use this runbook when the issue does not clearly match a specific runbook. Follow systematic triage to identify the correct domain and route to the appropriate runbook."
---

## Phase 1 — Triage

MUST:
- List projects: `aws lookoutvision list-projects --query 'Projects[*].{Name:ProjectName,Arn:ProjectArn}'`
- List models: `aws lookoutvision list-models --project-name <project> --query 'Models[*].{Version:ModelVersion,Status:Status}'`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=lookoutvision.amazonaws.com --max-results 20`

SHOULD:
- Check AWS Health Dashboard for service issues
- Verify IAM permissions for the calling principal
- Check S3 bucket accessibility

MAY:
- Review CloudWatch metrics for anomalies
- Check if the issue correlates with recent changes

## Phase 2 — Remediate

MUST:
- Identify the domain (project, dataset, training, inference, edge, quality, permissions) and route to the appropriate runbook
- If domain is unclear, check permissions first (most common), then data access, then model status
- Document findings for the specific runbook

SHOULD:
- Verify all prerequisites: project exists, dataset imported, model trained
- Check region consistency across all resources

MAY:
- Open AWS Support case if issue persists
- Check AWS documentation for known issues

## Common Issues

- symptoms: "Vague error with no clear cause"
  diagnosis: "Often IAM permissions or S3 access issue."
  resolution: "Start with S3 permissions (E1), then model status (C1), then dataset (A2)."

- symptoms: "Everything was working and suddenly stopped"
  diagnosis: "Model was stopped (costs), IAM policy changed, or S3 bucket modified."
  resolution: "Check model status (must be HOSTED for inference). Review CloudTrail for recent changes."

## Safety Ratings

```
safety_ratings:
  - "lookoutvision list-projects: GREEN — read-only enumeration"
  - "lookoutvision list-models: GREEN — read-only enumeration"
  - "cloudtrail lookup-events: GREEN — read-only event retrieval"
  - "lookoutvision describe-model: GREEN — read-only model details"
```

## Escalation Conditions

- Issue persists after checking all common causes
- Multiple Lookout for Vision features fail simultaneously
- AWS Health Dashboard shows active service event
- Issue affects production quality inspection pipeline

## Data Sensitivity

- `aws lookoutvision list-projects`: LOW — returns project metadata
- `aws lookoutvision list-models`: LOW — returns model metadata
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities
- Error messages: LOW — operational diagnostic information

## Prohibited Actions

- NEVER delete projects or models as a first troubleshooting step
- NEVER stop running models without confirming no production dependency
- NEVER modify production configurations without understanding root cause
- NEVER assume the issue is resolved without verification

## Phase 3 — Rollback

1. General troubleshooting should not require rollback — it is diagnostic
2. If any changes were made during investigation, document and revert
3. Route to the specific runbook for domain-specific rollback
4. Verify system state matches pre-investigation state
5. Document the investigation path for future reference

## Output Format

```yaml
root_cause: "general — <specific_cause_or_domain>"
evidence:
  - type: diagnostic
    content: "<findings from systematic triage>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Improve monitoring and alerting"
```

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
