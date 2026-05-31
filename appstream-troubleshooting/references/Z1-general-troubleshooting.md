---
title: "Z1 — General AppStream 2.0 Troubleshooting"
description: "Catch-all runbook for AppStream issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "AppStream not working"
  - "AppStream problem"
  - "AppStream issue"
  - "something wrong with AppStream"
owner: devops-agent
objective: "Systematically diagnose unspecified AppStream 2.0 issues"
context: "This catch-all runbook provides a systematic approach when the specific issue is unclear. It covers the most common AppStream problems and guides toward the appropriate specific runbook."
---

## Phase 1 — Triage

MUST:
- Check fleet status: `aws appstream describe-fleets --names <fleet-name>`
- Check stack status: `aws appstream describe-stacks --names <stack-name>`
- Check fleet capacity: `aws appstream describe-fleets --names <fleet-name> --query 'Fleets[*].ComputeCapacityStatus'`
- Check for errors: `aws cloudwatch get-metric-statistics --namespace AWS/AppStream --metric-name InsufficientCapacityError --dimensions Name=Fleet,Value=<fleet-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check active sessions: `aws appstream describe-sessions --stack-name <stack-name> --fleet-name <fleet-name>`
- Check image status: `aws appstream describe-images --names <image-name>`
- Review CloudTrail for recent events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appstream.amazonaws.com --max-results 20`
- Check image builder status if applicable: `aws appstream describe-image-builders`

MAY:
- Check VPC configuration for VPC fleets
- Verify user authentication configuration
- Check S3 buckets for home folders and app settings

## Phase 2 — Remediate

MUST:
- Address the identified issue based on triage findings
- Redirect to the appropriate specific runbook

SHOULD:
- Redirect based on findings:
  - Fleet issues → A1-A3
  - Image issues → B1-B2
  - Session issues → C1-C3
  - User issues → D1-D2
  - Network issues → E1-E2
  - Application issues → F1-F2
  - Storage issues → G1-G2

MAY:
- Perform a full AppStream configuration audit
- Document findings

## Common Issues

- symptoms: "General AppStream not working"
  diagnosis: "Multiple possible causes — follow systematic triage."
  resolution: "Check fleet, stack, capacity, sessions, and authentication in order."

- symptoms: "AppStream was working but stopped"
  diagnosis: "Fleet stopped, image deleted, or configuration changed."
  resolution: "Check fleet state. Verify image exists. Review CloudTrail for changes."

- symptoms: "Some users can connect but others cannot"
  diagnosis: "User-specific authentication or authorization issue."
  resolution: "Check user pool/SAML configuration. Verify user-stack associations."

## Output Format

```yaml
root_cause: "general_appstream_issue — <specific_cause>"
evidence:
  - type: fleet_status
    content: "<fleet state and capacity>"
  - type: stack_status
    content: "<stack configuration>"
  - type: session_status
    content: "<active sessions>"
  - type: error_metrics
    content: "<CloudWatch error metrics>"
severity: MEDIUM
mitigation:
  immediate: "Address the identified issue"
  long_term: "Implement comprehensive AppStream monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Check fleet status | GREEN | Read-only check |
| Check stack status | GREEN | Read-only check |
| Check fleet capacity | GREEN | Read-only check |
| Check for errors | GREEN | Read-only metrics |
| Check active sessions | GREEN | Read-only check |
| Check image status | GREEN | Read-only check |
| Review CloudTrail events | GREEN | Read-only audit |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Fleet status**: Fleet state and capacity — INTERNAL
- **Stack status**: Stack configuration — INTERNAL
- **Session data**: Active user sessions — CONFIDENTIAL
- **CloudTrail events**: API call history — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER make configuration changes during initial triage
- NEVER terminate active sessions without user notification

## Phase 3 — Rollback

1. Revert any configuration changes made during investigation
2. Verify fleet and stack are in expected state
3. Confirm active sessions are not disrupted
4. Restore previous configuration if changes were made
5. Document findings and escalate if root cause is unclassified

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
