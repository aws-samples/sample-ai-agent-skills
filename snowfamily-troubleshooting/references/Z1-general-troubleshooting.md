---
title: "Z1 — General Snow Family Troubleshooting"
description: "Catch-all runbook for AWS Snow Family issues"
status: active
severity: MEDIUM
triggers: ["Snow Family error", "Snowball not working", "Snow device issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Snow Family issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List jobs: `aws snowball list-jobs --query 'JobListEntries[*].{JobId:JobId,State:JobState,Type:JobType}'`
- Describe the relevant job: `aws snowball describe-job --job-id <job-id>`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=snowball.amazonaws.com --max-results 20`

SHOULD:
- Check device status if device is on-site: `snowballEdge describe-device --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Verify job type matches the intended use case

MAY:
- Check AWS Health Dashboard for Snow Family service issues
- Review AWS Support cases for related issues

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up job status notifications via SNS

## Safety Ratings

```
safety_ratings:
  - "list-jobs: GREEN — read-only job enumeration"
  - "describe-job: GREEN — read-only job details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "describe-device: GREEN — read-only device status"
```

## Escalation Conditions

- Snow Family service appears non-functional in the region
- Device is physically damaged or unresponsive
- CloudTrail shows unauthorized API calls to Snow Family
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws snowball list-jobs`: LOW — returns job metadata
- `aws snowball describe-job`: MEDIUM — reveals job configuration and shipping details
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER cancel jobs without confirming data has been transferred
- NEVER modify device configuration during general troubleshooting without identifying root cause
- NEVER dismiss the issue without documenting the investigation path
- NEVER assume device issues are software-related without checking physical connectivity

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore device settings if they were modified during troubleshooting
3. Verify device services are running normally
4. Document all investigation steps and findings
5. Verify the Snow Family service is functioning normally before closing

## Common Issues

- symptoms: "Job status not updating after device was returned"
  diagnosis: "Device is in transit or AWS processing is delayed."
  resolution: "Check carrier tracking. Allow 1-2 business days for AWS to receive and process the device."

## Output Format

```yaml
root_cause: "general_snowfamily — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
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
