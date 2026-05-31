---
title: "F2 — Parameter Store Throughput"
description: "Diagnose Parameter Store throughput throttling issues"
status: active
severity: MEDIUM
triggers:
  - "parameter throttling"
  - "ThrottlingException parameter"
  - "rate exceeded parameter"
  - "parameter store slow"
owner: devops-agent
objective: "Identify Parameter Store throughput issues and resolve throttling"
context: "Parameter Store has default throughput of 40 TPS (transactions per second) for standard throughput. Higher throughput (1,000 TPS) can be enabled but costs extra. Throttling occurs when API call rate exceeds the configured throughput limit."
---

## Phase 1 — Triage

MUST:
- Check current throughput setting: `aws ssm get-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/parameter-store/high-throughput-enabled`
- Check CloudWatch metrics for throttling: `aws cloudwatch get-metric-statistics --namespace AWS/SSM --metric-name ThrottledCount --start-time <time> --end-time <time> --period 300 --statistics Sum`
- Identify the API calls causing throttling from CloudTrail
- Check the rate of parameter API calls from the application

SHOULD:
- Review application parameter access patterns
- Check if multiple services are accessing parameters simultaneously
- Verify caching is implemented in the application

MAY:
- Check Parameter Store API call volume over time
- Review application code for unnecessary parameter fetches

## Phase 2 — Remediate

MUST:
- Implement parameter caching in the application (AWS SDK caching, Lambda extensions)
- Reduce unnecessary parameter API calls
- Enable higher throughput if caching is insufficient

SHOULD:
- Use GetParameters (batch) instead of individual GetParameter calls
- Implement exponential backoff for throttled requests
- Use AWS AppConfig for frequently changing configuration

MAY:
- Enable higher throughput: `aws ssm update-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/parameter-store/high-throughput-enabled --setting-value true`
- Consider using Secrets Manager for high-throughput secret access

## Common Issues

- symptoms: "ThrottlingException on GetParameter calls"
  diagnosis: "Application exceeding 40 TPS standard throughput limit."
  resolution: "Implement caching. Use batch GetParameters. Enable higher throughput if needed."

- symptoms: "Intermittent throttling during deployments"
  diagnosis: "Multiple instances fetching parameters simultaneously at startup."
  resolution: "Implement caching with TTL. Stagger instance startup. Use batch API."

- symptoms: "Higher throughput enabled but still throttling"
  diagnosis: "Exceeding 1,000 TPS higher throughput limit."
  resolution: "Implement aggressive caching. Reduce API call frequency."

## Output Format

```yaml
root_cause: "parameter_throughput — <specific_cause>"
evidence:
  - type: throughput_setting
    content: "<current throughput configuration>"
  - type: throttle_metrics
    content: "<CloudWatch throttling metrics>"
severity: MEDIUM
mitigation:
  immediate: "Implement caching and batch API calls"
  long_term: "Review application architecture for parameter access patterns"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Parameter Store may contain secrets, API keys, and credentials
- Command output may contain system configuration details
- Session logs may capture sensitive terminal activity
- Classification: HIGHLY CONFIDENTIAL - secrets management and system access data

## Prohibited Actions

- NEVER execute untested commands via Run Command on production instances
- NEVER modify patch baselines during active patching windows
- NEVER delete SSM parameters containing production secrets without rotation
- NEVER deregister managed instances without confirming no active sessions

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
