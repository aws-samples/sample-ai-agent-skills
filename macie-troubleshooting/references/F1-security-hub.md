---
title: "F1 — Security Hub Integration"
description: "Diagnose Macie integration issues with Security Hub"
status: active
severity: MEDIUM
triggers:
  - "findings not in Security Hub"
  - "Macie Security Hub"
  - "integration broken"
owner: devops-agent
objective: "Ensure Macie findings flow correctly to Security Hub"
context: "Macie automatically sends findings to Security Hub when both services are enabled in the same region. Issues arise when Security Hub is not enabled, the integration is disabled, or findings are filtered."
---

## Phase 1 — Triage

MUST:
- Check Security Hub enabled: `aws securityhub describe-hub`
- Check Macie integration: `aws securityhub list-enabled-products-for-import --query 'ProductSubscriptions[?contains(@, `macie`)]'`
- Verify Macie findings in Security Hub: `aws securityhub get-findings --filters '{"ProductName":[{"Value":"Macie","Comparison":"EQUALS"}]}' --max-results 5`

SHOULD:
- Verify both services in same region
- Check cross-region aggregation configuration
- Review Security Hub finding filters

MAY:
- Check CloudTrail for integration events
- Verify IAM permissions

## Phase 2 — Remediate

MUST:
- Enable Security Hub: `aws securityhub enable-security-hub`
- Enable Macie integration: `aws securityhub enable-import-findings-for-product --product-arn arn:aws:securityhub:<region>::product/aws/macie`

SHOULD:
- Configure cross-region aggregation
- Verify findings flow after enabling

MAY:
- Create Security Hub insights for Macie findings
- Set up automation rules

## Common Issues

- symptoms: "Macie findings not in Security Hub"
  diagnosis: "Integration not enabled."
  resolution: "Enable Macie product integration in Security Hub."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Security Hub | YELLOW | Enables new service with cost implications |
| Enable Macie integration | GREEN | Additive integration |
| Configure cross-region aggregation | GREEN | Centralization, no data modification |

## Escalation Conditions

- Macie findings not flowing to Security Hub for compliance — escalate to compliance team
- Cross-region aggregation needed — escalate to security operations
- Security Hub integration changes — escalate to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws securityhub describe-hub` | MEDIUM | Exposes Security Hub configuration |
| `aws securityhub get-findings` (Macie) | HIGH | Exposes sensitive data findings |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** disable Security Hub integration without security team approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled Security Hub | Disable: `aws securityhub disable-security-hub` (caution: deletes findings) |
| Enabled Macie integration | Disable: `aws securityhub disable-import-findings-for-product` |

## Output Format

```yaml
root_cause: "security_hub — <specific_cause>"
evidence:
  - type: integration_status
    content: "<Macie-Security Hub integration status>"
severity: MEDIUM
mitigation:
  immediate: "Enable integration"
  long_term: "Configure cross-region aggregation"
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
  - "NEVER suggest disabling Macie to reduce costs without review"
  - "NEVER suggest ignoring sensitive data findings"
  - "NEVER suggest removing S3 bucket monitoring"
