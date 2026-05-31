---
title: "A2 — Data Source Issues"
description: "Diagnose Detective data source enablement and ingestion problems"
status: active
severity: HIGH
triggers:
  - "data source"
  - "data not ingesting"
  - "missing data"
  - "data source package"
  - "vpc flow logs detective"
owner: devops-agent
objective: "Resolve data source configuration issues and ensure data ingestion"
context: "Detective ingests data from multiple sources: GuardDuty findings, CloudTrail logs, VPC Flow Logs, EKS audit logs, and S3 data events. Each is a separate data source package that must be enabled. Data takes up to 24 hours to appear."
---

## Phase 1 — Triage

MUST:
- Check enabled data sources: `aws detective list-datasource-packages --graph-arn <graph-arn>`
- Check member data source status: `aws detective batch-get-graph-member-datasources --graph-arn <graph-arn> --account-ids <account-id>`
- Verify GuardDuty is active and generating findings
- Check if 24 hours have passed since enablement

SHOULD:
- Verify CloudTrail is enabled and logging
- Check if VPC Flow Logs are enabled (if VPC data source is needed)
- Verify EKS Protection is enabled in GuardDuty (for EKS data)

MAY:
- Check data volume estimates in Detective console
- Verify no service quotas are being exceeded

## Phase 2 — Remediate

MUST:
- Enable missing data sources: `aws detective update-datasource-packages --graph-arn <graph-arn> --datasource-packages '{"DETECTIVE_CORE":{"Status":"ENABLED"},"EKS_AUDIT":{"Status":"ENABLED"},"ASFF_SECURITYHUB_FINDING":{"Status":"ENABLED"}}'`
- Wait 24 hours for data to appear after enabling
- Ensure prerequisite services are active (GuardDuty, CloudTrail)

SHOULD:
- Enable data sources for all member accounts
- Monitor data volume for cost management

## Output Format

```yaml
root_cause: "data_source — <specific_cause>"
evidence:
  - type: datasource_status
    content: "<data source package status>"
  - type: member_datasources
    content: "<member data source status>"
severity: HIGH
mitigation:
  immediate: "Enable required data source packages"
  long_term: "Auto-enable data sources for new member accounts"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Data sources ingest security-relevant activity data |
| Blast Radius | MEDIUM — Disabling data sources creates investigation blind spots |
| Rollback Complexity | LOW — Re-enable data sources; but 24-hour delay for data |
| Compliance Scope | HIGH — Complete data source coverage is required for thorough investigations |

## Escalation Conditions

- Escalate to security team if data sources are disabled or not ingesting
- Escalate to cost management if data volume costs are a concern (especially VPC Flow Logs)
- Escalate to AWS Support if data ingestion fails despite correct configuration

## Data Sensitivity

- **Ingested data** includes CloudTrail logs, VPC Flow Logs, and GuardDuty findings — security-sensitive
- **Data source configuration** reveals security monitoring scope

## PROHIBITED Actions

- Do NOT disable data sources without security team approval
- Do NOT enable VPC Flow Logs data source without understanding cost implications
- Do NOT assume data appears immediately — 24-hour ingestion delay

## Phase 3 — Rollback

- Re-enable disabled data sources: `aws detective update-datasource-packages --graph-arn <arn> --datasource-packages <config>`
- Wait 24 hours for data to appear
- Document any monitoring gaps during the disabled period

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
  - "NEVER suggest disabling Detective to reduce costs"
  - "NEVER suggest ignoring investigation findings"
  - "NEVER expose investigation details publicly"
