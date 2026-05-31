---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for MediaTailor issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["MediaTailor error", "MediaTailor not working", "general MediaTailor issue", "unknown MediaTailor problem"]
owner: devops-agent
objective: "Systematically diagnose and resolve general MediaTailor issues"
context: "Use this runbook when the issue does not clearly match a specific runbook category. Follow the systematic investigation workflow to identify the root cause domain."
---

## Phase 1 — Triage

MUST:
- List all configurations: `aws mediatailor list-playback-configurations`
- List all channels: `aws mediatailor list-channels`
- Check CloudWatch metrics for errors: `aws cloudwatch get-metric-statistics --namespace AWS/MediaTailor --metric-name AdDecisionServer.Errors --dimensions Name=Configuration,Value=<config-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check CloudTrail for recent MediaTailor API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=mediatailor.amazonaws.com --max-results 20`
- Verify IAM permissions for the calling principal
- Check service health dashboard for MediaTailor regional issues

MAY:
- Review CloudWatch Logs if logging is enabled
- Test playback URLs directly to reproduce the issue
- Check AWS Health Dashboard for service events

## Phase 2 — Remediate

MUST:
- Identify the specific domain (configuration, ad insertion, channel, CDN, manifest) and route to the appropriate runbook
- Fix IAM permission issues if API calls are failing
- Verify regional service availability

SHOULD:
- Document the issue pattern for future reference
- Check if the issue affects multiple configurations or is isolated

MAY:
- Open AWS Support case if issue appears to be service-side
- Review recent configuration changes that may have caused the issue

## Safety Ratings

```
safety_ratings:
  - "list-playback-configurations: GREEN — read-only enumeration"
  - "list-channels: GREEN — read-only enumeration"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "cloudtrail lookup-events: GREEN — read-only audit log"
  - "Any remediation action: varies — refer to specific runbook"
```

## Escalation Conditions

- Issue affects all MediaTailor configurations in a region
- Problem persists after following specific runbook remediation steps
- Issue appears to be an AWS service-side problem
- Multiple unrelated MediaTailor features failing simultaneously

## Data Sensitivity

- `aws mediatailor list-playback-configurations`: LOW — configuration metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals IAM principals and API actions
- CloudWatch metrics: LOW — aggregate statistics
- Playback session data: HIGH — may contain viewer information

## Prohibited Actions

- NEVER delete configurations or channels as a troubleshooting step
- NEVER modify production configurations without a rollback plan
- NEVER share CloudTrail logs containing IAM principal ARNs externally
- NEVER assume a service-side issue without first ruling out configuration problems

## Phase 3 — Rollback

1. If any configuration was modified during troubleshooting, revert to pre-investigation state
2. Document all changes made during investigation for audit trail
3. If issue was misdiagnosed and wrong runbook was followed, undo all remediation steps
4. Verify all configurations and channels are in their expected state after investigation

## Common Issues

- symptoms: "All MediaTailor API calls return AccessDeniedException"
  diagnosis: "IAM permissions missing or SCP blocking MediaTailor actions."
  resolution: "Verify IAM policy includes mediatailor:* permissions. Check SCPs for deny rules."

- symptoms: "MediaTailor works in one region but not another"
  diagnosis: "Configuration exists only in one region or regional service issue."
  resolution: "Verify configuration exists in the target region. Check AWS Health Dashboard for regional issues."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<triage findings and domain identification>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement comprehensive monitoring"
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
