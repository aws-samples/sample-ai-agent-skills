---
title: "D1 — Security Hub Integration"
description: "Diagnose Inspector integration issues with Security Hub"
status: active
severity: MEDIUM
triggers:
  - "findings not in Security Hub"
  - "Security Hub integration"
  - "ASFF findings missing"
  - "Inspector Security Hub"
owner: devops-agent
objective: "Ensure Inspector findings flow correctly to Security Hub"
context: "Inspector automatically sends findings to Security Hub in ASFF format when both services are enabled. Issues arise when Security Hub is not enabled, the Inspector integration is disabled, or cross-region aggregation is not configured."
---

## Phase 1 — Triage

MUST:
- Check Security Hub is enabled: `aws securityhub describe-hub`
- Check Inspector integration in Security Hub: `aws securityhub list-enabled-products-for-import --query 'ProductSubscriptions[?contains(@, `inspector`)]'`
- Verify Inspector is sending findings: `aws securityhub get-findings --filters '{"ProductName":[{"Value":"Inspector","Comparison":"EQUALS"}]}' --max-results 5`
- Check Inspector account status: `aws inspector2 batch-get-account-status`

SHOULD:
- Verify both services are in the same region
- Check if cross-region aggregation is configured in Security Hub
- Review Security Hub finding import settings

MAY:
- Check CloudTrail for integration-related events
- Verify IAM permissions for the integration

## Phase 2 — Remediate

MUST:
- Enable Security Hub if not active: `aws securityhub enable-security-hub`
- Enable Inspector integration: `aws securityhub enable-import-findings-for-product --product-arn arn:aws:securityhub:<region>::product/aws/inspector`
- Verify findings flow after enabling

SHOULD:
- Configure cross-region aggregation for centralized visibility
- Set up Security Hub insights for Inspector findings

MAY:
- Create custom actions for Inspector finding workflows
- Configure Security Hub automation rules for Inspector findings

## Common Issues

- symptoms: "Inspector findings not appearing in Security Hub"
  diagnosis: "Inspector product integration not enabled in Security Hub."
  resolution: "Enable the Inspector product integration in Security Hub."

- symptoms: "Findings appear in one region but not others"
  diagnosis: "Cross-region aggregation not configured."
  resolution: "Enable cross-region aggregation in Security Hub."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Security Hub | YELLOW | Enables a new service with potential cost implications |
| Enable Inspector integration | GREEN | Additive integration |
| Configure cross-region aggregation | GREEN | Centralization, no data modification |

## Escalation Conditions

- Inspector findings not flowing to Security Hub for compliance monitoring — escalate to compliance team
- Cross-region aggregation needed for centralized security view — escalate to security operations
- Security Hub integration changes — escalate to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws securityhub describe-hub` | MEDIUM | Exposes Security Hub configuration |
| `aws securityhub get-findings` (Inspector) | MEDIUM | Exposes vulnerability findings |
| `aws inspector2 batch-get-account-status` | MEDIUM | Exposes scanning status |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** disable Security Hub integration without security team approval
- **NEVER** delete Security Hub findings without review

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled Security Hub | Disable: `aws securityhub disable-security-hub` (caution: deletes all findings) |
| Enabled Inspector integration | Disable: `aws securityhub disable-import-findings-for-product` |
| Configured cross-region aggregation | Remove aggregation configuration |

## Output Format

```yaml
root_cause: "security_hub_integration — <specific_cause>"
evidence:
  - type: hub_status
    content: "<Security Hub status>"
  - type: integration_status
    content: "<Inspector integration status>"
severity: MEDIUM
mitigation:
  immediate: "Enable integration and verify finding flow"
  long_term: "Configure cross-region aggregation and automation"
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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
