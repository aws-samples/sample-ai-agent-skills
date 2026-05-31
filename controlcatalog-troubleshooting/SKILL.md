---
name: controlcatalog-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Control Catalog
  problems by analyzing control listing, control details, objective
  mapping, common controls, regional availability, API usage,
  and following structured runbooks.
  Activate when: control listing failures, control detail retrieval
  errors, objective mapping issues, common control problems, regional
  availability questions, API usage errors, or the user says something
  is wrong with Control Catalog.
compatibility: >
  Requires AWS CLI or SDK access with controlcatalog, controltower,
  iam, and cloudtrail permissions.
---

# AWS Control Catalog Diagnostics

## When to use

Any AWS Control Catalog investigation — control listing, control detail retrieval, objective mapping, common control identification, regional availability, or API usage issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws controlcatalog list-controls --query 'controls[*].{Arn:arn,Name:name}'
version: "1.0.0"
last_updated: "2025-04-12"
aws controlcatalog list-objectives --query 'objectives[*].{Arn:arn,Name:name}'
version: "1.0.0"
last_updated: "2025-04-12"
aws controlcatalog list-domains --query 'domains[*].{Arn:arn,Name:name}'
version: "1.0.0"
last_updated: "2025-04-12"
```

### Step 2 — Domain deep dive

```
aws controlcatalog get-control --control-arn <arn>
aws controlcatalog list-common-controls --common-control-filter '{"objectives":[{"arn":"<objective-arn>"}]}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=controlcatalog.amazonaws.com --max-results 20
aws controltower list-enabled-controls --target-identifier <ou-arn>
```

Read `references/guardrails.md` before concluding on any Control Catalog issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `controlcatalog list-controls` | List available controls |
| `controlcatalog get-control` | Get control details |
| `controlcatalog list-objectives` | List control objectives |
| `controlcatalog list-domains` | List control domains |
| `controlcatalog list-common-controls` | List common controls |

## Gotchas: AWS Control Catalog

- Control Catalog is a READ-ONLY service. It provides information about available controls but does NOT enable or enforce them. Use Control Tower or other services to enable controls.
- Controls in the catalog are organized by domains and objectives. Domains are broad categories, objectives are specific goals within domains.
- Common controls are AWS-recommended controls that map to multiple compliance frameworks. They are not custom controls.
- Control Catalog is a relatively new service. Not all controls from all services may be listed. The catalog is continuously expanding.
- Regional availability varies. Some controls are only available in specific regions. Check control details for regional support.
- Control Catalog API is separate from Control Tower API. Listing controls in the catalog doesn't mean they're enabled in your environment.

## Anti-hallucination rules

1. Always cite specific control ARNs, names, or API responses as evidence.
2. Control Catalog is READ-ONLY. Never suggest it enables or enforces controls.
3. Common controls are AWS-defined. Never suggest creating custom common controls.
4. Not all controls are in the catalog. Never assume comprehensive coverage.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Control Listing | A1-A2 | List controls, filter controls |
| B — Control Details | B1-B2 | Get details, control properties |
| C — Objectives | C1-C2 | Objective mapping, objective listing |
| D — Common Controls | D1-D2 | Common control listing, framework mapping |
| E — Regional | E1-E2 | Regional availability, region-specific controls |
| F — API Usage | F1 | API errors and usage |
| Z — Catch-All | Z1 | General troubleshooting |
