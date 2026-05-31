---
title: "Z1 — General Troubleshooting"
description: "Catch-all troubleshooting for Monitron issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Monitron error", "unknown issue", "general problem", "not sure what's wrong"]
owner: devops-agent
objective: "Systematically diagnose and resolve unclassified Monitron issues"
context: "Use this runbook when the issue does not clearly match a specific runbook. Follow systematic triage to identify the correct domain and route to the appropriate runbook."
---

## Phase 1 — Triage

MUST:
- Check project status: `aws monitron list-projects --query 'projectSummaries[*].{Name:name,Arn:projectArn}'`
- Get project details: `aws monitron get-project --project-id <project-id>`
- Review CloudTrail for recent Monitron events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=monitron.amazonaws.com --max-results 20`

SHOULD:
- Check AWS Health Dashboard for Monitron service issues
- Verify IAM permissions for the calling principal
- Check gateway LED status for hardware issues

MAY:
- Review CloudWatch metrics for anomalies
- Check if the issue correlates with recent changes

## Phase 2 — Remediate

MUST:
- Identify the domain (project, gateway, sensor, inference, connectivity, users, export) and route to the appropriate runbook
- If domain is unclear, check hardware first (gateway LED, sensor status), then software (permissions, configuration)
- Document findings for the specific runbook

SHOULD:
- Verify all prerequisites: project exists, gateway online, sensors paired, users provisioned
- Check region consistency

MAY:
- Open AWS Support case if issue persists
- Check AWS forums for known issues

## Common Issues

- symptoms: "Vague error with no clear cause"
  diagnosis: "Often gateway connectivity or IAM permission issue."
  resolution: "Start with gateway LED check (D1), then IAM permissions (E1), then sensor status (B2)."

- symptoms: "Everything was working and suddenly stopped"
  diagnosis: "WiFi change, power outage, or IAM policy modification."
  resolution: "Check gateway connectivity (D1). Review recent CloudTrail events for configuration changes."

## Safety Ratings

```
safety_ratings:
  - "monitron list-projects: GREEN — read-only enumeration"
  - "monitron get-project: GREEN — read-only detail retrieval"
  - "cloudtrail lookup-events: GREEN — read-only event retrieval"
  - "Physical gateway inspection: GREEN — visual observation only"
```

## Escalation Conditions

- Issue persists after checking all common causes
- Multiple Monitron components fail simultaneously
- AWS Health Dashboard shows active service event
- Issue affects safety-critical equipment monitoring

## Data Sensitivity

- `aws monitron list-projects`: LOW — returns project metadata
- `aws monitron get-project`: LOW — returns project details
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities
- Error messages: LOW — operational diagnostic information

## Prohibited Actions

- NEVER factory reset gateways as a first troubleshooting step
- NEVER delete projects or remove sensors without understanding the root cause
- NEVER ignore issues on safety-critical equipment
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
