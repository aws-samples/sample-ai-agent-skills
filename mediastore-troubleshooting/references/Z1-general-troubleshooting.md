---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for MediaStore issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["mediastore issue", "mediastore error", "mediastore problem", "general mediastore"]
owner: devops-agent
objective: "Provide a systematic approach for diagnosing unclassified MediaStore issues"
context: "Use this runbook when the issue does not clearly match a specific category. Follow the triage steps to classify and route to the appropriate runbook."
---

## Phase 1 — Triage

MUST:
- List all containers and their status: `aws mediastore list-containers --query 'Containers[*].{Name:Name,Status:Status,Endpoint:Endpoint}'`
- Check the specific container: `aws mediastore describe-container --container-name <name>`
- Review recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=mediastore.amazonaws.com --max-results 20`

SHOULD:
- Check CloudWatch metrics for errors: `aws cloudwatch get-metric-statistics --namespace AWS/MediaStore --metric-name 5xxErrorCount --dimensions Name=ContainerName,Value=<name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify IAM permissions for the caller

MAY:
- Check AWS Health Dashboard for MediaStore service issues
- Compare configuration with a known-working container

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings
- If container issue → A1 or A2
- If object operation issue → B1 or B2
- If CORS/lifecycle → C1 or C2
- If logging/metrics → D1 or D2
- If CDN/endpoint → E1 or E2
- If throttling → F1

SHOULD:
- Document the symptoms and triage findings before routing
- Verify the issue is MediaStore-specific and not a general AWS service issue

MAY:
- Open an AWS Support case if the issue appears to be a service-side problem

## Safety Ratings

```
safety_ratings:
  - "mediastore list-containers: GREEN — read-only container enumeration"
  - "mediastore describe-container: GREEN — read-only container details"
  - "cloudtrail lookup-events: GREEN — read-only event query"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric query"
```

## Escalation Conditions

- Issue cannot be classified into any specific runbook category
- Multiple symptoms spanning different categories simultaneously
- Suspected AWS service issue affecting MediaStore in the region
- Container is in an unexpected state (not ACTIVE or DELETING)

## Data Sensitivity

- Container list: LOW — names and status
- CloudTrail events: MEDIUM — reveals API actions and IAM principals
- CloudWatch metrics: LOW — numeric performance data
- Container configuration: MEDIUM — policies and settings

## Prohibited Actions

- NEVER delete containers as a first troubleshooting step — collect diagnostics first
- NEVER apply multiple changes simultaneously — change one thing at a time
- NEVER use S3 commands to troubleshoot MediaStore — they are different services
- NEVER assume the issue is MediaStore-specific without checking dependent services

## Phase 3 — Rollback

1. Revert any changes made during investigation
2. If policies were modified, restore previous versions
3. If containers were created for testing, clean them up
4. Document all changes made and their outcomes
5. Verify the original issue is resolved or properly escalated

## Common Issues

- symptoms: "All MediaStore operations return errors"
  diagnosis: "IAM permissions may have been revoked or an SCP is blocking MediaStore."
  resolution: "Check IAM policies and SCPs. Verify the caller has mediastore:* and mediastore-data:* permissions."

- symptoms: "MediaStore works from CLI but not from application"
  diagnosis: "Application is using S3 SDK/API instead of MediaStore SDK/API."
  resolution: "Use the MediaStore and MediaStoreData SDK clients, not the S3 client."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: diagnostic_finding
    content: "<finding details>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement comprehensive MediaStore monitoring"
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
