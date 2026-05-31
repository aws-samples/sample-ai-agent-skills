---
title: "Z1 — General DRS Troubleshooting"
description: "Catch-all runbook for DRS issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "DRS error"
  - "DRS not working"
  - "disaster recovery issue"
owner: devops-agent
objective: "Systematic investigation of unclassified DRS issues"
context: "Use this runbook when the DRS issue doesn't clearly match a specific runbook. Follow the systematic investigation workflow to identify the problem domain, then pivot to the appropriate specific runbook."
---

## Phase 1 — Triage

MUST:
- List all source servers and their states: `aws drs describe-source-servers --query 'items[*].{Id:sourceServerID,State:lifeCycle.state,Replication:dataReplicationInfo.dataReplicationState}'`
- Check for recent failed jobs: `aws drs describe-jobs --filters 'fromDate=2024-01-01T00:00:00Z' --query 'items[?status==FAILED]'`
- Review CloudTrail for DRS errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=drs.amazonaws.com --max-results 20`
- Verify DRS service initialization: `aws drs describe-replication-configuration-templates`

SHOULD:
- Check IAM permissions for the calling identity
- Verify DRS is initialized in the target region
- Review recent configuration changes

MAY:
- Check AWS Health Dashboard for DRS service issues
- Review CloudWatch metrics for DRS

## Phase 2 — Remediate

MUST:
- Identify the specific problem domain and pivot to the appropriate runbook
- If DRS not initialized, run: `aws drs initialize-service`

SHOULD:
- Document the issue and resolution for future reference
- Set up CloudWatch alarms for DRS replication state changes

MAY:
- Implement automated monitoring for DRS health

## Output Format

```yaml
root_cause: "general_drs — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings from systematic investigation>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and alerting for DRS health"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — General troubleshooting may expose DR architecture |
| Blast Radius | LOW — Diagnostic commands are read-only |
| Rollback Complexity | LOW — No changes during triage |
| Compliance Scope | HIGH — DRS troubleshooting supports DR compliance |

## Escalation Conditions

- Escalate to the specific runbook owner once the problem domain is identified
- Escalate to DR coordinator if DRS is non-functional
- Escalate to AWS Support if systematic issues affect DRS operations

## Data Sensitivity

- **Source server configurations** reveal protected infrastructure
- **Replication states** indicate DR readiness
- **CloudTrail events** contain API call details

## PROHIBITED Actions

- Do NOT modify DRS configurations during initial triage
- Do NOT disconnect source servers during troubleshooting
- Do NOT share DR architecture details outside authorized teams

## Phase 3 — Rollback

- No rollback needed for diagnostic operations
- Revert any configuration changes made during troubleshooting
- Route to the appropriate specific runbook for remediation

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
