---
title: "Z1 — General Security Hub Troubleshooting (Catch-All)"
description: "Fallback SOP for Security Hub issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Security Hub issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check Security Hub status: `aws securityhub describe-hub`
- List enabled standards: `aws securityhub get-enabled-standards`
- Get finding statistics: `aws securityhub get-findings --filters '{"ComplianceStatus":[{"Value":"FAILED","Comparison":"EQUALS"}]}' --max-results 10`
- List enabled product integrations: `aws securityhub list-enabled-products-for-import`
- Check Config recorder status: `aws configservice describe-configuration-recorder-status`
- List member accounts: `aws securityhub list-members`

SHOULD:
- Check finding aggregators: `aws securityhub list-finding-aggregators`
- List automation rules: `aws securityhub list-automation-rules`
- Check custom actions: `aws securityhub describe-action-targets`
- Check CloudTrail for Security Hub events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=securityhub.amazonaws.com --max-results 20`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Standard compliance issues → Standards (A1-A3)
- Finding management problems → Findings (B1-B3)
- Control evaluation failures → Controls (C1-C3)
- Service integration issues → Integration (D1-D3)
- Multi-account problems → Multi-Account (E1-E2)
- Automation failures → Automation (F1-F2)
- Cross-region issues → Cross-Region (G1-G2)
- Compliance reporting → Compliance (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Verify Config is enabled with all resource types
- Check all standard control statuses for patterns
- Verify IAM permissions for Security Hub operations
- Review recent configuration changes in CloudTrail

SHOULD:
- Check AWS Health Dashboard for Security Hub service events
- Verify no SCPs are blocking Security Hub operations
- Check for recent changes to Config, GuardDuty, or other integrated services
- Review finding retention and archival status

MAY:
- Check all regions for Security Hub status
- Verify organization configuration
- Review EventBridge rules for Security Hub events

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Route to specific SOP | GREEN | Classification only; no state change |
| Investigate and collect evidence | GREEN | Read-only analysis |

## Escalation Conditions

- Standard enablement/disablement
- Cross-account configuration changes
- Unclassified issues after investigation

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-hub` | LOW | Hub status |
| `get-enabled-standards` | LOW | Standard subscription status |
| `get-findings` | HIGH | Security findings |
| `list-members` | MEDIUM | Member account IDs |
| CloudTrail events | MEDIUM | API call history |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Route to specific SOP | Follow that SOP's rollback procedures |
| No direct remediation in catch-all | N/A — remediation happens in specific SOPs |

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<standards|findings|controls|integration|multi_account|automation|cross_region|compliance|unknown>"
investigation_path: "hub status → standards → controls → integrations → <domain_classification>"
evidence:
  - type: hub_status
    content: "<Security Hub status>"
  - type: standards
    content: "<enabled standards and scores>"
  - type: controls
    content: "<control status summary>"
  - type: integrations
    content: "<product integration status>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
