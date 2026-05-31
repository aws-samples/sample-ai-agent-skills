---
title: "D1 — Automated Evidence Collection"
description: "Diagnose automated evidence collection failures"
status: active
severity: HIGH
triggers: ["no automated evidence", "evidence not collected", "Config evidence missing"]
owner: devops-agent
objective: "Resolve automated evidence collection issues"
context: "Automated evidence comes from Config rules, Security Hub findings, CloudTrail events, and API calls. Each source must be enabled and configured."
---

## Phase 1 — Triage

MUST:
- Verify AWS Config is enabled: `aws config describe-configuration-recorders`
- Check Config rules are active: `aws config describe-config-rules`
- Verify Security Hub is enabled if controls use it
- Check evidence collection timing (may take up to 24 hours)

SHOULD:
- Verify CloudTrail is logging in the region
- Check if evidence source matches control type

MAY:
- Review evidence collection logs

## Phase 2 — Remediate

MUST:
- Enable AWS Config with required rules
- Enable Security Hub with relevant standards
- Wait for evidence collection cycle

SHOULD:
- Verify all evidence sources before assessment creation

MAY:
- Manually trigger Config evaluation for faster evidence

## Common Issues

- symptoms: "Config-based controls show no evidence"
  diagnosis: "AWS Config recorder not enabled or Config rules not active."
  resolution: "Enable Config recorder and activate required Config rules."

## Output Format

```yaml
root_cause: "automated_evidence — <specific_cause>"
evidence:
  - type: evidence_source
    content: "<evidence source status>"
severity: HIGH
mitigation:
  immediate: "Enable required evidence sources"
  long_term: "Validate evidence sources as part of assessment setup"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable AWS Config with required rules | YELLOW | Enables service and rules — account-wide impact |
| Enable Security Hub with relevant standards | YELLOW | Enables service — account-wide security monitoring |
| Wait for evidence collection cycle | GREEN | No action — passive waiting |
| Manually trigger Config evaluation | GREEN | Triggers existing rule — non-destructive |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Enabling Config/Security Hub affects account-wide monitoring and costs
- Evidence collection changes affect compliance reporting timelines

## Data Sensitivity

- Automated evidence captures resource configurations that may contain sensitive settings
- Config rule evaluations may reveal security vulnerabilities and misconfigurations
- Security Hub findings may contain sensitive security posture information
- CloudTrail evidence may contain API call details with sensitive parameters

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER disable automated evidence sources during active assessment periods
- NEVER modify Config rules without understanding impact on evidence collection

## Phase 3 — Rollback

- If AWS Config was enabled incorrectly: disable the recorder (stops evidence collection)
- If Security Hub was enabled incorrectly: disable Security Hub (stops security findings evidence)
- If Config rules were activated incorrectly: deactivate specific rules
- Document all evidence source changes for compliance audit trail

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
