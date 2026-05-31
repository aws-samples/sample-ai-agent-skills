# Amazon GuardDuty Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon GuardDuty problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon GuardDuty when the console alone isn't enough — finding analysis and triage, false positive management, detector configuration debugging, protection feature troubleshooting, malware scanning issues, multi-account management, threat intelligence configuration, Security Hub and EventBridge integration, and runtime monitoring failures.

### Activate When

- Finding analysis and investigation
- False positive identification and management
- Suppression rule configuration issues
- Detector not detecting or misconfigured
- Data source enablement problems
- Feature enablement failures
- S3 protection not detecting threats
- EKS protection or audit log issues
- Lambda protection configuration problems
- RDS protection not working
- Malware scan failures or incomplete scans
- Delegated administrator setup issues
- Member account management problems
- Custom threat list upload or activation failures
- IP reputation list issues
- Security Hub finding export failures
- EventBridge rule automation not triggering
- Runtime monitoring coverage gaps
- GuardDuty agent deployment failures

---

## Skill Structure

```
guardduty-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-finding-analysis.md
    ├── A2-false-positives.md
    ├── A3-suppression-rules.md
    ├── B1-detector-configuration.md
    ├── B2-data-sources.md
    ├── B3-feature-enablement.md
    ├── C1-s3-protection.md
    ├── C2-eks-protection.md
    ├── C3-lambda-protection.md
    ├── C4-rds-protection.md
    ├── D1-malware-scanning.md
    ├── D2-scan-failures.md
    ├── E1-delegated-administrator.md
    ├── E2-member-accounts.md
    ├── F1-custom-threat-lists.md
    ├── F2-ip-reputation.md
    ├── G1-security-hub-export.md
    ├── G2-eventbridge-automation.md
    ├── H1-runtime-monitoring.md
    ├── H2-agent-deployment.md
    ├── Z1-general-troubleshooting.md
    ├── guardduty-guardrails.md
    └── guardduty-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Findings** | A1–A3 | Finding analysis, false positives, suppression rules |
| **B — Detector** | B1–B3 | Detector configuration, data sources, feature enablement |
| **C — Protection** | C1–C4 | S3 protection, EKS protection, Lambda protection, RDS protection |
| **D — Malware** | D1–D2 | Malware scanning, scan failures |
| **E — Multi-Account** | E1–E2 | Delegated administrator, member accounts |
| **F — Threat Intel** | F1–F2 | Custom threat lists, IP reputation |
| **G — Integration** | G1–G2 | Security Hub export, EventBridge automation |
| **H — Runtime** | H1–H2 | Runtime monitoring, agent deployment |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardduty-guardrails.md` covering independent data source analysis, regional finding scope, suppression vs deletion distinction, S3 protection data event dependency, EKS audit log requirement, runtime agent requirement, malware scan EBS snapshot costs, delegated admin authority scope, finding auto-archive behavior, trusted IP list limitations, finding severity classification, and detector suspension vs disabling.

---

## Investigation Workflow

1. **Triage** — List detectors, check detector status, list high-severity findings, check finding statistics
2. **Deep Dive** — Examine detector features, member accounts, publishing destinations, threat intel sets
3. **Detailed** — Review malware scans, runtime coverage, organization configuration, CloudTrail events

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `guardduty:*`, `ec2:Describe*`, `eks:Describe*`, `lambda:GetFunction`, `rds:Describe*`, `s3:GetBucketPolicy`, `iam:GetRole`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `organizations:Describe*`, `securityhub:*`, `events:Describe*`
- GuardDuty detector enabled in the target region
- CloudTrail logging enabled (recommended for correlation)

---

## Usage Examples

```
# List detectors
aws guardduty list-detectors

# Get detector configuration
aws guardduty get-detector --detector-id <detector-id>

# List high-severity findings
aws guardduty list-findings --detector-id <detector-id> --finding-criteria '{"Criterion":{"severity":{"Gte":7}}}'

# Get finding details
aws guardduty get-findings --detector-id <detector-id> --finding-ids <finding-id>

# Check malware scan status
aws guardduty describe-malware-scans --detector-id <detector-id>

# List member accounts
aws guardduty list-members --detector-id <detector-id>
```

---

## License

MIT-0
