---
name: resiliencehub-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Resilience Hub
  problems by analyzing application creation, resiliency policies,
  assessments, recommendations, drift detection, SOP generation,
  and following structured runbooks.
  Activate when: application import failures, policy configuration errors,
  assessment failures, recommendation issues, drift detection problems,
  SOP generation errors, or the user says something is wrong with
  Resilience Hub without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with resiliencehub, cloudformation,
  cloudwatch, ssm, iam, and resource-groups permissions.
---

# AWS Resilience Hub Diagnostics

## When to use

Any AWS Resilience Hub investigation — application creation/import failures, resiliency policy configuration, assessment errors, recommendation interpretation, drift detection issues, or SOP generation problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws resiliencehub list-apps --query 'appSummaries[*].{Arn:appArn,Name:name,Status:status,ComplianceStatus:complianceStatus}'
version: "1.0.0"
last_updated: "2025-04-12"
aws resiliencehub list-resiliency-policies --query 'resiliencyPolicies[*].{Arn:policyArn,Name:policyName,Tier:tier}'
version: "1.0.0"
last_updated: "2025-04-12"
aws resiliencehub list-app-assessments --query 'assessmentSummaries[*].{Arn:assessmentArn,Status:assessmentStatus,Compliance:complianceStatus}'
```

### Step 2 — Domain deep dive

```
aws resiliencehub describe-app --app-arn <app-arn>
aws resiliencehub describe-app-assessment --assessment-arn <assessment-arn>
aws resiliencehub list-app-component-compliances --assessment-arn <assessment-arn>
aws resiliencehub list-app-component-recommendations --assessment-arn <assessment-arn>
```

### Step 3 — Detailed investigation

```
aws resiliencehub describe-resiliency-policy --policy-arn <policy-arn>
aws resiliencehub describe-draft-app-version-resources-import-status --app-arn <app-arn>
aws resiliencehub list-sop-recommendations --assessment-arn <assessment-arn>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=resiliencehub.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Resilience Hub issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `resiliencehub list-apps` | List all applications |
| `resiliencehub describe-app` | Get application details |
| `resiliencehub list-app-assessments` | List assessments |
| `resiliencehub describe-app-assessment` | Get assessment details |
| `resiliencehub list-resiliency-policies` | List policies |
| `resiliencehub list-app-component-compliances` | Check component compliance |
| `resiliencehub list-app-component-recommendations` | Get recommendations |
| `resiliencehub list-sop-recommendations` | Get SOP recommendations |

## Gotchas: AWS Resilience Hub

- Resilience Hub discovers resources through CloudFormation stacks, Resource Groups, or Terraform state files. Resources not in these sources won't be included in assessments.
- Resiliency policies define RTO/RPO targets per disruption type (Software, Hardware, AZ, Region). Each application must have a policy attached before assessment.
- Assessments evaluate current architecture against the policy. They don't make changes — they only report compliance status and recommendations.
- Drift detection compares current resource configuration against the last successful assessment. It does NOT detect application-level changes, only infrastructure drift.
- SOP generation creates SSM documents for recovery procedures. These are templates that need customization for your specific environment.
- Assessment can take several minutes depending on application complexity and number of resources.

## Anti-hallucination rules

1. Always cite specific application ARNs, assessment results, or API responses as evidence.
2. Resilience Hub assesses but does NOT remediate. Never claim it auto-fixes issues.
3. Resources must be discoverable via CFN, Resource Groups, or Terraform. Never assume auto-discovery.
4. RTO/RPO targets are per disruption type. Never conflate AZ and Region targets.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Application | A1-A3 | Creation, import, resource discovery |
| B — Policy | B1-B2 | Policy configuration, policy assignment |
| C — Assessment | C1-C3 | Assessment failures, compliance, component analysis |
| D — Recommendations | D1-D2 | Architecture recommendations, cost optimization |
| E — Drift | E1-E2 | Drift detection, drift resolution |
| F — SOP | F1-F2 | SOP generation, SOP execution |
| Z — Catch-All | Z1 | General troubleshooting |
