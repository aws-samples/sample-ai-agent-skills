# AWS Security Hub Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Security Hub problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Security Hub when the console alone isn't enough — security standard compliance debugging, finding management and aggregation, control evaluation troubleshooting, service integration configuration, multi-account and cross-region aggregation, automated response setup, and compliance scoring and reporting issues.

### Activate When

- CIS benchmark control failures
- PCI DSS compliance issues
- NIST 800-53 control evaluation problems
- Finding aggregation not working
- ASFF format errors on finding import
- Finding workflow status issues
- Control evaluation failures or NO_DATA
- Disabled controls not re-enabling
- Custom control configuration problems
- GuardDuty findings not appearing in Security Hub
- Config integration gaps
- Third-party integration failures
- Cross-account aggregation issues
- Delegated administrator setup problems
- Custom action not triggering EventBridge
- Automated remediation failures
- Cross-region aggregation not working
- Finding replication delays
- Compliance score incorrect or not updating
- Compliance reporting gaps

---

## Skill Structure

```
securityhub-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-cis-benchmark-failures.md
    ├── A2-pci-dss.md
    ├── A3-nist-800-53.md
    ├── B1-finding-aggregation.md
    ├── B2-asff-format.md
    ├── B3-finding-workflow.md
    ├── C1-control-failures.md
    ├── C2-disabled-controls.md
    ├── C3-custom-controls.md
    ├── D1-guardduty-integration.md
    ├── D2-config-integration.md
    ├── D3-third-party-integration.md
    ├── E1-cross-account-aggregation.md
    ├── E2-delegated-administrator.md
    ├── F1-custom-actions.md
    ├── F2-automated-remediation.md
    ├── G1-cross-region-aggregation.md
    ├── G2-finding-replication.md
    ├── H1-compliance-scoring.md
    ├── H2-reporting.md
    ├── Z1-general-troubleshooting.md
    ├── securityhub-guardrails.md
    └── securityhub-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Standards** | A1–A3 | CIS benchmark failures, PCI DSS, NIST 800-53 |
| **B — Findings** | B1–B3 | Finding aggregation, ASFF format, finding workflow |
| **C — Controls** | C1–C3 | Control failures, disabled controls, custom controls |
| **D — Integration** | D1–D3 | GuardDuty integration, Config integration, third-party |
| **E — Multi-Account** | E1–E2 | Cross-account aggregation, delegated administrator |
| **F — Automation** | F1–F2 | Custom actions, automated remediation |
| **G — Cross-Region** | G1–G2 | Cross-region aggregation, finding replication |
| **H — Compliance** | H1–H2 | Compliance scoring, reporting |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/securityhub-guardrails.md` covering Config dependency for controls, control-Config rule relationship, finding deduplication logic, ASFF format requirements, cross-region aggregation setup, compliance score calculation, disabled control finding retention, custom action EventBridge dependency, pricing model, standard enablement scope, finding retention limits, and consolidated control findings behavior.

---

## Investigation Workflow

1. **Triage** — Check Security Hub status, list enabled standards, check control compliance, query failed findings
2. **Deep Dive** — Examine control associations, integrations, member accounts, finding aggregators
3. **Detailed** — Review automation rules, custom actions, Config recorder status, CloudTrail events

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `securityhub:*`, `config:Describe*`, `config:Get*`, `guardduty:Get*`, `guardduty:List*`, `iam:GetRole`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `organizations:Describe*`, `events:Describe*`, `events:List*`
- AWS Config enabled with recorder running (required for most controls)
- CloudTrail logging enabled (recommended)

---

## Usage Examples

```
# Check Security Hub status
aws securityhub describe-hub

# List enabled standards
aws securityhub get-enabled-standards

# Get failed findings
aws securityhub get-findings --filters '{"ComplianceStatus":[{"Value":"FAILED","Comparison":"EQUALS"}]}' --max-results 10

# Check controls for a standard
aws securityhub describe-standards-controls --standards-subscription-arn <standard-arn>

# List integrations
aws securityhub list-enabled-products-for-import

# Check cross-region aggregation
aws securityhub list-finding-aggregators
```

---

## License

MIT-0
