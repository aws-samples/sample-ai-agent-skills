---
title: "Z1 — General Troubleshooting"
description: "General edge function troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["edge function error", "CloudFront function not working", "Lambda@Edge issue", "general edge compute problem"]
owner: devops-agent
objective: "Systematically diagnose and resolve general edge function issues"
context: "Catch-all runbook for edge function issues. Covers both CloudFront Functions and Lambda@Edge with systematic elimination approach."
---

## Phase 1 — Triage

MUST:
- Identify function type (CF Function or Lambda@Edge): `aws cloudfront get-distribution-config --id <distribution-id>`
- Check function health and error rates: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name FunctionExecutionErrors --dimensions Name=DistributionId,Value=<dist-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check distribution status: `aws cloudfront get-distribution --id <distribution-id> --query 'Distribution.Status'`

SHOULD:
- Review recent changes in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudfront.amazonaws.com --max-results 20`
- Check AWS Health Dashboard for CloudFront service issues
- Verify the function is associated with the correct cache behavior and event type

MAY:
- Test CF Functions with test-function API
- Check Lambda@Edge logs across multiple regions
- Compare current configuration with a known-good state

## Phase 2 — Remediate

MUST:
- Address the specific error identified in triage
- Verify function code is correct and handles all event types
- Ensure function is within resource limits (time, memory, package size)

SHOULD:
- Test the fix in DEVELOPMENT stage (CF Functions) or test distribution
- Monitor error rates after applying the fix
- Document the root cause and resolution

MAY:
- Set up CloudWatch alarms for function error rates
- Implement automated health checks
- Create a runbook for the specific issue if recurring

## Common Issues

- symptoms: "Edge function worked yesterday but fails today"
  diagnosis: "Recent deployment, configuration change, or AWS service issue."
  resolution: "Check CloudTrail for recent changes. Check AWS Health Dashboard. Roll back recent changes."

- symptoms: "Function works in testing but fails in production"
  diagnosis: "Test events don't match production request patterns, or production has edge cases not covered."
  resolution: "Capture production request patterns from logs. Update test events. Add error handling for edge cases."

## Safety Ratings

```
safety_ratings:
  - "cloudfront get-distribution-config: GREEN — read-only"
  - "cloudwatch get-metric-statistics: GREEN — read-only"
  - "cloudtrail lookup-events: GREEN — read-only"
  - "cloudfront update-distribution: RED — modifies production CDN"
```

## Escalation Conditions

- Issue cannot be diagnosed with available logs and metrics
- Problem appears to be an AWS service issue
- Multiple distributions affected simultaneously
- Issue requires AWS Support engagement

## Data Sensitivity

- `aws cloudfront get-distribution-config`: MEDIUM — reveals distribution settings
- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history
- Function execution logs: HIGH — may contain request data

## Prohibited Actions

- NEVER delete distributions or functions as a troubleshooting step
- NEVER modify production functions without a rollback plan
- NEVER assume the issue is with AWS — verify configuration first
- NEVER share distribution configurations externally without redaction

## Phase 3 — Rollback

1. If any changes were made during troubleshooting, revert them
2. Restore previous function versions and distribution configuration
3. Wait for distribution to reach 'Deployed' status
4. Verify the system returns to its pre-troubleshooting state
5. Document all changes made and their outcomes

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: diagnostic_finding
    content: "<finding details>"
severity: MEDIUM
mitigation:
  immediate: "Apply targeted fix"
  long_term: "Implement monitoring and alerting"
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
