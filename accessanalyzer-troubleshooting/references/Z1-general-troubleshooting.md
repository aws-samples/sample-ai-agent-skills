---
title: "Z1 — General Access Analyzer Troubleshooting"
description: "Catch-all runbook for Access Analyzer issues"
status: active
severity: MEDIUM
triggers: ["Access Analyzer error", "access analyzer not working"]
owner: devops-agent
objective: "Systematic investigation of unclassified Access Analyzer issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List analyzers: `aws accessanalyzer list-analyzers`
- Check for active findings: `aws accessanalyzer list-findings --analyzer-arn <arn> --filter '{"status":{"eq":["ACTIVE"]}}'`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=access-analyzer.amazonaws.com --max-results 20`

SHOULD:
- Check IAM permissions
- Verify analyzer is ACTIVE

MAY:
- Check AWS Health Dashboard

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings

MAY:
- Set up monitoring

## Safety Ratings

```
safety_ratings:
  - "list-analyzers: GREEN — read-only enumeration"
  - "list-findings: GREEN — read-only finding retrieval"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "check AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Access Analyzer service appears non-functional across multiple regions
- No findings generated despite known external access patterns in the account
- CloudTrail shows unauthorized API calls to Access Analyzer
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws accessanalyzer list-analyzers`: LOW — returns analyzer metadata
- `aws accessanalyzer list-findings`: MEDIUM — reveals external access patterns
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER make configuration changes during general troubleshooting without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path and findings

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them to the pre-investigation state
2. Restore analyzer settings if they were modified: verify analyzer type, status, and archive rules
3. If findings were archived during investigation, unarchive them: `aws accessanalyzer update-findings --analyzer-arn <arn> --ids <finding-ids> --status ACTIVE`
4. Document all investigation steps and findings for the security team
5. Verify the Access Analyzer service is functioning normally before closing the investigation

## Output Format

```yaml
root_cause: "general_accessanalyzer — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
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
