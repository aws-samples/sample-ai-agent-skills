---
title: "C2 — Data Collection Issues"
description: "Diagnose data collection failures, missing data, and collection scheme problems"
status: active
severity: HIGH
triggers: ["no data collected", "missing vehicle data", "collection not working", "data gaps", "stale data"]
owner: devops-agent
objective: "Resolve data collection issues and ensure vehicle telemetry flows to destinations"
context: "Data collection requires the full chain: signal catalog → model manifest → decoder manifest → vehicle → fleet → campaign (RUNNING) → edge agent → destination. A break at any point stops data flow."
---

## Phase 1 — Triage

MUST:
- Verify campaign is RUNNING: `aws iotfleetwise get-campaign --name <name> --query '{Status:status}'`
- Check target fleet has vehicles: `aws iotfleetwise list-vehicles-in-fleet --fleet-id <fleet-id>`
- Verify destination has data: check S3 bucket or Timestream table for recent records

SHOULD:
- Check vehicle connectivity (edge agent online): verify vehicle's IoT thing connectivity
- Verify model manifest and decoder manifest are both ACTIVE
- Check collection scheme triggers (time interval or condition expression)

MAY:
- Review CloudWatch metrics for FleetWise data ingestion
- Check edge agent logs on the vehicle for collection errors
- Verify network connectivity between vehicle and AWS IoT endpoint

## Phase 2 — Remediate

MUST:
- Ensure campaign status is RUNNING (not SUSPENDED or WAITING_FOR_APPROVAL)
- Verify edge agent is running on the vehicle and connected to IoT Core
- Confirm destination (S3/Timestream) is accessible and IAM role has write permissions

SHOULD:
- Check collection scheme: time-based schemes need appropriate interval; condition-based need valid expression
- Verify decoder manifest correctly maps vehicle network signals to cloud signals
- Ensure vehicle is associated with the campaign's target fleet

MAY:
- Restart edge agent on the vehicle to force reconnection
- Enable FleetWise logging: `aws iotfleetwise put-logging-options --cloud-watch-log-delivery '{logType: "ERROR", logGroupName: "/aws/iotfleetwise"}'`
- Test with a simple time-based collection scheme to isolate condition-based issues

## Safety Ratings

```
safety_ratings:
  - "get-campaign: GREEN — read-only inspection"
  - "list-vehicles-in-fleet: GREEN — read-only enumeration"
  - "put-logging-options: YELLOW — enables logging, may increase CloudWatch costs"
  - "update-campaign --action SUSPEND: YELLOW — pauses data collection"
  - "update-campaign --action RESUME: YELLOW — resumes data collection"
```

## Escalation Conditions

- No data collected from any vehicle in the fleet for more than 1 hour
- Data collection works for some vehicles but not others in the same fleet
- Collected data values are incorrect or corrupted
- Edge agent crashes repeatedly on vehicles during data collection

## Data Sensitivity

- `aws iotfleetwise get-campaign`: MEDIUM — contains collection configuration
- Vehicle telemetry data: HIGH — contains operational vehicle data
- Edge agent logs: HIGH — may contain vehicle diagnostic information
- S3/Timestream destination data: HIGH — contains collected vehicle signals

## Prohibited Actions

- NEVER assume data collection is working without verifying data in the destination
- NEVER modify collection schemes on production campaigns without testing
- NEVER ignore data gaps — they may indicate vehicle or network issues
- NEVER collect more signals than needed — excessive collection increases costs and bandwidth

## Phase 3 — Rollback

1. If collection scheme was changed, suspend and recreate campaign with previous configuration
2. If edge agent was reconfigured, restore previous configuration on the vehicle
3. If destination was changed, verify data continuity in the new destination
4. Resume campaign after fixes: `aws iotfleetwise update-campaign --name <name> --action RESUME`
5. Monitor data flow for 30 minutes to confirm collection is stable

## Common Issues

- symptoms: "Campaign is RUNNING but no data appears in S3 or Timestream"
  diagnosis: "Edge agent is not running, not connected, or decoder manifest incorrectly maps signals."
  resolution: "Verify edge agent status on vehicle. Check decoder manifest signal mappings. Verify destination IAM permissions."

- symptoms: "Data collected intermittently with gaps"
  diagnosis: "Vehicle connectivity is unstable, or condition-based collection scheme triggers infrequently."
  resolution: "Check vehicle network connectivity. For condition-based schemes, verify the condition is met during expected intervals."

## Output Format

```yaml
root_cause: "data_collection — <specific_cause>"
evidence:
  - type: collection_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix data collection chain"
  long_term: "Implement end-to-end data flow monitoring"
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
