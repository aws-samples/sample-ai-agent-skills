---
name: trustedadvisor-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Trusted Advisor
  problems by analyzing check failures, stale results, refresh issues,
  category-specific recommendations, priority settings, EventBridge
  integration, organizational view, support plan requirements, and API
  access following structured runbooks. Activate when: check failures,
  stale check results, refresh not working, cost optimization checks,
  security check findings, performance recommendations, priority
  recommendation issues, risk level configuration, EventBridge
  integration problems, organizational view setup, support plan
  access limitations, API access errors, or the user says something
  is wrong with Trusted Advisor without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Trusted Advisor (support API),
  IAM, EventBridge, Organizations, and optionally CloudWatch permissions.
  Business or Enterprise Support plan required for full API access.
---

# Trusted Advisor Diagnostics

## When to use

Any Trusted Advisor investigation where the console alone is insufficient — check failures, stale results, refresh issues, category recommendations, priority settings, EventBridge integration, organizational view, or API access problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws support describe-trusted-advisor-checks --language en
aws support describe-trusted-advisor-check-result --check-id <check-id> --language en
aws support describe-trusted-advisor-check-summaries --check-ids <check-id-1> <check-id-2>
aws support refresh-trusted-advisor-check --check-id <check-id>
```

### Step 2 — Domain deep dive

```
aws support describe-trusted-advisor-check-refresh-statuses --check-ids <check-id>
aws events list-rules --name-prefix TrustedAdvisor
aws organizations describe-organization
aws support describe-services
```

Read `references/guardrails.md` before concluding on any Trusted Advisor issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `support describe-trusted-advisor-checks` | List all available TA checks |
| `support describe-trusted-advisor-check-result` | Detailed check results with flagged resources |
| `support describe-trusted-advisor-check-summaries` | Summary status of multiple checks |
| `support refresh-trusted-advisor-check` | Trigger check refresh |
| `support describe-trusted-advisor-check-refresh-statuses` | Check refresh progress |
| `events list-rules` | EventBridge rules for TA events |
| `organizations describe-organization` | Organization setup for org view |

## Gotchas: Trusted Advisor

- Full Trusted Advisor API access requires Business or Enterprise Support plan. Basic and Developer plans have limited checks.
- Check results can be stale. Results are not real-time and depend on the last refresh time. Some checks refresh automatically, others require manual refresh.
- Trusted Advisor checks have a minimum refresh interval of 5 minutes. Requesting refresh too frequently returns an error.
- The Trusted Advisor API is only available in us-east-1 region. All API calls must target this endpoint.
- Organizational view requires AWS Organizations with all features enabled and Trusted Advisor enabled as a trusted service.
- Priority recommendations are only available with Trusted Advisor Priority (Enterprise Support with Technical Account Manager).
- Some checks are region-specific while others are global. A "green" check in one region doesn't mean all regions are clean.
- Trusted Advisor checks for service limits may not reflect recent quota increases. There can be a delay in updating.
- EventBridge integration publishes check status changes, not individual resource findings.
- Excluding resources from checks only hides them from the dashboard. It does not fix the underlying issue.

## Anti-hallucination rules

1. Always cite specific check IDs, check results, and flagged resources as evidence.
2. Trusted Advisor API requires Business/Enterprise Support. Never suggest API calls for Basic/Developer plans.
3. The API endpoint is us-east-1 only. Never suggest using other regions for TA API calls.
4. Check results may be stale. Always verify the last refresh timestamp before acting on results.
5. Priority recommendations require Enterprise Support with TAM. Never suggest Priority for lower support tiers.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 18 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Checks | A1–A3 | Check failures, stale results, refresh issues |
| B — Categories | B1–B3 | Cost optimization, security, performance |
| C — Priority | C1–C2 | Priority recommendations, risk levels |
| D — Integration | D1–D2 | EventBridge, organizational view |
| E — Access | E1–E2 | Support plan requirements, API access |
| Z — Catch-All | Z1 | General troubleshooting |
