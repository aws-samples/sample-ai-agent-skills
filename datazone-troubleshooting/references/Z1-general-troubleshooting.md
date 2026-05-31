# Z1 — DataZone General Troubleshooting

## When No Specific Runbook Matches

## General Diagnostic Commands

```bash
aws datazone list-domains --max-results 20
aws datazone get-domain --identifier <domain-id>
aws datazone list-projects --domain-identifier <domain-id> --max-results 20
aws datazone list-environments --domain-identifier <domain-id> --project-identifier <project-id>
aws datazone search --domain-identifier <domain-id> --search-scope ASSET --search-text "*" --max-results 10
aws datazone list-subscriptions --domain-identifier <domain-id> --max-results 20
aws datazone list-subscription-requests --domain-identifier <domain-id> --max-results 20
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=datazone.amazonaws.com --max-results 20
```

## Escalation Checklist
1. Domain ID and status
2. Project IDs and member configuration
3. Environment IDs and blueprint types
4. Asset IDs and publishing status
5. Subscription request and grant status
6. IAM roles and DataZone permissions
7. Underlying data source status (Glue/Redshift/Athena)
8. CloudTrail events for recent changes


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — General troubleshooting may expose domain and project configurations |
| Blast Radius | LOW — Diagnostic commands are read-only |
| Rollback Complexity | LOW — No changes during triage |
| Compliance Scope | MEDIUM — DataZone troubleshooting supports data governance operations |

## Escalation Conditions

- Escalate to the specific runbook owner once the problem domain is identified
- Escalate to data governance team if troubleshooting reveals governance gaps
- Escalate to AWS Support if systematic issues affect DataZone operations

## Data Sensitivity

- **Domain and project configurations** reveal data governance structure
- **CloudTrail events** contain API call details and account identifiers
- **Subscription and asset data** may reveal data access patterns

## PROHIBITED Actions

- Do NOT modify domain or project configurations during initial triage
- Do NOT share data governance structure details outside authorized teams
- Do NOT approve or reject subscription requests during troubleshooting

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
