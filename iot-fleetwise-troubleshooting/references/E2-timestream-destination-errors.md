---
title: "E2 — Timestream Destination Errors"
description: "Diagnose Timestream destination configuration, permission, and data ingestion failures"
status: active
severity: MEDIUM
triggers: ["Timestream destination error", "data not in Timestream", "Timestream write failed", "Timestream table not found"]
owner: devops-agent
objective: "Resolve Timestream destination issues for near-real-time vehicle data access"
context: "Timestream destinations provide near-real-time data access for vehicle telemetry. Requires Timestream database and table, IAM role with write permissions. Data uses vehicle name as a dimension."
---

## Phase 1 — Triage

MUST:
- Check campaign destination configuration: `aws iotfleetwise get-campaign --name <name> --query 'dataDestinationConfigs'`
- Verify Timestream database exists: `aws timestream-write describe-database --database-name <name>`
- Verify Timestream table exists: `aws timestream-write describe-table --database-name <db> --table-name <table>`

SHOULD:
- Check IAM role for Timestream write permissions: `aws iam get-role --role-name <role>`
- Query Timestream for recent data: `aws timestream-query query --query-string "SELECT * FROM \"<db>\".\"<table>\" ORDER BY time DESC LIMIT 10"`
- Verify IAM role trust policy allows iotfleetwise.amazonaws.com

MAY:
- Check Timestream table retention settings
- Review CloudTrail for Timestream WriteRecords failures
- Verify Timestream is available in the campaign's region

## Phase 2 — Remediate

MUST:
- Ensure Timestream database and table exist in the same region as the campaign
- Verify IAM role has timestream:WriteRecords and timestream:DescribeEndpoints permissions
- Ensure IAM role trust policy allows FleetWise service to assume it

SHOULD:
- Configure appropriate memory store and magnetic store retention periods
- Verify table schema is compatible with FleetWise data format
- Check Timestream service quotas for write throughput

MAY:
- Set up Timestream scheduled queries for data aggregation
- Configure Grafana dashboards for real-time vehicle data visualization
- Implement Timestream cost monitoring

## Safety Ratings

```
safety_ratings:
  - "get-campaign: GREEN — read-only inspection"
  - "timestream-write describe-database: GREEN — read-only inspection"
  - "timestream-write describe-table: GREEN — read-only inspection"
  - "timestream-query query: GREEN — read-only data query"
  - "timestream-write update-table: YELLOW — modifies retention settings"
```

## Escalation Conditions

- Timestream write throttling causing data loss
- Timestream table retention too short, losing historical data
- Cross-account Timestream access required
- Timestream costs exceeding budget due to high data volume

## Data Sensitivity

- `aws iotfleetwise get-campaign`: MEDIUM — contains destination configuration
- `aws timestream-query query`: HIGH — returns vehicle telemetry data
- `aws timestream-write describe-table`: LOW — returns table metadata
- `aws iam get-role`: MEDIUM — reveals permission configuration

## Prohibited Actions

- NEVER delete a Timestream table with active campaign data without backup
- NEVER reduce memory store retention below the campaign's data freshness requirements
- NEVER grant timestream:* permissions — use least privilege for write operations
- NEVER modify Timestream table schema while campaigns are actively writing data

## Phase 3 — Rollback

1. If IAM role was modified, restore previous permissions
2. If Timestream table was deleted, recreate with same configuration
3. If retention was changed, restore previous retention settings: `aws timestream-write update-table --database-name <db> --table-name <table> --retention-properties MemoryStoreRetentionPeriodInHours=<hours>,MagneticStoreRetentionPeriodInDays=<days>`
4. Verify data ingestion resumes by querying for recent records
5. Confirm no data was lost during the outage

## Common Issues

- symptoms: "Campaign RUNNING but Timestream table has no recent records"
  diagnosis: "IAM role missing timestream:WriteRecords permission, or trust policy does not allow FleetWise to assume the role."
  resolution: "Add timestream:WriteRecords and timestream:DescribeEndpoints to IAM role. Add iotfleetwise.amazonaws.com to trust policy."

- symptoms: "Timestream WriteRecords returns RejectedRecordsException"
  diagnosis: "Data timestamps are outside the memory store retention window, or records have duplicate dimensions and timestamps."
  resolution: "Check memory store retention period. Ensure edge agent clock is synchronized. Verify no duplicate data submissions."

## Output Format

```yaml
root_cause: "timestream_destination — <specific_cause>"
evidence:
  - type: destination_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Timestream destination configuration"
  long_term: "Implement destination monitoring and data ingestion alerts"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
