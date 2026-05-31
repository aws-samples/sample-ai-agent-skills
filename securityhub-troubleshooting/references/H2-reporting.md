---
title: "H2 — Compliance Reporting Issues"
description: "Diagnose Security Hub compliance reporting and insight issues"
status: active
severity: MEDIUM
triggers:
  - "compliance report"
  - "security report"
  - "insight results"
  - "compliance dashboard"
  - "reporting issues"
owner: devops-agent
objective: "Identify and resolve compliance reporting and insight configuration issues"
context: "Security Hub provides compliance reporting through insights (managed and custom), finding exports, and integration with other services. Issues arise from insight configuration, finding filters, export setup, or data completeness."
---

## Phase 1 — Triage

MUST:
- Check available insights: `aws securityhub get-insights`
- Get insight results: `aws securityhub get-insight-results --insight-arn <insight-arn>`
- Verify findings are being generated: `aws securityhub get-findings --max-results 10`
- Check enabled standards for compliance data: `aws securityhub get-enabled-standards`
- Verify Config is enabled for control evaluation

SHOULD:
- Check custom insights for configuration issues
- Verify finding filters are not excluding relevant data
- Check cross-region and cross-account aggregation for complete reporting

MAY:
- Check EventBridge rules for finding export to S3
- Verify third-party reporting tool integrations

## Phase 2 — Remediate

MUST:
- Fix insight filters to include the correct finding scope
- Ensure all relevant standards are enabled for compliance reporting
- Verify Config is enabled for complete control evaluation

SHOULD:
- Create custom insights for specific reporting needs: `aws securityhub create-insight --name <name> --filters <filters> --group-by-attribute <attribute>`
- Set up finding export to S3 for long-term reporting
- Configure cross-region aggregation for comprehensive reports

MAY:
- Create automated reporting workflows with EventBridge and Lambda
- Set up scheduled finding exports for compliance audits

## Common Issues

- symptoms: "Insight shows no results or incomplete data"
  diagnosis: "The insight filters are too restrictive, or findings are not being generated for the filtered criteria."
  resolution: "Review and broaden insight filters. Verify findings exist for the criteria."

- symptoms: "Compliance report missing data from some accounts or regions"
  diagnosis: "Cross-account or cross-region aggregation is not configured."
  resolution: "Set up cross-account member associations and cross-region finding aggregation."

- symptoms: "Report shows stale compliance data"
  diagnosis: "Config rules have not re-evaluated, or finding updates are delayed."
  resolution: "Force Config rule re-evaluation. Allow time for finding propagation."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix insight filters | YELLOW | Insight modification; reversible |
| Create custom insights | YELLOW | New resource; can be deleted |
| Set up finding export | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Cross-account configuration changes
- Reporting pipeline changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-insights` | LOW | Insight configuration |
| `get-insight-results` | MEDIUM | Aggregated finding data |
| `get-findings` | HIGH | Security finding details |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Insight filter fix | Revert via `update-insight` |
| Custom insight creation | Delete via `delete-insight` |
| Export configuration | Remove export configuration |

## Output Format

```yaml
root_cause: "reporting — <specific_cause>"
evidence:
  - type: insights
    content: "<insight configuration and results>"
  - type: findings
    content: "<finding data completeness>"
  - type: aggregation
    content: "<cross-region/cross-account coverage>"
severity: MEDIUM
mitigation:
  immediate: "Fix insight filters and verify data completeness"
  long_term: "Set up comprehensive aggregation and automated reporting"
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
