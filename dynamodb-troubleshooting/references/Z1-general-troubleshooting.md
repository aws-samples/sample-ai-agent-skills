---
title: "Z1 — General DynamoDB Troubleshooting"
description: "Catch-all systematic investigation for DynamoDB issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "DynamoDB issue"
  - "DynamoDB error"
  - "DynamoDB problem"
  - "DynamoDB not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate DynamoDB issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for DynamoDB issues that don't clearly match a specific runbook. It covers table health checks, capacity analysis, error pattern identification, and systematic elimination of common causes. Use this as a starting point when the issue category is unclear, then pivot to a specific runbook once the problem area is identified."
---

## Phase 1 — Triage

MUST:
- Check table status and configuration: `aws dynamodb describe-table --table-name <table>`
- Check all error metrics:
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name SystemErrors --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name UserErrors --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ThrottledRequests --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check latency metrics:
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name SuccessfulRequestLatency --dimensions Name=TableName,Value=<table> Name=Operation,Value=GetItem --start-time <start> --end-time <end> --period 300 --statistics Average`
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name SuccessfulRequestLatency --dimensions Name=TableName,Value=<table> Name=Operation,Value=Query --start-time <start> --end-time <end> --period 300 --statistics Average`
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name SuccessfulRequestLatency --dimensions Name=TableName,Value=<table> Name=Operation,Value=PutItem --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check capacity utilization:
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check CloudTrail for recent table modifications: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::DynamoDB::Table --start-time <start> --end-time <end>`
- Check all GSI status and metrics: `aws dynamodb describe-table --table-name <table> --query 'Table.GlobalSecondaryIndexes[].{IndexName:IndexName,IndexStatus:IndexStatus}'`
- Check DynamoDB Streams status if applicable: `aws dynamodb describe-table --table-name <table> --query 'Table.StreamSpecification'`
- Check auto-scaling configuration: `aws application-autoscaling describe-scalable-targets --service-namespace dynamodb --resource-ids table/<table>`
- Check TTL configuration: `aws dynamodb describe-time-to-live --table-name <table>`

MAY:
- Enable Contributor Insights for deeper analysis: `aws dynamodb update-contributor-insights --table-name <table> --contributor-insights-action ENABLE`
- Check AWS Health Dashboard for service-level issues
- Review application logs for DynamoDB error patterns
- Check VPC endpoint configuration if using VPC endpoints: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.dynamodb`

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Throttling → A1 (read), A2 (write), A3 (GSI), A4 (on-demand)
  - Hot partitions → B1 (hot partitions), B2 (key design)
  - Query/Scan performance → C1 (query), C2 (scan), C3 (pagination)
  - Capacity issues → D1 (auto-scaling), D2 (billing mode), D3 (burst)
  - Index issues → E1 (GSI creation), E2 (LSI limits), E3 (projections)
  - Stream issues → F1 (processing), F2 (Lambda), F3 (consumer lag)
  - TTL issues → G1 (not expiring), G2 (filter expressions)
  - Global table issues → H1 (replication), H2 (conflicts), H3 (failover)
  - Transaction issues → I1 (failures), I2 (idempotency)
  - DAX issues → J1 (cache), J2 (consistency)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues (e.g., throttling + hot partition)
- Review recent changes that may have caused the issue
- Verify the fix resolves the issue without introducing new problems

MAY:
- Set up comprehensive monitoring to prevent recurrence
- Create a post-incident report for significant issues
- Review DynamoDB best practices documentation

## Common Issues

- symptoms: "Intermittent 500 errors from DynamoDB"
  diagnosis: "SystemErrors metric shows internal DynamoDB errors. These are transient and should be retried."
  resolution: "Implement retry with exponential backoff. If persistent, check AWS Health Dashboard and contact AWS Support."

- symptoms: "Application suddenly slower but no errors"
  diagnosis: "Latency increase without errors. Could be item size growth, missing indexes, or network issues."
  resolution: "Check SuccessfulRequestLatency per operation. Compare with baseline. Check if queries are scanning more data."

- symptoms: "ValidationException on API calls"
  diagnosis: "Request format is invalid. Common causes: item size > 400KB, too many attributes, invalid key schema."
  resolution: "Check the specific ValidationException message. Verify item sizes, attribute names, and request format."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Pivot to specific runbook | GREEN | Diagnostic routing; no infrastructure change |
| Enable Contributor Insights | GREEN | Read-only analysis; no data impact |
| Check CloudTrail for changes | GREEN | Read-only audit; no operational impact |
| Document findings | GREEN | No operational impact |

## Escalation Conditions

- Issue cannot be classified into any specific failure domain
- Multiple concurrent issues detected across different domains
- SystemErrors metric elevated (potential DynamoDB service issue)
- Issue persists after following specific runbook remediation
- Customer-facing impact with no clear root cause identified

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` | MEDIUM | Exposes full table configuration and schema |
| `get-metric-statistics` (all metrics) | LOW | Operational metrics only |
| `lookup-events` (CloudTrail) | MEDIUM | Exposes modification history and IAM principals |
| `describe-vpc-endpoints` | LOW | Network configuration only |

## Prohibited Actions

- NEVER suggest deleting a table as a troubleshooting step
- NEVER suggest disabling encryption during investigation
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER make infrastructure changes during general troubleshooting without identifying root cause first

## Phase 3 — Rollback

General rollback guidance for unclassified issues:
1. If any configuration changes were made during investigation, document and revert them
2. If capacity was temporarily increased, schedule a review to right-size after root cause is found
3. If Contributor Insights was enabled, it can remain active (low cost, high diagnostic value)
4. If the issue was escalated, provide all collected evidence to the escalation team
5. Verify all metrics return to baseline after any investigative changes are reverted

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: table_status
    content: "<table configuration and health>"
  - type: error_metrics
    content: "<SystemErrors, UserErrors, ThrottledRequests>"
  - type: latency_metrics
    content: "<SuccessfulRequestLatency per operation>"
  - type: capacity_metrics
    content: "<consumed vs provisioned capacity>"
severity: MEDIUM
general_analysis:
  table_status: "ACTIVE | CREATING | UPDATING | DELETING"
  billing_mode: "PROVISIONED | PAY_PER_REQUEST"
  error_pattern: "<identified pattern>"
  latency_baseline: "<normal vs current>"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and alerting"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting a table to fix issues"
  - "NEVER run full table scans on production tables"
