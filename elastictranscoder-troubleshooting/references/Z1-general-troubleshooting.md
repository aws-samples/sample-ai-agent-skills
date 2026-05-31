---
title: "Z1 — General Elastic Transcoder Troubleshooting"
description: "Catch-all runbook for Elastic Transcoder issues"
status: active
severity: MEDIUM
triggers: ["Elastic Transcoder error", "transcoder not working", "transcoding issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Elastic Transcoder issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List pipelines: `aws elastictranscoder list-pipelines`
- Check pipeline status for each pipeline
- List recent jobs by status: `aws elastictranscoder list-jobs-by-status --status Error --ascending`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elastictranscoder.amazonaws.com --max-results 20`

SHOULD:
- Check IAM permissions for the caller
- Verify service is available in the current region
- Check AWS Service Health Dashboard for Elastic Transcoder

MAY:
- List presets to verify availability: `aws elastictranscoder list-presets`
- Check CloudWatch metrics for anomalies

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1-F1)
- Document findings from triage

SHOULD:
- Categorize the issue: pipeline, job, preset, format, notification, or permission
- Collect relevant IDs (pipeline ID, job ID, preset ID) for the specific runbook

MAY:
- Set up monitoring for the identified issue category
- Create a test pipeline to isolate the problem

## Safety Ratings

```
safety_ratings:
  - "list-pipelines: GREEN — read-only enumeration"
  - "list-jobs-by-status: GREEN — read-only job listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "list-presets: GREEN — read-only preset listing"
```

## Escalation Conditions

- Elastic Transcoder service appears non-functional in the region
- Multiple pipelines experiencing failures simultaneously
- CloudTrail shows unauthorized API calls to Elastic Transcoder
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws elastictranscoder list-pipelines`: LOW — returns pipeline metadata
- `aws elastictranscoder list-jobs-by-status`: LOW — returns job metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER delete pipelines or presets during general troubleshooting
- NEVER cancel active jobs without understanding the impact
- NEVER make configuration changes without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them to pre-investigation state
2. Restore pipeline settings if they were modified during troubleshooting
3. If test jobs were submitted, cancel any still in progress: `aws elastictranscoder cancel-job --id <job-id>`
4. Clean up test output files: `aws s3 rm s3://<output-bucket>/<test-prefix> --recursive`
5. Document all investigation steps and findings for the team

## Common Issues

- symptoms: "All API calls return 'Service Unavailable'"
  diagnosis: "Elastic Transcoder service may be experiencing an outage in the region."
  resolution: "Check AWS Service Health Dashboard. If confirmed outage, wait for resolution or use another region."

- symptoms: "Cannot determine which runbook applies"
  diagnosis: "Issue spans multiple domains or is a new failure mode."
  resolution: "Start with pipeline status check, then job status, then permissions. Follow the error messages to the specific domain."

## Output Format

```yaml
root_cause: "general_elastictranscoder — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
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
