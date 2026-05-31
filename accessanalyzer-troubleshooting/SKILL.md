---
name: accessanalyzer-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot IAM Access Analyzer
  problems by analyzing analyzer creation, findings, external access,
  unused access, policy validation, policy generation, archive rules,
  and following structured runbooks.
  Activate when: analyzer creation failures, finding issues, external
  access detection problems, unused access analysis errors, policy
  validation failures, policy generation issues, archive rule problems,
  or the user says something is wrong with Access Analyzer.
compatibility: >
  Requires AWS CLI or SDK access with access-analyzer, iam, s3,
  organizations, cloudtrail, and cloudwatch permissions.
---

# IAM Access Analyzer Diagnostics

## When to use

Any IAM Access Analyzer investigation — analyzer creation, finding management, external access detection, unused access analysis, policy validation, policy generation, or archive rule configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws accessanalyzer list-analyzers --query 'analyzers[*].{Name:name,Arn:arn,Type:type,Status:status}'
version: "1.0.0"
last_updated: "2025-04-12"
aws accessanalyzer list-findings --analyzer-arn <arn> --filter '{"status":{"eq":["ACTIVE"]}}'
aws accessanalyzer list-archive-rules --analyzer-name <name>
```

### Step 2 — Domain deep dive

```
aws accessanalyzer get-analyzer --analyzer-name <name>
aws accessanalyzer get-finding --analyzer-arn <arn> --id <finding-id>
aws accessanalyzer validate-policy --policy-document file://policy.json --policy-type IDENTITY_POLICY
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=access-analyzer.amazonaws.com --max-results 20
aws accessanalyzer get-generated-policy --job-id <job-id>
aws accessanalyzer list-findings --analyzer-arn <arn> --filter '{"resourceType":{"eq":["AWS::S3::Bucket"]}}'
```

Read `references/guardrails.md` before concluding on any Access Analyzer issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `accessanalyzer list-analyzers` | List analyzers |
| `accessanalyzer get-analyzer` | Get analyzer details |
| `accessanalyzer list-findings` | List findings with filters |
| `accessanalyzer get-finding` | Get finding details |
| `accessanalyzer validate-policy` | Validate IAM policy |
| `accessanalyzer start-policy-generation` | Generate policy from activity |
| `accessanalyzer create-archive-rule` | Create archive rule |

## Gotchas: IAM Access Analyzer

- Access Analyzer has TWO types: ACCOUNT (analyzes within account) and ORGANIZATION (analyzes across organization). Organization type requires Organizations admin.
- External access findings identify resources shared with external principals (other accounts, public). This is the original Access Analyzer capability.
- Unused access findings identify unused IAM roles, access keys, passwords, and permissions. This is a newer capability with separate pricing.
- Policy validation checks IAM policies against best practices and grammar. It does NOT check if the policy is effective — only if it's well-formed.
- Policy generation creates policies based on CloudTrail activity. It requires CloudTrail logging and sufficient activity history (90 days recommended).
- Archive rules automatically archive findings matching criteria. Archived findings are NOT deleted — they're hidden from active view.
- Findings are generated asynchronously. New resources may take up to 24 hours to be analyzed.

## Anti-hallucination rules

1. Always cite specific analyzer ARNs, finding IDs, or API responses as evidence.
2. ACCOUNT vs ORGANIZATION analyzers have different scopes. Never conflate them.
3. External access and unused access are separate capabilities. Never mix them up.
4. Policy validation checks grammar, not effectiveness. Never claim it validates permissions.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Analyzer | A1-A2 | Creation, configuration |
| B — Findings | B1-B2 | Finding management, finding resolution |
| C — External Access | C1-C2 | External access detection, cross-account |
| D — Unused Access | D1-D2 | Unused access analysis, unused permissions |
| E — Policy Validation | E1-E2 | Validation errors, validation best practices |
| F — Policy Generation | F1-F2 | Generation setup, generation results |
| G — Archive Rules | G1-G2 | Archive rule creation, archive management |
| Z — Catch-All | Z1 | General troubleshooting |
