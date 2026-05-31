---
title: "D2 — Security Hub Integration"
description: "Diagnose Detective integration issues with AWS Security Hub"
status: active
severity: MEDIUM
triggers:
  - "security hub detective"
  - "security hub integration"
  - "asff findings"
  - "security hub findings detective"
owner: devops-agent
objective: "Resolve Security Hub integration issues with Detective"
context: "Detective can ingest Security Hub findings (ASFF format) as a data source. This enables investigation of findings from multiple security services. The ASFF data source package must be explicitly enabled."
---

## Phase 1 — Triage

MUST:
- Check if ASFF data source is enabled: `aws detective list-datasource-packages --graph-arn <graph-arn>`
- Verify Security Hub is enabled: `aws securityhub describe-hub`
- Check if Security Hub findings exist: `aws securityhub get-findings --max-results 5`
- Verify the account is a member of both Security Hub and Detective

SHOULD:
- Check if Security Hub aggregation region matches Detective region
- Verify Security Hub integrations are enabled for the finding source
- Check data source status for the specific member account

MAY:
- Check CloudTrail for Security Hub and Detective integration events
- Verify no service issues affecting data flow

## Phase 2 — Remediate

MUST:
- Enable ASFF data source: `aws detective update-datasource-packages --graph-arn <graph-arn> --datasource-packages '{"ASFF_SECURITYHUB_FINDING":{"Status":"ENABLED"}}'`
- Ensure Security Hub is enabled in the same region
- Wait 24 hours for findings to appear in Detective

SHOULD:
- Enable Security Hub in all regions where Detective is active
- Configure Security Hub finding aggregation for centralized view

## Output Format

```yaml
root_cause: "security_hub_integration — <specific_cause>"
evidence:
  - type: asff_datasource
    content: "<ASFF data source status>"
  - type: securityhub_status
    content: "<Security Hub status>"
severity: MEDIUM
mitigation:
  immediate: "Enable ASFF data source package"
  long_term: "Ensure Security Hub and Detective are aligned across regions"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Security Hub findings contain security assessment data |
| Blast Radius | LOW — Integration changes affect data ingestion only |
| Rollback Complexity | LOW — Data source can be re-enabled |
| Compliance Scope | HIGH — Security Hub integration provides comprehensive finding coverage |

## Escalation Conditions

- Escalate to security team if Security Hub findings are not reaching Detective
- Escalate to operations if Security Hub is not enabled in Detective's region

## Data Sensitivity

- **ASFF findings** contain security assessment results — security-sensitive
- **Integration configuration** reveals security monitoring architecture

## PROHIBITED Actions

- Do NOT disable ASFF data source without security team approval
- Do NOT assume Security Hub findings appear immediately — 24-hour delay

## Phase 3 — Rollback

- Re-enable ASFF data source: `aws detective update-datasource-packages --graph-arn <arn> --datasource-packages '{"ASFF_SECURITYHUB_FINDING":{"Status":"ENABLED"}}'`
- Verify Security Hub is enabled in the same region
- Wait 24 hours for findings to appear

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
