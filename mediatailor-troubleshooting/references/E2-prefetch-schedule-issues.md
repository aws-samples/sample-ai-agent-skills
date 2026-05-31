---
title: "E2 — Prefetch Schedule Issues"
description: "Diagnose prefetch schedule creation and execution issues"
status: active
severity: MEDIUM
triggers: ["prefetch not working", "prefetch schedule error", "ad prefetch failed", "prefetch expired"]
owner: devops-agent
objective: "Resolve prefetch schedule issues to improve ad fill rates"
context: "Prefetch schedules pre-fetch ad decisions before ad breaks to reduce latency and improve fill rates. They define retrieval windows (when to fetch) and consumption windows (when to use the fetched decision)."
---

## Phase 1 — Triage

MUST:
- List prefetch schedules: `aws mediatailor list-prefetch-schedules --PlaybackConfigurationName <config-name>`
- Get prefetch schedule details: `aws mediatailor get-prefetch-schedule --PlaybackConfigurationName <config-name> --Name <schedule-name>`
- Verify retrieval and consumption window times are correct

SHOULD:
- Check if prefetch retrieval window has not expired
- Verify consumption window aligns with expected ad break times
- Check ADS availability during retrieval window

MAY:
- Monitor prefetch-related metrics in CloudWatch
- Review ADS response times during prefetch retrieval

## Phase 2 — Remediate

MUST:
- Fix retrieval window to be in the future and before consumption window
- Ensure consumption window covers the expected ad break time
- Verify ADS URL is accessible during retrieval window

SHOULD:
- Create prefetch schedules with sufficient lead time (retrieval before consumption)
- Set appropriate dynamic variables for ADS request customization

MAY:
- Implement automated prefetch schedule creation via API
- Configure multiple prefetch schedules for different ad break patterns

## Safety Ratings

```
safety_ratings:
  - "list-prefetch-schedules: GREEN — read-only enumeration"
  - "get-prefetch-schedule: GREEN — read-only schedule details"
  - "create-prefetch-schedule: YELLOW — creates new prefetch behavior"
  - "delete-prefetch-schedule: YELLOW — removes prefetch, may reduce fill rate"
```

## Escalation Conditions

- Prefetch schedules consistently fail to improve fill rates
- ADS cannot handle prefetch request volume during retrieval windows
- Prefetch timing misalignment causes stale ad decisions in live streams
- Automated prefetch creation system is generating invalid schedules

## Data Sensitivity

- Prefetch schedule configuration: LOW — timing windows and configuration
- ADS request parameters in prefetch: MEDIUM — may contain targeting data
- Prefetch response data: MEDIUM — contains ad decision details
- Dynamic variables: LOW — parameter substitution templates

## Prohibited Actions

- NEVER delete all prefetch schedules during an active ad campaign
- NEVER set retrieval windows that overlap with peak ADS load periods without capacity planning
- NEVER create prefetch schedules with consumption windows in the past
- NEVER use prefetch to circumvent ADS rate limiting

## Phase 3 — Rollback

1. If new prefetch schedule reduced fill rate, delete it: `aws mediatailor delete-prefetch-schedule --PlaybackConfigurationName <config-name> --Name <schedule-name>`
2. If prefetch timing was changed, recreate schedule with correct windows
3. If dynamic variables were misconfigured, update schedule with correct variables
4. Monitor fill rate metrics for 30 minutes after rollback

## Common Issues

- symptoms: "Prefetch schedule exists but ads are not pre-fetched"
  diagnosis: "Retrieval window has expired or is in the past."
  resolution: "Create new prefetch schedule with future retrieval window. Ensure retrieval window starts before consumption window."

- symptoms: "Prefetched ads do not match expected content"
  diagnosis: "Dynamic variables in prefetch do not match actual session parameters."
  resolution: "Review and fix dynamic variable configuration in prefetch schedule."

## Output Format

```yaml
root_cause: "prefetch_schedule — <specific_cause>"
evidence:
  - type: prefetch_detail
    content: "<prefetch schedule configuration and timing>"
severity: MEDIUM
mitigation:
  immediate: "Fix prefetch schedule timing"
  long_term: "Automate prefetch schedule management"
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
