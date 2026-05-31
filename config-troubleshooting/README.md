# AWS Config Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Config problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Config when the console alone isn't enough — rule evaluation failure analysis, custom rule Lambda debugging, conformance pack deployment issues, aggregator configuration problems, remediation automation failures, recorder and delivery channel issues, advanced query errors, multi-account setup, and integration troubleshooting.

### Activate When

- Config rule evaluation failures
- Custom rule Lambda errors or timeouts
- Managed rule not evaluating as expected
- Conformance pack deployment failures
- Organization conformance pack issues
- Aggregator not collecting data
- Cross-account authorization failures
- Auto-remediation not executing
- SSM Automation errors from remediation
- Recorder stopped or misconfigured
- Delivery channel errors
- Resource types not being recorded
- Advanced query syntax errors
- Query performance or timeout issues
- Organization rule deployment failures
- Delegated administrator setup problems
- Security Hub integration gaps
- S3 delivery failures for configuration snapshots

---

## Skill Structure

```
config-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-rule-evaluation-failures.md
    ├── A2-custom-rule-lambda-errors.md
    ├── A3-managed-rule-issues.md
    ├── B1-conformance-pack-failures.md
    ├── B2-organization-conformance.md
    ├── C1-aggregator-issues.md
    ├── C2-cross-account-authorization.md
    ├── D1-auto-remediation-failures.md
    ├── D2-ssm-automation-errors.md
    ├── E1-recorder-configuration.md
    ├── E2-delivery-channel.md
    ├── E3-resource-types.md
    ├── F1-query-errors.md
    ├── F2-query-performance.md
    ├── G1-organization-rules.md
    ├── G2-delegated-administrator.md
    ├── H1-security-hub-integration.md
    ├── H2-s3-delivery.md
    ├── Z1-general-troubleshooting.md
    ├── config-guardrails.md
    └── config-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Rules** | A1–A3 | Rule evaluation failures, custom rule Lambda errors, managed rule issues |
| **B — Conformance** | B1–B2 | Conformance pack failures, organization conformance |
| **C — Aggregator** | C1–C2 | Aggregator issues, cross-account authorization |
| **D — Remediation** | D1–D2 | Auto-remediation failures, SSM automation errors |
| **E — Recorder** | E1–E3 | Recorder configuration, delivery channel, resource types |
| **F — Advanced Query** | F1–F2 | Query errors, query performance |
| **G — Multi-Account** | G1–G2 | Organization rules, delegated administrator |
| **H — Integration** | H1–H2 | Security Hub integration, S3 delivery |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/config-guardrails.md` covering recorder-must-be-on requirement, delivery channel dependency, resource type recording defaults, custom rule Lambda limits, conformance pack CloudFormation basis, remediation SSM dependency, aggregator authorization requirement, advanced query limitations, organization rule deployment scope, Config cost model, rule evaluation ordering, and configuration item retention.

---

## Investigation Workflow

1. **Triage** — Check recorder status, delivery channel, list rules and compliance, check CloudWatch metrics
2. **Deep Dive** — Examine rule evaluation status, conformance packs, aggregators, remediation configurations
3. **Detailed** — Run advanced queries, check Lambda functions, SSM executions, CloudTrail events, organization settings

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `config:*`, `lambda:GetFunction`, `lambda:InvokeFunction`, `ssm:DescribeAutomationExecutions`, `s3:GetBucketPolicy`, `sns:GetTopicAttributes`, `iam:GetRole`, `iam:GetRolePolicy`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `organizations:Describe*`, `securityhub:Describe*`
- CloudWatch metrics enabled for AWS Config
- CloudTrail logging enabled (recommended)

---

## Usage Examples

```
# Check recorder status
aws configservice describe-configuration-recorder-status

# List non-compliant rules
aws configservice describe-compliance-by-config-rule --compliance-types NON_COMPLIANT

# Check rule evaluation status
aws configservice describe-config-rule-evaluation-status --config-rule-names <rule-name>

# Describe conformance pack status
aws configservice describe-conformance-pack-status

# Check aggregator configuration
aws configservice describe-configuration-aggregators

# Run an advanced query
aws configservice select-resource-config --expression "SELECT resourceId WHERE resourceType = 'AWS::EC2::Instance'"
```

---

## License

MIT-0
