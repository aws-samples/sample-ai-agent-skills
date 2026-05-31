---
title: "Z1 — General Mainframe Modernization Troubleshooting"
description: "Catch-all runbook for Mainframe Modernization issues"
status: active
severity: MEDIUM
triggers: ["M2 error", "mainframe modernization not working", "M2 issue", "modernization problem"]
owner: devops-agent
objective: "Systematic investigation of unclassified Mainframe Modernization issues"
context: "Use when the issue doesn't clearly match a specific runbook. Covers general diagnostic steps for M2 environments and applications."
---

## Phase 1 — Triage

MUST:
- List environments: `aws m2 list-environments --query 'environments[*].{Name:name,Id:environmentId,Status:status,EngineType:engineType}'`
- List applications: `aws m2 list-applications --query 'applications[*].{Name:name,Id:applicationId,Status:status,EngineType:engineType}'`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=m2.amazonaws.com --max-results 20`

SHOULD:
- Check application logs for errors
- Verify environment health metrics
- Review recent deployments

MAY:
- Check AWS Health Dashboard for M2 service issues
- Review recent configuration changes

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1-F1)

SHOULD:
- Document findings and investigation path

MAY:
- Set up comprehensive monitoring if not already configured

## Safety Ratings

```
safety_ratings:
  - "list-environments: GREEN — read-only enumeration"
  - "list-applications: GREEN — read-only enumeration"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- M2 service appears non-functional in the Region
- Multiple environments or applications in Failed state
- No clear error messages in logs or CloudTrail
- Issue cannot be classified into any specific runbook after investigation

## Data Sensitivity

- `aws m2 list-environments`: LOW — returns environment metadata
- `aws m2 list-applications`: LOW — returns application metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER delete environments or applications during investigation
- NEVER dismiss the issue without documenting the investigation path
- NEVER modify application definitions as a first troubleshooting step

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore environment and application settings if modified
3. Verify all environments and applications are functioning after investigation
4. Document all investigation steps and findings
5. Verify M2 service is functioning normally before closing

## Output Format

```yaml
root_cause: "general_m2 — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and documentation"
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
