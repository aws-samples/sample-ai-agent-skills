---
title: "Z1 — General MediaConvert Troubleshooting (Catch-All)"
description: "Fallback SOP for MediaConvert issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown MediaConvert issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Get endpoint: `aws mediaconvert describe-endpoints --query 'Endpoints[0].Url' --output text`
- List recent jobs: `aws mediaconvert list-jobs --max-results 10 --endpoint-url <endpoint>`
- Check for failed jobs: `aws mediaconvert list-jobs --status ERROR --max-results 10 --endpoint-url <endpoint>`
- List queues: `aws mediaconvert list-queues --endpoint-url <endpoint>`
- Check CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConvert --metric-name JobsErrored --start-time <start> --end-time <end> --period 3600 --statistics Sum`

SHOULD:
- Get details of any failed job: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>`
- Check IAM role configuration
- Review CloudTrail for recent API calls
- Check AWS Health Dashboard for service events

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Job errors or stuck jobs → Jobs (A1-A3)
- Input file or codec issues → Input (B1-B2)
- Output format or quality → Output (C1-C2)
- Queue or pricing issues → Queue (D1-D2)
- S3 or IAM issues → Integration (E1-E2)
- Speed or throughput → Performance (F1-F2)
- DRM or encryption → DRM (G1-G2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Review job settings in detail
- Check all error messages and codes
- Verify end-to-end pipeline (input → encoding → output)
- Check for recent configuration changes in CloudTrail

SHOULD:
- Test with a simple known-good job to isolate the issue
- Compare with previously successful job configurations
- Check AWS documentation for known issues

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<jobs|input|output|queue|integration|performance|drm|unknown>"
investigation_path: "endpoint → jobs → errors → <domain_classification>"
evidence:
  - type: job_status
    content: "<job details>"
  - type: cloudwatch
    content: "<error metrics>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete or modify a production job queue while jobs are actively processing
- **NEVER** change IAM roles on job templates without verifying S3 input/output bucket access
- **NEVER** disable DRM or content protection settings on outputs destined for licensed content
- **NEVER** cancel reserved transcoding capacity without assessing active workload commitments
- **NEVER** modify output presets for compliance-regulated content without re-validation

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
