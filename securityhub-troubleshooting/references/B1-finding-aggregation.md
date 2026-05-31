---
title: "B1 — Finding Aggregation Issues"
description: "Diagnose Security Hub finding aggregation and collection problems"
status: active
severity: MEDIUM
triggers:
  - "finding aggregation"
  - "findings missing"
  - "findings not appearing"
  - "finding collection"
  - "finding import"
owner: devops-agent
objective: "Identify and resolve finding aggregation issues in Security Hub"
context: "Security Hub aggregates findings from multiple sources: AWS services (GuardDuty, Config, etc.), third-party products, and custom integrations. Issues arise from disabled integrations, ASFF format errors, import failures, or cross-account/cross-region aggregation gaps."
---

## Phase 1 — Triage

MUST:
- Check Security Hub status: `aws securityhub describe-hub`
- List enabled product integrations: `aws securityhub list-enabled-products-for-import`
- Get recent findings: `aws securityhub get-findings --max-results 20`
- Check for finding import failures in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=BatchImportFindings --max-results 10`
- Verify the expected source products are enabled

SHOULD:
- Check finding count by product: `aws securityhub get-insight-results --insight-arn arn:aws:securityhub:::insight/securityhub/default/2`
- Verify cross-region aggregation: `aws securityhub list-finding-aggregators`
- Check member accounts: `aws securityhub list-members`

MAY:
- Check CloudWatch metrics for finding ingestion rates
- Verify third-party product configurations

## Phase 2 — Remediate

MUST:
- Enable missing product integrations
- Fix ASFF format errors in custom finding imports
- Verify findings start appearing from all expected sources

SHOULD:
- Set up cross-region aggregation if centralized view is needed
- Configure cross-account aggregation for multi-account environments

MAY:
- Create custom insights to monitor finding sources
- Set up alerts for finding ingestion failures

## Common Issues

- symptoms: "Findings from GuardDuty not appearing in Security Hub"
  diagnosis: "GuardDuty integration is not enabled, or GuardDuty is not enabled in the same region."
  resolution: "Enable GuardDuty and verify the Security Hub integration is active in list-enabled-products-for-import."

- symptoms: "Custom findings failing to import with BatchImportFindings"
  diagnosis: "The findings do not conform to ASFF format or are missing required fields."
  resolution: "Validate ASFF format. Ensure all required fields are present: SchemaVersion, Id, ProductArn, GeneratorId, AwsAccountId, Types, CreatedAt, UpdatedAt, Severity, Title, Resources."

- symptoms: "Findings from member accounts not visible in admin account"
  diagnosis: "Cross-account aggregation is not configured or member accounts are not associated."
  resolution: "Associate member accounts and verify the admin-member relationship."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable product integrations | YELLOW | Configuration change; reversible |
| Fix ASFF format errors | GREEN | Data correction; no state change |
| Configure cross-region aggregation | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Cross-account configuration changes
- Standard enablement/disablement
- Integration changes affecting finding visibility

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-hub` | LOW | Hub status |
| `list-enabled-products-for-import` | LOW | Product integration list |
| `get-findings` | HIGH | Security findings with resource details |
| CloudTrail events | MEDIUM | API call history |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Product integration enablement | Disable via `disable-import-findings-for-product` |
| Cross-region aggregation | Delete aggregator via `delete-finding-aggregator` |

## Output Format

```yaml
root_cause: "finding_aggregation — <specific_cause>"
evidence:
  - type: integrations
    content: "<enabled product integrations>"
  - type: findings
    content: "<finding sources and counts>"
  - type: aggregation
    content: "<cross-region/cross-account configuration>"
severity: MEDIUM
mitigation:
  immediate: "Enable missing integrations and fix import errors"
  long_term: "Monitor finding ingestion and set up comprehensive aggregation"
```

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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
