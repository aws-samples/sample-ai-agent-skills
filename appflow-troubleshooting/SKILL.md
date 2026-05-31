---
name: appflow-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon AppFlow
  problems by analyzing flow creation, connector profiles, flow
  execution, data mapping, scheduling, destinations, custom connectors,
  private connections, encryption, and following structured runbooks.
  Activate when: flow creation failures, connector profile errors,
  flow execution failures, data mapping issues, scheduling problems,
  destination errors, custom connector issues, private connection
  failures, encryption problems, or the user says something is wrong
  with AppFlow.
compatibility: >
  Requires AWS CLI or SDK access with appflow, s3, redshift, kms,
  iam, cloudwatch, and cloudtrail permissions.
---

# Amazon AppFlow Diagnostics

## When to use

Any Amazon AppFlow investigation — flow creation, connector profiles, flow execution, data mapping, scheduling, trigger configuration, S3/Redshift destinations, custom connectors, private connections, or data encryption.

## Investigation workflow

### Step 1 — Collect and triage

```
aws appflow list-flows --query 'flows[*].{Name:flowName,Status:flowStatus,SourceType:sourceConnectorType,DestType:destinationConnectorType}'
aws appflow list-connector-profiles --query 'connectorProfileProperties[*].{Name:connectorProfileName,Type:connectorType}'
aws appflow describe-flow --flow-name <flow-name>
```

### Step 2 — Domain deep dive

```
aws appflow describe-flow-execution-records --flow-name <flow-name> --max-results 10
aws appflow describe-connector-profiles --connector-profile-names <profile-name>
aws cloudwatch get-metric-statistics --namespace AWS/AppFlow --metric-name FlowExecutionsFailed --dimensions Name=FlowName,Value=<flow-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appflow.amazonaws.com --max-results 20
aws appflow describe-connectors --connector-type <type>
aws logs filter-log-events --log-group-name /aws/appflow/<flow-name> --start-time <epoch-ms> --limit 50
```

Read `references/guardrails.md` before concluding on any AppFlow issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `appflow list-flows` | List all flows in the account |
| `appflow describe-flow` | Get flow configuration details |
| `appflow describe-flow-execution-records` | Review execution history |
| `appflow list-connector-profiles` | List connector profiles |
| `appflow describe-connector-profiles` | Get connector profile details |
| `appflow describe-connectors` | Get connector capabilities |
| `appflow start-flow` | Manually trigger a flow run |

## Gotchas: Amazon AppFlow

- AppFlow has a limit of 100 flows per account per region by default. Request a quota increase for more.
- Salesforce connector requires a Connected App with specific OAuth scopes. Refresh tokens expire if the Connected App is modified.
- S3 destination writes use a specific prefix pattern: `<prefix>/<flow-name>/<execution-id>/`. Do not assume flat file output.
- Redshift destination requires an intermediate S3 bucket for staging. The Redshift cluster must have COPY permissions on that bucket.
- Scheduled flows use UTC time. Cron expressions follow AWS EventBridge syntax, not standard Unix cron.
- Custom connectors using Lambda must return responses in the exact AppFlow connector SDK format. Malformed responses cause silent failures.
- Private connections use AWS PrivateLink. The VPC endpoint must be in the same region as the AppFlow flow.

## Anti-hallucination rules

1. Always cite specific flow names, execution IDs, or API responses as evidence.
2. Salesforce OAuth tokens and Connected App configurations are service-specific. Never assume generic OAuth troubleshooting applies.
3. S3 and Redshift destinations have different error patterns. Never conflate destination types.
4. Scheduled flows and on-demand flows have different trigger mechanisms. Never mix troubleshooting steps.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Flow Management | A1-A2 | Flow creation failures, flow configuration |
| B — Connectors | B1-B2 | Connector profile errors, Salesforce connector |
| C — Execution | C1-C2 | Flow execution failures, data mapping errors |
| D — Scheduling | D1-D2 | Scheduling issues, trigger configuration |
| E — Destinations | E1-E2 | S3 destination errors, Redshift destination errors |
| F — Advanced | F1-F2 | Custom connector issues, private connection |
| G — Security | G1 | Data encryption |
| Z — Catch-All | Z1 | General troubleshooting |
