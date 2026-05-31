---
title: "C2 — Scan Optimization"
description: "Optimize DynamoDB scan operations including parallel scan, segments, and rate limiting"
status: active
severity: MEDIUM
triggers:
  - "slow scan"
  - "scan performance"
  - "full table scan"
  - "scan consuming too much capacity"
  - "parallel scan"
owner: devops-agent
objective: "Optimize scan operations to minimize impact on table performance"
context: "DynamoDB Scan reads every item in the table and returns up to 1MB per call. Scans consume RCU proportional to the data read, not the data returned (FilterExpression does not reduce RCU). Parallel scan uses multiple workers with TotalSegments and Segment parameters. Rate limiting prevents scans from consuming all provisioned capacity and starving other operations."
---

## Phase 1 — Triage

MUST:
- Check scan latency and frequency: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name SuccessfulRequestLatency --dimensions Name=TableName,Value=<table> Name=Operation,Value=Scan --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check consumed RCU during scan operations: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check table size to estimate scan cost: `aws dynamodb describe-table --table-name <table> --query 'Table.{ItemCount:ItemCount,TableSizeBytes:TableSizeBytes}'`
- Check if scans are causing throttling on other operations

SHOULD:
- Check if a Query could replace the Scan by using a GSI
- Verify if parallel scan is being used for large tables
- Check if rate limiting is implemented to prevent scan from consuming all capacity

MAY:
- Test parallel scan with different segment counts: `aws dynamodb scan --table-name <table> --total-segments 4 --segment 0 --return-consumed-capacity TOTAL`
- Check if the scan can be replaced with an export to S3: `aws dynamodb export-table-to-point-in-time --table-arn <table-arn> --s3-bucket <bucket> --export-format DYNAMODB_JSON`
- Review if DynamoDB Streams could provide the needed data instead of scanning

## Phase 2 — Remediate

MUST:
- Replace Scan with Query wherever possible by creating appropriate GSIs
- Implement rate limiting on scans to protect other operations
- Use ProjectionExpression to reduce data read per item

SHOULD:
- Use parallel scan for large tables: split into TotalSegments and process concurrently
- Add a Limit parameter to control items per page and add delays between pages
- Use FilterExpression to reduce data transferred (but note it does not reduce RCU)

MAY:
- Use DynamoDB Export to S3 for analytics workloads instead of scanning
- Schedule scans during off-peak hours
- Consider using a separate read replica or GSI dedicated to scan workloads

## Common Issues

- symptoms: "Scan takes hours to complete on a large table"
  diagnosis: "Sequential scan on a large table processes one 1MB page at a time."
  resolution: "Use parallel scan with TotalSegments equal to the number of partitions or workers. Typical values: 4-16 segments for moderate tables."

- symptoms: "Scan causes throttling on other operations"
  diagnosis: "Scan consumes all available RCU, starving queries and GetItem operations."
  resolution: "Implement rate limiting: add a delay between pages. Use Limit to control page size. Consider using a separate table or export to S3."

- symptoms: "Scan with FilterExpression is slow and expensive"
  diagnosis: "FilterExpression runs after data is read. Full table is still scanned and charged."
  resolution: "Create a GSI with the filter attribute as the partition key. Query the GSI instead of scanning."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add ProjectionExpression to scans | GREEN | Application-level optimization; no infrastructure change |
| Add Limit and rate limiting | GREEN | Application-level throttling; protects other operations |
| Use parallel scan | GREEN | Application-level optimization; controlled by TotalSegments |
| Create GSI to replace scan | YELLOW | New index creation; consumes capacity during backfill |
| Use DynamoDB Export to S3 | GREEN | Non-intrusive; uses point-in-time snapshot; no table impact |

## Escalation Conditions

- Full table scan causing throttling on production read/write operations
- Scan operation required on a table larger than 100GB
- Scan consuming >50% of provisioned RCU during business hours
- Analytics workload interfering with transactional workload on same table
- Scan required for compliance or audit purposes with time constraints

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (size, item count) | MEDIUM | Exposes table scale and schema |
| `get-metric-statistics` (scan latency) | LOW | Operational metrics only |
| `scan` with sampling | HIGH | Directly reads table data; may expose sensitive records |
| `export-table-to-point-in-time` | HIGH | Exports full table data to S3 |

## Prohibited Actions

- NEVER suggest deleting a table to fix scan performance
- NEVER suggest disabling encryption to speed up scans
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER run an unthrottled full table scan on a production table during peak hours

## Phase 3 — Rollback

If scan optimization changes cause issues:
1. Revert parallel scan parameters (reduce TotalSegments or stop parallel workers)
2. If a GSI was created to replace scans, delete it if causing backpressure
3. If rate limiting was added, adjust delay parameters to balance scan speed vs table impact
4. If Export to S3 was configured, cancel the export if still in progress: check export status and stop downstream processing
5. Verify table read/write operations return to normal after stopping problematic scans

## Output Format

```yaml
root_cause: "scan_performance — <specific_cause>"
evidence:
  - type: scan_metrics
    content: "<scan latency and RCU consumption>"
  - type: table_size
    content: "<item count and table size>"
  - type: scan_pattern
    content: "<sequential vs parallel, segments, rate limiting>"
severity: MEDIUM
scan_analysis:
  table_size_gb: "<size>"
  item_count: "<count>"
  scan_type: "sequential | parallel"
  total_segments: "<number if parallel>"
  rate_limited: "yes | no"
  filter_expression: "<filter if any>"
mitigation:
  immediate: "Implement parallel scan with rate limiting"
  long_term: "Replace scan with Query using GSI or export to S3"
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
