---
name: auditmanager-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Audit Manager
  problems by analyzing assessment creation, frameworks, controls,
  evidence collection, delegation, reports, custom frameworks,
  and following structured runbooks.
  Activate when: assessment creation failures, framework issues,
  control evaluation errors, evidence collection problems, delegation
  failures, report generation errors, custom framework issues, or
  the user says something is wrong with Audit Manager.
compatibility: >
  Requires AWS CLI or SDK access with auditmanager, config, cloudtrail,
  securityhub, iam, and s3 permissions.
---

# AWS Audit Manager Diagnostics

## When to use

Any AWS Audit Manager investigation — assessment creation, framework selection, control evaluation, evidence collection, delegation management, report generation, or custom framework configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws auditmanager get-settings --attribute ALL
aws auditmanager list-assessments --query 'assessmentMetadata[*].{Id:id,Name:name,Status:status}'
version: "1.0.0"
last_updated: "2025-04-12"
aws auditmanager list-assessment-frameworks --frameworkType Standard --query 'frameworkMetadataList[*].{Id:id,Name:name,Type:type}'
version: "1.0.0"
last_updated: "2025-04-12"
```

### Step 2 — Domain deep dive

```
aws auditmanager get-assessment --assessment-id <id>
aws auditmanager get-evidence-folder --assessment-id <id> --control-set-id <set-id> --evidence-folder-id <folder-id>
aws auditmanager list-assessment-control-insights-by-control-domain --control-domain-id <domain-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=auditmanager.amazonaws.com --max-results 20
aws auditmanager get-assessment-report-url --assessment-id <id> --assessment-report-id <report-id>
aws config describe-config-rules --query 'ConfigRules[*].{Name:ConfigRuleName,State:ConfigRuleState}'
version: "1.0.0"
last_updated: "2025-04-12"
```

Read `references/guardrails.md` before concluding on any Audit Manager issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `auditmanager list-assessments` | List assessments |
| `auditmanager get-assessment` | Get assessment details |
| `auditmanager list-assessment-frameworks` | List frameworks |
| `auditmanager get-evidence-folder` | Get evidence details |
| `auditmanager create-assessment` | Create new assessment |
| `auditmanager get-settings` | Check Audit Manager settings |

## Gotchas: AWS Audit Manager

- Audit Manager requires AWS Config to be enabled for automated evidence collection. Without Config, many controls cannot collect evidence automatically.
- Evidence collection is NOT real-time. It follows a schedule based on the control type. Manual evidence must be uploaded separately.
- Frameworks are templates — they define which controls to evaluate. Standard frameworks (CIS, PCI DSS, SOC 2) cannot be modified. Create custom frameworks for modifications.
- Delegation assigns control sets to team members for review. Delegates need IAM permissions to access Audit Manager and the delegated assessment.
- Assessment reports are generated as PDF/CSV and stored in the configured S3 bucket. The S3 bucket must exist and have proper permissions.
- Audit Manager is regional. Assessments in one region don't include resources from other regions.

## Anti-hallucination rules

1. Always cite specific assessment IDs, control statuses, or API responses as evidence.
2. AWS Config must be enabled for automated evidence. Never assume evidence collection works without Config.
3. Standard frameworks cannot be modified. Never suggest editing built-in frameworks.
4. Evidence collection is scheduled, not real-time. Never claim instant evidence.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Assessment | A1-A2 | Creation, configuration |
| B — Frameworks | B1-B2 | Standard frameworks, custom frameworks |
| C — Controls | C1-C2 | Control evaluation, control status |
| D — Evidence | D1-D3 | Automated collection, manual evidence, evidence quality |
| E — Delegation | E1-E2 | Delegation setup, delegate access |
| F — Reports | F1-F2 | Report generation, report access |
| Z — Catch-All | Z1 | General troubleshooting |
