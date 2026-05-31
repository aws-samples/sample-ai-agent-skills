---
title: "E1 — Metering Failures"
description: "Diagnose Marketplace metering and usage reporting failures"
status: active
severity: CRITICAL
triggers: ["metering failed", "batch-meter-usage error", "usage not reported", "metering service"]
owner: devops-agent
objective: "Resolve metering failures to ensure accurate billing"
context: "Sellers must report usage via Marketplace Metering Service within 6 hours. Failed metering means lost revenue for sellers and potential billing discrepancies for buyers."
---

## Phase 1 — Triage

MUST:
- Test metering with dry run: `aws marketplace-metering batch-meter-usage --product-code <code> --usage-records '[{"Timestamp":"2025-04-12T00:00:00Z","CustomerIdentifier":"<id>","Dimension":"<dim>","Quantity":1}]' --dry-run`
- Check IAM permissions for metering: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names aws-marketplace:MeterUsage aws-marketplace:BatchMeterUsage`
- Verify product code and dimension names are correct

SHOULD:
- Review application logs for metering API call failures
- Check CloudTrail for MeterUsage/BatchMeterUsage events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=BatchMeterUsage --max-results 10`

MAY:
- Verify the metering endpoint is reachable from the application environment
- Check for clock skew on the reporting instance

## Phase 2 — Remediate

MUST:
- Fix IAM permissions — ensure aws-marketplace:MeterUsage and aws-marketplace:BatchMeterUsage are granted
- Correct product code and dimension names to match Marketplace configuration
- Implement retry logic with exponential backoff for failed metering calls

SHOULD:
- Send metering records within 1 hour of usage (well within the 6-hour window)
- Implement dead-letter queue for failed metering records
- Add monitoring and alerting for metering failures

MAY:
- Implement local usage caching for resilience
- Set up CloudWatch alarms for metering error rates

## Safety Ratings

```
safety_ratings:
  - "marketplace-metering batch-meter-usage --dry-run: GREEN — validation only, no billing impact"
  - "marketplace-metering batch-meter-usage: RED — submits billing records, financial impact"
  - "marketplace-metering meter-usage: RED — submits billing records, financial impact"
  - "iam simulate-principal-policy: GREEN — read-only permission check"
```

## Escalation Conditions

- Metering records rejected for valid usage within the 6-hour window
- Duplicate metering records causing overbilling
- Metering service returns persistent 5xx errors
- Customer disputes billing due to metering discrepancies

## Data Sensitivity

- Metering records: HIGH — contain customer identifiers and usage quantities
- Customer identifiers: HIGH — can be used to identify specific buyers
- Usage patterns: MEDIUM — reveals customer consumption behavior
- `aws cloudtrail lookup-events`: MEDIUM — reveals metering activity

## Prohibited Actions

- NEVER submit metering records with inflated quantities
- NEVER use production customer identifiers in test metering calls (use --dry-run)
- NEVER ignore metering failures — they result in lost revenue
- NEVER submit metering records older than 6 hours — they will be rejected

## Phase 3 — Rollback

1. If duplicate metering records were submitted, contact AWS Support for correction
2. If incorrect quantities were reported, submit correcting records for subsequent periods
3. If metering was sent with wrong dimensions, fix the dimension mapping and re-report
4. Review and flush any queued metering records that may be stale
5. Document the metering failure timeline for billing reconciliation

## Common Issues

- symptoms: "BatchMeterUsage returns InvalidProductCodeException"
  diagnosis: "Product code does not match any registered Marketplace product."
  resolution: "Verify product code in Marketplace Management Portal. Product codes are case-sensitive."

- symptoms: "BatchMeterUsage returns TimestampOutOfBoundsException"
  diagnosis: "Usage record timestamp is older than 6 hours."
  resolution: "Submit records within 6 hours of usage. Implement retry logic to prevent stale records."

## Output Format

```yaml
root_cause: "metering_failure — <specific_cause>"
evidence:
  - type: metering_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix metering and resubmit records"
  long_term: "Implement robust metering with retries and dead-letter queues"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
