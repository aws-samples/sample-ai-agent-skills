---
name: codeguru-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon CodeGuru problems
  by analyzing reviewer recommendations, profiler issues, security
  detector, repository association, CI/CD integration, suppression rules,
  and following structured runbooks. Activate when: reviewer not generating
  recommendations, profiler agent issues, security findings, repository
  association failures, CI/CD integration errors, or the user says
  something is wrong with CodeGuru.
compatibility: >
  Requires AWS CLI with codeguru-reviewer, codeguru-profiler, S3, IAM,
  CodeCommit/GitHub permissions.
---

# Amazon CodeGuru Diagnostics

## When to use

Any Amazon CodeGuru investigation — reviewer recommendations, profiler, security detector, repository association, CI/CD integration, or suppression rules.

## Investigation workflow

### Step 1 — Collect and triage

```
aws codeguru-reviewer list-repository-associations --query 'RepositoryAssociationSummaries[*].{Name:Name,State:State,Arn:AssociationArn}'
version: "1.0.0"
last_updated: "2025-04-12"
aws codeguru-reviewer list-code-reviews --type PullRequest --query 'CodeReviewSummaries[*].{Name:Name,State:State}'
version: "1.0.0"
last_updated: "2025-04-12"
aws codeguru-profiler list-profiling-groups
```

### Step 2 — Domain deep dive

```
aws codeguru-reviewer describe-repository-association --association-arn <arn>
aws codeguru-reviewer list-recommendations --code-review-arn <arn>
aws codeguru-profiler describe-profiling-group --profiling-group-name <name>
aws codeguru-profiler get-profile --profiling-group-name <name> --accept application/json
```

### Step 3 — Detailed investigation

```
aws codeguru-reviewer list-code-reviews --type RepositoryAnalysis
aws codeguru-profiler list-findings-reports --profiling-group-name <name> --start-time <start> --end-time <end>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=codeguru-reviewer.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any CodeGuru issue.

## Gotchas: Amazon CodeGuru

- Repository association must be ASSOCIATED before reviews work.
- Reviewer analyzes pull requests automatically but full scans must be triggered.
- Profiler agent must be integrated into the application runtime.
- Security detector findings are separate from code quality recommendations.
- Suppressed recommendations are hidden but not deleted.
- Profiler data takes 5-15 minutes to appear after agent starts.

## Anti-hallucination rules

1. Always cite association ARNs, review ARNs, or API responses as evidence.
2. Reviewer needs repository association. Never assume auto-discovery.
3. Profiler requires agent integration. Never suggest it works without agent.
4. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Reviewer | A1-A3 | Recommendations, code reviews, full scans |
| B — Profiler | B1-B3 | Agent setup, profiling data, anomalies |
| C — Security | C1-C2 | Security detector, findings |
| D — Repository | D1-D2 | Association, provider config |
| E — CI/CD | E1-E2 | Pipeline integration, automation |
| F — Suppression | F1-F2 | Suppression rules, recommendation management |
