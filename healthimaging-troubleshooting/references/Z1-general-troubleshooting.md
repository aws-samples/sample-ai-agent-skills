---
title: "Z1 — General HealthImaging Troubleshooting"
description: "Catch-all runbook for HealthImaging issues"
status: active
severity: MEDIUM
triggers: ["HealthImaging error", "medical-imaging not working", "HealthImaging issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified HealthImaging issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List datastores: `aws medical-imaging list-datastores`
- Check recent import jobs: `aws medical-imaging list-dicom-import-jobs --datastore-id <datastore-id>`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=medical-imaging.amazonaws.com --max-results 20`

SHOULD:
- Check IAM permissions for the calling principal
- Verify service quotas: `aws service-quotas list-service-quotas --service-code medical-imaging`

MAY:
- Check AWS Health Dashboard for service issues
- Review CloudWatch metrics for anomalies

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for recurring issues

## Safety Ratings

```
safety_ratings:
  - "list-datastores: GREEN — read-only enumeration"
  - "list-dicom-import-jobs: GREEN — read-only job listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "service-quotas list-service-quotas: GREEN — read-only quota check"
```

## Escalation Conditions

- HealthImaging service appears non-functional across multiple regions
- Data integrity concerns with stored medical imaging data
- CloudTrail shows unauthorized API calls to HealthImaging
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws medical-imaging list-datastores`: LOW — returns datastore metadata
- `aws medical-imaging list-dicom-import-jobs`: LOW — returns job metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER delete datastores or image sets during general troubleshooting
- NEVER access PHI data without proper authorization during investigation
- NEVER make configuration changes without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path and findings

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them to the pre-investigation state
2. Restore datastore settings if they were modified
3. If image sets were accessed during investigation, document access for HIPAA audit trail
4. Document all investigation steps and findings for the compliance team
5. Verify the HealthImaging service is functioning normally before closing the investigation

## Output Format

```yaml
root_cause: "general_healthimaging — <specific_cause>"
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
