---
title: "B1 — Finding Analysis"
description: "Analyze and prioritize Inspector findings"
status: active
severity: MEDIUM
triggers:
  - "too many findings"
  - "prioritize findings"
  - "critical findings"
  - "finding details"
owner: devops-agent
objective: "Analyze Inspector findings to prioritize remediation efforts"
context: "Inspector findings include CVE details, severity scores, affected resources, and remediation guidance. Prioritization should consider severity, exploit availability, network exposure, and business criticality of affected resources."
---

## Phase 1 — Triage

MUST:
- Get finding summary by severity: `aws inspector2 list-finding-aggregations --aggregation-type SEVERITY`
- List critical findings: `aws inspector2 list-findings --filter-criteria '{"severity":[{"comparison":"EQUALS","value":"CRITICAL"}]}' --max-results 20`
- Get finding details: `aws inspector2 get-finding --finding-arn <finding-arn>`
- Check findings by resource: `aws inspector2 list-finding-aggregations --aggregation-type RESOURCE_TYPE`

SHOULD:
- Filter findings with known exploits: `aws inspector2 list-findings --filter-criteria '{"exploitAvailable":[{"comparison":"EQUALS","value":"YES"}]}' --max-results 20`
- Check findings by package: `aws inspector2 list-finding-aggregations --aggregation-type PACKAGE`
- Review network reachability findings separately

MAY:
- Export findings report: `aws inspector2 create-findings-report --report-format CSV --s3-destination bucketName=<bucket>,keyPrefix=<prefix>`
- Aggregate findings by account for multi-account environments

## Phase 2 — Remediate

MUST:
- Prioritize: Critical + exploit available + network exposed first
- Follow remediation guidance in finding details
- Track remediation progress by re-checking finding status

SHOULD:
- Group findings by package for batch patching
- Create remediation tickets for high/critical findings
- Set up EventBridge rules for new critical findings

MAY:
- Use finding aggregations to identify systemic issues
- Create dashboards for finding trends over time

## Common Issues

- symptoms: "Overwhelmed by number of findings"
  diagnosis: "Need to prioritize by severity and exploitability."
  resolution: "Filter by CRITICAL severity + exploit available. Address those first."

- symptoms: "Same CVE across many resources"
  diagnosis: "Common package vulnerability across fleet."
  resolution: "Aggregate by package. Patch the package across all affected resources."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Prioritize and remediate findings | GREEN | Analysis and patching, no Inspector config changes |
| Create EventBridge rules for notifications | GREEN | Monitoring-only addition |
| Export findings report | GREEN | Read-only export |

## Escalation Conditions

- Critical findings with known exploits on internet-facing resources — escalate immediately to security team
- Large number of critical findings across fleet — escalate to infrastructure team for batch patching
- Findings in compliance-critical systems — escalate to compliance team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 list-findings` | MEDIUM | Exposes vulnerability details and affected resources |
| `aws inspector2 get-finding` | MEDIUM | Exposes detailed CVE and remediation information |
| `aws inspector2 list-finding-aggregations` | MEDIUM | Exposes finding statistics |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** ignore critical findings with known exploits
- **NEVER** share finding details externally without redacting resource identifiers

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Applied patches to remediate findings | Revert patches if they cause application issues |
| Created EventBridge rules | Delete the rules: `aws events delete-rule` |
| No Inspector config changes in finding analysis | No rollback needed |

## Output Format

```yaml
root_cause: "finding_analysis — <specific_focus>"
evidence:
  - type: severity_summary
    content: "<finding counts by severity>"
  - type: critical_findings
    content: "<critical finding details>"
severity: MEDIUM
mitigation:
  immediate: "Address critical findings with known exploits"
  long_term: "Establish regular patching cadence and finding review process"
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
