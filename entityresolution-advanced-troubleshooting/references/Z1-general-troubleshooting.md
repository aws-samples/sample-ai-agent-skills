---
title: "Z1 — General Troubleshooting"
description: "General advanced Entity Resolution troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["entity resolution issue", "entity resolution not working", "general entity resolution problem", "entity resolution help"]
owner: devops-agent
objective: "Provide general troubleshooting guidance for advanced Entity Resolution issues"
context: "This catch-all runbook covers general advanced Entity Resolution troubleshooting when the issue does not clearly map to a specific runbook. Start here and escalate to specific runbooks as the issue becomes clearer."
---

## Phase 1 — Triage

MUST:
- List matching workflows: `aws entityresolution list-matching-workflows`
- List ID mapping workflows: `aws entityresolution list-id-mapping-workflows`
- List schema mappings: `aws entityresolution list-schema-mappings`

SHOULD:
- Check recent job statuses for failures
- Review CloudTrail for Entity Resolution events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=entityresolution.amazonaws.com --max-results 20`
- Verify IAM permissions for the service role

MAY:
- Check AWS Health Dashboard for Entity Resolution service issues
- Review service quotas: `aws service-quotas list-service-quotas --service-code entityresolution`
- Test basic workflow execution with sample data

## Phase 2 — Remediate

MUST:
- Address the specific error identified during triage
- Refer to the appropriate specific runbook (A1-F1) based on findings
- Verify the fix resolves the issue

SHOULD:
- Document the issue and resolution
- Set up monitoring for the specific failure mode
- Review related configurations for similar issues

MAY:
- Implement preventive measures
- Create data quality dashboards
- Review and update workflows based on lessons learned

## Safety Ratings

```
safety_ratings:
  - "list-matching-workflows: GREEN — read-only workflow listing"
  - "list-id-mapping-workflows: GREEN — read-only workflow listing"
  - "list-schema-mappings: GREEN — read-only schema listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log access"
```

## Escalation Conditions

- Issue cannot be categorized into any specific runbook after initial triage
- Multiple Entity Resolution features failing simultaneously
- Entity Resolution service appears to be experiencing regional issues
- Issue requires AWS Support engagement

## Data Sensitivity

- `aws entityresolution list-matching-workflows`: LOW — returns workflow names
- `aws entityresolution list-id-mapping-workflows`: LOW — returns workflow names
- `aws entityresolution list-schema-mappings`: LOW — returns schema names
- `aws cloudtrail lookup-events`: MEDIUM — reveals IAM principals and actions

## Prohibited Actions

- NEVER make changes to production workflows without understanding the root cause
- NEVER delete workflows or schema mappings to troubleshoot
- NEVER ignore recurring job failures
- NEVER modify IAM permissions broadly to fix specific Entity Resolution issues

## Phase 3 — Rollback

1. Revert any diagnostic changes made during troubleshooting
2. Restore original workflow configurations if modified
3. Verify all Entity Resolution workflows function correctly after troubleshooting
4. Document findings and share with the team
5. Schedule a review if the root cause was not fully identified

## Common Issues

- symptoms: "All Entity Resolution jobs suddenly failing"
  diagnosis: "IAM policy change, Glue catalog issue, or Entity Resolution service issue."
  resolution: "Check recent IAM changes. Verify Glue catalog access. Review CloudTrail for policy modifications."

- symptoms: "Entity Resolution works for one workflow but not another"
  diagnosis: "Workflow-specific configuration issue — different schema mappings, roles, or data sources."
  resolution: "Compare working and failing workflow configurations. Check schema mappings and service roles."

- symptoms: "Job execution time significantly increased"
  diagnosis: "Input data volume increased, or data quality degraded causing more comparison operations."
  resolution: "Check input data volume. Review data quality. Consider optimizing matching rules."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details and diagnostic findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and preventive measures"
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
