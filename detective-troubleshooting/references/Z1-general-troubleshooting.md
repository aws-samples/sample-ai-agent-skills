---
title: "Z1 — General Detective Troubleshooting (Catch-All)"
description: "Fallback SOP for Detective issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Detective issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List behavior graphs: `aws detective list-graphs`
- If graph exists, list members: `aws detective list-members --graph-arn <graph-arn>`
- Check data sources: `aws detective list-datasource-packages --graph-arn <graph-arn>`
- Verify GuardDuty is enabled: `aws guardduty list-detectors`
- Check recent events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=detective.amazonaws.com --max-results 10`

SHOULD:
- Check delegated admin configuration
- Verify IAM permissions
- Check Organizations integration status

MAY:
- Check AWS Health Dashboard for Detective service events
- Review Security Hub integration status

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Graph creation/config issues → Graph (A1)
- Data source problems → Graph (A2)
- Finding investigation failures → Investigation (B1)
- Entity profile issues → Investigation (B2)
- Anomaly detection problems → Investigation (B3)
- Admin account issues → Multi-Account (C1)
- Member invitation problems → Multi-Account (C2)
- GuardDuty integration → Integration (D1)
- Security Hub integration → Integration (D2)
- Data volume concerns → Data (E1)
- Retention questions → Data (E2)
- IAM permission errors → Access (F1)
- Cross-account access → Access (F2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all graph and member configurations
- Review CloudTrail for recent Detective changes
- Verify all prerequisite services are enabled
- Test basic operations: list-graphs, list-members

SHOULD:
- Compare with known-good configuration
- Check AWS Health Dashboard

MAY:
- Check for regional service outages
- Review Detective console for visual indicators

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<graph|investigation|multi_account|integration|data|access|unknown>"
investigation_path: "graph status → members → data sources → <domain_classification>"
evidence:
  - type: graph_config
    content: "<graph configuration summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — General troubleshooting may expose security investigation infrastructure |
| Blast Radius | LOW — Diagnostic commands are read-only |
| Rollback Complexity | LOW — No changes during triage |
| Compliance Scope | HIGH — Detective supports security incident response requirements |

## Escalation Conditions

- Escalate to the specific runbook owner once the failure domain is identified
- Escalate to security team if Detective is non-functional during an active security incident
- Escalate to AWS Support if systematic issues affect Detective operations

## Data Sensitivity

- **Graph and member configurations** reveal security monitoring architecture
- **CloudTrail events** contain API call details
- **Data source status** reveals security monitoring coverage

## PROHIBITED Actions

- Do NOT modify graph or member configurations during initial triage
- Do NOT share security monitoring architecture details outside the security team
- Do NOT delete or disable any Detective resources during troubleshooting

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
  - "NEVER suggest disabling Detective to reduce costs"
  - "NEVER suggest ignoring investigation findings"
  - "NEVER expose investigation details publicly"
