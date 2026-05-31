---
title: "D1 — CloudFront Integration"
description: "Diagnose CloudFront distribution integration issues with Internet Monitor"
status: active
severity: HIGH
triggers: ["CloudFront not monitored", "distribution not showing", "CloudFront integration failed"]
owner: devops-agent
objective: "Resolve CloudFront distribution integration issues with Internet Monitor"
context: "CloudFront distributions are a primary resource type for Internet Monitor. They provide both performance and availability metrics."
---

## Phase 1 — Triage

MUST:
- Check monitor resources: `aws internetmonitor get-monitor --monitor-name <name> --query 'Resources'`
- Verify CloudFront distribution exists and is deployed: `aws cloudfront get-distribution --id <distribution-id> --query 'Distribution.{Status:Status,DomainName:DomainName}'`
- Confirm the distribution ARN format: arn:aws:cloudfront::<account-id>:distribution/<distribution-id>

SHOULD:
- Check if the distribution has active traffic (check CloudFront access logs or CloudWatch metrics)
- Verify the distribution is in Deployed status, not InProgress
- Check if the distribution is in the same account as the monitor

MAY:
- Review CloudFront real-time metrics: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name Requests --dimensions Name=DistributionId,Value=<id> --start-time <time> --end-time <time> --period 300 --statistics Sum`

## Phase 2 — Remediate

MUST:
- Add distribution to monitor if missing: `aws internetmonitor update-monitor --monitor-name <name> --resources-to-add arn:aws:cloudfront::<account-id>:distribution/<id>`
- Verify distribution ARN is correctly formatted
- Ensure distribution is in Deployed status before adding

SHOULD:
- Verify traffic is flowing through the distribution after adding to monitor
- Allow time for Internet Monitor to begin collecting data (ProcessingStatus: COLLECTING_DATA)

MAY:
- Add multiple distributions to a single monitor for consolidated monitoring
- Configure log delivery to capture CloudFront-specific insights

## Common Issues

- symptoms: "Distribution added to monitor but no data appearing"
  diagnosis: "Distribution may have no traffic or monitor is still in COLLECTING_DATA phase."
  resolution: "Check ProcessingStatus. Allow up to 30 minutes for initial data collection. Verify the distribution has active traffic using CloudFront metrics."

## Safety Ratings

```
safety_ratings:
  - "get-monitor: GREEN — read-only monitor details"
  - "get-distribution: GREEN — read-only distribution details"
  - "update-monitor resources-to-add: YELLOW — adds resource to monitoring, may increase cost"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric retrieval"
```

## Escalation Conditions

- Distribution consistently fails to produce Internet Monitor data despite active traffic
- Cross-account distribution monitoring not working with correct permissions
- Multiple distributions showing integration issues simultaneously
- Internet Monitor data contradicts CloudFront native metrics

## Data Sensitivity

- `aws cloudfront get-distribution`: MEDIUM — reveals distribution configuration, origins, and behaviors
- `aws internetmonitor get-monitor`: LOW — returns monitor configuration
- `aws cloudwatch get-metric-statistics`: LOW — returns aggregated metric data

## Prohibited Actions

- NEVER remove a CloudFront distribution from a production monitor without explicit approval
- NEVER modify CloudFront distribution settings to troubleshoot Internet Monitor issues
- NEVER add distributions from other accounts without verifying cross-account permissions
- NEVER assume CloudFront issues based solely on Internet Monitor data without checking CloudFront metrics

## Phase 3 — Rollback

1. If a distribution was added in error, remove it: `aws internetmonitor update-monitor --monitor-name <name> --resources-to-remove <arn>`
2. If monitor configuration was changed, revert to previous settings
3. Verify remaining monitored resources are still producing data
4. Document the integration troubleshooting steps and outcome
5. Confirm no CloudFront configuration was inadvertently modified

## Output Format

```yaml
root_cause: "cloudfront_integration — <specific_cause>"
evidence:
  - type: integration_issue
    content: "<CloudFront and monitor details>"
severity: HIGH
mitigation:
  immediate: "Fix CloudFront integration"
  long_term: "Automate distribution registration via IaC"
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
